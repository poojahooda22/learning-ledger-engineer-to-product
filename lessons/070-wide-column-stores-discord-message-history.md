# Day 70 — How does Discord answer "give me the last 50 messages in this channel" in single-digit milliseconds, for a channel that has been running for a decade, inside a server with 200,000 members, when the whole platform now holds trillions of messages?

**Date:** 2026-09-07
**Difficulty:** Expert
**Topic:** Wide-column store data modeling and operation at extreme scale, via Discord's message history: Cassandra, then ScyllaDB. This lesson builds directly on pieces this ledger already has in hand: Day 21 covered the LSM-tree write path that makes sustained high-throughput inserts cheap, Day 26 covered Snowflake IDs as a sortable, collision-free identifier, Day 16 covered what happens when one key gets disproportionate traffic, and Day 13 covered breaking retry-driven feedback loops with backpressure instead of raw capacity. Discord's message store is the case where all four ideas had to work together, at a scale where getting any one of them wrong meant an outage.
**Stack relevance:** Rare.lab does not store chat messages, but it will accumulate its own unbounded, per-object event history: every edit to a node graph, every telemetry ping from an embedded runtime instance, every shader compile log, keyed by scene ID or session ID, growing for as long as that scene stays alive and embedded. The question Discord answers for "the last 50 messages in this channel" is structurally the question Rare.lab will eventually have to answer for "the last 50 edits to this scene" or "the last N runtime events from this embed," and Supabase Postgres, unpartitioned, is exactly the naive design this lesson shows failing first.

---

## 1. The company and the breaking number

**Discord, and the number 100,000,000.** In November 2015, eight months after launch, Discord's single MongoDB replica set had stored 100 million messages, and the working set, the actual data and indexes a live query needs touched, no longer fit in RAM. Discord's core read is not "fetch message by ID," it is "give me the last 50 messages in channel X, before message Y," a query that walks a descending index. The moment that index stopped fitting in memory, every one of those queries started hitting disk, and disk seeks are milliseconds where RAM access is nanoseconds, a difference of roughly six orders of magnitude, on the single most frequent query the whole product makes.

**The second, larger number: 177 nodes, and one `@everyone`.** Discord moved to Apache Cassandra in 2017 with a 12-node cluster and never looked back on the choice of database, only on how much of it they needed. By around 2022 that cluster had grown to 177 nodes holding trillions of messages, and it was breaking in a different, more specific way. Take a Discord server with 200,000 members and a busy announcement channel. Someone posts a message that pings `@everyone`. Every online member's client is notified at once and immediately fetches that same message, by the same message ID, which lives in the same Cassandra partition, on the same physical node. Thousands of near-simultaneous, independent reads for identical data all land on one node at the same instant. That node's latency climbs as it falls further behind, and because Cassandra spreads data by partition key, every other query that happens to hash to that same node, unrelated messages in unrelated channels, gets slower too. Nothing about the request rate looked unusual in aggregate. It was unusual in concentration, all of it aimed at one row on one machine.

**Third: the maintenance the cluster could no longer afford to run.** At 177 nodes, Cassandra's own housekeeping, repair (reconciling replicas) and compaction (merging SSTables and cleaning up deleted data), had become expensive enough in CPU and I/O that Discord's team had to scale some of it back just to keep the cluster stable day to day. A database's safety mechanisms becoming too costly to actually run is its own kind of breaking number: the cluster wasn't down, but the thing keeping it correct and fast was being rationed.

---

## 2. Why the naive (demo) design dies

**The obvious version:** one database, one `messages` table or collection, primary key on message ID, a secondary index on `(channel_id, created_at)` to support "recent messages in this channel." This is exactly what Discord shipped in 2015, on MongoDB, and it is exactly what almost every chat app tutorial ships today.

**Death one: the working set outgrows RAM, and the failure mode is silent until it isn't.** A database is fast while its hot index and hot data fit in memory. Nothing about crossing that threshold produces an error. Queries simply, gradually, start touching disk instead of RAM, and a random disk read for a B-tree index lookup is not amortized across other reads, each one pays the seek cost individually. Discord's 100-million-message threshold in 2015 is this exact failure: no code changed, no traffic spike happened, the data just grew past the size of the machine's memory, and the single most common query in the app got dramatically slower for everyone, all at once.

**Death two: one un-bucketed partition key turns "this channel is popular" into "this channel is unqueryable."** Suppose the next design collapses the index problem by moving to a wide-column store like Cassandra and using `channel_id` alone as the partition key, one obvious improvement. This still dies, because a wide-column store's partition is the unit that must live together on one set of replica nodes and often be read close to entirely to satisfy a query. A channel that's been open for years in an active server accumulates a partition that keeps growing, unbounded, for the entire life of the channel. Compaction has to keep rewriting that same enormous, ever-growing partition. Reads into it get slower as it grows, because more of that one partition's data must be scanned even to answer "just give me the last 50." The channel's popularity, the very thing that makes it valuable, is what kills it.

**Death three: identical concurrent reads have no memory of each other.** In the naive design, every client request is independent all the way down to the database. When `@everyone` fires and thousands of clients ask for the same message at once, none of those requests know the other 4,999 are asking the identical question. The database gets hit 5,000 times for one answer. This isn't a capacity problem that more replicas quietly fix, because all 5,000 requests hash to the same partition key regardless of how many nodes the cluster has; adding nodes spreads unrelated traffic, not this traffic.

**The real-world version:** a 200,000-member Discord server's moderator posts an `@everyone` announcement at 9am. In the naive design, that single, ordinary act of posting one message triggers a synchronized stampede of thousands of identical reads against one row, on one node, at the same instant every other channel in the product is also relying on that node staying responsive.

---

## 3. The architecture

```
Client (Discord desktop/mobile app)
  - job: render messages, send new ones, request pages of history
  - analogy: someone calling a shared office line and asking the
    receptionist for "the last 50 messages left for extension 4"

        |
        v
API / gateway monolith
  - job: authentication, permissions, routing; does not talk to the
    message database directly
  - analogy: the receptionist who takes the call but doesn't personally
    walk to the filing room

        |
        v
Rust data-service layer (one gRPC endpoint per query shape)
  - job: sit between the API and the database as the ONLY thing allowed
    to query it for this data; coalesce concurrent identical requests
    (same channel, same message range) into a single in-flight database
    query, and hand the one result to every waiter
  - analogy: a single clerk who, when five callers ask for the exact
    same filed memo within the same second, walks to the filing room
    once and reads the answer back to all five, instead of making five
    separate trips

        |
        v
ScyllaDB (Cassandra-wire-compatible, written in C++, shard-per-core)
  - partition key: (channel_id, bucket) where bucket is a 10-day time
    window, keeping any single partition under roughly 100MB regardless
    of how long the channel has existed
  - clustering key: message_id, a Snowflake ID (Day 26): a 64-bit,
    time-encoded, chronologically sortable integer, so "before message
    Y" is a plain descending range scan on the clustering key, no
    secondary index needed
  - job: durable, replicated storage with an LSM-tree write path (Day
    21): writes go to an in-memory memtable, flush to immutable SSTables
    on disk, so insert throughput stays flat regardless of total table
    size; a shard-per-core design (one OS thread pinned per CPU core,
    each owning its own slice of data and its own memory, no shared
    heap) removes the JVM garbage-collector pauses that used to spike
    Cassandra's own tail latency
  - analogy: instead of one filing cabinet growing forever, a fresh
    labeled folder is started for each channel every 10 days, and every
    page dropped into a folder is stamped with a timestamp-ordered
    number the instant it's filed, so "show me the newest 50 pages"
    never requires re-sorting the folder, just reading from the back

        |
        v
Background compaction and repair (per-node, continuous)
  - job: merge SSTables, physically remove data past tombstones
    (deleted or edited messages), reconcile replicas
  - analogy: an overnight crew that periodically re-files a folder's
    loose pages into a clean, single stack, and finally shreds the
    pages someone marked "discard" days ago
```

---

## 4. The transferable mechanisms

- **Composite partition key = natural key plus a bounded bucket.** `(channel_id, bucket)` instead of bare `channel_id` is the general fix for "one logical entity, unboundedly growing over time": pick a bucket width (Discord chose 10 days, sized to keep partitions under about 100MB) so no single partition can grow forever just because the thing it represents is popular or long-lived. This is manual sharding applied inside a single table, and it applies to anything keyed by a parent that outlives any fixed size budget: a user's activity feed, a device's telemetry stream, a document's edit history.

- **A sortable, collision-free ID doubles as both primary key and pagination cursor.** Snowflake IDs (Day 26) encode a timestamp in the high bits, so ordering by ID is ordering by time, with no ties even at massive concurrent write volume. Discord's `created_at` column was tried first as a clustering key and rejected for exactly this reason: two messages can share a timestamp, an ID cannot. This is why "give me messages before X" needs no secondary index at all, it's a native range scan on the key that's already there for other reasons.

- **The LSM-tree write path (Day 21) absorbs sustained insert-heavy traffic that a B-tree-based design cannot.** Every message is a pure insert, an append to an in-memory memtable that flushes to disk as an immutable SSTable, so write latency stays flat whether the table holds a million rows or a trillion. The cost is deferred to background compaction, and to reads, which is exactly why the *next* two mechanisms exist.

- **Tombstones are the deferred cost of "delete" and "edit" in an append-only store.** A wide-column store never overwrites data in place; a delete or edit writes a new marker (a tombstone) that a later compaction pass has to physically reconcile. A workload with heavy message deletion or editing accumulates tombstones a read must skip past until compaction catches up, the same "cost is paid later, not now" trade this ledger has already seen in Day 21's LSM-trees and Day 69's table formats, just at the level of individual rows instead of whole files.

- **Request coalescing (singleflight) turns N identical concurrent reads into 1.** Discord's Rust data-service layer sits as the sole path to the database and deduplicates simultaneous requests for the same key, so an `@everyone` stampede produces one database query with thousands of waiters attached to its result, not thousands of queries. This is the general fix for any thundering-herd-on-one-key situation (Day 16's hot-key problem), and it belongs at the narrowest possible chokepoint, one layer, one gRPC endpoint per query shape, not scattered across every caller.

- **Removing a garbage collector removes a self-inflicted latency generator.** Cassandra's JVM-based garbage collection pauses the world on its own schedule, unrelated to actual query load; ScyllaDB's C++, shard-per-core design (one thread, one slice of memory, per CPU core, no shared heap to collect) eliminates that source of tail latency entirely. The transferable lesson: some of your worst p99 spikes may come from your runtime's own memory management, not from your workload, and the fix is architectural, not "add more replicas."

---

## 5. The trade-offs

**Consistency per data type, not per system.** Cassandra and ScyllaDB offer tunable consistency per query: Discord can accept a write with acknowledgment from a subset of replicas (favoring availability and low write latency) for ordinary message sends, where a message that takes an extra moment to be visible on every replica is a non-event, while a feature like a unique username or a payment would need a different, stronger consistency guarantee and, in practice, a different data store entirely. The lesson isn't "Cassandra is eventually consistent," it's that a single product routes different fields to different consistency guarantees based on what an inconsistent read of that specific field would actually cost the user.

**Cost versus latency, with an unusual double win.** Moving from 177 JVM-based Cassandra nodes to 72 C++ ScyllaDB nodes cut node count by roughly 60% (each ScyllaDB node running around 9TB of disk against an average of about 4TB per Cassandra node) while also cutting read latency (p99 fell from roughly 40 to 125 milliseconds down to a steady ~15 milliseconds) and write latency (p99 from roughly 5 to 70 milliseconds down to a steady ~5 milliseconds). This isn't the usual cost-versus-latency dial where you pay more for less latency; it happened because the thing being removed, JVM garbage-collection overhead, was pure operational tax with no benefit to the workload, not a genuine trade-off. Genuine trade-offs (like the 10-day bucket width itself: smaller buckets mean more, smaller partitions and more cross-partition queries when reading far enough back; larger buckets risk the original unbounded-partition problem returning) still apply and had to be tuned.

**A migration of this size accepted a real, bounded risk for zero downtime.** Rewriting the storage engine under trillions of live messages, with a Rust-built migrator sustaining roughly 3.2 million messages per second and completing the full cutover in about nine days, is itself a trade: extra engineering investment in a purpose-built, throwaway migration tool, in exchange for never taking chat offline for the whole platform during the move.

---

## 6. The systems-thinking lens

The feedback loop here is a **thundering herd concentrated by a hot key, not by aggregate load**: an `@everyone` ping is a single, ordinary write that fans out into thousands of near-simultaneous, identical reads, all hashing to the same partition on the same node. That node slows down under the concentrated load; slower responses risk client-side retries or app-level re-fetches; retries land on the exact same hot partition, because the key didn't change, only reinforcing the spike. This is the same shape as Day 13's retry-death-spiral and Day 16's hot-key problem: the failure isn't "too much traffic for the system," it's "too much traffic for one narrow, specific place in the system," and every unrelated query that happens to share that node pays for it too.

The naive fix, adding more nodes to the cluster, does not touch this loop at all, because more nodes spread traffic across *different* partition keys; a stampede on one key still lands entirely on whichever single node owns that key. The senior fix is structural: put a request-coalescing layer directly in front of the database as the one mandatory path every read must take, so that N simultaneous identical requests become exactly 1 database query with N waiters attached to its result, and the fan-in happens before the database is touched, not as a queue building up inside it. Combine that with the 10-day bucketing that already bounds worst-case partition size, and a JVM-free storage engine that removes a second, entirely load-independent source of latency spikes, and the same event that used to threaten one node's stability becomes one query with a large number of listeners, indistinguishable from ordinary traffic to everything downstream.

---

## Sources

- [How Discord Stores Trillions of Messages, Discord Blog](https://discord.com/blog/how-discord-stores-trillions-of-messages): the primary 2023 source for the Cassandra-to-ScyllaDB migration, node counts, and the Rust data-service architecture. Direct fetch was blocked by this session's network egress policy (`discord.com` is not reachable from this environment), so the figures below are drawn from multiple independent secondary write-ups that quote and summarize it, and are treated as reliable specifically because they converge on the same exact numbers from different authors.
- [How Discord Stores Billions of Messages, Stanislav Vishnevskiy, Discord Engineering (2017), mirrored on Medium](https://medium.com/discord-engineering/how-discord-stores-billions-of-messages-7fa6ec7ee4c7): the primary source for the 2015 MongoDB failure (100 million messages, working set exceeding RAM), the 2017 move to a 12-node Cassandra cluster at replication factor 3, and the original `(channel_id, bucket)` partition key with a Snowflake-ID clustering key. Direct fetch was blocked by this session's network egress policy (`medium.com` is not reachable from this environment); figures are drawn from converging secondary summaries.
- [How Discord Stores Trillions of Messages, ByteByteGo](https://blog.bytebytego.com/p/how-discord-stores-trillions-of-messages): secondary source corroborating the 177-to-72 node reduction, the per-node disk figures (9TB ScyllaDB versus roughly 4TB average Cassandra), and the p99 read/write latency figures before and after migration. Direct fetch blocked by this session's network egress policy; used via search-indexed summary.
- [System Design Case Study: How Discord solved the Hot Partition problem, Engineering at Scale](https://engineeringatscale.substack.com/p/how-discord-solved-hot-partition-problem): secondary source for the concrete `@everyone`-in-a-200,000-member-server hot-partition scenario and the request-coalescing fix in the Rust data-service layer. Direct fetch blocked by this session's network egress policy; used via search-indexed summary.
- [Discord: From Billions to Trillions of Messages, A Three-Database Journey, Sujeet Jaiswal](https://sujeet.pro/articles/discord-message-storage): secondary source corroborating the three-generation storage history (MongoDB, Cassandra, ScyllaDB) and the 10-day, sub-100MB bucketing rationale. Direct fetch blocked by this session's network egress policy; used via search-indexed summary.
- Day 13 (this ledger, backpressure and load shedding), Day 16 (the hot-key/celebrity problem), Day 21 (LSM-trees and write-optimized storage), Day 26 (distributed ID generation and Snowflake IDs): the ledger's own prior lessons this one directly combines and applies to a single, named, real production incident shape.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of both of Discord's own engineering blog posts (on `discord.com` and mirrored on `medium.com`) and every third-party analysis site attempted (ByteByteGo, ScyllaDB's own resources page, and several independent engineering blogs). Every specific number in this lesson (12 nodes in 2017, 177 nodes and roughly 4TB average per node before migration, 72 ScyllaDB nodes at roughly 9TB each after, p99 read latency of 40 to 125 milliseconds falling to a steady ~15 milliseconds, p99 write latency of 5 to 70 milliseconds falling to a steady ~5 milliseconds, a Rust migrator sustaining about 3.2 million messages per second, and a roughly nine-day zero-downtime cutover) is therefore drawn from search-indexed summaries rather than a direct first-party read, consistent with how this ledger has flagged network-blocked sources before (Day 69). These figures are reported here because multiple independent secondary write-ups, published by different authors at different times, converge on the same exact values, which is the strongest corroboration available without direct access to the primary post.
