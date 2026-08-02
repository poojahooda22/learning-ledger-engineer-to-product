# Day 45 — How do you find a row by a field that isn't your shard key, without asking every shard at once?

**Date:** 2026-08-02
**Difficulty:** Expert
**Topic:** Global and secondary indexes in sharded databases: Uber's Schemaless (2014), DynamoDB Global Secondary Indexes, Vitess lookup Vindexes, and Cloud Spanner interleaved indexes, and why sharding a table by one key quietly breaks every query that filters by a different one
**Stack relevance:** Supabase Postgres holding scene/project/node rows for Rare.lab's node-based editor, plus the R2 content-addressed scene-JSON manifest from Day 23; the day either of those needs to answer "find me X by a field that isn't the row's own ID" at real scale

---

## 1. The company and the breaking number

**Uber, early 2014, running its entire trips table on a single Postgres instance.** Uber's own engineering blog is direct about how bad this had gotten: trip growth was outpacing that one database's capacity so fast that, without a new storage layer, **Uber's infrastructure would fail to function by the end of the year.** The fix, built as an internal project code-named Mezzanine and later published as "Schemaless," launched in October 2014 and split the trips data across a **fixed 4,096 shards**, each an independent MySQL database.

Sharding by itself fixes the write-throughput problem: 4,096 independent MySQL instances can absorb far more inserts per second than one Postgres box ever could. But it creates a sharper problem the instant any query needs to find a row by something other than the column you chose to shard on. "Find every trip for driver X" or "find the currently open trip in city C" no longer has an address, because if `driver_id` isn't the shard key, that driver's trips could be sitting in any of the 4,096 shards. Ask all of them, every time, and a single point lookup, the kind of query a rider or driver app fires constantly, turns into a **4,096-way fan-out**. That is the breaking number: one query becomes 4,096 queries, and the two things that matter most in a distributed system, tail latency and total backend load, both get roughly 4,096 times worse, at exactly the moment sharding was supposed to make things faster, not slower.

AWS DynamoDB puts a second, independent number on the same failure mode. A single DynamoDB partition is hard-capped at **3,000 read capacity units or 1,000 write capacity units, and 10 GB of data**, regardless of how much total capacity the table has been provisioned with. A Global Secondary Index (GSI) is partitioned by its own key, which is usually not the base table's key, so a base table that is perfectly load-balanced across its partitions can still concentrate all its writes onto one hot partition inside its own index. AWS's documentation states the consequence in one sentence: when a GSI can't keep up, **DynamoDB throttles writes to the base table itself**, even if the base table has spare capacity sitting right next to the overloaded index partition.

---

## 2. Why the naive (demo) design dies

The demo version of "let me query by something other than the shard key" is one of three shortcuts, and each one dies for a specific, concrete reason at production scale.

**Naive design A: scatter-gather, ask every shard.** For any query that doesn't filter by the shard key, fan the same query out to all shards in parallel and merge the results. This works in a demo with 4 shards. At Uber's 4,096 shards it means every such query opens up to 4,096 concurrent connections, no matter how selective the real answer is (maybe exactly one shard actually has a match). Total backend load for that one query class multiplies by the shard count, and it is usually the query class the product needs most: "my trips," "my messages," "the driver currently assigned to me" are all lookups by something other than a random shard key.

**Naive design B: keep a separate index table, but write to it and the base table independently, with no consistency mechanism between the two.** Write the row to its shard, then write a second, un-coordinated copy to an index table keyed by the field you actually want to query. This dies on partial failure: the base write succeeds, then the process crashes or the network blips before the index write lands, and the index silently drops a row forever. A driver's trip quietly disappears from "my trips" while still existing in the base table. It also dies on races: two concurrent updates to the same base row can interleave with their two index writes in the wrong order, leaving the index permanently pointed at stale data with no way to detect it happened.

**Naive design C: fix B by wrapping both writes in a real cross-shard transaction, two-phase commit, on every single write.** This restores correctness, the index can never be wrong, but it reintroduces exactly the coordination bottleneck sharding existed to remove. 2PC needs a prepare round trip to both shards, then a commit round trip, and holds row locks on both shards for the full duration of both round trips, on every write that touches an indexed field. Uber's own Schemaless writeup states this directly: because a cell and its index entries typically live on different shards, "offering consistent indexes would require introducing 2PC in the writes, which would incur significant overhead." At Uber's write volume, that overhead was not something they were willing to pay on the hot path for every trip update.

---

## 3. The architecture

The shape Schemaless, DynamoDB GSIs, and Vitess lookup Vindexes all converge on, top to bottom:

```
[Client asks: "find trips for driver_id = X"]
   |
   v
[Stateless app tier / query router]
   analogy: the front desk of a library. It does not walk every
   aisle itself; it knows whether to check the shelf directly or
   consult the card catalog first
   |
   v
[Does this query filter by the shard key?]
   YES --> hash/route directly to the ONE owning shard
           (this is the fast, common case: primary-key lookups
           never need an index at all)
   NO  --> consult the index / lookup table first
   |
   v
[Index / lookup table: field_value -> shard ID (or full
 denormalized copy of the row's needed columns)]
   analogy: the card catalog itself. It doesn't hold the whole
   book, just enough (the value you searched, plus the shelf
   number, or a full summary card if the query never needs the
   book itself) to send you straight to the right shelf, once,
   instead of walking every aisle
   |
   v
[Targeted read to the 1 (or few) owning shard(s) the index pointed at]
   total shards touched: O(matches), not O(all shards)
   |
   v
[Result returned]

Meanwhile, on every WRITE to the base row:

[Base row written to its owning shard]
   |
   v
[Propagation to the index: synchronous-in-transaction (Vitess
 consistent lookup Vindex's SELECT FOR UPDATE) OR asynchronous,
 fed by a trigger/replication stream (Schemaless's "trigger"
 framework reading the MySQL binlog; DynamoDB's internal GSI
 replication)]
   analogy: synchronous = the librarian re-files the card catalog
   entry before letting go of the book. asynchronous = the
   librarian drops a note in a pneumatic tube to a back-office
   clerk who updates the catalog a few milliseconds later, while
   the book itself is already back on the shelf
   |
   v
[Index table updated: eventually consistent (Schemaless measures
 this lag as usually well under 20ms) or transactionally
 guaranteed (Vitess, at the cost of the lock held above)]
```

The one expensive step, cross-shard coordination, is pushed either off the hot path entirely (async trigger/replication) or into a deliberately narrow, well-understood lock (Vitess's row-level `SELECT FOR UPDATE` on the owner row), instead of wrapping every write in a full 2PC transaction the way naive design C did.

---

## 4. The transferable mechanisms

**a. Route by matching first, fan out only as a last resort.** The query router needs to know, for every query it sees, whether the filter is on the shard key. If yes, it's a direct O(1) shard lookup, no index consulted at all. If no, it goes through the index, which itself resolves to a small, known number of shards. Scatter-gather across every shard should be the option of last resort for genuinely ad hoc queries, not the default path for "find my trips," and even then it deserves a hard shard-count or timeout cap so one unindexed query pattern can't silently cost 4,096x.

**b. Denormalize into the index row so the index alone answers the query, no follow-up join.** Schemaless copies the actual data columns the query needs directly into the index cell, so finding the index entry and retrieving the answer are the same shard round trip, not two. DynamoDB's GSIs offer the same choice explicitly: you pick `KEYS_ONLY`, `INCLUDE`, or `ALL` projected attributes, trading index storage size for skipping a second fetch back to the base table.

**c. Choose the index's consistency model on purpose, per use case, not by accident.** Eventually consistent (Schemaless's async trigger, DynamoDB's GSI default) keeps write throughput high and needs no cross-shard locking, at the cost of a real, measured staleness window (Schemaless: usually under 20ms). Synchronous-in-transaction (Vitess's consistent lookup Vindex, which locks the owner row with `SELECT FOR UPDATE` before the insert) guarantees the index is never wrong, at the cost of holding that lock for the write's full duration. Physically colocated (Spanner's interleaved index, storing index rows in the same physical split as their parent row) buys the fastest possible read for queries that filter by the parent key, and nothing extra for queries that don't.

**d. A hot value in the indexed field creates a hot partition inside the index, even when the base table is perfectly balanced.** Index partitioning is keyed by the distribution of the field being indexed, not the base table's distribution. A GSI on a low-cardinality field like `status`, where 99% of live rows sit in `"active"`, concentrates almost all index writes onto one index partition no matter how evenly the base table's own primary key spreads writes across its own partitions. This is Day 16's hot-key problem, reappearing one layer removed, inside a component that isn't the one you were thinking about when you designed the base table's key.

**e. Let the index push backpressure onto the write path on purpose, rather than silently falling behind.** DynamoDB's choice to throttle base table writes when a GSI can't keep up looks, at first glance, like a bug: why should a healthy base table partition get punished for a struggling index partition? It's a deliberate trade: slow down the source of truth rather than let a derived index quietly drift further and further out of date or drop updates outright.

**f. A fixed shard or index count decided up front is a real, load-bearing constraint.** Schemaless's 4,096 shards, like Instagram's 8,192-shard ceiling from Day 26, is chosen with headroom in advance, because changing it later is a genuine migration, not a config change. The number itself is a bet on future scale, made once, that everything downstream depends on.

---

## 5. The trade-offs

**Consistency versus throughput, scoped specifically to the index, not the whole system.** An eventually consistent index (Schemaless, DynamoDB GSI) never blocks the base write on cross-shard coordination, so write throughput stays high, but a read that goes through the index microseconds after a write can miss it (a lookup by the primary key would still find it fine). A synchronously consistent index (Vitess's consistent lookup Vindex) guarantees the index is never stale, but every write that touches the indexed field now holds a row lock for the duration of that write, and Vitess's own design notes call out that the alternative, full 2PC across shards for every DML, was rejected specifically because of that overhead. Neither choice is universal: Vitess ships both a plain (eventually consistent) lookup Vindex and the consistent one, and expects each table's owner to pick per use case.

**Storage and write amplification versus read latency.** Denormalizing full row data into the index (mechanism b) means every base-row update also rewrites its index copy, more total writes, more total storage, in exchange for collapsing a base-shard-then-index-shard double hop into one shard round trip on read. A `KEYS_ONLY` index avoids that duplication but forces a second fetch back to the base table on every read.

**Physical colocation versus general-purpose usefulness.** Spanner's interleaved index is the fastest possible answer for the exact query shape it's built for, filtering by the parent row's key, because the whole index lives in the same physical split as the parent and Spanner can satisfy the query from one split instead of consulting several. Google's own Spanner documentation gives the counter-example directly: querying songs by duration without filtering by the singer is a query interleaving does not help with at all, and a plain, non-interleaved global index serves it better. The design is a bet on which query shape actually dominates real traffic, not a free win across the board.

---

## 6. The systems-thinking lens

**The feedback loop: an index you never designed to be a bottleneck becomes one, and the retries it triggers land exactly where the damage already is.** Trace it: a secondary field has real-world skew (most rows share one `status` value, one city, one plan tier) → the index partitioned on that field concentrates almost all its writes onto a single partition, even though the base table itself is evenly sharded → that index partition hits its fixed per-partition capacity first, because the limit (DynamoDB's 1,000 WCU) applies per partition, not per table → DynamoDB backpressures the base table write for any row whose update would touch that hot index partition → the client retries the now-throttled write → the retry lands on the exact same hot index partition, adding load to the one place already failing, the same retry-amplifies-load shape Day 13's backpressure and load-shedding lesson names directly, except here the trigger is buried inside a derived index nobody was watching, because the team's key-design attention went into the base table, not into the index the base table quietly spawned.

**The senior fix does not add index capacity or retry harder, it removes the skew or removes the coupling.** Throwing more provisioned capacity at the hot index partition raises cost without touching the underlying skew, and retrying more aggressively feeds the loop directly. The actual fixes available are structural: give the low-cardinality indexed field enough real cardinality that no single value dominates one partition, the same move Day 16 used to shard Lady Gaga's single hot cache key into 16 sub-keys, applied here to an index partition key instead of a cache key; or move index maintenance off the synchronous write path entirely onto an asynchronous stream, which is exactly Uber's own choice: Schemaless's index updates run through a dedicated trigger framework consuming the MySQL replication log, not inline with the request that wrote the row, so a struggling index degrades its own staleness, not the rider's or driver's request latency. Both fixes are the same underlying move this ledger keeps returning to: break the loop structurally, at its source, rather than adding capacity or cleverness downstream of where it's already failing.

---

## Map to Rare.lab's stack

**Where this is still invisible today, and where it stops being invisible.** Rare.lab's scene/project/node data lives in a single, unsharded Supabase Postgres instance, so right now "find every node using shader function X across all projects" or "find every scene that references R2 asset hash Y" is a plain B-tree secondary index and a join, and it works fine, because there's only one shard: the whole database. That comfort is exactly what naive design A and B in section 2 look like before they break: they also work fine on one shard. The day the node/edge tables need to be sharded, by project ID or by workspace, to keep write throughput up as the editor and the embeddable runtime both scale, every query that currently filters by shader function ID or asset hash instead of by the shard key becomes the 4,096-shard problem from section 1, at whatever Rare.lab's own shard count turns out to be.

**The concrete move to make before that day arrives, not after:** for the specific lookups Rare.lab already knows it needs at scale, "which scenes use asset X" being the clearest one, since it ties directly to Day 23's content-addressed R2 manifest and matters for cache invalidation and garbage collection, build a dedicated `asset_usage_index` table now, even while everything still lives on one Postgres instance, keyed by the R2 content hash rather than by project ID. Populate it the same way Uber populates Schemaless's index cells: via a Postgres trigger or a logical-replication-based worker (the exact mechanism Day 17's WAL/CDC lesson covers) reacting to scene-JSON writes, not via a live join computed at query time. This costs a small amount of write amplification today, for a table that isn't remotely under write pressure yet, and in exchange, the day project data needs to shard, the asset-usage lookup already has the shape of an index table with its own independent key, and re-partitioning it is a capacity change, not a redesign, exactly the difference between Uber's async trigger framework and the naive un-coordinated dual-write from section 2 that a team under real deadline pressure reaches for first.

---

## References and summaries

**Uber Engineering Blog: "Designing Schemaless, Uber Engineering's Scalable Datastore Using MySQL"** (Part One)
https://www.uber.com/us/en/blog/schemaless-part-one-mysql-datastore/
Primary source for this lesson's core problem and consistency trade-off: Uber's single Postgres trips database could not survive projected 2014 growth, Schemaless shards data across a fixed number of shards, and secondary indexes are kept eventually consistent rather than transactionally exact because cross-shard 2PC on every write "would incur significant overhead."

**Uber Engineering Blog: "The Architecture of Schemaless, Uber Engineering's Trip Datastore Using MySQL"** (Part Two)
https://www.uber.com/us/en/blog/schemaless-part-two-architecture/
Source for the 4,096-shard configuration, the per-shard entity table plus per-index MySQL table layout, and the design goal that an index query should only ever need to consult a single shard because the needed data is denormalized directly into the index row.

**Uber Engineering Blog: "Using Triggers On Schemaless, Uber Engineering's Datastore Using MySQL"** (Part Three)
https://www.uber.com/in/en/blog/schemaless-part-three-datastore-triggers/
Source for the asynchronous trigger/streamer framework that reads the MySQL replication stream to propagate base-row changes into index tables off the synchronous write path.

**Amazon DynamoDB Developer Guide: "Understanding Global Secondary Index (GSI) write throttling and back pressure in DynamoDB"**
https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/gsi-throttling.html
Primary source for this lesson's DynamoDB numbers and mechanism: GSIs are updated asynchronously and can develop hot partitions independent of base table balance, and DynamoDB deliberately throttles base table writes when a GSI can't keep up, rather than letting the index silently fall behind.

**Amazon DynamoDB Developer Guide: "Best practices for designing and using partition keys effectively in DynamoDB"**
https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html
Source for the per-partition hard limits (3,000 RCU, 1,000 WCU, 10 GB) that apply independently to the base table and to every GSI.

**Vitess Docs: "Vindexes"**
https://vitess.io/docs/22.0/reference/features/vindexes/
Source for the general Vindex model: a primary Vindex determines shard placement, and secondary (lookup) Vindexes use a dedicated mapping table so `SELECT`/`UPDATE`/`DELETE` queries can be routed to the owning shard instead of scattering to all shards.

**Vitess Blog: "Consistent Lookup Vindex: Achieving Data Consistency without 2PC"** (April 29, 2024)
https://vitess.io/blog/2024-04-29-consistent-lookup-vindex/
Source for the synchronous-consistency mechanism used in this lesson: locking the owner row with `SELECT FOR UPDATE` inside the same transaction as the lookup-table write, avoiding full two-phase commit while still guaranteeing the index is never stale.

**Google Cloud Spanner Documentation: "Secondary indexes"**
https://docs.cloud.google.com/spanner/docs/secondary-indexes
Source for the interleaved-index trade-off: physically colocating index rows with their parent row for queries that filter by the parent key, and the documented counter-example (querying by a non-parent field like song duration) where interleaving provides no benefit over a plain global index.
