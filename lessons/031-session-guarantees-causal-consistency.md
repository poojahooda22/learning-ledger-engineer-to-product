# Day 31 — How does a user see their own comment the instant after posting it, when the next read can land on a replica that hasn't heard about the write yet?

*2026-07-13*

---

## 1. The company and the number that breaks a naive design

Facebook's TAO, the caching and storage layer behind the social graph (posts, comments, likes, friend edges), serves **more than 1 billion reads per second** against a read:write ratio of roughly 500:1, at a 96.4% cache hit rate. That volume is only survivable because almost none of those reads ever touch a database: they are answered by regional caches sitting in front of a MySQL tier organized as one **leader region** and several **follower regions**, connected by asynchronous cross-region replication.

The number that breaks a naive version of this design is TAO's own measured replica lag: followers trail their region's leader by **less than 1 second 85% of the time, less than 3 seconds 99% of the time, and less than 10 seconds 99.8% of the time**. That sounds tiny. It is not, once you multiply it by traffic. Picture a single, ordinary action: a user comments on a friend's photo, then immediately taps back to look at the thread, the way essentially every user does after posting. If the read that follows the write is routed to *any available replica* with no memory of which replica just accepted that user's write, then for roughly 1% to 15% of comments (whatever fraction land inside that lag window on a random follower), the user's own comment will be invisible on the very next screen they look at. At a billion-plus reads a second, "a small percentage of the time" is not a rare edge case, it is a routine, constant background rate of users watching their own actions apparently fail.

This is not a partition, a crash, or an outage. Every replica is healthy and will converge to the correct state within seconds. The problem is purely about **which order a user is allowed to observe events in**, and it is exactly the kind of bug that is invisible in a demo (one laptop, one database, zero replication lag) and guaranteed at scale (many regions, asynchronous replication, network latency that does not go away no matter how much money you spend, because the round trip is bounded by the speed of light between data centers).

---

## 2. Why the naive design dies

The naive design is: stateless app servers, a load balancer that round-robins every request across all read replicas with zero affinity, and "eventual consistency" left to mean whatever the replication lag happens to be at that moment. It fails in three concrete ways.

**a. Read-your-writes breaks first, and it is the one users notice immediately.** A user posts a comment, the write lands on the leader (or their region's primary), and is acknowledged. Their next page load is routed, by round-robin, to a follower that has not yet replayed that write. The comment is gone from their own view. This is not an abstract correctness violation, it is a support-ticket generator: users conclude the post "didn't go through" and either resubmit it (creating duplicate comments the moment the lagging replica does catch up) or lose trust that the product works at all.

**b. Monotonic reads breaks next, and it is worse because it is non-deterministic per refresh.** Round-robin load balancing means the *same user*, hitting refresh twice in a row, can land on two different followers at two different points in replication lag. The second read can show *less* recent data than the first read, that is, time can appear to run backward for one person within one session. A comment count can go from 47 to 46 and back to 47 across three refreshes. Nothing in a naive design prevents this, because nothing tracks which version of the data any given client has already seen.

**c. Causal ordering across different objects breaks, and this one is silent and dangerous.** Suppose a user changes a photo album's visibility from "public" to "friends-only," then immediately uploads a new photo into it. Those are two separate writes, to two separate keys, possibly replicated on two independent paths. If replica B has already replicated the new photo but not yet the visibility change, a stranger reading from replica B briefly sees the new photo under the old, public visibility setting, a genuine privacy leak produced entirely by replication ordering, not by any bug in the visibility-check code itself. Per-key eventual consistency (each individual object eventually converges) says nothing about the *order* in which causally related writes to different keys become visible together.

A single-database, no-replicas demo never surfaces any of this, because there is only one copy of the data and every read seeing every write instantly is trivially guaranteed. The failure mode is a direct, unavoidable consequence of the thing that makes the system survive a billion reads a second in the first place: caching reads regionally and replicating asynchronously so that no single database has to answer every request.

---

## 3. The architecture, top to bottom

```
Client (browser / app)
   |  carries a per-session token: last-write logical clock ("cluster time" / vector-clock stamp)
   v
Edge / CDN
   |  serves only static, non-personalized assets; never caches per-user reads
   v
Load balancer  ->  STICKY, not round-robin
   |  routes a session by a hash of (user id / session token), not by "next server in line,"
   |  the way a call center routes you back to the same agent who has your ticket open
   v
Stateless app tier
   |  reads the client's carried clock on every request, decides where this read is *allowed* to go
   v
Regional cache (TAO-style leader cache + follower caches)
   |  L1: per-region in-memory cache of hot objects, tagged with the write version that produced them
   |  a librarian's front desk: most requests are answered from the desk, not the stacks
   v
DB leader region  +  DB follower regions
   |  one region is the leader for a given shard of the social graph; writes go there first
   |  followers replicate asynchronously, like regional newspaper print plants getting the
   |  day's page proofs couriered from head office, arriving seconds after the presses run centrally
   v
Async cross-region replication pipeline
   |  ships committed writes leader -> followers; this is the pipe whose lag (the 1s / 3s / 10s
   |  percentiles above) is the entire reason this lesson exists
   v
Read-after-write gate  (the mechanism that actually fixes section 2's failures)
   |  before serving a read, check: has *this session's* last-known write reached the replica
   |  we're about to read from? if not, route to the leader, or to a replica proven caught up,
   |  for this one session, for a bounded window, then fall back to normal routing
```

The layer that does not exist in the naive design, and is the entire point of this lesson, is the **read-after-write gate**. It is not a new database or a new replication protocol; it is a small piece of routing logic sitting between the stateless app tier and the cache/DB tier, and it is where all four of section 4's mechanisms actually live.

---

## 4. The transferable mechanisms

**a. Session guarantees: four named, separately purchasable consistency levels, not one all-or-nothing dial.** The formal vocabulary here comes from a 1994 paper out of Xerox PARC, Terry, Demers, Petersen, Spreitzer, Theimer, and Welch's *Session Guarantees for Weakly Consistent Replicated Data*, written for Bayou, an early mobile, intermittently-connected replicated database. It named four guarantees a session can be given independently: **read your writes** (a session always sees its own prior writes), **monotonic reads** (a session never sees the data move backward in time), **writes follow reads** (a session's writes are ordered after whatever it has already read, so causally dependent writes cannot be observed out of order), and **monotonic writes** (a session's writes are applied in the order it issued them). The insight that survives 30 years later, still exactly how MongoDB's causal sessions and TAO's routing both work, is that you do not have to choose one global consistency level for an entire system. You attach guarantees *to a session*, cheaply, and leave the rest of the system eventually consistent.

**b. A logical clock the client carries, not a wall-clock timestamp.** MongoDB's causally consistent sessions implement this literally: every write returns a `clusterTime`, a Lamport clock value, and the driver stores the highest clusterTime the session has seen. Every subsequent read in that session is issued with `afterClusterTime` set to that value, an instruction to the server: "do not answer this read from any replica that has not yet applied everything up through this logical timestamp." This is exactly Lamport's insight from 1978 (*Time, Clocks, and the Ordering of Events in a Distributed System*) made operational: you do not need synchronized real clocks to establish "happened-before," you need a counter that increments on every causally relevant event and gets carried forward by whoever depends on it.

**c. Sticky routing: pin a session to a specific replica or region, not to "whatever is free right now."** TAO's actual production mechanism for read-after-write is closer to this than to per-request clock-checking: clients are sticky to a specific group of followers, and writes are synchronously applied to the write-issuing tier's own leader and follower caches before being acknowledged, so a session reading back through the same tier it wrote through sees its own write, even though the *asynchronous* leader-to-other-regions replication (the 1s/3s/10s lag) is still happening in the background for everyone else. This is the same shape as sticky sessions in any load balancer, and it is cheap: it costs a routing-table lookup, not a synchronous cross-region round trip.

**d. Read-repair / bounded staleness as the fallback when sticky routing alone is not enough.** When a session's clock genuinely cannot be satisfied by the replica it would normally be routed to (say, its own region actually failed over), the system has two honest choices: route that one read to the leader (slower, but correct) or block briefly for the replica to catch up to the required clock value (also slower, but correct). What it must not do is silently serve stale data while claiming otherwise. This is the same "pay latency, not correctness" trade Day 19's cache stampede protection and Day 24's fencing tokens both make: when in doubt, the system chooses to be visibly slow rather than invisibly wrong.

**e. Per-data-type consistency assignment, matching the guarantee to what the data actually needs.** Facebook's own instrumentation (Lu et al., *Existential Consistency: Measuring and Understanding Consistency at Facebook*, SOSP 2015) found that even without universal strong consistency, real observed anomaly rates were extremely low, on the order of a handful of violations per million reads, precisely because the mechanisms above are applied selectively to the data that actually needs them (a user's own recent actions) and left off the vast majority of reads (someone else's public content, which tolerates a few seconds of staleness without anyone noticing or caring). Applying read-your-writes to *every* read in the system would mean routing all traffic through the leader and destroying the 96.4% cache hit rate that makes a billion reads a second possible in the first place. Applying it only to a session's own recent writes costs almost nothing and fixes almost all of the user-visible pain.

**f. Vector clocks / dependency tracking for writes-follow-reads across different keys.** Section 2c's privacy-then-photo bug needs more than a single per-session counter, it needs the *write* to the photo to carry a dependency on the *write* to the visibility setting, so a replica cannot apply the photo write until it has applied everything that write causally depended on. This is the same vector-clock machinery Day 22's leaderless replication (Dynamo-style quorums) uses to detect concurrent versus causally ordered updates, applied here not to resolve write conflicts but to sequence *visibility* of unrelated keys that a single user's actions have entangled.

---

## 5. The trade-offs

**CAP made concrete, per data type, not per system.** TAO chooses availability and low latency over global strong consistency for the graph as a whole (eventual consistency, billions of cheap cache reads), and then buys back just enough consistency, read-your-writes and monotonic reads for the acting session, to make that choice invisible to the person it would otherwise burn. This is a strictly better trade than picking one CAP point for the entire system: uniform strong consistency would mean every read is a synchronous, cross-region, leader-serialized operation, which at TAO's actual traffic would require an amount of leader capacity and inter-region bandwidth that does not exist at any price point that keeps the product's unit economics sane. Uniform weak consistency, the naive design, is cheap but user-hostile in exactly the way section 2 describes.

**DynamoDB prices this trade explicitly, in the same table, on the same data.** A DynamoDB `GetItem` call with `ConsistentRead: false` (eventually consistent) costs half the read-capacity units of the same call with `ConsistentRead: true` (strongly consistent); AWS literally sells you consistency as a per-request line item, 2x the price for a guarantee that the read reflects every prior successful write. And it is capped by geography: standard DynamoDB global tables only support eventually consistent reads *across* regions, full stop, no amount of money buys a strongly consistent cross-region read in that mode, because a real speed-of-light round trip between regions is the floor, not a configuration choice. (AWS's newer multi-Region strong consistency mode for global tables buys this back by synchronously replicating before acknowledging the write, at the direct cost of higher write latency, the same trade, just moved from read time to write time.)

**Cost vs. latency, stated plainly: consistency bookkeeping is cheap; consistency enforcement is not.** Carrying a Lamport clock value in a session token costs a few bytes per request. Routing a session stickily to the replica that has its writes costs a hash lookup. Actually *enforcing* strong consistency on every read, by contrast, costs a synchronous round trip to a leader or a quorum on every single one of a billion daily reads. The entire lesson here is: the bookkeeping (b, c, f above) is nearly free and should be applied broadly; the enforcement (routing to the leader, blocking for replica catch-up) is genuinely expensive and should be reserved for exactly the reads that need it, a specific session, reading back its own specific recent write.

---

## 6. The systems-thinking lens

**The feedback loop here is not a classic overload spiral, it is a perception-driven retry storm, and it is worth naming separately because it is easy to miss.** Trace it: a user's own write lands on a lagging replica's blind spot → their next read appears to show the write "failed" → they resubmit the same action (a duplicate comment, a repeated form submission) or refresh repeatedly hoping to see it appear → every one of those extra reads and duplicate writes is *additional, unnecessary load* on the exact same lagging replication path that caused the problem in the first place → under real traffic, thousands of users independently doing this at once during a spike (a viral post, a breaking-news comment thread) manufactures a genuine thundering herd out of what began as a pure consistency illusion, not an actual capacity shortfall. This is a close cousin of Day 9 and Day 13's retry-storm and backpressure patterns, but the trigger is different: no server is actually failing or overloaded at the start of this loop, the system is lying to the user about the state of their own actions, and the user's rational response to that lie is what turns a non-event into real, additional load.

**The senior fix is not "replicate faster."** Replication lag has a floor set by physical network latency between regions, no amount of engineering effort drives a cross-continent round trip to zero, and no amount of extra hardware changes that. The fix that actually breaks the loop is to stop the *perception* problem at its source: guarantee read-your-writes and monotonic reads for the one session that just acted, via sticky routing and a carried logical clock, so the user's own next read never lands in the blind spot that provoked the resubmission in the first place. That is a targeted, cheap intervention (section 4's mechanisms) applied exactly where the feedback loop starts, rather than an attempt to eliminate replication lag everywhere, which is both impossible and unnecessary, because the vast majority of reads, someone else's public content, were never going to provoke a retry storm regardless of how stale they briefly are.

---

## Map to Rare.lab's stack

**Rare.lab's own multiplayer runtime already sits close to this exact problem, one layer removed.** The CRDT-based node-graph sync (Day 15) guarantees eventual convergence between collaborators, but convergence alone does not guarantee that the person who *just made an edit* sees their own edit reflected the instant their own client re-renders, if that re-render path ever reads from a different cached snapshot than the one their own edit was applied to. The same class of bug from section 2a, "my own action appears to have vanished," is exactly as support-ticket-generating in a node editor as in a comment thread: a user drags a node, the panel briefly shows the old position because a stale cached read of the graph state raced the local optimistic update, and the user assumes the drag failed and repeats it, the same retry-storm seed as section 6, just local rather than cross-region.

**The concrete, proportionate fix: give every editing session a monotonically increasing local edit counter, and gate the runtime's own read path on it, before ever reaching for anything as heavy as a full causal-consistency protocol.** Rare.lab does not operate at TAO's scale yet, and does not need MongoDB-style cluster-time bookkeeping across a fleet of replicas. What transfers directly, cheaply, today: every local edit increments a per-session sequence number; any render or state read the local client performs must be gated on "have I applied at least sequence number N," the exact read-your-writes guarantee from section 4a, just enforced locally against the client's own optimistic-update queue rather than against a remote replica. The moment Rare.lab's runtime grows a server-authoritative cache layer sitting between the client and Supabase Postgres (a natural next step once concurrent-editor counts outgrow what a single shared WebGL context and direct Postgres reads can serve), that same sequence number becomes the seed for exactly the sticky-routing and logical-clock mechanisms this lesson describes, applied at the moment it actually starts to matter, not before.

---

## References and summaries

**Bronson, Amsden, Cabrera, et al.: "TAO: Facebook's Distributed Data Store for the Social Graph"** (USENIX ATC 2013)
https://www.usenix.org/conference/atc13/technical-sessions/presentation/bronson
The primary source for this lesson's breaking number: TAO's billion-plus reads/second at a roughly 500:1 read:write ratio and 96.4% cache hit rate, its leader-region/follower-region topology, and its measured cross-region replica lag (85% under 1 second, 99% under 3 seconds, 99.8% under 10 seconds). Also the source for TAO's actual read-after-write mechanism: sticky client routing to a tier whose leader and follower caches are updated synchronously at write time, even while broader cross-region replication remains asynchronous.

**Lu, Merchant, Chandra (Facebook / Princeton): "Existential Consistency: Measuring and Understanding Consistency at Facebook"** (SOSP 2015)
https://www.cs.princeton.edu/~wlloyd/papers/existential-sosp15.pdf
Facebook's own production measurement of how often eventual consistency actually produces visible anomalies once session-scoped read-your-writes mechanisms are in place, the source for this lesson's claim that real-world violation rates land in the single digits per million reads, evidence for section 4e's argument that targeted consistency mechanisms, not blanket strong consistency, are what make the low anomaly rate affordable.

**Terry, Demers, Petersen, Spreitzer, Theimer, Welch: "Session Guarantees for Weakly Consistent Replicated Data"** (PDIS 1994)
https://www.researchgate.net/publication/3561300_Session_guarantees_for_weakly_consistent_replicated_data
The original source of the four named session guarantees this lesson builds on: read-your-writes, monotonic reads, writes-follow-reads, and monotonic writes, developed for Bayou, an early weakly-connected mobile replicated database at Xerox PARC. The paper that established the idea of attaching consistency guarantees to a session rather than to the whole system.

**MongoDB, Inc.: "Causal Consistency and Read and Write Concerns"** (Database Manual)
https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/
The concrete, shipping implementation of section 4b's logical clock mechanism: MongoDB's `clusterTime` (a Lamport clock) and `afterClusterTime` read concern, used inside causally consistent sessions to give read-your-writes, monotonic reads, monotonic writes, and writes-follow-reads guarantees, the modern production descendant of the 1994 Bayou paper's terminology.

**Amazon Web Services: "DynamoDB Read Consistency"** and **"Global Tables: How It Works"** (Developer Guide)
https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html
https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html
The source for this lesson's section 5 cost example: strongly consistent reads costing twice the read-capacity units of eventually consistent reads on the same table, and the documented limitation that standard DynamoDB global tables support only eventually consistent reads across regions, with multi-Region strong consistency (MRSC) as the newer, explicitly higher-write-latency alternative that buys the same guarantee back at write time instead of read time.
