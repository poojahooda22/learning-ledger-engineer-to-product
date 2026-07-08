# Day 26 — How do you hand every tweet a unique ID, when no single database is allowed to be the one place that says "next"?

**Date:** 2026-07-08
**Difficulty:** Expert
**Topic:** Distributed unique ID generation: Twitter's Snowflake service (announced June 2010) and Instagram's in-database composite-ID scheme (2012), and why a random UUIDv4 or a single auto-increment column both fail this problem in opposite ways
**Stack relevance:** Supabase Postgres as the single row-identity authority for scene/project/node objects, and the client-side, node-based editor that wants to create many new nodes and edges in one offline-feeling burst without a network round trip per object

---

## 1. The company and the breaking number

**Twitter, in 2010, running on sharded MySQL and starting to adopt Cassandra.** Twitter's own engineering blog stated the problem in one line when they announced Snowflake on June 1, 2010: neither sharded MySQL nor Cassandra has any built-in way to generate a unique ID once your data lives on more than one database, and Twitter needed something that could mint **"tens of thousands of ids per second, in a highly available manner."**

Here is the number that makes "just shard the database" insufficient on its own. By 2013 Twitter's own engineering blog put the platform's steady-state load at roughly **5,700 tweets per second**, and documented a real, observed peak of **143,199 tweets per second**, hit on August 2, 2013 (11:21:50 PM JST, August 3) when Japanese viewers live-tweeted a broadcast of the film *Castle in the Sky*, roughly **25 times** the normal rate, in the same single second. That peak did not blip the site. Hold that number next to the naive alternative: a single MySQL table with an `AUTO_INCREMENT` primary key can only ever process inserts one at a time, in strict order, on one master. There is no version of "buy a bigger master" that absorbs a 25x, multi-tens-of-thousands-per-second spike arriving in a single second, and there is no way to shard that table across many masters without immediately reintroducing the question this lesson is about: if tweet IDs now come from a hundred different database shards instead of one, what stops two different shards from independently generating tweet `#4,502,981` at the same moment?

That is the actual breaking number, and it is not really a raw-throughput number, it is a **coordination** number: however many IDs per second you need, the mechanism that hands them out cannot itself become the new single serialization point, or you have just rebuilt the exact bottleneck sharding was supposed to remove, one layer up.

---

## 2. Why the naive (demo) design dies

The demo version of "give every row a unique ID" is whichever of these two options a tutorial reaches for first, and each one dies for a different, specific reason once the system is sharded across many machines.

**Naive design A: a single `AUTO_INCREMENT` column on one MySQL table.** This is exactly Twitter's actual starting point in 2006. It fails in three concrete ways the instant you try to scale past one database:

**a. The counter is a lock, and every single insert across the whole system has to wait its turn for it.** `AUTO_INCREMENT` guarantees uniqueness by serializing every insert through one number, on one server. Total system write throughput is capped at whatever that one master can do, no matter how many read replicas or application servers sit in front of it, because reads don't need the counter, only writes do, and every write needs it.

**b. You cannot shard the table without the counters colliding.** The entire reason to move to sharded MySQL or Cassandra is to spread writes across many machines. But `AUTO_INCREMENT` counts locally, per table, per server. Shard 1's table and shard 42's table each start counting from 1 independently. The moment a client needs a single tweet ID to be globally unique, so that a merged timeline or a `WHERE id > X` range query works across shards, two different shards handing out the same small integer is a guaranteed collision, not a rare one.

**c. "Just add a central ID-issuing service" relocates the bottleneck instead of removing it.** The obvious patch is to keep one dedicated service that does nothing but hand out the next global ID (a single `SELECT nextval()` service, or a single Redis `INCR`). But now every insert, on every shard, has to make a synchronous network round trip to that one service before it can proceed. You have re-created naive design A's single point of serialization and failure, just moved it into its own box, and added a network hop's worth of latency to every single write in exchange for nothing.

**Naive design B, the usual "fix": generate a random UUIDv4 for every row instead.** This solves uniqueness (collision probability is astronomically low) and needs zero coordination between machines, which looks like exactly what the problem wants. It trades the coordination bottleneck for a different, quieter one: **index locality**. A production, well-documented Percona benchmark found a real 1-billion-row table using a `CHAR(36)` UUIDv4 primary key came out to **roughly 1 TB**, and shrank to **about 450 GB, roughly half**, once rebuilt with an ordered key instead. The mechanism is that InnoDB (and most B-tree-backed storage engines) keep pages efficiently packed, close to 94% full, when new keys arrive in roughly increasing order, because each insert lands at the end of the tree. A UUIDv4 is, by design, uniformly random across the entire keyspace, so each insert lands at a random point in the tree instead, forcing constant page splits, leaving pages only around 50% full, and turning every insert into a likely disk read for a table too large to fit in the buffer pool. A random ID also throws away the one piece of information a chronological product almost always wants for free: **which row came first**, without a separate `created_at` index.

Twitter and Instagram were each solving the same underlying problem, "unique IDs, minted independently on many machines, with no shared counter to wait on", but neither naive design gets there: A recreates the old bottleneck, B trades it for silent write-amplification and loses ordering.

---

## 3. The architecture

The shape Twitter's Snowflake and Instagram's Postgres-native scheme both converge on, top to bottom:

```
[Client posts a tweet / creates a row]
   |
   v
[Stateless app tier: whichever API host happens to handle this request]
   analogy: whichever teller window you walk up to at the bank; it
   does not matter which one, because none of them need to ask a
   central office for permission to serve you
   |
   v
[Local, in-process ID minting: no network call, no lock, no other
 machine consulted]
   reads: (1) current time in milliseconds, (2) this machine's own
   pre-assigned worker/shard identity, (3) a sequence counter that
   only this process touches, held purely in local memory
   analogy: a numbered-ticket dispenser bolted to the wall at every
   single counter in the building, each pre-loaded at install time
   with a different starting letter (A-107, B-107, C-107...) so two
   machines can never print the same ticket, without ever calling
   each other to check
   |
   v
[Clock-sanity guard: is current_time_ms >= last_time_ms seen by
 THIS worker?]
   if the local clock has jumped backward (NTP correction, VM
   migration, hypervisor pause) since the last ID this worker
   minted, REFUSE to mint rather than silently emit a duplicate or
   out-of-order ID
   analogy: the ticket dispenser jamming and demanding a technician
   rather than quietly reprinting a ticket number it already gave
   out five minutes ago
   |
   v
[64-bit composite ID assembled: timestamp bits (high order, for
 rough chronological sort) | worker or shard-identity bits (assigned
 ONCE, at process/shard startup, via ZooKeeper for Snowflake, or
 statically for a fixed Postgres shard) | per-millisecond sequence
 bits]
   the three fields occupy disjoint bit ranges, so uniqueness is
   guaranteed by CONSTRUCTION, not by checking after the fact
   |
   v
[Row written to whichever shard owns it: MySQL/Cassandra shard
 (Twitter) or the Postgres schema for this logical shard (Instagram)]
   the ID doubles as useful routing/ordering metadata downstream:
   decode the timestamp bits and you know roughly when any row was
   created without a separate column or index lookup
```

The one-time coordination step, assigning each worker or shard its identity bits, is the only place ZooKeeper (or, in Instagram's case, a fixed, statically-assigned shard map) appears at all. It happens once, at boot, never again per request, which is exactly what makes the hot path (the actual ID mint, happening tens of thousands of times a second) able to run with zero network calls and zero shared locks.

---

## 4. The transferable mechanisms

**a. Uniqueness by disjoint bit ranges beats uniqueness by probability.** A random UUIDv4 is unique because collisions are astronomically unlikely across 122 random bits; that is a probabilistic guarantee. A Snowflake-style ID is unique because the timestamp, worker-identity, and sequence fields each occupy their own fixed slice of the same 64-bit integer, and no two workers are ever assigned the same identity bits at the same time; that is a structural guarantee, true by construction, not by odds. The general pattern: whenever you need many independent producers to generate non-colliding identifiers with zero coordination per item, partition the identifier space itself so that each producer owns a disjoint slice of it, rather than relying on random chance or a shared counter.

**b. Coordinate once, at startup, never on the hot path.** Twitter's original Snowflake implementation assigns each worker a 5-bit datacenter ID and a 5-bit worker ID via ZooKeeper exactly once, when the process comes up; after that, minting an ID touches only local memory. Instagram made the equivalent trade even cheaper: each Postgres schema (logical shard) already knows its own shard number statically, so there is no runtime coordination step at all, ever, for ID generation. The transferable lesson: push coordination to the boundary of a component's lifecycle (startup, shard assignment, deployment), not into the path of every individual operation, and the operation itself becomes something that can run tens of thousands of times a second with no shared bottleneck.

**c. Treat the system clock as an external dependency you must guard, not a fact you can trust.** Because the timestamp occupies the high-order bits (41 bits, enough for roughly 69 years from a custom epoch before it needs redefining, the same category of deliberate finite-lifespan trade-off as the Unix Y2038 problem), a clock that jumps backward, from an NTP correction, a VM pause, or a hypervisor migration, could otherwise cause a worker to mint an ID smaller than one it already handed out, silently breaking both uniqueness and the chronological-sort guarantee. The documented fix is to detect it explicitly and refuse to mint rather than limp forward: treat "is my own clock currently trustworthy" as a check with the same seriousness as checking whether a remote dependency is up.

**d. Rough sortability is a deliberate design constraint, paid for with real bits, not a free side effect.** Twitter's blog post is explicit that tweet IDs need to be "roughly sortable", because that is how Twitter's own clients order tweets, so the timestamp goes in the highest bits on purpose. This is the direct fix for naive design B's problem: an ordered-enough key keeps B-tree inserts landing near the end of the index rather than scattered randomly across it, which is the mechanical reason the Percona benchmark showed roughly double the storage size and far more random disk I/O for a UUIDv4 primary key versus an ordered one.

**e. When running a separate ID service is not worth the operational cost, fold the same idea into infrastructure you already run.** Instagram evaluated Snowflake and explicitly decided that standing up and operating a whole separate ID-generating service was more complexity than they wanted, and instead generated the exact same shape of composite ID (41 bits timestamp from a custom Jan 1, 2011 epoch, 13 bits logical shard ID allowing up to 8,192 shards, of which they used about 2,000, and 10 bits of sequence, reusing Postgres's own existing auto-increment machinery, good for 1,024 IDs per shard per millisecond) directly inside a PL/pgSQL function on each shard. Same mechanism, zero new infrastructure, because the pieces they needed (a known shard number, a working auto-increment primitive) already existed inside the database they were already running.

---

## 5. The trade-offs

**Consistency of global ordering versus availability of minting, made concrete.** A Snowflake-style ID is only *roughly* time-sortable across different workers: two tweets posted in the same millisecond, one on worker 3 and one on worker 12, could come out with worker 12's ID numerically smaller even though worker 3's tweet was posted a fraction of a millisecond earlier, because the sequence and worker-identity bits, not just the timestamp, determine final ordering within that millisecond. Twitter and Instagram both explicitly accept this. The alternative, a single authoritative counter that guarantees perfectly strict global ordering (a lone SQL sequence, a single Redis `INCR`), is available: it gives you a stronger consistency guarantee on ordering, at the direct cost of becoming the single dependency every write must reach, which is precisely naive design A's bottleneck. Both companies chose availability and horizontal independence over perfect global ordering, because "which of two tweets posted in the same millisecond technically comes first" is a much cheaper thing to get slightly wrong than "the entire write path for the whole product depends on one box staying up."

**Cost of a standalone service versus latency and coupling of an in-database scheme.** Twitter's Snowflake runs as its own fleet of processes, a genuine piece of extra infrastructure to deploy, monitor, and keep ZooKeeper-coordinated, but in exchange it decouples ID generation entirely from any one storage technology; the same Snowflake service can hand IDs to MySQL, Cassandra, or anything else, and worker count can scale independently of how the data itself is sharded. Instagram's in-database function costs nothing extra to operate, it rides on Postgres they already run, but it hard-couples the ID format to a shard count decided up front (2,000 of a maximum possible 8,192 with 13 bits); changing the number of logical shards later is a real migration, not a config change. Neither choice is universally correct: it is a bet on which axis, operational surface area or future re-sharding flexibility, is more expensive for that specific company to pay down the line.

---

## 6. The systems-thinking lens

**The feedback loop: a clock is a distributed dependency wearing a disguise, and treating it as infallible is what turns a rare NTP hiccup into a silent, hard-to-trace correctness bug.** Trace the failure path: a worker's local clock, for any of a dozen ordinary reasons (leap-second smearing done wrong, a VM live-migration pause, a manual NTP correction after drift), momentarily reports a time earlier than the last timestamp that same worker already used → if the ID-minting code does not check for this, it happily assembles a new ID using the smaller timestamp → that new ID can now collide with, or numerically precede, an ID minted moments earlier by the very same worker → the collision, if it reaches a unique-index constraint, surfaces as a mysterious insert failure far downstream, disconnected in time and in the logs from the clock event that actually caused it, which is exactly the kind of bug that costs a debugging session tracing backward through layers that all look individually correct.

**The senior fix does not add more monitoring or a smarter retry, it removes the need to trust the clock at the moment of use.** The instinctive patch, "add alerting on clock drift" or "retry the insert on a unique-constraint violation," treats the symptom: it still lets a bad clock read reach the ID-assembly step, and only catches the damage after something downstream already broke. The actual fix documented in Snowflake's own design is a hard, local, synchronous guard exactly at the point of use: compare this millisecond to the last millisecond this worker minted against, and if it goes backward, refuse to proceed at all rather than proceed on faith. That is the same structural move this ledger keeps returning to: Day 24's fencing tokens made the *protected resource* the final judge of staleness instead of trusting a lock's holder to know it was still valid; here, the *ID-minting code itself* is made the final judge of its own clock's validity instead of trusting the OS clock to always move forward. In both cases, the fix is not "detect the bad case more cleverly downstream", it is "make the one component closest to the risk refuse to act on unverified state", which is cheaper, faster, and closes the hole instead of shrinking it.

---

## Map to Rare.lab's stack

**Where this bites concretely:** Rare.lab's node-based editor is exactly the workload that makes a single Postgres `IDENTITY`/`SERIAL` column for node and edge IDs feel fine in a demo and then slow in real use. A user dragging out a new subgraph, say a 40-node procedural noise network, generates dozens of new node and edge rows in a tight burst, from a single browser session, and each one currently needs its ID assigned by asking Supabase, meaning the client either waits on a round trip per node before it can even locally reference the new node in an edge, or it has to fake a temporary local ID and rewrite every reference once the server's real ID comes back, undo/redo history included. That is naive design A's cost, paid on every drag-and-drop, not just at extreme scale: one authority, reached over the network, for an operation that conceptually has nothing to do with any other client.

**The fix is the same shape as section 4a and 4b:** assign each active editor session (or each embeddable-runtime instance) a small, disjoint worker-identity slice once, at session start, the same one-time coordination Snowflake does via ZooKeeper and Instagram does via a static shard number, for instance a short session ID handed out when the client authenticates against Supabase. From then on, the client mints its own composite node and edge IDs entirely locally: a timestamp for rough chronological ordering (useful for undo/redo and for diffing scene versions), the session's worker bits, and a local per-session sequence, with zero network round trips for the actual ID assignment. New nodes and edges become referenceable, linkable, and undo-able the instant they're created client-side, and still merge into Supabase safely afterward, because uniqueness was guaranteed by construction, not by whichever row the server happened to insert first. This is the complementary primitive to Day 23's content-addressed R2 manifest: a content hash is the right identity for an immutable scene-JSON blob, because two identical blobs should collapse to one address, but it is the wrong identity for a live, mutable Postgres row like a node or a project, where you want a stable, sortable, unique reference that survives edits, exactly the gap Snowflake-style IDs fill.

---

## References and summaries

**Twitter Engineering Blog: "Announcing Snowflake"** (June 1, 2010)
https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake
The primary source for this lesson's core problem statement and design: neither sharded MySQL nor Cassandra generates cross-shard unique IDs on its own, Twitter needed "tens of thousands of ids per second, in a highly available manner," IDs needed to be roughly time-sortable and fit in 64 bits, and the resulting design composes a timestamp, a worker number (assigned at startup via ZooKeeper), and a per-thread sequence number, generated with no runtime coordination between machines.

**Twitter Engineering Blog: "New Tweets per second record, and how!"** (August 16, 2013)
https://blog.twitter.com/engineering/en_us/a/2013/new-tweets-per-second-record-and-how
Source for this lesson's concrete scale numbers: a steady-state rate of roughly 5,700 tweets per second, and an observed peak of 143,199 tweets per second during a Japanese broadcast of *Castle in the Sky* on August 2-3, 2013, about 25 times the normal rate, handled without the site stuttering.

**Instagram Engineering: "Sharding & IDs at Instagram"** (2012)
https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c
The primary source for this lesson's alternative, in-database approach: Instagram considered Snowflake, decided the operational cost of a standalone ID service wasn't worth it, and instead generates composite 64-bit IDs directly inside each sharded Postgres schema via a PL/pgSQL function, using 41 bits of timestamp from a custom January 1, 2011 epoch, 13 bits of logical shard ID (up to 8,192 possible, roughly 2,000 used), and a 10-bit sequence built on Postgres's existing auto-increment, good for 1,024 IDs per shard per millisecond.

**Wikipedia: "Snowflake ID"**
https://en.wikipedia.org/wiki/Snowflake_ID
Consolidated reference for the generic 64-bit layout (1 sign bit, 41 timestamp bits, 10 node-ID bits, 12 sequence bits, 4,096 IDs per millisecond per node by default) and for later adopters of the same scheme, including Discord (custom epoch of January 1, 2015) and Mastodon.

**Percona Database Performance Blog: "MySQL UUIDs - Bad For Performance? Let's Discuss"** (November 22, 2019)
https://www.percona.com/blog/2019/11/22/uuids-are-popular-but-bad-for-performance-lets-discuss/
Source for this lesson's naive-design-B numbers: a real 1-billion-row InnoDB table using a random UUIDv4 `CHAR(36)` primary key measured at roughly 1 TB, versus roughly 450 GB after rebuilding with an ordered key, and the underlying mechanism of random inserts causing B-tree page splits and low page-fill factor versus the roughly 94%-full pages achieved by sequential insert order.
