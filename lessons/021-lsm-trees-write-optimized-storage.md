# Day 21 — How does a database survive trillions of writes without a compaction meltdown?

**Date:** 2026-07-02
**Difficulty:** Expert
**Topic:** Write-optimized storage engines — LSM-trees, memtables, SSTables, compaction strategies (size-tiered vs leveled), tombstones, shard-per-core — via Discord's Cassandra-to-ScyllaDB migration
**Stack relevance:** Cloudflare R2 (content-addressed immutable scene JSON + manifest), Supabase Postgres (B-tree, single-writer today), any future per-edit collaboration event log in Rare.lab's multiplayer editor

---

## 1. The company and the breaking number

**Discord. By 2022 the platform stored trillions of messages, and its single largest Cassandra cluster had grown to 177 nodes while the team was, in their own words, "frequently firefighting."**

The story has two acts, both publicly documented by Discord's own engineers.

Act one, 2017: Discord started in 2015 on a single MongoDB replica set. By early 2017 they were pushing past **120 million messages a day** and MongoDB's single-primary replica set could not keep up, so they moved to Apache Cassandra with **12 nodes**. Cassandra was chosen specifically because it is a write-optimized, log-structured store: automatic failover, no separate caching layer needed, and — the detail that matters for this lesson — a storage engine built to absorb a firehose of small, constant writes without buckling.

Act two, 2022 to 2023: the same design that made Cassandra the right choice at 12 nodes started to show its seams at 177 nodes and trillions of rows. The **breaking number** is not the node count itself, it is what showed up in the tail latencies: **p99 read latency ranged 40 to 125ms, and p99 write latency ranged 5 to 70ms**, a swing of up to 14x between the best case and the worst case on the exact same cluster, for the exact same kind of query. That spread is not random jitter. It is the visible fingerprint of two background costs colliding on the same machines as live traffic: JVM garbage-collection pauses (Cassandra runs on the JVM) and compaction I/O (the background process that keeps a write-optimized store fast to read from). Once both of those started competing with user-facing reads and writes for the same CPU and disk, adding more of the same kind of node stopped fixing the tail. Discord ultimately rewrote the storage layer on ScyllaDB (a C++, Cassandra-API-compatible engine with no JVM) and cut the same trillions-of-rows workload down to **72 nodes**, with p99 read latency down to **15ms** and p99 write latency down to **5ms**.

Hold onto that number: **a 14x p99 latency swing (5 to 70ms) on a single storage engine at trillions-of-rows scale**, caused not by more data than the disks could hold, but by background housekeeping (compaction, GC) stealing the same CPU and disk the live writes needed.

---

## 2. Why the naive (demo) design dies

The demo version of "store every message" looks exactly like what you'd build in an afternoon:

```sql
CREATE TABLE messages (
  id BIGINT PRIMARY KEY,
  channel_id BIGINT,
  author_id BIGINT,
  content TEXT,
  created_at TIMESTAMP
);
CREATE INDEX idx_channel ON messages(channel_id, created_at);
```

One Postgres (or MySQL) table, a B-tree index on `channel_id`. This works perfectly for a demo server with a few thousand messages. It falls over for three concrete reasons once the write rate climbs into the hundreds of thousands of messages per second Discord actually sees:

**a. B-trees are read-optimized, in-place structures, and every insert pays a random-access cost.** A B-tree index keeps data sorted so a lookup is fast (`O(log n)` page reads), but that means every insert has to find the *correct* leaf page on disk and rewrite it in place, wherever it happens to live. At low volume this is invisible. At Discord's volume, every single message insert becomes a random disk seek somewhere in a multi-terabyte index, and random-write I/O saturates long before storage capacity does. This is the storage-engine version of the hot-row problem: not one row contended, but every write forced through a slow, scattered access pattern.

**b. A single writable primary is a hard ceiling, not a soft one.** You can bolt on read replicas, but every message, from every channel, on every server, worldwide, still funnels through one primary's write-ahead log and one primary's disk. There is no "shard by channel" story on a stock Postgres box — you'd have to build sharding, routing, and rebalancing yourself, which is precisely what a distributed LSM store like Cassandra already does out of the box.

**c. Hot channels behave exactly like Day 16's hot key, one layer down.** A wildly popular server generates far more reads and writes for its channels than an average server does. If every row for a channel sits in the same index range on the same primary, one celebrity AMA can pin the CPU while every quiet server's queries starve behind it, because they are all competing for the same single writer's attention.

The naive design's real mistake is applying a *read-optimized* data structure to a workload that is overwhelmingly *write-then-scan-later*: people write one message and then, much later, scroll through thousands of old ones. A B-tree pays its cost on every write to make every read cheap. Discord's workload wants the opposite trade: pay a small, deferred, background cost so writes are always fast.

---

## 3. The architecture

Top to bottom, the shape Discord converged on (Cassandra originally, then ScyllaDB, same LSM shape, different engine underneath):

```
[Client: Discord app]
   sends a message
   |
   v
[Gateway / API tier — stateless]
   accepts the write, hands it downward, no state kept here
   analogy: a receptionist who takes your form and passes it on, keeps nothing
   |
   v
[Data services layer — Rust, gRPC, added specifically to fix the hot-partition problem]
   two jobs bolted onto the front of the database:
   1. Request coalescing: if 500 users in a busy channel all ask for the same
      recent messages within the same instant, run ONE database query and hand
      all 500 the same answer, instead of 500 identical queries.
   2. Consistent-hash routing: always send a given channel's traffic to the
      SAME data-service instance, so coalescing above actually has repeat
      customers to merge, instead of 500 requests landing on 500 different
      stateless boxes with nothing to coalesce.
   analogy: one teller window that notices ten people in line all asking the
   same balance, and photocopies one receipt for all ten — but only works if
   all ten are actually standing in THIS line, not scattered across ten windows
   |
   v
[Storage engine node: Cassandra / ScyllaDB]
   |
   +--> [Commitlog / WAL] — sequential append-only log, survives a crash
   |     analogy: the insurance policy — same job as Day 17's WAL
   |
   +--> [Memtable] — in-memory, sorted structure; the write lands HERE first
   |     analogy: a scratchpad; writing here is just an append, no disk seek
   |
   +--> [SSTable] — once the memtable fills, flush it to disk as one
   |     immutable, sorted, indexed file
   |     analogy: a frozen photograph — once printed it is never edited again,
   |     only replaced by a newer photograph
   |
   +--> [Compaction] — a background process that merges several SSTables into
         fewer, larger, still-sorted SSTables, dropping overwritten and
         tombstoned (deleted) rows along the way
         analogy: a librarian who re-shelves overnight, merging today's returns
         into the main stacks so tomorrow's reader doesn't have to check 50
         separate return-carts to find one book
   |
   v
[Shard-per-core layout — the ScyllaDB-specific upgrade]
   one OS thread pinned to one CPU core, its own memory, its own SSTables, its
   own I/O queue; requests are routed to the correct shard instead of every
   core contending a shared structure
   analogy: instead of one shared kitchen where every chef queues for the same
   walk-in fridge (shared lock, or a JVM garbage collector pausing everyone),
   each chef gets a personal mini-fridge stocked in advance — no queueing,
   no stop-the-world pause
```

Partitioning detail that matters: the primary key is **(channel_id + time bucket, message_id)**. `channel_id` (plus a bucket, so one ancient, enormous channel doesn't become one unbounded partition) is the partition key that decides *which node* owns the data; `message_id` is a Snowflake ID — a 64-bit ID with a timestamp baked into its high bits (the exact same trick Day 27's Instagram lesson uses for story expiry) — so rows arrive at the node already sorted by time, and "give me the last 50 messages" is a sequential scan of already-sorted data, never a sort at read time.

---

## 4. The transferable mechanisms

**a. LSM-tree write path: turn random writes into sequential writes by deferring the sort.** The core trick is simple to state and easy to underrate: buffer new writes in a sorted in-memory structure (the memtable), and when it fills, flush it as one immutable, already-sorted file (the SSTable) with one sequential disk write. No insert ever seeks to "the right place" on disk; the right place is wherever the next SSTable happens to land, because sorting and merging is a background job, not a per-write cost. This is why LSM-based engines (Cassandra, ScyllaDB, RocksDB, LevelDB, HBase) sustain dramatically higher write throughput than B-tree engines (Postgres/MySQL's default index) under heavy insert load: the cost of "keep everything sorted" is paid later, in bulk, in a job you can throttle, instead of synchronously on every single write's critical path.

**b. Compaction strategy is a dial you choose, not a database default you accept.** Size-tiered compaction (STCS) merges similarly-sized SSTables together, which keeps write amplification low (data gets rewritten relatively few times) but lets read amplification and space amplification grow, because many overlapping SSTables can accumulate and a single read may need to check several of them, and space is not reclaimed until a merge happens to touch it. Leveled compaction (LCS) organizes SSTables into levels with non-overlapping key ranges, so a read touches far fewer files — at the cost of much higher write amplification, because data gets rewritten across levels many more times over its life. Discord's workload (write once per message, read the same history over and over as people scroll) leans toward the STCS-flavored trade: keep writes cheap, accept some read/space overhead, and let hardware and a smarter access layer absorb the difference.

**c. Tombstones are a liability with a shelf life, not a deletion.** In an LSM store, deleting a row does not erase it, it writes a *new* row: a tombstone marker that says "ignore anything older with this key." That tombstone has to survive until compaction physically removes both the tombstone and the data it shadows, and until then, reads pay the cost of skipping over it. A heavily-deleted partition (a channel where people constantly delete messages) can accumulate so many tombstones that a single read has to skip past *millions* of dead rows before finding a live one. This is exactly what stalled Discord's Cassandra-to-ScyllaDB migration at **99.9999% complete**: the very last token ranges scanned were the most tombstone-heavy, and the fix was to force-compact those ranges before the scan could finish.

**d. Request coalescing plus consistent-hash routing is cache-adjacent deduplication without a cache.** Before traffic even reaches the storage engine, collapse N simultaneous identical reads into 1 real query, and route by key so those N simultaneous identical reads actually land on the same process to be collapsed in the first place. Consistent hashing (Day 10) is doing double duty here: not just "which shard owns this data" but "which stateless service instance should see all of this channel's traffic so coalescing has something to coalesce."

**e. Shard-per-core removes a whole category of coordination cost, not just "more threads."** Pin one thread to one core, give it its own memory and its own set of SSTables, and route each request to the shard that owns the relevant key range instead of letting every core contend a shared, lockable structure. Dropping the JVM (and its stop-the-world garbage collector) alongside this is what let ScyllaDB cut Discord's p99s by roughly 3 to 8x on the same shape of workload: compaction and GC pauses stopped competing with live traffic for the same core.

**f. Bound partitions by adding time into the key, before they grow unbounded.** `channel_id` alone as a partition key means one old, extremely active channel eventually becomes one enormous, hot partition living on the same few replicas forever. Adding a time bucket into the partition key (`channel_id + week-number`, for instance) caps how large any single partition can get, at the cost of a slightly more complex read path (query N buckets instead of 1 for a long history scan).

---

## 5. The trade-offs

**Consistency vs availability, per data type.** Message history itself tolerates eventual consistency comfortably — Cassandra and ScyllaDB both use tunable consistency (for example, quorum reads and writes), so a message being visible on one replica a few hundred milliseconds before another replica is invisible to a user scrolling chat. What *cannot* be eventually consistent is the ordering within a channel, and that's solved without any read-time coordination at all: the Snowflake ID baked into the clustering key is already time-sortable, so "what happened after what" is answered by comparing integers, not by asking every replica what time it thinks it is.

**Cost vs latency, paid once to save forever.** Going from 177 JVM-based Cassandra nodes to 72 C++ ScyllaDB nodes for the identical dataset is both a latency win (15ms vs 40-125ms p99 read) and an infrastructure cost win (fewer, denser machines) — but that trade was bought with a real, one-time, expensive migration: Discord's first migration-tool estimate (Spark-based) was **three months**; they rewrote the migrator in Rust, checkpointing progress locally via SQLite and streaming ("firehosing") token ranges into ScyllaDB at **3.2 million messages per second**, cutting the estimate to **nine days**. The lesson: a storage-engine swap at trillions-of-rows scale is not a config change, it is a project, and the payoff is a permanently cheaper, faster steady state purchased with a large, bounded, one-time cost.

**Write amplification vs read amplification is CAP's "it depends on the data type" idea, recursed one layer down into the storage engine itself.** STCS vs LCS is not a universally correct choice, it is a bet about your access pattern: write-heavy-and-read-later workloads (chat messages, event logs) favor STCS's cheap writes; read-heavy-with-occasional-writes workloads favor LCS's cheap, narrow reads. Choosing wrong doesn't crash the system immediately, it shows up months later as a slow bleed in p99 latency that looks like "we need more nodes" when the actual fix is "we chose the wrong compaction strategy for this table."

---

## 6. The systems-thinking lens

**The feedback loop: compaction falling behind is a metastable failure, not a single spike.** Under sustained load, compaction can't keep pace with incoming SSTables → more unmerged SSTables pile up → each read now has to check more files to find the current value → reads get slower → the app tier retries slow reads → retry traffic adds more read load → compaction gets even less CPU because reads (and, on Cassandra, GC pauses) are consuming it → compaction falls further behind. Nothing about this loop requires a traffic spike to trigger it; once compaction is behind, the system can stay in the bad state indefinitely, even after the original spike has passed, because the loop is self-reinforcing rather than self-correcting. That is the textbook shape of a metastable failure: a system that has two stable operating points (compaction keeping up, compaction perpetually behind) and, once nudged into the second one, does not return to the first on its own.

**The senior fix breaks the loop, it does not add nodes into it.** Adding more identically-configured Cassandra/JVM nodes just spreads the same GC-vs-compaction contention across more machines running the same loop; it buys time, not a fix. Discord's actual fix operated on the loop's structure in three places at once: (1) dampen read pressure *before* it reaches compaction-starved nodes, via request coalescing and consistent-hash routing, so N identical reads become 1; (2) remove the JVM's stop-the-world GC pause entirely by moving to a shard-per-core C++ engine, so compaction stops fighting a pause for the same core; (3) bound tombstone accumulation (proactive pruning, and forcing compaction on token ranges that had drifted into tombstone-heavy territory) so no single partition can silently become the one that stalls a full-cluster scan at 99.9999% complete. None of those three is "buy more hardware" — each one removes a source of contention from the loop itself.

---

## Map to Rare.lab's stack

Rare.lab is on the naive side of section 2 for anything that becomes append-heavy, and closer than it looks to the LSM side for anything content-addressed.

**Where Rare.lab already has the LSM idea, without naming it that:** Cloudflare R2 storing immutable, content-addressed scene JSON plus a manifest *is* the SSTable pattern. An SSTable is never edited in place, only replaced by a newer, merged file, and a small index says which files are current — that's precisely "immutable object in R2, manifest points at the current merged view." The manifest is doing the job of Cassandra's SSTable index: "here is what's current," instead of "go rewrite the old object." That is a genuinely good foundation, not a coincidence to fix.

**Where the naive B-tree ceiling is waiting:** Supabase Postgres today is a single-writer, B-tree engine, exactly section 2's demo design. That's fine for scene metadata, user accounts, and billing, all low-write-rate, relational-by-nature data. It stops being fine the moment Rare.lab's multiplayer editor logs individual edit events at full granularity — one row per node-graph edit, per collaborator, per frame of drag — rather than periodic whole-scene snapshots. **The specific next ceiling:** a naive "insert one row per edit event" table on Supabase Postgres starts contending on random-page writes competing with every other write on the same instance once event volume crosses roughly the order of magnitude Discord hit with just 12 Cassandra nodes (~120 million events/day) — a number that arrives far sooner than Rare.lab's *user* count would suggest, because full-granularity edit logging multiplies event count per user by orders of magnitude versus one message per human action.

**The concrete, actionable move when that day comes:** don't put the raw edit-event stream in Postgres rows at all. Buffer edits client-side or in a short-lived queue (the memtable equivalent), flush them periodically as immutable batched objects into R2 (the SSTable equivalent, which the manifest pattern already supports), and keep only small, queryable "current state" pointers in Postgres — the compacted, current view — rather than every individual edit as a live row. That is the same shape as Discord's fix: keep the write-hot path off the B-tree entirely, and let a log-structured, immutable-file layer absorb the volume.

---

## References and summaries

**Discord Engineering Blog: "How Discord Stores Trillions of Messages"** (Bo Ingram, 2023)
https://discord.com/blog/how-discord-stores-trillions-of-messages
The primary source for this lesson's core numbers: the 177-node Cassandra cluster, the migration to 72 ScyllaDB nodes, the p99 latency comparison (40-125ms → 15ms read, 5-70ms → 5ms write), the Rust-rewritten migrator hitting 3.2M messages/sec and cutting the estimate from three months to nine days, and the data-services layer built for request coalescing and consistent-hash routing.

**Discord Engineering Blog: "How Discord Stores Billions of Messages"** (Stanislav Vishnevskiy, 2017)
https://discord.com/blog/how-discord-stores-billions-of-messages
The original 2017 post: the move from a single MongoDB replica set to a 12-node Cassandra cluster at 120M+ messages/day, and the (channel_id, message_id)-with-Snowflake-ID data model this lesson's architecture section is built on.

**Engineering at Scale (Substack): "How Discord solved the Hot partition problem"**
https://engineeringatscale.substack.com/p/how-discord-solved-hot-partition-problem
A clear, diagram-driven walkthrough of why popular channels hotspot a single Cassandra partition, and how the request-coalescing + consistent-hash-routing data-services layer fixes it — the source for section 4d's mechanism.

**ScyllaDB Tech Talk: "How Discord Migrated Trillions of Messages from Cassandra to ScyllaDB"**
https://www.scylladb.com/tech-talk/how-discord-migrated-trillions-of-messages-from-cassandra-to-scylladb/
ScyllaDB's own recorded talk covering the same migration from the vendor side, including the shard-per-core architecture detail (one thread per core, no JVM) that underlies section 3's last diagram layer and section 4e.

**ScyllaDB Blog: "Compaction Series: Space Amplification"** and **"Compaction Series: Write Amplification in Leveled Compaction"** (2018)
https://www.scylladb.com/2018/01/17/compaction-series-space-amplification/
https://www.scylladb.com/2018/01/31/compaction-series-leveled-compaction/
The two-part technical breakdown behind section 4b's STCS-vs-LCS trade-off: size-tiered compaction's low write amplification but growing space/read amplification, versus leveled compaction's non-overlapping levels that cut read amplification at the cost of far higher write amplification.

**Apache Cassandra Documentation: Compaction**
https://cassandra.apache.org/doc/latest/cassandra/managing/operating/compaction/ucs.html
The vendor-neutral, current reference for how compaction strategies are configured and how tombstones are physically removed during a compaction pass, underlying section 4c.

**O'Neil, Cheng, Gawlick, O'Neil: "The Log-Structured Merge-Tree (LSM-Tree)"** (Acta Informatica, 1996)
https://www.cs.umb.edu/~poneil/lsmtree.pdf
The original academic paper that named and formalized the data structure this entire lesson is about: a disk-based index designed specifically for a file experiencing a high rate of inserts and deletes over an extended period, by deferring the cost of sorting to a background merge process. This is the primary source for section 4a.

**Chang et al.: "Bigtable: A Distributed Storage System for Structured Data"** (Google, OSDI 2006)
https://www.usenix.org/legacy/event/osdi06/tech/chang/chang.pdf
The paper that popularized the SSTable (Sorted String Table) as an immutable, ordered, indexed on-disk file format — the exact building block Cassandra and ScyllaDB both inherited, and the direct ancestor of section 3's "SSTable = frozen photograph" layer.

**Hacker News discussion: "How Discord Stores Billions of Messages"** (2017)
https://news.ycombinator.com/item?id=13442457
Independent, adversarial community discussion of the original 2017 post, including practitioner pushback on the tombstone problem being partly a misuse issue rather than a pure Cassandra limitation — useful as a reminder, in the same spirit as Day 20's Jepsen references, that a vendor's own success story is worth reading alongside independent scrutiny.
