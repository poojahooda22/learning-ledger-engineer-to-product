# Day 24 — How does a lock survive a client that everyone has given up for dead, but that isn't actually dead yet?

**Date:** 2026-07-06
**Difficulty:** Expert
**Topic:** Distributed locks, leases, and fencing tokens: why a mutex that works perfectly on one machine becomes a correctness hazard across a network, via Google's Chubby lock service (Burrows, OSDI 2006), Apache ZooKeeper's session model as used by HBase, and the 2016 Kleppmann-versus-antirez public debate over Redis's Redlock algorithm
**Stack relevance:** Supabase Postgres as the coordination point for "only one worker compiles/publishes this shader graph right now," and Cloudflare R2's content-addressed manifest pointer (Day 23) as the exact kind of shared resource a stale lock holder could corrupt with a late write

---

## 1. The company and the breaking number

**HBase, running on Apache ZooKeeper, in production, at companies including Yahoo, Facebook (in its early years), and countless smaller shops running JVM-based distributed storage.** HBase uses ZooKeeper to decide which single "region server" is currently allowed to own and serve a given slice of a table. That ownership is implemented as a session-backed lock: a region server holds a ZooKeeper session, and as long as that session is alive, HBase treats it as the sole owner of its regions, free to accept writes.

Here is the breaking number, and it is not a network number, it is a **pause** number. Martin Kleppmann's widely cited 2016 analysis of distributed locking states plainly that "for example, in the JVM, stop-the-world garbage collection pauses have sometimes been known to last for several minutes." A **region server's ZooKeeper session lease is typically configured on the order of tens of seconds** (Kleppmann's own worked example uses a client that "pauses for 40 seconds"). Several minutes of GC pause against a lease measured in tens of seconds is not a close call, it is a pause **five to ten times longer** than the window the whole locking scheme assumed could never be exceeded.

Walk through what that actually does to a "the lock says I still own this" assumption. Region server A holds the ZooKeeper session lock for region R. A full garbage collection kicks off, A's JVM stops all application threads dead, including the heartbeat thread that renews its ZooKeeper session. From ZooKeeper's point of view, A has gone silent, exactly indistinguishable from a crashed process, a severed network cable, or a killed container: ZooKeeper cannot see *why* a client stopped heartbeating, only *that* it did. After the session timeout elapses, ZooKeeper expires A's session, releases the lock, and region server B acquires it and starts legitimately serving region R, accepting new writes. Then A's garbage collection finishes. A's application threads resume exactly where they left off, mid-operation, still holding a reference to what it believes is a valid lock, with **no idea that any of the last several minutes happened from ZooKeeper's perspective.** If A's very next instruction is to write to the storage backing region R, that write lands, right alongside B's writes, to the same region, from two processes that each individually believe they are the only writer alive. That is not a hypothetical: it is precisely the failure mode Kleppmann's article names to motivate the entire design he proposes, and precisely why real ZooKeeper deployments (documented in the ZooKeeper programmer's guide's own recipes for locks) had to build a fix for exactly this race.

Hold onto that number: **a session timeout in the tens of seconds, against a real, documented, repeatable pause of several minutes.** Every mechanism in this lesson exists to answer one question: given that any process can be paused for arbitrarily long, by the CPU scheduler, a garbage collector, a hypervisor, or a slow disk, and can wake up with zero awareness of how much time passed, how do you stop that woken-up process from doing damage, without simply making the timeout longer (which never actually solves the problem, it only moves the goalposts)?

---

## 2. Why the naive (demo) design dies

The demo version of a distributed lock looks almost exactly like a local mutex, just implemented against a shared remote store:

```
1. client sends SET lock_key = my_id, "if not already set"
2. if success: client believes it holds the lock
3. client does its work
4. client sends DEL lock_key when done
```

Add a timeout so a crashed client's lock does not hang the system forever: `SET lock_key = my_id NX PX 10000` (Redis's own real syntax), meaning "set the key only if absent, and expire it automatically after 10 seconds." This is the standard, honest, recommended single-node Redis lock pattern. It fails in three concrete ways once it is asked to protect something that actually matters if two clients briefly believe they both hold it.

**a. The lock's timeout and the client's actual runtime are two unrelated clocks, and nothing enforces that the first outlasts the second.** The 10-second expiry is a bet: "whatever this client is doing will always finish inside 10 seconds." Section 1's number shows that bet can be wrong by an order of magnitude with nothing abnormal happening, no bug, no attack, just an ordinary JVM garbage collector doing its ordinary job on an ordinary day. A lock's safety cannot depend on an assumption about how long code takes to run, because that assumption is not something the lock service can verify, only hope for.

**b. The lock being "held" tells you nothing about whether the holder is still the one thing acting on the world.** The instant region server A's session lease expires, HBase's own logic correctly promotes region server B, and B is now, correctly, the authority. The bug is not that B acted, the bug is that **A does not know it has been demoted**, and A's very next write proceeds as if the promotion never happened. A lock is a statement about the past ("I successfully acquired this at time T"), but the code holding it treats it as a statement about the present ("I still hold this right now"), and nothing in the naive design ever asks the shared resource itself to check which of those two statements is actually true at write time.

**c. There is no way for the thing being protected to independently reject a stale actor.** In the naive design, the storage backend (the region's data files, a database row, an S3 object) has no concept of "which lock generation wrote this." It accepts whatever byte stream arrives from whichever process sends it, in whatever order the network happens to deliver it, which under an arbitrary pause can easily be **the old, stale client's write arriving after the new, correct client's write**, silently overwriting the correct data with stale data and leaving no trace that anything went wrong.

The naive design's mistake is treating "acquired the lock" as equivalent to "safe to act," when the only thing actually guaranteed is "was, at some point in the past, the sole holder of the lock, for some finite length of time that nobody can bound in advance."

---

## 3. The architecture

Top to bottom, the shape Chubby, ZooKeeper (as HBase and Kafka's older versions use it), and etcd all converge on, once you fix the naive design properly:

```
[Client wants exclusive access: "compile and publish this build,"
 "own this region," "run this cron job once"]
   |
   v
[Lock / lease service: Chubby cell, ZooKeeper ensemble, etcd cluster]
   itself a Raft/Paxos-replicated state machine (Day 11), so the lock
   service's own answer to "who holds this lock right now" cannot
   split-brain even if some of ITS replicas fail
   analogy: a single, always-agreed-upon ledger book that several
   backup clerks all keep in perfect lockstep, so asking any one of
   them "who has the key" always gets the same answer
   |
   v
[Session / heartbeat: KeepAlive (Chubby) or ping (ZooKeeper)]
   the client must actively prove it is still alive on a schedule;
   silence for longer than the session timeout means the service
   ASSUMES death, it can never know for certain
   analogy: a night watchman who must press a button every few
   minutes or the alarm assumes he has collapsed, even though he
   might just be stuck in a stairwell, very much alive
   |
   v
[Lock granted WITH a monotonically increasing number: Chubby's
 "sequencer" (lock name + generation number + mode), ZooKeeper's
 zxid or znode version, etcd's revision]
   this number is the entire fix: it increases by exactly one every
   single time the lock transitions from free to held, with no gaps,
   no reuse, ever
   analogy: a numbered ticket at a deli counter; ticket 34 is
   objectively, unambiguously later than ticket 33, no clock involved
   |
   v
[Client does its work, carrying that number]
   |
   v
[The PROTECTED RESOURCE checks the number itself: storage server,
 database row, R2 manifest write]
   "have I already accepted a write stamped with a HIGHER number
   than this one?" if yes, REJECT this write outright, regardless of
   who sent it or how confident it is
   analogy: the storage clerk who only accepts deliveries in ticket
   order and hands a stale-ticket delivery straight back, unopened
   |
   v
[Grace period / lock-delay for resources that cannot check a number:
 Chubby's ~1 minute lock-delay after a session loss]
   a coarser fallback: simply refuse to hand the lock to anyone else
   for a fixed window after the old holder is presumed dead, giving
   any of its in-flight, stale operations time to either land harmlessly
   or time out on their own
   analogy: a hotel that will not re-issue a checked-out room's key
   for one hour, in case the previous guest is still inside packing
```

The one non-negotiable box in that diagram is the fourth one: the resource being protected has to participate. A lock service that only tells you "you have the lock" and never hands you a comparable, checkable number gives the *protected resource* nothing to verify against, which is exactly the hole Section 2c describes.

---

## 4. The transferable mechanisms

**a. A lease is a time-bounded promise, not a permanent fact, and code must never silently upgrade one into the other.** Chubby's own paper is explicit about this: sessions are maintained through **KeepAlive RPCs**, each extending a **session lease**, and a client only continues to consider itself the session's owner for as long as that lease is provably still valid, not indefinitely. The moment any code path treats "I acquired the lock earlier" as equivalent to "I hold the lock now," section 2's bug is already latent in the design, waiting for a long-enough pause to trigger it.

**b. Fencing tokens: hand out a number that only ever goes up, and make the resource itself the judge.** This is the single mechanism that actually closes the hole, and it is the core of Kleppmann's 2016 proposal: the lock service issues a token that strictly increases on every acquisition (his worked example: client 1 gets token 33, its lease expires, client 2 gets token 34), and the client is required to present that token with every write to the thing being protected. The storage server "remembers that it has already processed a write with a higher token number, and so it rejects" any write arriving later with a lower one, even if that write is perfectly well-formed and comes from a client that, moments ago, legitimately held the lock. Chubby ships this exact idea under the name **sequencer**, a string encoding the lock's name, a generation number that changes on every free-to-held transition, and the mode of acquisition, explicitly designed to be handed onward to any service the lock is meant to protect. ZooKeeper deployments that need the same guarantee reuse a value ZooKeeper already produces for free: the **zxid** (the globally ordered transaction ID of the write that granted the lock) or a znode's **version number**, both already monotonic by construction.

**c. When the protected resource cannot check a token, buy time instead of certainty: the lock-delay fallback.** Not every legacy file server or storage backend can be modified to check a fencing token. Chubby's answer for exactly that case is a **lock-delay**, typically on the order of **one minute**, during which the lock service will not hand the same lock to any other client after the previous holder's session is lost, specifically to let any in-flight operations from the presumed-dead holder either complete harmlessly or time out on their own before a new holder starts acting. This is strictly weaker than a real fencing token (it is a probabilistic "probably long enough," not a guarantee), and Chubby's own designers present it as the fallback for when the stronger mechanism is not available, not as the preferred solution.

**d. Distinguish "the lock service disagrees with itself" from "the lock service is being lied to by a stale client."** The lock/session service itself must be internally consistent, which is exactly Day 11's consensus problem: Chubby and ZooKeeper are each built on a replicated, Paxos-or-Raft-family state machine specifically so that the ensemble never gives two different clients contradictory answers about who currently holds a lock. But solving *that* problem does not solve section 1's problem at all, a perfectly consistent lock service can still be told the truth ("client A's session expired, client B now holds the lock") while client A, off doing its own thing, is the one operating on false information. Fencing tokens are the fix for the second problem; consensus is the fix for the first; a real system needs both, and confusing them is a common design mistake.

**e. Clients maintain their own local timer, specifically because clocks between machines cannot be perfectly trusted either.** Chubby's session model has the client track its own estimate of when its lease will expire locally, and if that local timer runs out before the server confirms an extension, the client waits out an additional **grace period, roughly 45 seconds** in Chubby's implementation, before it gives up on the session entirely, rather than immediately assuming failure the instant one KeepAlive round-trip is late. This exists because a single slow KeepAlive is a normal network hiccup, not proof of death, and treating every delay as a fatal timeout would make an already-fragile system flap constantly. Chubby also **raises its lease time from a default of about 12 seconds up toward 60 seconds under heavy master load**, specifically to reduce how many KeepAlive RPCs a busy master has to process, a direct load/safety trade-off: a longer lease is cheaper to maintain at scale and gives a paused client a longer window in which to be legitimately dangerous.

---

## 5. The trade-offs

**Redlock is the sharpest real illustration of this trade-off, because it was fought out in public between two credible experts and neither side was simply wrong.** Redis's own Redlock algorithm acquires a lock by writing the same key, with the same random value and a TTL, to a majority of independent Redis master nodes within a bounded time budget, and declares success if a majority accepted it inside that budget. Kleppmann's 2016 critique names two concrete problems: first, "Redlock does not have any facility for generating fencing tokens," so even if Redlock's mutual-exclusion logic is perfectly correct, any resource it protects is exactly as exposed to section 2c's stale-write race as the naive design was, because there is still no monotonic number for that resource to check. Second, Redlock's safety argument leans on assumptions about **bounded network delay, bounded clock drift, and bounded process pause**, which is precisely the class of assumption section 1's several-minute GC pause violates in the real world. Salvatore Sanfilippo (antirez, Redis's creator) responded that Redlock explicitly checks elapsed wall-clock time before and after the majority-acquisition step specifically to detect (not prevent) delays during that step, and defended Redlock as adequate for its intended use cases, things like deduplicating a cron job or preventing a cache-stampede thundering herd (Day 19), where the cost of an occasional double-execution is a wasted computation, not corrupted data.

**The actual trade-off underneath that public argument is CAP made concrete, applied per use case rather than per system.** For "make sure this cron job doesn't run twice this hour," the cost of an occasional false negative (both replicas skip it) or false positive (both run it) is low, so Redlock's availability-leaning, best-effort mutual exclusion is a perfectly reasonable choice, cheaper to run (no dedicated Chubby/ZooKeeper cell, just N ordinary Redis instances) and lower-latency to acquire. For "make sure two region servers never both write the same region" or "make sure a build worker's stale artifact never overwrites a newer one," the cost of that same false positive is silent data corruption, so the system must pay for a real fencing token, an extra check at the protected resource, and a lock service built on actual consensus (Chubby, ZooKeeper, etcd), even though that is strictly slower and more infrastructure to run. There is no universally correct choice here, the correct choice is a property of what you are protecting, and the biggest real-world mistake is picking the cheap option for a resource that actually needed the expensive one.

---

## 6. The systems-thinking lens

**The feedback loop: a paused-but-alive process is functionally a zombie that the rest of the system has already buried, and every safeguard except fencing only postpones the moment it can do damage, it never removes it.** Trace it through: normal operation, client A holds a lease, heartbeats on schedule, does its work, releases cleanly → under GC pressure, memory fragmentation, a noisy neighbor's CPU steal, or a hypervisor migration, A's pause grows past the lease timeout → the lock service, unable to distinguish "paused" from "dead," correctly reassigns the lock to client B → **A resumes with stale state and no signal that anything changed**, and if the very next thing A's code does is act on the outside world (write a file, charge a card, publish a build artifact), it does so exactly as if no time had passed. The tempting fix, "just make the lease longer so this pause fits inside it," does not break this loop at all, it only raises the bar for the next pause that will eventually exceed it, while simultaneously making every legitimate failure detector slower to react to an actually-dead client. A system tuned this way oscillates between two bad states: leases too short (false expirations under ordinary load, exactly HBase's original documented pain with ZooKeeper) and leases too long (a genuinely crashed client's lock sits unusable for longer, and a paused-but-alive client has an even bigger window to cause damage when it wakes up).

**The senior fix does not touch the timeout at all, it moves the correctness check to the one place a stale actor cannot talk its way past: the resource it is trying to act on.** Making the lease longer, adding more heartbeat retries, or building a smarter failure detector are all attempts to make the "is the client actually dead" judgment more accurate, and that judgment is fundamentally undecidable across an asynchronous network (this is the same impossibility Day 11's consensus lesson names for leader election). Fencing tokens sidestep the undecidable question entirely: the system stops trying to guess whether client A is really dead, and instead guarantees that **even if A is alive and acts anyway, its action cannot be accepted once a newer generation has taken over**, because the resource itself, not the lock service, is the final arbiter, and it only has to compare two numbers, a decidable, cheap, local check. That is the same structural move as this ledger's circuit breakers and backpressure valves (Day 13): stop trying to predict or prevent the bad event upstream, and instead make the downstream consumer of that event unconditionally safe against it arriving late.

---

## Map to Rare.lab's stack

**Where this bites concretely:** picture a build-worker pool where any available worker can claim "compile and publish this shader graph" via a `SELECT ... FOR UPDATE` style lease row in Supabase Postgres, with an `expires_at` column standing in for a lock TTL. That is section 2's naive design, transplanted directly onto Rare.lab's stack, and it fails the identical way: a worker pod gets preempted, hits a slow cold-start, or pauses on a GC-heavy runtime step mid-compile; its lease row expires; a second worker correctly claims the job and publishes a compiled artifact; then the first worker resumes, oblivious, finishes its own compile, and writes **its own, now-stale artifact** as the new pointer in the R2 manifest, silently reverting a shader graph to an older, already-superseded version. This is not a hypothetical edge case for a system whose entire mutable state, per Day 23, concentrates into exactly one pointer: that concentration is precisely what makes a stale write to it so dangerous, one wrong write there overwrites the single source of truth for the whole scene.

**The concrete fix, in the same shape as section 4b:** add a monotonically increasing epoch to the lease row itself, a plain Postgres sequence (`nextval('build_epoch_seq')`) bumped every time a lease on that job is (re)granted, playing the exact role of Chubby's sequencer or ZooKeeper's zxid. Hand that epoch to the worker when it claims the job, and, critically, require the R2 manifest write path (or the Postgres row that names the current manifest pointer) to store the epoch that produced its current value and **reject any incoming publish whose epoch is lower than the one already recorded**, the same check section 3's protected-resource box performs. This is a small, cheap addition, one integer column and one comparison, on top of infrastructure Rare.lab already has (Postgres, and the manifest pointer from Day 23), and it converts an unbounded, silent corruption risk into a loud, safe rejection: the stale worker's publish simply bounces, instead of quietly winning.

---

## References and summaries

**Martin Kleppmann: "How to do distributed locking"** (2016)
https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
The primary source for this lesson's fencing-token mechanism and its critique of Redlock: the worked example of a client whose lease expires during a pause, the token-33-then-34 diagram showing a storage service correctly rejecting a stale write, the explicit statement that "stop-the-world garbage collection pauses have sometimes been known to last for several minutes," and the argument that Redlock's safety leans on bounded-delay assumptions that a real asynchronous network, and a real paused process, will eventually violate.

**Salvatore Sanfilippo (antirez): "Is Redlock safe?"** (2016)
https://antirez.com/news/101
Redis creator's direct rebuttal to Kleppmann, arguing Redlock's explicit elapsed-time check around the majority-acquisition step protects against a specific class of delay, and defending Redlock's design goals as adequate for use cases like deduplicating scheduled jobs, where an occasional double-execution is cheap, rather than for cases demanding hard data-safety guarantees.

**Mike Burrows: "The Chubby lock service for loosely-coupled distributed systems"** (Google, OSDI 2006)
https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/
The primary source for this lesson's session-lease model, the sequencer mechanism (lock name plus a generation number that changes on every free-to-held transition, plus acquisition mode) as Chubby's built-in fencing token, the lock-delay fallback (on the order of one minute) for clients of locks that cannot check a sequencer, the default 12-second session lease extended up to roughly 60 seconds under heavy master load, and the roughly 45-second local client grace period before a session is given up as lost. Chubby is Google's production lock/leader-election service underpinning GFS master election and BigTable.

**Apache ZooKeeper Programmer's Guide: recipes for locks and leader election**
https://zookeeper.apache.org/doc/current/recipes.html
The documented basis for using ZooKeeper's own monotonic zxid or a znode's version number as a fencing token, and the session-timeout model HBase's region-server locking is built on, the source for this lesson's HBase/GC-pause breaking scenario.

**Redis documentation: "Distributed Locks with Redis"**
https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
Redis's own current documentation of the single-instance `SET key value NX PX ttl` lock pattern used in section 2's naive design, and of the Redlock algorithm itself, the majority-of-N-independent-masters acquisition scheme this lesson's section 5 trade-off discussion is built on.
