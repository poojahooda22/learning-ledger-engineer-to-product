# Day 46 — How do you JOIN two tables when the matching rows live on different machines?

**Date:** 2026-08-03
**Difficulty:** Expert
**Topic:** Distributed SQL query execution across shards: Google F1's hierarchical schema over Spanner (the AdWords database), Vitess/PlanetScale's vtgate nested-loop join engine (built at YouTube), Citus's colocated versus repartition joins, and CockroachDB's DistSQL hash/merge/lookup join strategies. Why sharding a table for write throughput quietly turns every JOIN between two tables into a network problem.
**Stack relevance:** Supabase Postgres holds Rare.lab's project, scene, node, and shader-function tables today as one database, so every JOIN is free. The day node or scene data has to shard to keep write throughput up, "which nodes use shader function X" or "which scenes reference asset Y" stops being a free JOIN and becomes exactly the problem this lesson is about.

---

## 1. The company and the breaking number

**Google, running F1, the database behind the entire AdWords advertising business, since early 2012.** The published numbers from the F1 paper (VLDB 2013) are blunt: the AdWords database is **over 100 TB**, is queried by **hundreds of applications and thousands of users**, and serves **more than 100,000 queries per second** while scanning **tens of trillions of rows a day**. F1 sits on top of Spanner, which replicates every write synchronously across datacenters for durability, and that synchronous replication has a real, measured cost: **commit latencies of 50 to 150 milliseconds**, an order of magnitude higher than a single local database commit. At 100,000+ QPS, if every one of those queries that joined two related tables, say an ad campaign row and its child ad-group rows, needed its own extra round of cross-machine coordination on top of that 50-150ms commit tax, the system would not survive its own launch traffic.

A second, independent number comes from the opposite end of the same problem: what happens when a join is not designed around, but simply executed the naive way. Vitess, the MySQL sharding layer built at YouTube and now maintained by PlanetScale, documents its own join engine's basic strategy in plain terms: **a nested loop join executes the query on the left-hand side of the join, then, using that result, issues one query on the right-hand side per row returned by the left side.** A join whose left side returns a modest 50,000 rows is not one query anymore, it is **50,001 queries**: one to fetch the left side, then 50,000 individual point lookups fired at the right-hand shards. A Vitess GitHub proposal from the maintainers themselves, filed to fix exactly this ("Proposal to Optimize Join Operations with Chunking Using VALUES Statement," issue #16508), states the consequence directly: this pattern "can result in excessive network round-trips, particularly with large join result sets, negatively impacting performance." That is the breaking number for this lesson: **a JOIN, the single most basic operation in relational databases, silently turns into N+1 network calls the moment the two tables it touches live on different machines**, and neither table being large nor the query being unusual is required to trigger it. It happens by default.

---

## 2. Why the naive (demo) design dies

The demo version of a JOIN assumes both tables sit in one process, on one disk, reachable by one pointer chase. Once the tables are sharded across machines for write throughput, that assumption breaks in three specific ways, each a different naive attempt at a fix.

**Naive design A: pretend the shards are still one database and just write the JOIN.** This is not even a bad strategy, it is not a strategy: a plain SQL engine has no operator that reaches across a network socket to another machine's storage engine mid-query. Something new has to sit between the client and the shards translating "JOIN these two tables" into a sequence of network operations. Every distributed SQL system in this lesson, Citus, Vitess, F1/Spanner, CockroachDB, exists because of this one gap.

**Naive design B: broadcast the whole query to every shard and merge everything in the application.** Fetch every row of both tables from every shard, ship it all to one client or app-tier process, and do the join there in memory. This works in a demo with a few hundred rows per shard. At real scale it dies twice over: total network traffic becomes the full size of both tables, moved once per query, and the app-tier process doing the merge needs enough memory to hold both full result sets at once, so a query someone thought was "give me my active campaigns and their ad groups" turns into an out-of-memory crash on whichever server drew the short straw. Vitess's own operational guardrail for this failure mode is a configurable `warn_memory_rows` threshold specifically because in-memory join buffering at the vtgate layer is a real, measured risk, not a theoretical one.

**Naive design C: the uncontrolled nested loop, one query per row, with no batching and no limit.** This is what Vitess does by default the moment a join spans two keyspaces that are not colocated, and it is not wrong, it is exactly right for a left side with 5 rows. The failure mode is that nothing in the naive version notices when the left side stops being 5 rows and becomes 5 million: the system keeps issuing one query per row, and total query latency becomes roughly N times a single round trip, serialized or lightly parallelized, instead of one query's worth of network cost. The Vitess maintainers' own BlockJoin proposal exists to fix this specific failure by batching many left-side rows into a single right-hand query using SQL's `VALUES` clause, precisely because the unbatched version does not degrade gracefully, it degrades linearly with row count, forever.

---

## 3. The architecture

The shape Citus, Vitess, F1/Spanner, and CockroachDB all converge on for a cross-shard join, top to bottom:

```
[Client sends: SELECT ... FROM nodes JOIN shader_functions ON ...]
   |
   v
[Query router / gateway node: vtgate (Vitess), coordinator (Citus),
 gateway node (CockroachDB DistSQL), or F1 server]
   analogy: an air-traffic controller. It never flies the plane
   itself; it decides which runway (shard/node) handles which
   piece of the flight plan (query fragment)
   |
   v
[Logical plan -> physical plan: is this a colocated join, a
 broadcast join, a repartition/shuffle join, or a nested-loop
 lookup join?]
   this decision is the entire ballgame; everything below follows
   from which of these four the planner picks
   |
   v
   -- CASE 1: colocated (both tables sharded by the join key,
      matching rows already live on the same node) --
   [Each node independently joins its own local slice of both
    tables, no network traffic between nodes at all]
      analogy: two filing cabinets that were organized by the
      same label scheme from day one; you never have to walk
      files from cabinet A to cabinet B to match them up
   |
   -- CASE 2: broadcast (one side is small) --
   [The small table is copied in full to every node holding a
    shard of the big table; each node joins its local big-table
    shard against the small table it now has a copy of]
      analogy: instead of moving the warehouse to the cashier,
      hand every cashier a copy of the same one-page price list
   |
   -- CASE 3: repartition / shuffle join (neither side is
      colocated and neither is small) --
   [Exchange operator: every node hashes its rows of BOTH tables
    by the join key and redistributes them across the network so
    matching keys land on the same node, THEN each node joins its
    now-locally-complete slice]
      analogy: a massive card-sorting operation where everyone in
      a room throws their cards into the correct pile across the
      room by the card's suit, then each pile is searched alone
   |
   -- CASE 4: nested-loop / lookup join (Vitess's default for
      non-colocated joins; CockroachDB's "batched lookup join") --
   [Run the left side first (one query), then issue one indexed
    point lookup per left-side row (or, batched, per chunk of
    rows) against the right-hand shards]
      analogy: a librarian who checks out your first book, reads
      its bibliography, then walks back to the shelves once per
      citation instead of pulling the whole bibliography's worth
      of books in one trip
   |
   v
[Per-node local join result streamed back to the gateway/
 coordinator over exchange streams (CockroachDB literally calls
 this a DAG of "processors" connected by "streams")]
   |
   v
[Gateway merges, re-sorts if needed, applies any remaining
 filter/aggregate, returns to client]
```

F1 sidesteps cases 2 through 4 for its most common query shape entirely, by choosing schema, not query-time cleverness: child rows (ad groups under a campaign) are physically **interleaved** inside the same Spanner storage range as their parent row. A join between a campaign and its ad groups is not case 3 or case 4 at all, it is a **single ordered range read**, because the rows were never apart on disk in the first place.

---

## 4. The transferable mechanisms

**a. Colocation: shard both tables by the join key so matching rows are never apart.** Citus's documentation states it plainly: hash-distribute both tables on the join column with the same shard count, and "each shard will join with exactly one shard of the other table," with the shard-placement logic guaranteeing matching shards land on the same worker. F1/Spanner's interleaved tables are the same idea taken to the physical storage layer: a child row is not just sharded the same way as its parent, it lives inside the parent's own storage range. This is the only one of the four strategies that produces **zero** network traffic for the join itself, which is exactly why it is the first thing every one of these systems recommends designing for, not discovering after the fact.

**b. Repartition / shuffle: when colocation isn't possible, redistribute both sides by the join key on the fly.** This is literally the MapReduce shuffle pattern, reused as a query operator. Citus calls this out as expensive enough that it is **disabled by default**: attempting a repartition join without explicitly setting `citus.enable_repartition_joins = true` simply fails with an error, forcing an engineer to consciously opt into the cost rather than triggering it by accident. CockroachDB's DistSQL engine calls the same strategy "shuffle by hash." The cost is proportional to the size of **both** tables being redistributed, not the size of the answer.

**c. Broadcast join: move the small side, not the big side.** If one table is small relative to the other, ship the whole small table to every node holding a shard of the big table instead of shuffling the big table across the network. This converts the network cost from "proportional to the big table" to "proportional to the small table times the number of nodes," which is a much better trade whenever the size gap between the two tables is large.

**d. Nested-loop / lookup join: cheap when the left side is small, catastrophic when it silently isn't.** Push the join down to per-row indexed lookups instead of moving bulk data at all. Vitess makes this its default strategy for joins that cross keyspace boundaries specifically because it needs no upfront planning about data distribution, but its own maintainers' BlockJoin proposal is the tell: an unbounded, unbatched version of this mechanism degrades linearly with the left side's row count, with no ceiling, which is exactly the shape of failure a system should never leave undefended.

**e. Merge join: free once both sides already arrive in join-key order.** If both sides are already sorted by the join key, streaming through them once in lockstep costs a single linear pass with no hash table and no shuffle at all. CockroachDB's optimizer defaults to a merge join over a hash join whenever an index already provides that ordering, precisely because it is cheaper once the ordering exists; the mechanism does nothing to conjure that ordering into being, it only exploits it when a schema or index has already produced it.

**f. Push filters and column projection down before any of the above.** Every one of these systems tries to apply `WHERE` clauses and drop unused columns at the source shard, before data crosses the network for a shuffle or a broadcast. Vitess's query planner explicitly performs "extensive tree rewriting to push as much work down under Routes as possible," repeated to a fixed point. The cheapest byte to move across a shuffle is the one that never had to move because it was filtered out one hop earlier.

---

## 5. The trade-offs

**Consistency of the read snapshot versus availability of every shard, at query time, not just at write time.** A cross-shard join is only correct if every shard involved is read as of the same logical moment; if shard A answers with data from a millisecond later than shard B, the join can silently produce phantom matches or miss real ones. Spanner and F1 buy this correctness with TrueTime, a globally synchronized clock (the mechanism this ledger covered in depth on Day 27) so every node can agree on "as of this timestamp" without a coordination round trip per query. Systems without a synchronized global clock pay a different cost: either a coordination round trip to agree on a snapshot before the join runs, or an accepted risk of slightly-inconsistent cross-shard reads. And if one shard involved in a join is simply slow or down, the system must decide, per query, whether to wait for it (consistency of the answer) or return a partial or stale result (availability of an answer at all). Neither choice is free, and different systems in this lesson make different defaults.

**Cost and latency, split by which of the four join strategies gets picked.** Colocation is free at query time but is a schema decision made once, in advance, that constrains every future query on that table. Broadcast trades a small, repeated network cost (re-sending the small table) for skipping a much larger one. Repartition/shuffle joins pay the full cost of moving both tables' relevant columns across the network, every single time the query runs, unless the result is cached; this is precisely why Citus makes an engineer opt in explicitly rather than silently absorbing that cost. Nested-loop/lookup joins pay in round-trip count, not bytes, so they are cheap when the left side is small and brutal when it is not, and that brutality scales with data growth in a way that is invisible until the day the left side crosses whatever threshold turns "a few extra queries" into "tens of thousands of them."

---

## 6. The systems-thinking lens

**The feedback loop: a skewed join key concentrates the shuffle onto one node, that node falls behind, the query times out, and the retry re-triggers the exact same skewed shuffle onto the exact same hot node.** Trace it end to end: a repartition join redistributes rows by hashing the join key, so every node ends up owning the slice of the hash space its rows landed in → if the join key has real-world skew (one advertiser ID, one popular shader-function ID, one celebrity account, the same hot-key shape this ledger named on Day 16 and again inside a secondary index on Day 45) then the shuffle concentrates a disproportionate share of both tables' rows onto whichever node owns that one hot hash bucket → that node runs long past every other participant in the same query, because a distributed join finishes only when its slowest participant does → the client or application layer sees a timeout and retries the whole query → the retry recomputes the identical hash distribution and lands the identical disproportionate share back on the identical node, which was already the bottleneck and has had no chance to recover. This is the same retry-amplifies-load shape this ledger has now named three separate times, in three different components (backpressure and load shedding on Day 13, a hot secondary-index partition on Day 45, and now a hot shuffle node inside a distributed join), because it is not really three different bugs, it is one structural failure mode that reappears wherever a system routes work by a key with real-world skew and then lets a naive client retry blindly on failure.

**The senior fix does not add shuffle capacity or retry harder, it changes which strategy runs or breaks the retry loop structurally.** Throwing more nodes at a repartition join does not touch the skew, the same hot hash bucket still exists, it is just now surrounded by more idle neighbors. The real fixes are choices made earlier in the pipeline: pick colocation (mechanism a) for the join shapes that are known in advance and common, so there is no shuffle to skew in the first place, exactly F1's bet with interleaved tables; recognize when one side is genuinely small and force a broadcast join (mechanism c) instead of letting the planner default to a shuffle; and, for the retry itself, apply backoff and a query-level circuit breaker rather than an immediate blind retry, so a struggling hot node gets a chance to drain before the next attempt lands on it again. All three are the same underlying move this ledger keeps returning to: fix the routing or the schema at the source of the skew, rather than adding capacity or retrying downstream of where the damage is already concentrated.

---

## Map to Rare.lab's stack

**Where this is still invisible today, and where it stops being invisible.** Rare.lab's scene, node, and shader-function-library data lives in one Supabase Postgres instance, so a query like "which nodes in this scene reference shader function X" is a plain, free, single-machine join, exactly the comfort naive design A in section 2 enjoys right up until the day it doesn't. That day arrives specifically when node or edge data has to shard, most likely by `project_id`, the same shard key Day 45's lesson already flagged as the natural boundary for Rare.lab's write-throughput problem. The moment that happens, "which nodes use shader function X" stops being free: nodes are sharded by project, but shader functions are a shared, cross-project library, so this join has no natural colocation, and it becomes exactly the choice in section 3, case 2 versus case 3 versus case 4, that this lesson exists to make on purpose instead of by accident.

**The concrete move to make before that day arrives, not after:** design that specific join to be a **broadcast join (mechanism c), not a repartition join,** from the start. Rare.lab's shader-function library is almost certainly small and slow-changing, a few thousand rows at most, compared to a node table that could reach millions of rows once projects and the embeddable runtime both scale. That size gap is exactly the shape mechanism c exists for: replicate the full shader-function table to every shard (or serve it from a small, fully-replicated read cache sitting in front of whichever shard owns the query) instead of ever hashing and shuffling the much larger node table by shader-function ID. This is the direct counterpart to Day 45's `asset_usage_index` recommendation: that lesson built a dedicated index so an R2 asset-hash lookup never needs a live join at all; this lesson's version is choosing, in advance, which of the four join strategies a genuinely necessary cross-shard join should use, rather than letting the query planner default into a full repartition shuffle on the day the node table is finally large enough for that default to hurt.

---

## References and summaries

**Jeff Shute et al., "F1: A Distributed SQL Database That Scales" (Google, VLDB 2013)**
https://research.google.com/pubs/archive/41344.pdf
The primary source for this lesson's core numbers and mechanism: the AdWords database is over 100 TB, served by F1 at more than 100,000 queries per second with commit latencies of 50-150ms from Spanner's synchronous cross-datacenter replication, and F1's hierarchical schema interleaves child table rows inside their parent's Spanner storage range so a join across the hierarchy costs a single ordered range read instead of a distributed query.

**Vitess Docs: query plan classification and nested-loop join behavior**
https://vitess.io/blog/2021-09-07-examine-query-plan/ and https://systay.github.io/2021/08/27/explain-a-query.html
Source for Vitess's default cross-keyspace join strategy: execute the left-hand side once, then issue one query per row on the right-hand side, a nested loop join at the vtgate layer.

**Vitess GitHub Issue #16508: "Proposal to Optimize Join Operations with Chunking Using VALUES Statement"**
https://github.com/vitessio/vitess/issues/16508
Primary source for this lesson's breaking number and its documented consequence: unbatched nested-loop joins across shards "can result in excessive network round-trips, particularly with large join result sets," and the maintainers' own proposed fix batches left-side rows via SQL's VALUES clause.

**Vitess Docs: VTGate memory guardrails and query planning**
https://vitess.io/blog/2024-11-05-optimizing-query-planning-in-vitess-a-step-by-step-approach/
Source for the `warn_memory_rows` threshold and the description of Vitess's planner pushing filters and projections down under Routes to a fixed point before executing a join.

**Citus Docs: "Querying Distributed Tables (SQL)"**
https://docs.citusdata.com/en/v11.2/develop/reference_sql.html
Primary source for colocated versus repartition joins: hash-distributing both tables on the join key with matching shard counts produces one-to-one shard pairing and zero network shuffle, while repartition joins dynamically redistribute data across workers and are disabled by default (`citus.enable_repartition_joins` must be explicitly set to true).

**Citus Blog: "How Distributed Outer Joins on PostgreSQL with Citus Work" (2016)**
https://www.citusdata.com/blog/2016/10/10/outer-joins-in-citus/
Background source on Citus's general join execution model and the colocation requirement for avoiding a repartition shuffle.

**CockroachDB Blog: "Local and distributed query processing in CockroachDB"**
https://www.cockroachlabs.com/blog/local-and-distributed-processing-in-cockroachdb/
Source for the DistSQL physical planner: a gateway node builds a distributed physical plan as a DAG of processors connected by streams across nodes, and distributed joins run as either a hash join (shuffle by hash) or a merge join, with merge joins preferred by default whenever the data is already ordered.

**CockroachDB Docs: "JOIN expressions"**
https://www.cockroachlabs.com/docs/stable/joins
Source for CockroachDB's join strategy selection, including batched lookup joins for indexed joins and the default preference for merge joins over hash joins when input ordering already exists.

**Google Cloud Docs: "Cloud Spanner's Table Interleaving: A Query Optimization Feature"**
https://medium.com/google-cloud/cloud-spanners-table-interleaving-a-query-optimization-feature-b8a87059da16
Source for the general interleaved-table mechanism this lesson's Rare.lab mapping draws on: physically colocating a child table's rows with their parent row's storage range so parent-child joins never require cross-node coordination.
