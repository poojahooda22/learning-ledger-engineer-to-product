# Day 27 — How do you put a single global order on events across five continents, when no two of your clocks agree on what time it is?

**Date:** 2026-07-09
**Difficulty:** Expert
**Topic:** External consistency in globally-distributed databases: Google Spanner's TrueTime API and commit-wait (OSDI 2012), contrasted with CockroachDB's Hybrid Logical Clocks and uncertainty-driven transaction restarts, and why a naive "just use each server's wall clock" design silently reorders transactions that never should have been reordered
**Stack relevance:** Rare.lab's Supabase Postgres is a single region today, so it gets external consistency for free from one primary; this lesson is the map for what breaks the day a shader project needs a second region, and why "add a read replica in Europe" is not a free upgrade

---

## 1. The company and the breaking number

**Google, running F1, the SQL layer Google built on top of Spanner to replace a sharded MySQL fleet for the AdWords billing and reporting system.** Before Spanner, AdWords ran on manually-sharded MySQL. Resharding it, splitting one shard into two as it grew, was, in Google's own words in the F1 paper, taking **multiple months of intense effort by a full team of engineers**, and was so risky it was normally scheduled around the times of year advertisers cared about least. F1 needed a database that could hold AdWords' data, by the time of the OSDI paper already **over 100 TB**, spread across **five datacenters in the mainland United States**, replicated **five ways with Paxos** for availability, serving **hundreds of thousands of requests per second**, while still answering the same SQL queries a single MySQL box would, including scans across **tens of trillions of rows a day** for reporting.

Here is the specific number that makes this hard in a way sharding alone does not fix. **CockroachDB, a database explicitly modeled on Spanner's design, ships with a default maximum tolerated clock offset of 500 milliseconds between any two nodes in the cluster**, and its own docs recommend node self-termination if a node's clock drifts by more than 80% of that budget from the majority of its peers. That number is not a typo or an act of excessive caution: it is a realistic, load-bearing estimate of how far apart two ordinary, NTP-synchronized servers' wall clocks can actually be in production, once you account for NTP polling intervals, virtual machine clock drift, leap-second smearing, and network jitter to the time source. Five hundred milliseconds is enormous next to the actual work a transaction does. A single Paxos-replicated write inside one datacenter commits in single-digit milliseconds. If the *ambiguity in what time it even is* on two different machines is 100 times bigger than the *time the actual work takes*, then simply reading each machine's local clock and using it to decide "did transaction A happen before transaction B" is not a rounding error. It is a coin flip, and for a system whose entire job is charging advertisers the correct amount exactly once, a coin flip on transaction order is a correctness bug, not a performance one.

---

## 2. Why the naive (demo) design dies

**The demo version: one MySQL primary in one datacenter, an `AUTO_INCREMENT` or wall-clock `updated_at` column decides order, done.** This is exactly F1's starting point, and it works precisely because there is only one clock in the whole system: the one on that single box. Every transaction that box handles really did happen in the order its own clock recorded, because nothing else's opinion about time ever entered the picture.

The instant you go multi-region, three things break, in order of how quickly they bite:

**a. Latency, if you keep a single writer.** AdWords billing serves advertisers on every continent. A single MySQL primary sitting in, say, Iowa means every write from Europe or Asia pays a full round trip, commonly **100-150ms each way**, before it's even acknowledged. That is not a hidden cost that shows up later; it shows up on the very first cross-ocean write, and multiplies across every retry, every dependent read, every user waiting on a page load.

**b. Availability, if that single writer's datacenter has a bad day.** One primary is one blast radius. A fiber cut, a power event, a bad deploy, anything that takes that region offline takes writes offline everywhere, globally, for a business (ad billing) where "briefly unable to write" is directly "briefly unable to bill correctly," which is the thing you built the whole database to guarantee.

**c. Ordering, the moment you add a second writer to fix (a) and (b), and this is the one that is easy to miss in a demo.** Say you put a Paxos-replicated shard leader in the US and another in Europe, so each region can accept local writes without crossing an ocean. Now you need a way to say, globally, "transaction A really did commit before transaction B," not just "A's local clock read an earlier number than B's." Naive fix: stamp every committed transaction with `time.Now()` from whichever machine committed it, and use that to order everything, including deciding what a later read "should" see. But because ordinary NTP-synchronized clocks across a real fleet can legitimately disagree by up to that same ~500ms window, two transactions that are 50ms apart in real wall-clock time, one committed in the US, one in Europe, can come out of this scheme with their timestamps in the *wrong* relative order. Concretely: an advertiser in London updates their daily budget cap at 09:00:00.100 UTC (their read), a batch billing job in the US reads the account 40ms later, at 09:00:00.140, and because of clock skew the US machine's local clock happens to be running 100ms behind, it stamps its read with a timestamp *earlier* than the London write. The billing job now silently reads the account's *old* budget cap, not the new one the advertiser just set, even though in real, physical time the write unambiguously happened first. Nothing crashed. Nothing threw an error. The system just quietly served the wrong answer, and because it is a timing bug, it is intermittent, non-reproducible on demand, and exactly the kind of thing that shows up as "advertiser says we billed them under the old budget" tickets weeks later with no obvious root cause in the logs.

This property, "if transaction A finishes in real time before transaction B starts, then the system-wide order must place A before B", is called **external consistency** (Spanner's term; the more general database-theory name is **strict serializability**). A single machine gets it for free. The moment you have more than one clock deciding order, you have to *build* it, and building it is the entire subject of this lesson.

---

## 3. The architecture

Spanner's answer, and F1's use of it, drawn top to bottom:

```
[Client: F1 query layer / AdWords frontend]
   sends a read-write SQL transaction (e.g. "update this campaign's
   daily budget") or a read-only query (e.g. "run this billing report")
   |
   v
[Stateless query/routing tier]
   parses SQL, figures out which shard(s) / key ranges the query
   touches, has zero memory of any prior request
   analogy: a bank teller with no memory of you between visits, who
   looks your account up fresh, at whichever branch you happen to walk
   into, every single time
   |
   v
[Paxos replica groups, one per shard ("tablet")]
   each shard is a key range, replicated across (in F1's case) 5
   datacenters via Paxos; one replica is elected leader and serializes
   writes to that shard
   analogy: a committee of clerks in five different cities who must
   get a majority "yes" vote before any single change to the ledger is
   considered official, so no one city burning down loses the record
   |
   v
[TrueTime API, called on every machine, at every commit]
   NOT a single number. Returns an INTERVAL: TT.now() = [earliest,
   latest], guaranteed to contain the true current time somewhere
   inside it. Backed in Google's datacenters by GPS receivers and
   atomic clocks, cross-checked against each other, with software
   detecting and rejecting any single clock source that starts
   disagreeing with its peers
   analogy: a wall clock that, instead of lying to you with false
   confidence ("it is exactly 3:00:00pm"), honestly tells you "it is
   somewhere between 2:59:996pm and 3:00:004pm, and I am certain the
   real time is in that window"
   |
   v
[Commit-wait: before the leader releases a read-write transaction's
 result to the client, it waits until TT.after(commit_timestamp) is
 true]
   i.e. it waits out its OWN stated uncertainty window before telling
   anyone the transaction happened, so that by the time any other
   transaction anywhere in the world could possibly start, this one's
   timestamp is guaranteed to already be safely in the past
   analogy: a cautious teller who, after stamping a check with a
   time, deliberately pauses for the length of the clock's own error
   bar before handing it back, so no other check that was truly
   earlier can still be "in flight" with a timestamp that would beat it
   |
   v
[Two-phase commit across Paxos leaders, for cross-shard transactions]
   if a transaction touches more than one shard (e.g. moving budget
   between two campaigns in different shards), one shard's Paxos
   leader acts as 2PC coordinator, the others as participants; each
   participant is itself fault-tolerant (a Paxos group, not a single
   box), so 2PC's classic weakness, a single coordinator or
   participant crashing mid-protocol and blocking everyone, is
   contained: a crashed leader is replaced by its own Paxos group's
   next election, not by a human paged at 3am
   |
   v
[Snapshot reads for read-only transactions, at a chosen timestamp,
 with zero locks]
   a read-only transaction picks a safe timestamp and reads whichever
   version of every row was committed as of that instant, using
   multi-version storage; it never blocks a writer and is never
   blocked by one
   analogy: photocopying every page of a ledger at exactly 3:00pm and
   reading only from the photocopy, so nobody flipping pages after
   you started can possibly confuse what you see
```

The one governing rule that makes all of this line up: **every timestamp Spanner ever hands out for a committed transaction is chosen, and then waited out, so that it can never be ambiguous relative to real time.** That single discipline, repeated at every shard, every region, every commit, is what lets a read-only report generated in Asia and a write committed a millisecond earlier in the US agree on which one came first, without either one ever calling the other over the network to ask.

---

## 4. The transferable mechanisms

**a. Bounded uncertainty beats false precision.** The core move is refusing to pretend a clock reading is a single, exact point. TrueTime returns `[earliest, latest]`, an honest interval, not a number with hidden error. This generalizes past clocks entirely: whenever a system is tempted to treat an estimate as exact (a load average, a queue depth, a "current time"), the more robust move is to carry the uncertainty forward explicitly and design the protocol to be correct *even in the worst case inside that interval*, rather than betting the estimate is close enough.

**b. Commit-wait: pay latency, in a bounded and known amount, to buy a global guarantee.** Instead of coordinating with every other machine on every write (which would not scale), Spanner pays a small, fixed, local cost, waiting out its own epsilon, once, at commit time. In Google's original single-datacenter-replica experiments this added roughly 5-8ms per write; by 2023 Google reported TrueTime's uncertainty at under 1ms at the 99th percentile, and Google's own engineering notes that in production, the Paxos replication round trip usually already takes longer than the uncertainty window, so the "wait" frequently costs nothing extra at all, it overlaps work that was happening anyway. The transferable lesson: a bounded, local wait that produces a global guarantee is often far cheaper than a network round trip that tries to get the same guarantee by asking someone else.

**c. Snapshot isolation via multi-version storage removes reads from the contention fight entirely.** Because every write is stored with the timestamp it committed at, a read-only transaction can pick a timestamp and read a fully consistent, lock-free snapshot as of that instant, never blocking behind a writer and never getting blocked by one. This is the same idea underneath Postgres's MVCC, just extended across shards and continents: separate "what a reader sees" from "what a writer is currently doing" by keeping versions, not by taking turns.

**d. Two-phase commit becomes safe again once each participant is itself replicated.** Classic 2PC's reputation for fragility comes from a single coordinator or participant crashing mid-protocol and leaving everyone else blocked, holding locks, forever. Spanner's fix is not to avoid 2PC, it is to make every participant in the 2PC protocol a full Paxos group instead of a single machine, so "the coordinator crashed" becomes "the coordinator's Paxos group elects a new leader and continues," a problem Raft/Paxos already solved (see Day 11), composed one layer up.

**e. Hybrid Logical Clocks: get most of the same ordering guarantee without owning atomic clocks.** CockroachDB, built explicitly as "Spanner without Google's hardware budget," uses NTP-synchronized wall clocks plus a Hybrid Logical Clock (a wall-clock value that also carries a logical counter, so it can be nudged forward when causality demands it) plus an explicit **uncertainty interval** per transaction (wall time to wall time + `max_offset`, default 500ms). Any value read whose timestamp falls inside that uncertainty window is treated as ambiguously ordered, and the transaction is simply **restarted at a higher timestamp** rather than being allowed to guess. The trade this makes explicit: no atomic clocks or GPS hardware required, at the cost of more transaction restarts under contention, and a correctness-critical dependency on `max_offset` actually holding, enforced by nodes self-terminating if their clock drifts too far from their peers' consensus. This is the general pattern: **when you cannot afford to shrink uncertainty with better hardware, shrink your tolerance for it instead, by detecting when you have hit the ambiguous case and refusing to proceed rather than proceeding on a guess.**

---

## 5. The trade-offs

**Consistency of global order versus latency of every write, made concrete.** Spanner spends real, measured milliseconds (commit-wait) on every read-write transaction, worldwide, to buy external consistency: any two transactions, anywhere on Earth, can be placed in a single global order that matches real time, with no exceptions and no "usually." A system that skips this (plain multi-region MySQL with local timestamps, or naive last-write-wins across regions) is faster on paper and cheaper to build, but the London-budget-update example in section 2 is the real, silent cost: it can and will serve stale data as if it were current, with no error raised, exactly when clock skew and timing happen to line up against you. Spanner's PACELC answer is explicit: under normal operation, be as available as a 5-way-replicated Paxos group allows (survive 2 of 5 replicas failing), and under a genuine network partition, favor consistency over availability for the affected shard, rather than let two partitions both accept writes and reconcile later.

**Hardware cost versus tolerance for restarts.** Spanner's TrueTime needs GPS receivers and atomic clocks physically installed in every datacenter, a genuine capital and operations expense most companies will never justify. That buys an uncertainty window measured in single-digit milliseconds, which keeps commit-wait cheap and restarts rare. CockroachDB's alternative costs nothing extra in hardware, riding on ordinary NTP, but accepts a much wider uncertainty window (hundreds of milliseconds by default) as the price, which shows up as more transaction restarts under contention and a hard operational requirement to actually keep clocks within that budget, or the whole correctness argument stops holding. Neither choice is free; it is a bet on which is cheaper for your business to carry: buying better clocks, or eating more restarts and a stricter ops runbook.

**Read-only versus read-write gets genuinely different treatment, on purpose.** Snapshot reads never take a lock and never wait on commit-wait at all, because a read of the past cannot violate external consistency the way choosing a *new* commit timestamp for a *write* can. This is a deliberate, load-bearing split: pay the expensive guarantee only where it is actually needed (assigning new commit order to writes), and let the cheap path (reading an already-settled past) stay cheap.

---

## 6. The systems-thinking lens

**The feedback loop: clock uncertainty and lock contention amplify each other.** Trace it through: a node's clock uncertainty grows for any ordinary reason (an NTP source degrades, a VM host is under load and steals CPU cycles from the clock, a GPS antenna briefly loses lock) → transactions must wait longer (Spanner's commit-wait) or restart more often (CockroachDB's uncertainty-interval restarts) to stay correct → those slower or restarting transactions hold locks or retry longer → more transactions pile up behind the same contended rows or ranges → under load, that pile-up itself can further degrade the machine's ability to keep its own clock disciplined (a starved NTP daemon, a scheduler too busy to service a time-sync thread promptly) → uncertainty grows further. This is the same shape as every metastable failure this ledger has covered (Day 13's backpressure, Day 16's hot key): a small degradation in one dimension (time) creates load in another dimension (contention), and that load feeds back into the first dimension, and a system that looked healthy a minute ago tips into a state that does not recover on its own even after the original trigger clears.

**The senior fix does not add more hardware or a smarter retry, it removes the dependency on trusting an estimate at the exact moment of use.** The instinctive patches, "add more NTP servers," "retry failed transactions faster", both still let an uncertain clock reading reach the commit-ordering decision and hope it happens to be fine. The actual fix, in both Spanner and CockroachDB, despite their very different hardware budgets, is structurally identical: **make the moment of commit itself refuse to proceed on an unverified assumption about time.** Spanner does this by waiting out its own stated uncertainty before ever revealing a commit timestamp to anyone. CockroachDB does it by treating any read that falls inside the uncertainty window as inherently ambiguous and restarting rather than guessing. Both are the same move this ledger keeps returning to (Day 24's fencing tokens, Day 26's clock-sanity guard on Snowflake IDs): push the correctness check to the single component closest to the risk, at the exact instant it matters, instead of trying to prevent the underlying uncertainty from ever occurring at all, which you cannot fully do in a distributed system regardless of budget.

---

## Map to Rare.lab's stack

**Where this doesn't bite yet, and exactly why.** Rare.lab currently runs a single Supabase Postgres primary. That means, today, Rare.lab already has Spanner's hardest-won property for free: one clock, one primary, one true order of every commit to a scene's node graph, with no cross-region ambiguity possible, because there is nothing to disagree with. This is worth naming explicitly so the temptation to prematurely add multi-region infrastructure is resisted: paying commit-wait-style latency, or CockroachDB-style restart complexity, buys you nothing when you have exactly one writer already producing a real, unambiguous order.

**Where the ceiling actually is.** The day Rare.lab adds a second write region, for lower latency to a new user base, or a second Supabase project for isolation, the same London-budget bug becomes a live risk: two artists in different regions editing the same shared shader graph, or two embeddable-runtime instances writing telemetry back to project state, could have their edits reordered by nothing more than which region's clock happened to read fast or slow that millisecond, silently overwriting a newer node-graph edit with an older one and showing no error to either artist. The fix, when that day comes, is not "install atomic clocks", that is Google-scale infrastructure Rare.lab will likely never need, it is the CockroachDB-shaped answer: adopt a Hybrid Logical Clock for any cross-region write path, define an explicit, monitored maximum-offset budget between regions, and make the commit path for cross-region writes explicitly detect the ambiguous case (two writes to the same node within the uncertainty window) and resolve it deterministically (by explicit merge, the same shape as Day 15's CRDTs for the node graph itself) rather than trusting whichever timestamp arrived first to the database. The one thing to build now, cheaply, before it's needed: every write to shared scene state should already carry its own logical version or a monotonic per-object counter, not just rely on Postgres's own commit ordering, so that the day a second write region exists, "which edit wins" is a question the data model can already answer without redesigning it under pressure.

---

## References and summaries

**Corbett, James C. et al., "Spanner: Google's Globally-Distributed Database"** (OSDI 2012)
https://research.google/pubs/spanner-googles-globally-distributed-database/
The primary source for this entire lesson: introduces TrueTime as an interval-returning API (`TT.now()` returns `[earliest, latest]`), describes commit-wait (a leader holds a read-write transaction's result until `TT.after(commit_timestamp)` is true, guaranteeing external consistency), reports single-replica experimental commit-wait times around 5-8ms, and documents Spanner's directory-sharded, Paxos-replicated architecture used as the storage layer under F1.

**Kevin Sookocheff, "TrueTime"**
https://sookocheff.com/post/time/truetime/
A clear, secondary explainer of TrueTime's interval semantics and why commit-wait is the mechanism that turns a bounded, honest uncertainty window into a hard external-consistency guarantee; useful for the "why wait at all" intuition without wading through the full paper's proof structure.

**Google Cloud, "Life of Spanner reads and writes"** (Spanner documentation whitepaper)
https://cloud.google.com/spanner/docs/whitepapers/life-of-reads-and-writes
Google's own current-day walkthrough of how a Spanner read-write transaction acquires locks, chooses a commit timestamp via TrueTime, performs commit-wait, and how read-only transactions instead pick a safe snapshot timestamp and read lock-free; the source for the read-only-versus-read-write split described in sections 3 and 5 of this lesson.

**Cockroach Labs, "Living Without Atomic Clocks"**
https://www.cockroachlabs.com/blog/living-without-atomic-clocks/
CockroachDB's own account of why it diverges from Spanner: instead of TrueTime's hardware-backed interval, it uses NTP-synchronized wall clocks plus a Hybrid Logical Clock and an explicit per-transaction uncertainty interval (reader's timestamp to reader's timestamp plus `max_offset`), restarting any transaction whose read lands on a value inside that ambiguous window, at the cost of more restarts under contention rather than a hardware bill.

**Cockroach Labs, "Clock Management in CockroachDB: Good Timekeeping is Key"** and CockroachDB Operational FAQs
https://www.cockroachlabs.com/blog/clock-management-cockroachdb/
https://www.cockroachlabs.com/docs/stable/operational-faqs
Source for the concrete operational numbers used in section 1 and section 5: the default 500ms `max_offset`, the recommendation to lower it to 250ms for multi-region deployments using global tables, and the self-termination rule (a node shuts itself down if its clock diverges from a majority of peers by more than 80% of the configured maximum offset), which is the mechanism that keeps the entire correctness argument from silently breaking in production.

**Ji, Xiang and Demirbas, Murat, "Use of Time in Distributed Databases (part 4): Synchronized clocks in production databases"**
http://muratbuffalo.blogspot.com/2025/01/use-of-time-in-distributed-databases.html
A 2025 survey comparing how TrueTime, Hybrid Logical Clocks, and other synchronized-clock schemes are actually used across production distributed databases today, including updated TrueTime uncertainty figures (sub-millisecond at the 99th percentile in Google's more recent infrastructure, down from the original paper's 1-7ms range), useful for seeing how far clock hardware has improved since the 2012 paper.
