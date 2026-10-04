# Day 80: How does Discord store trillions of chat messages and still load a channel in milliseconds? Time-bucketed partitions, hot partitions, and request coalescing

**Date:** 2026-10-04
**Difficulty:** Advanced (partition design, hot-key defense, GC-free storage engines, live migration)
**Topic:** Append-heavy, read-recent data at planetary scale. Pick the partition key so one question touches one partition, then stop a thundering crowd from hammering that one partition.
**Stack relevance:** Rare.lab will have comments, activity feeds, version history and render logs per scene. Same shape as chat. Section 7 maps it.

---

## 0. The framework (same six steps as Day 74)

1. Functional: send a message to a channel, load the latest messages of a channel, scroll back to older ones, edit and delete, jump to a message by id.
2. Non-functional: huge write volume, reads dominated by "most recent N", p99 latency stable (no random 100 ms spikes), no single point of failure, cheap enough to keep every message forever.
3. Entities: Guild (server), Channel, Message, User.
4. API: `POST /channels/:id/messages`, `GET /channels/:id/messages?before=<id>&limit=50`, `GET /channels/:id/messages/:id`.
5. Naive design: one relational table `messages(id, channel_id, author_id, body, created_at)` on one Postgres, with an index on `(channel_id, id)`.
6. Deep dives: partition key design, time buckets, hot partitions, request coalescing and consistent-hash routing, why the JVM garbage collector hurt, live migration of trillions of rows.

---

## 1. The company and the breaking number

**Discord, and one number: a single hot channel that 100,000 people open in the same minute, on top of a store that has to absorb hundreds of thousands of writes per second.**

- Secondary sources quote Discord at about **40 billion messages per day** (see Section 9, not verified against a primary source). Plain arithmetic: 40,000,000,000 / 86,400 seconds is about **463,000 writes per second on average**. Peak is a multiple of that. No single Postgres primary takes that (roughly 10k to 50k simple writes/s is a realistic ceiling for one box).
- Stored volume: Discord's own engineering posts moved from "billions" (2017) to "trillions" (2022) of messages. Cassandra cluster grew to **177 nodes**, then shrank to **72 ScyllaDB nodes with 9 TB each** after the move (per Discord's 2022 post as summarized by several sources).
- The hot read: a big server posts "event starts now" in a channel with 1,000,000 members. Say 100,000 of them open that channel within 60 seconds. That is about **1,700 reads per second asking for exactly the same 50 rows**. Same partition, same data, same answer.

Analogy: a library with every book ever written (storage problem) where, one morning, 100,000 people queue for the same newspaper page (hot key problem). Two different problems. Discord had to solve both.

---

## 2. Why the naive design dies

**Naive: one Postgres table, index on `(channel_id, id)`.**
- **Write ceiling.** 463k average writes/s needs dozens of primaries. One box stalls on WAL fsync and index maintenance long before that. You are forced to shard, and the shard key is the whole game.
- **Storage ceiling.** Trillions of rows at about 1 KB each is petabytes. A B-tree index over that does not fit in RAM, so "latest 50 of a channel" becomes random disk I/O.
- **Bad shard keys.** Shard by `message_id` (hash): the latest 50 messages of one channel scatter across every shard, so one page load becomes dozens of network calls. Shard by `channel_id` alone: one channel grows without bound into one giant partition, and one busy channel melts one node.
- **Hot key.** Even with a perfect key, 1,700 identical reads/s land on one node's replicas. That node's latency rises, the app retries, load rises again. That is a retry death spiral (Section 6).

---

## 3. The architecture, top to bottom

```
Clients (phones, desktop, browser)
   |   WebSocket gateway pushes new messages live; HTTP for history
   v
Edge / load balancer
   |
   v
API tier (stateless: auth, permissions, rate limits)
   |   "give me channel 123, before message X, limit 50"
   v
Data services tier  (Rust, one per data type)       <- 2022 addition
   |   * consistent-hash route by channel_id: same channel always lands on same instance
   |   * request coalescing: 1,700 identical reads become 1 DB query
   v
Message store: ScyllaDB (was Cassandra), 72 nodes, shard-per-core
   |   partition key = (channel_id, time bucket), clustering key = message_id (descending)
   v
Replicas (3x) across racks / zones
```

- **Gateway (WebSocket):** pushes new messages to people already in the channel, so most users never poll. Analogy: a doorbell instead of walking to the door every 5 seconds.
- **API tier:** stateless, so you add servers freely. Job: who is allowed, how fast, then forward.
- **Data services (Rust):** a thin gatekeeper in front of the database. Analogy: a librarian at one desk who hears 100,000 people ask for the same page and photocopies it once.
- **Message store:** the partitioned, replicated, append-friendly database. Analogy: a wall of numbered lockers, where the locker number is (channel, time window).
- **Replication (3 copies):** if one machine dies the data is still on two others. Analogy: three photocopies in three buildings.

---

## 4. The transferable mechanisms

### 4.1 Partition key that matches the read ("one question, one partition")
- Discord's schema (from the 2017 post, from memory): primary key `((channel_id, bucket), message_id)`. `channel_id` and `bucket` together pick the partition. `message_id` orders rows inside it.
- The most common query is "last 50 messages in this channel". With this key, that is **one partition, one sequential read**, newest first.
- Real query walked end to end: user opens `#announcements` (channel 8841). Message ids are Snowflake-style (time-sortable 64 bit ids, see Day 26), so the current bucket is computed from the id's timestamp. Fetch 50 rows from bucket `(8841, current)`. If fewer than 50 come back, step to the previous bucket and fetch the rest.

### 4.2 Time buckets to cap partition size
- A partition that grows forever is a bug. Discord adds a **bucket**, a static time window of the channel's history (the 2017 post used about 10 days, from memory).
- Why: Cassandra-style stores are happiest with partitions in the tens to low hundreds of MB. Back-of-envelope: a hot channel at 100 messages/s over 10 days is 86.4M messages. At about 1 KB each that is **86 GB in one partition**. Far too big. A quiet channel at 5 messages/day produces 50 rows in 10 days, which is tiny. So bucket width is a trade: too wide and hot channels bloat, too narrow and quiet channels need many bucket hops to fill a screen.
- Lesson: the bucket is a safety rail against the growth of the worst key, not a performance trick for the average key.

### 4.3 Append-only storage engine (LSM tree)
- Writes go to a commit log plus an in-memory table, then flush to immutable sorted files. No in-place updates, so writes are sequential and fast. Deep dive in Day 21.
- Cost: reads may check several files, and deletes become **tombstones** (markers) that live until compaction. Mass deletes plus reads through tombstones was a known pain point for Discord (they describe it in their posts; from memory).

### 4.4 Request coalescing plus consistent-hash routing (the hot-partition fix)
- Problem: 1,700 identical reads/s on one partition.
- Fix: route every request for channel X to the **same data-service instance** (consistent hashing on `channel_id`, Day 10). That instance keeps a map `in_flight[query] -> future`. The first request starts the DB query and registers a future. Requests 2 through 1,700 find the future and just wait on it. When the DB answers, all 1,700 get the same bytes.
- Result: **1,700 DB reads become 1**. This is the "singleflight" pattern. It only works if identical requests meet in the same process, which is why consistent-hash routing comes first.
- Analogy: a coffee shop where 30 people order "the daily special" and the barista makes one big pot instead of 30 cups.

### 4.5 Remove the garbage collector from the hot path
- Cassandra runs on the JVM. A stop-the-world GC pause on one node makes it look dead or slow to peers, requests time out, retries pile onto the remaining replicas. Discord reports unpredictable p99 read latency of roughly **40 to 125 ms** on Cassandra.
- ScyllaDB is C++ with a **shard-per-core** design (each CPU core owns its slice of data, no shared locks). Reported result: p99 reads about **15 ms**, p99 writes about **5 ms**, and a smaller cluster (177 to 72 nodes). Numbers per secondary summaries of Discord's post (Section 9).
- Lesson: at scale the tail (p99) is set by pauses, not by averages. Day 76 covers why.

### 4.6 Live migration without downtime
- Discord reports a custom **Rust migrator** reading token ranges, checkpointing progress to a local SQLite file for crash recovery, and writing at about **3.2 million records per second**. The migration finished in about **9 days** versus an early estimate of about 3 months using a Spark-based tool.
- Pattern: dual write new traffic to both stores, backfill history in the background, verify, then flip reads. Same idea as Day 77 (live resharding).

---

## 5. The trade-offs accepted

| Data type | Choice | Why |
|---|---|---|
| Message history | **Availability over consistency (AP)**, tunable quorum per query | Seeing a message 100 ms late is fine. A chat that refuses to load is not. Replicated 3x, reads and writes at quorum where it matters. |
| Message ordering | Eventual, ordered by Snowflake id inside a partition | Ids carry time, so sort order is stable without a global lock (Day 26). |
| Edits and deletes | Eventually consistent | Brief flicker is acceptable. |
| Permissions | Stronger, checked in the API tier before the data call | Showing the wrong person private messages is the one error that is not tolerable. |

- **Cost vs latency:** fewer, larger ScyllaDB nodes (72 x 9 TB) cost less than 177 smaller Cassandra nodes and give a flatter p99. The data-services tier costs engineering time and one extra network hop, bought back by shielding the database from crowds.
- **No joins, no ad hoc queries.** The partition key fixes which questions are cheap. "Show all messages by user U across channels" needs a separate index or a search system. You choose the query up front and pay forever for choices you did not make.

---

## 6. The systems-thinking lens

**The loop: hot partition, slow node, retries, hotter partition (a retry death spiral, a flavour of metastable failure).**

```
hot channel -> one replica set slows -> client timeouts -> retries x3
     ^                                                        |
     +---------- more load on the same slow replicas <--------+
```

- Adding nodes does not help, because the key lives on three specific replicas. This is why "just add capacity" fails for hot keys (Day 16).
- The senior fix **breaks the loop at its source**:
  1. **Coalesce:** identical concurrent reads collapse to one, so the crowd size no longer multiplies DB load.
  2. **Route consistently:** so coalescing actually catches the duplicates.
  3. **Shard-per-core and no GC:** one slow request cannot stall a whole node.
  4. **Backoff and backpressure** between tiers (Day 13): retries must not be free.
- Metastable failure test: if load drops back to normal and the system stays broken, a loop is feeding itself. Coalescing and bounded retries make the system return to health on its own.

---

## 7. Map to Rare.lab's stack (Supabase Postgres with RLS, R2 immutable scene JSON plus manifest, one shared WebGL context)

| Discord pattern | Rare.lab touchpoint | Status | Action |
|---|---|---|---|
| Immutable data is cheap to serve | Content-addressed scene JSON on R2 | **Already doing this, and it is the strongest hot-key defense there is** | Hash-keyed objects never change, so a viral shared scene is served by the CDN edge, not your origin. Keep `Cache-Control: immutable`. |
| Hot key on the mutable pointer | The manifest (name -> current hash) | Real risk | A manifest is the one mutable, hot object. Cache it at the edge with a short TTL (1 to 5 s) and use stale-while-revalidate. A thousand embeds loading the same scene should cost one origin fetch, not a thousand. |
| Request coalescing | Edge Worker fetching the manifest | Not yet | In a Cloudflare Worker, keep an in-isolate `Map<url, Promise>` so concurrent requests share one fetch. A Durable Object per scene id gives you the consistent-hash routing for free. |
| Partition key + time bucket | Future comments, activity, render logs per scene | Design now | Use a Postgres partitioned table by time range (monthly) and index `(scene_id, created_at desc)`. "Latest 50 for scene X" then hits one small partition. Do not use a single ever-growing table. |
| Append-only | Version history of a graph | Natural fit | Store versions as immutable rows or R2 objects. Never update in place. |
| Pooling and shed load | Supabase connection limits | See Day 78 | Hot reads must not each hold a DB connection. Coalesce first, pool second. |

**Where the next ceiling is (inference, not measured):** the Postgres primary. Event-style tables (activity, logs, comments) grow fastest and are write-heavy. First fix: partition by time and move cold partitions to R2 as Parquet. Second: move the append-only firehose off Postgres only when write rate nears roughly 5k to 10k rows/s sustained. Until then Postgres is the right, boring choice. Discord ran on MongoDB, then Cassandra, then ScyllaDB as each ceiling arrived. They did not start with the fanciest one.

**One-line lesson for Rare.lab:** choose the partition key from your most common read ("latest N for this scene"), cap partition size with a time bucket, and put a request-coalescing layer in front of anything a crowd can hit at once, because the first thing that breaks at scale is not storage, it is one popular key.

---

## 8. What is inside the video you shared (recap, already covered)

The "Design LeetCode" mock interview (Vanika Agarwal, Google) is fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md), so I did not repeat it. One-minute spine for your notes:
1. Requirements first (functional, out of scope, non-functional), then entities, then REST API, then a simple high level design, then fix it layer by layer.
2. Her fixes: never run user code on the API server, containers per language, queue plus retries between API and runners, cache the leaderboard, DB replicas with failover.
3. **Link to today:** her `GET /leaderboard/:competitionId` and the submissions table partitioned by competition id is the same idea as `(channel_id, bucket)`. Her cached leaderboard is the same idea as coalescing: do the work once, serve it to 100,000.
4. Her recommended resources: Hello Interview (blog and videos) and Shreyansh Jain's YouTube channel.

---

## 9. References and what is actually in them

**Honest note on access:** discord.com, hellointerview.com and thenewstack.io were blocked by the network proxy in this session. I could not read the primary Discord posts. Facts below came through web search excerpts and are marked. Verify numbers before quoting them.

- **Discord Engineering, "How Discord Stores Trillions of Messages" (2023)** (discord.com/blog/how-discord-stores-trillions-of-messages). The primary source. Summary (via excerpts): 177 Cassandra nodes, hot partitions causing cascading latency, Rust data services with request coalescing, ScyllaDB with 72 nodes at 9 TB, a Rust migrator.
- **Discord Engineering, "How Discord Stores Billions of Messages" (2017)** (discord.com/blog/how-discord-stores-billions-of-messages). Primary source for the MongoDB to Cassandra move and the `(channel_id, bucket)` key. Bucket width and schema here are from memory, not re-read today.
- [How Discord Migrated Trillions of Messages to ScyllaDB (The New Stack)](https://thenewstack.io/how-discord-migrated-trillions-of-messages-to-scylladb/). Journalistic walk-through of the same migration.
- [How Discord Moved Trillions of Messages to ScyllaDB (Hello Interview)](https://www.hellointerview.com/learn/system-design/in-the-wild/discord-messages-scylladb). Interview-prep style breakdown, good for the problem, fix, result structure.
- [How Discord Migrated Trillions of Messages and Fired Their Garbage Collector (DEV Community)](https://dev.to/techlogstack/how-discord-migrated-trillions-of-messages-and-fired-their-garbage-collector-4818). Source of the 40 to 125 ms to 15 ms p99 read, 3.2M records/s, 9 day figures (via search excerpt).
- [Discord: From Billions to Trillions of Messages, A Three-Database Journey (Sujeet Jaiswal)](https://sujeet.pro/articles/discord-message-storage) and [Why Discord Moved From Cassandra to ScyllaDB (Medium)](https://medium.com/@amit.agarwal0422/how-discord-stores-trillions-of-messages-524865d20ffc). Medium-style secondary write-ups, lower authority, useful for the MongoDB, Cassandra, ScyllaDB timeline.
- [How Discord Handles 40 Billion Messages Per Day (Substack)](https://grindengineer.substack.com/p/how-discord-handles-40-billion-messages-per-day). Source of the 40B/day figure. Treat as unverified.
- **Talk to search for on YouTube:** Discord engineers at ScyllaDB Summit 2023 on storing trillions of messages (not opened today, so no link given).

**Inference, labeled:** the 463k writes/s arithmetic, the 1,700 reads/s crowd example, the 86 GB hot-channel partition example, the one-Postgres write ceiling, and all of Section 7.

**Related ledger lessons:** Day 10 (consistent hashing), Day 13 (backpressure), Day 16 (hot key), Day 19 (caching), Day 21 (LSM trees), Day 26 (Snowflake ids), Day 74 (LeetCode), Day 76 (tail latency), Day 77 (live resharding), Day 78 (connection pooling).
