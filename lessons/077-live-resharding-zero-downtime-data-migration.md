# Day 77: How do you move a live database to new servers while users keep writing to it, and never lose or double a single row? Live resharding with Notion, Stripe and Vitess

**Date:** 2026-10-01
**Difficulty:** Advanced (a copy, a change stream, a verifier and an atomic switch, all while traffic runs)
**Topic:** Online resharding and zero-downtime data movement. The five-phase recipe (snapshot, change capture, verify, cut over, keep a way back), why logical shards beat physical ones, and the feedback loop that makes migrations fail.
**Stack relevance:** Rare.lab will outgrow one Supabase Postgres. Section 7 maps when, and how to make that day boring.

---

## 0. The framework (same six steps as Day 74)

1. Functional requirements: move a slice of live data (a shard, a tenant, a table) from database A to database B. Reads and writes keep working the whole time.
2. Non-functional: zero or near-zero downtime, no lost writes, no duplicate or stale rows, and a way to undo.
3. Entities: Source shard, Target shard, Routing map (key to shard), Change stream, Verifier.
4. API: `move(range, source, target)` and a router that answers `where_is(key)`.
5. Naive design: stop the app, copy the data, point the app at the new database, start the app.
6. Deep dives: the five phases, the cutover race, logical versus physical shards, rollback, the migration feedback loop.

---

## 1. The company and the breaking number

**Notion, 2021. One number: Postgres `VACUUM` could not keep up on a monolith holding every block of every workspace.**

From Notion's engineering post "Herding elephants" (summarized from search excerpts, see the sourcing note): the single Postgres was hitting the point where `VACUUM` (the housekeeping that reclaims dead rows) was stalling, and transaction ID wraparound (a counter Postgres can run out of) became a real risk. Notion split the data into **480 logical shards on 32 physical databases**, 15 per machine, keyed by workspace ID. The migration used a **three day backfill** and ended with **about five minutes of scheduled downtime**. By 2023 the same 480 logical shards were spread over **96 physical instances** with no change to routing logic (per the same coverage).

**Stripe, the sequel.** Stripe's DocDB (a database service built on MongoDB community edition) runs thousands of shards. Its Data Movement Platform moves chunks between them with **traffic cutovers that typically finish in milliseconds**. In 2023 Stripe says it bin-packed underused databases by moving **1.5 petabytes** and cut its shard count by about **three quarters**, with product apps unaware.

The breaking number is not load. It is **the clock**: a copy of a big database takes days (Notion: 3 days), and the data keeps changing for all of those days. Analogy: moving house while the family keeps buying groceries. By the time the truck is loaded, the fridge has new food in it.

---

## 2. Why the naive design dies

**Naive version:** announce maintenance, stop writes, copy everything, flip the config, resume.

- **Downtime scales with data size.** Copy speed is fixed, data keeps growing. A 20 TB copy at a very generous 1 GB/s is about 5.5 hours of a dead product. Stripe cannot be down for payments, and Notion's users live in their workspace all day.
- **Copy-then-flip loses writes.** Start copying at 10:00. A user edits a page at 10:05, after that row was already copied. The new database never hears about it. Analogy: photocopying a ledger while the clerk is still writing in it.
- **The flip is not atomic across machines.** If app server 1 sees the new map and server 2 still has the old one, for a moment two machines both accept writes for the same key on different databases. That is a split brain (two writers, two truths).
- **No way back.** After the flip, new writes land only on the new database. If it is wrong or slow, going back means losing everything written since.
- **The load of the copy itself.** A full-speed backfill competes with live traffic for IO and CPU on the source, and can slow production. This is how a "safe" migration causes an outage.

---

## 3. The architecture

```
Clients
   |
   v
App tier (stateless)
   |   every query asks the router: "which shard owns key K?"
   v
Router / proxy  <------ Routing map (versioned): key -> logical shard -> physical DB
   |                    (a small, strongly consistent config store)
   |-------------------------------|
   v                               v
SOURCE shard (live, takes writes)  TARGET shard (being filled)
   |                               ^
   | 1. snapshot / bulk copy ------|
   | 2. change stream (log tail) --|   <- keeps TARGET in sync
   |
   v
Verifier (diffs source vs target)    <- "dark reads" or row/checksum compare
   |
   v
Coordinator (state machine)          <- runs the phases, flips the map, keeps rollback armed
```

Box by box, with analogies:
- **Router and routing map.** The single place that knows where each key lives. Analogy: a post office sorting table with a lookup sheet. To move a family, you change one line on the sheet, you do not re-teach every postman.
- **Source shard.** Keeps serving until the last second.
- **Bulk copy.** A consistent snapshot read, the moving truck.
- **Change stream.** A tail of the source's changes (a write-ahead log, a binlog, or an audit table) replayed on the target. See Day 17 (WAL and CDC). Analogy: a courier carrying every new grocery receipt to the new house.
- **Verifier.** Proves the two copies match before anyone relies on the new one.
- **Coordinator.** A state machine (a program that is always in exactly one named phase). It makes the move resumable and auditable.

---

## 4. The transferable mechanisms (the five-phase recipe)

**1. Snapshot, then catch up from a marker (the "copy plus tail" trick).**
Record a position in the change log (a log sequence number, binlog position, or timestamp), copy a consistent snapshot as of that position, then replay every change after it. The copy can take days because the tail makes up the difference. Worked example: Notion's backfill took 3 days. During those 3 days the source took millions of edits. They were all queued in the audit log and applied after. Stripe's published detail adds that bulk import was made about 10x faster by sorting rows into the target's index order before inserting, which matches how B-tree storage likes to be written.

**2. Change capture, and the choice of how.**
Notion first tried Postgres logical replication, then chose its own **audit log table** because replication could not keep up with the write volume of the block table during the snapshot step (per the post). Stripe uses change data capture in its Data Movement Platform. Vitess uses **VReplication**, which tails MySQL binlogs and can filter and transform per table. Mechanism: whichever way you tail changes, it must be ordered and resumable from a saved position. This is the same idea as Day 42 (Kafka commit log).

**3. Verify before you trust: dark reads and checksums.**
Notion ran **dark reads**: read from both old and new, return only the old result to the user, log any mismatch. It added API latency, and it bought confidence. Vitess ships **VDiff** for the same job. Stripe has an explicit correctness check phase. Rule: never cut over on faith. Analogy: a bank runs the new ledger in shadow for a month and compares to the old one before closing the old one.

**4. The atomic, versioned cutover.**
The cutover needs a short pause on writes to the moving range: stop accepting writes, let the change stream drain to zero lag, flip the routing map to a new version, resume. Stripe describes **versioned gating** across its proxy, coordinator, routing service and replication service: requests carry the map version they were routed with, and a shard rejects a request whose version is stale. So a slow app server with an old map gets refused and retries with the new map, instead of writing to the wrong place. Vitess does the equivalent: the old primary is marked to reject queries (`query_service_disabled`), and binlog positions are saved to a journal table so downstream streams resume from the right spot. That pause is why Stripe's cutover is milliseconds and Notion's was minutes: Stripe automated the drain and gating, Notion in 2021 used a scheduled window.

**5. Reverse replication: keep the door open.**
After the cutover, Vitess starts a **reverse stream** from target back to source (`_reverse` workflow), so `ReverseTraffic` can roll back with no data loss. Cost: you keep the old shard alive, and pay for it, for a soak period (days). Benefit: the migration stops being a one-way door. Senior rule: any step that cannot be undone needs a bigger review than one that can.

**Bonus mechanism: logical shards first, physical shards second.**
Notion made 480 shards up front and mapped them onto machines. 480 divides evenly by many numbers (1, 2, 3, 4, 5, 6, 8, 10, 12, 15, 16, 20, 24, 30, 32, 40, 48, 60...), so going from 32 to 40 to 48 to 60 to 96 machines only moves whole logical shards. No rehashing of keys, no change to the key to shard function, only the shard to machine table changes. Compare Day 10 (consistent hashing): the same goal, different tool. Analogy: pre-cut a pizza into 480 slices, then decide how many plates to put them on.

---

## 5. The trade-offs

**CAP, made concrete per data type.**
- **Routing map: consistency over availability.** If a router cannot confirm the current map version, refuse the request. A wrong write is worse than a slow one. Keep the map tiny and put it in a strongly consistent store (Day 11, Raft).
- **User data during the move: availability wins, except for one short pause.** Reads and writes continue for the whole copy. Only the cutover window blocks writes to the moving range, for milliseconds (Stripe) to minutes (Notion 2021).
- **Verification data: eventual consistency is fine.** The target may lag the source by seconds while catching up. It only has to be at zero lag at the instant of the flip.

**Cost versus latency.**
- Dual writes (writing to both systems) cost latency and double the write load. Change-log tailing costs less on the request path but adds replication lag.
- Dark reads add read latency (Notion saw API latency go up) and double read load. Run them on a sample or in a short window, not forever.
- Keeping the old shard for rollback costs real money for days. That is the price of undo.
- Over-provisioning logical shards (480 up front) costs a bit of overhead per shard (connections, vacuum workers) and buys cheap future moves.

**Choice made by these systems:** copy plus tail, verify, millisecond-to-minutes write pause, keep the way back open.

---

## 6. The systems-thinking lens

**The loop that actually causes migration failures: the backfill that slows production, which slows the change stream, which grows the lag, which lengthens the migration.**

```
Backfill reads hammer the source
        -> source gets slower, app latency rises
        -> change-stream replay falls behind (replication lag grows)
        -> cutover window needs longer to drain
        -> pressure to "just cut over now"
        -> inconsistent data or an outage
```

Add the second loop: **dual-write retries**. If a write to the new system fails and the app retries, and the old system already accepted it, you now have duplicates unless the write is idempotent (Day 12).

The senior fix breaks the loop, it does not add hardware:
- **Throttle the backfill on a lag signal.** Speed up when replication lag is near zero, slow down when it rises. This is backpressure (Day 13) applied to your own migration.
- **Make every replayed change idempotent**, keyed by row ID plus version, so a retry or replay never double applies.
- **Cut over only at zero lag**, enforced by the coordinator, not by a human who is tired.
- **Gate by map version** so stale routers get refused, not obeyed.
- **Keep reverse replication** so a bad cutover costs a rollback, not an incident.
- **Move small units.** Stripe moves chunks, Notion moves logical shards. A small unit has a small blast radius and a short copy.

---

## 7. Map to Rare.lab's stack

Rare.lab's current shape: Supabase Postgres with RLS, Cloudflare R2 with content-addressed immutable scene JSON plus a manifest, and a runtime with one shared WebGL context.

| Mechanism | Rare.lab touchpoint | Status | Action |
|---|---|---|---|
| Immutable, content-addressed blobs | Scene JSON in R2 | Already the easiest case to move | Nothing to dual-write. Copy bucket to bucket, verify by hash (the key IS the checksum), flip the manifest pointer. A manifest flip is a one-row atomic cutover. See Day 23. |
| Routing map | Manifest in R2, plus any future "which DB has this tenant" lookup | Partly | If you ever shard Postgres, store `workspace_id -> logical shard -> database` as one small versioned table. Never hash directly to a machine. |
| Logical shards first | Postgres tables keyed by workspace or project | Not yet needed | Put `workspace_id` on every table now (RLS policies already imply it). Retrofitting a tenant key later is the hardest part of Notion's story. |
| Change capture | Supabase Realtime and logical replication slots | Available | Supabase can stream changes from the WAL. Use it as your future tail, but watch slot lag: an unconsumed replication slot makes Postgres keep WAL on disk and can fill it. |
| Verify before cutover | Scene metadata rows | Not yet | When you do move, run a checksum compare per project (row count plus a hash of ordered rows) before the flip. |
| Reverse path | Rollback | Not yet | Keep the old database read-only for a week after any migration. |

**Where the next ceiling is (inference, not measured):** the first wall is unlikely to be raw data size. It is more likely connection count or a hot project (Day 16) on one Postgres, solved first by the Supabase pooler, then read replicas, then a bigger instance. Sharding is the last step, and Notion reached it at a size far above an early product. Do the cheap preparation (a tenant key on every table, no cross-tenant joins in the hot path) and skip the migration until the numbers say so.

**One-line lesson for Rare.lab:** put a tenant key on every table and a versioned routing indirection in front of anything that could one day move, so the future migration is a boring five-phase script and not a rewrite.

---

## 8. What is inside the video you shared in this task (recap, already covered)

The "Design LeetCode" mock interview is fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md). It follows the same spine: requirements, out of scope, non-functional, entities, API, then a basic design improved in layers (containers for isolation, a queue between API and runners, a cache for the leaderboard, replicas for the submissions database, partition key by competition ID). I did not repeat that topic. A link to today: the interviewer's last point there, "promote a replica when the primary dies", is failover. Today is the planned sibling: moving data on purpose while it is live, which is harder because both sides are healthy and both are being written to. The interview's choice of competition ID as a partition key is exactly the "pick the key that most queries stay inside" decision Notion made with workspace ID.

---

## 9. References and what is actually in them

**Honest note on access:** the network proxy in this session blocked notion.com, stripe.com and vitess.io. Everything below is summarized from web search excerpts and secondary write-ups, not a direct read of the primary pages. Numbers are quoted as reported there. Check them against the originals before citing.

- [Herding elephants: lessons learned from sharding Postgres at Notion](https://www.notion.com/blog/sharding-postgres-at-notion) (primary). Content: why they sharded (VACUUM stalls, wraparound risk), why workspace ID is the key, 480 logical shards on 32 databases, audit log plus catch-up over logical replication, dark reads, three day backfill, short scheduled downtime. Retellings: [Quastor](https://blog.quastor.org/p/notion-sharded-postgres-database-8af4), [Changelog](https://changelog.com/news/herding-elephants-lessons-learned-from-sharding-postgres-at-notion-039Z), [Hacker News thread](https://news.ycombinator.com/item?id=28776786).
- [How Notion Scaled to Handle 200 Billion Blocks](https://www.lorenzopalaia.com/blog/how-notion-scaled-to-handle-billions-of-blocks) and [DEV Community: Lessons from Notion's 480 shards](https://dev.to/souvick_20/scaling-postgresql-without-microservices-lessons-from-notions-480-shards-32id). Secondary sources for the 2023 move from 32 to 96 instances and the "480 has many factors" reasoning.
- [Stripe: How Stripe's document databases supported 99.999% uptime with zero-downtime data migrations](https://stripe.com/blog/how-stripes-document-databases-supported-99.999-uptime-with-zero-downtime-data-migrations) (primary, Jimmy Morzaria and Suraj Narkhede, June 2024). Content: the Coordinator and its phases (register, bulk import, async replication, correctness check, traffic switch, deregister), 1.5 PB moved in 2023, shard count cut by about three quarters. Retellings: [ByteByteGo](https://blog.bytebytego.com/p/how-stripe-scaled-to-5-million-database), [DEV Community](https://dev.to/techlogstack/how-stripe-moves-petabytes-between-database-shards-without-stopping-the-money), [InfoQ on the QCon SF 2025 talk](https://infoq.com/news/2025/11/stripe-zero-downtime-date-move/). The talk page: [QCon SF 2025](https://qconsf.com/presentation/nov2025/stripes-docdb-how-zero-downtime-data-movement-powers-trillion-dollar-payment). Source of the versioned gating and millisecond cutover claims.
- [Vitess docs: Reshard](https://vitess.io/docs/22.0/reference/vreplication/reshard/), [VReplication cutover internals](https://vitess.io/docs/archive/22.0/reference/vreplication/internal/cutover/), [SwitchTraffic](https://vitess.io/docs/25.0/reference/programs/vtctldclient/vtctldclient_reshard/vtctldclient_reshard_switchtraffic/) and [ReverseTraffic](https://vitess.io/docs/25.0/reference/programs/vtctldclient/vtctldclient_reshard/vtctldclient_reshard_reversetraffic/). The open-source reference implementation of the whole recipe: split or merge shards on a live cluster, VDiff to verify, journal table for binlog positions, `query_service_disabled` on the old primary at cutover, reverse replication for rollback. Read these to see the steps as commands.
- [7 Real-Time Migrations With Vitess That Don't Wake SREs (Medium)](https://medium.com/@2nick2patel2/7-real-time-migrations-with-vitess-that-dont-wake-sres-dd5d100b7d2c). A practitioner list. Lower authority than the docs.

**Inference, labeled:** the 20 TB at 1 GB/s arithmetic in Section 2, the backfill-lag feedback loop diagram in Section 6 (the components are real, the loop framing is mine), the claim that Stripe's millisecond cutover is due to automated drain and gating while Notion's minutes were due to a manual window, and all Rare.lab mappings and ceiling estimates in Section 7.

**Related ledger lessons:** Day 10 (consistent hashing and sharding), Day 11 (Raft), Day 12 (idempotency), Day 13 (backpressure), Day 16 (hot keys), Day 17 (WAL and CDC), Day 23 (content-addressed storage), Day 45 (secondary indexes in sharded databases), Day 58 (online schema migrations), Day 62 (global uniqueness in sharded databases).
