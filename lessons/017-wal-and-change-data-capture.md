# Day 17 — How does a database broadcast every write to 50 downstream systems in real time without destroying itself?

**Date:** 2026-06-28
**Difficulty:** Advanced
**Topic:** Write-Ahead Logs (WAL) and Change Data Capture (CDC)
**Stack relevance:** Supabase Realtime (already uses WAL), Cloudflare Queues as CDC sink, search index sync, audit logs for Rare.lab

---

## 1. The company and the breaking number

**LinkedIn, 2012. 50+ downstream systems. 500,000 events per second. One MySQL primary.**

LinkedIn's social graph (connections, job views, message reads) lived in Oracle and MySQL. Every "profile viewed," every "connection accepted," every "message read" had to reach:
- The search index (for typeahead)
- The activity feed ("Pooja viewed your profile")
- The recommendations engine
- Analytics dashboards
- 47 other consumers

The naive approach each of those 50 systems ran: `SELECT * FROM events WHERE updated_at > last_poll`. At 1-second polling intervals, that is 50 full-table scans per second on one MySQL primary touching 100 million+ rows.

**The breaking number: 50 downstream pollers x 100M-row table x 1-second interval = primary CPU at 100% within 3 minutes.**

And the worst part: deleted rows simply disappear. The polling query finds nothing. The search index keeps serving a user who no longer exists.

This is the problem that Write-Ahead Logs and Change Data Capture solve.

---

## 2. Why the naive (demo) design dies

The one-server demo looks like this:

```
App writes row -> MySQL primary
Each downstream system: SELECT * FROM table WHERE updated_at > last_poll
```

Three concrete ways it collapses at scale:

**a. Poll storm.**
50 systems, each opening a connection and running a scan every second, equal 50 separate read transactions per second competing with live writes. Even with an index on `updated_at`, at 50 systems this is a DDoS you launch against yourself.

**b. Deletes are invisible.**
`WHERE updated_at > last_poll` only finds rows that still exist. A `DELETE` removes the row. There is nothing left to find. Your downstream search index keeps serving data that no longer exists. The only workaround is a soft-delete column (`deleted_at`), but now you must filter it everywhere and the table grows forever, never shrinking.

**c. Missed intermediate states.**
Between poll T1 and poll T2, a row can be updated twice. Your poll sees only the second state. You missed the intermediate. For most analytics pipelines this is acceptable. For an audit log it is a regulatory violation.

**The demo fails the same way every time: it treats the database as a query engine when it should be treated as a log.**

---

## 3. The architecture

Top to bottom, annotated with each layer's single job:

```
[Application Code]
     |  INSERT / UPDATE / DELETE
     v
[Database Primary]  <---- writes data pages to disk
     |
     +----> [WAL / Binlog]
            sequential append-only file on disk
            written BEFORE data pages change
            analogy: the black box of the plane
               |
               v
     [Replication Slot (Postgres) / Binlog Position (MySQL)]
            a bookmark: "consumer X has read up to byte N"
            DB holds WAL segments until all slots have consumed them
            analogy: a library bookmark; the book cannot be reshelfed until returned
               |
               v
     [CDC Connector]
            Debezium / LinkedIn Databus / Netflix DBLog
            pretends to be a read replica; reads WAL via replication protocol
            decodes raw bytes into structured events: {op: INSERT, table: orders,
            before: null, after: {id:42, total:99.00}, lsn: 0/16B3748}
            zero extra queries on the primary
            analogy: a court reporter who transcribes every word without asking anyone to repeat
               |
               v
     [Message Broker]
            Kafka / Kinesis / Redpanda
            one topic per table: orders, users, connections
            partitioned by primary key: all events for row ID 42 go to the same partition,
            so per-row order is preserved with parallelism across rows
            durable, replayable: a new consumer can replay from offset 0
            analogy: a post office with unlimited retry and a permanent archive of every letter
               |
        +------+------+------+------+------+------+
        v      v      v      v      v      v      v
  [Search  [Feed  [Analytics [Cache  [Audit  [Data  [New
  Index]   Svc]   Pipeline]  Warmer] Log]   WH]    Consumer]
            each reads at its own pace; none affects the others
```

**The key insight:** the database primary sees exactly ONE consumer (the CDC connector). That connector reads via the native replication protocol, which is how read replicas work anyway. The primary does not know or care that 50 real consumers exist downstream. You have replaced 50 database connections with 1.

---

## 4. The transferable mechanisms

### a. Log as the source of truth

The WAL is not a debugging feature. It IS the database. The data pages on disk are a materialized view of the log. You can rebuild any table from scratch by replaying WAL from the beginning. This is the same idea as event sourcing in application code: the log is ground truth, and state is derived from it.

**PostgreSQL WAL in practice:**
- Default segment size: 16 MB per WAL file.
- WAL is written sequentially; sequential disk writes are 10 to 100 times faster than random writes, which is why WAL is a performance win, not just a safety net.
- `wal_level = logical` is required for CDC (the default `replica` level does not include column-level before/after values).
- `REPLICA IDENTITY FULL` on a table is required to get before-images on UPDATE (by default Postgres only logs the primary key).

### b. The Outbox Pattern (app-level CDC without WAL access)

Sometimes you cannot tap the WAL: your DB is managed, the replication protocol is blocked, or you want business-level events rather than raw row diffs. The fix:

```sql
BEGIN;
  INSERT INTO orders (id, total) VALUES (42, 99.00);
  INSERT INTO outbox (event_type, payload, sent)
    VALUES ('order_created', '{"id": 42, "total": 99.00}', false);
COMMIT;
```

Both writes are in the same transaction. Atomicity is free. A separate publisher reads unsent outbox rows, publishes to Kafka, marks them sent. It does not matter if the app crashes after the commit: the outbox row survives and the publisher picks it up on restart. This beats application-level pub/sub because it survives crashes with zero message loss.

### c. Replication slots as natural backpressure

A slot tells the DB: "do not delete this WAL until I confirm I have consumed it." If your CDC consumer slows down, WAL accumulates on disk. This is backpressure: the DB applies natural resistance rather than dropping events. The risk is the inverse: a stalled slot fills the disk and crashes the primary.

Production rule: always set `max_slot_wal_keep_size` (introduced in Postgres 13) to cap WAL retention per slot. If a slot exceeds the limit, Postgres invalidates it. The consumer misses events and must resync, but the primary stays alive. This is the right trade: consumer correctness is recoverable; a crashed primary is not.

### d. Watermark-based full dumps without locks (Netflix DBLog)

When a new consumer starts, it needs the current state of every existing row, not just future changes. The naive approach: pause the CDC stream, dump the table, resume. Problem: the dump of a 500 GB table takes 45 minutes. You cannot pause the stream for 45 minutes.

Netflix's solution in DBLog: write a sentinel "watermark" row to the table (which appears in the WAL as a regular write), take a chunk of rows between two watermarks, and interleave those chunk rows into the live event stream at the correct WAL position. No lock. No gap. No paused consumers. The new consumer bootstraps by consuming interleaved dump chunks alongside live events, and converges to a consistent state without ever pausing anyone.

### e. Idempotent downstream consumers

CDC delivers at-least-once. If Debezium crashes and restarts, it re-delivers events from the last confirmed offset. Every consumer must be idempotent.

- **Search index:** "index document id=42 with content X" is naturally idempotent; indexing it twice has no effect.
- **Counter update:** use the WAL sequence number (LSN in Postgres) as an optimistic lock; reject writes with an LSN older than the last processed.
- **Audit log:** deduplicate by `(table, pk, lsn)` before inserting.

---

## 5. The trade-offs

### Consistency vs. availability, per data type

| Data type | CDC choice | Why |
|-----------|-----------|-----|
| Search index | Eventual (seconds behind) | Stale search results are tolerable; freshness is not worth locking |
| Analytics | Eventual (minutes behind) | Batch windows dwarf CDC lag |
| Feed notifications | Near-real-time (<500ms lag) | The user expects to see the notification promptly |
| Audit log | At-least-once, deduped | Must not miss events; duplicates are filtered by LSN |
| Cross-table consistency | Saga or Kafka transactions | CDC gives per-table order, not cross-table atomicity |

**Replication lag** means all consumers see a slightly stale view. For "immediately reflect the user's own write," bridge the lag with a write-through cache: the app updates the cache directly after writing to the DB; CDC eventually overwrites it with the canonical DB value. The window is typically 50-200 ms.

### Cost vs. latency

- Polling is cheap to implement. At 5+ downstream consumers or 10k+ events/min it becomes the most expensive thing you operate.
- WAL-based CDC is expensive to set up (Debezium cluster, Kafka cluster, schema registry, slot monitoring) but nearly free to operate per consumer added: adding consumer 51 costs zero additional load on the primary.
- The crossover point in practice is roughly 3 to 5 downstream consumers or any consumer that needs deletes.

### Schema evolution risk

Every CDC consumer decodes WAL events using a schema. Adding a column is backward-compatible (new field in the event, old consumers ignore it). Renaming or removing a column breaks all consumers simultaneously unless you use a schema registry. Use Confluent Schema Registry or AWS Glue Schema Registry to version event schemas and enforce compatibility rules before deployment.

---

## 6. The systems-thinking lens

**The feedback loop that causes failure: the replication slot death spiral.**

1. CDC consumer (Debezium) slows down because Kafka is under load.
2. Replication slot falls behind. Postgres accumulates WAL on disk.
3. At 90% disk, Postgres starts rejecting writes. App errors spike.
4. Ops team kills the stalled consumer to free disk space.
5. Consumer restarts from scratch. It must re-read gigabytes of WAL. Falls even further behind.
6. The loop tightens. Primary disk fills again faster than before.

**The named pattern: WAL pressure -> disk fill -> primary crash -> cascade.**

This is a metastable failure: the system cannot recover by itself once the loop starts. Adding more consumer capacity helps but does not break the loop because the slot keeps accumulating during the capacity ramp.

**The senior fix breaks the loop at three places:**

1. **Alert early.** Monitor slot lag in bytes (`pg_replication_slots.wal_status`, `pg_replication_slots.safe_wal_size`), not just minutes behind. Alert at 20% disk used by WAL, not 90%.
2. **Cap WAL retention.** Set `max_slot_wal_keep_size = 10GB`. If a slot exceeds this, Postgres invalidates it (the consumer must resync), but the primary survives. Survivable correctness failure beats total outage.
3. **Decouple with Kafka.** Put Kafka between Debezium and every slow consumer. Debezium writes to Kafka quickly. Kafka's retention is cheap disk on separate nodes. Slow analytics pipelines read from Kafka, never from the DB. The DB slot stays current. The slow work drains independently. This is the same "shock absorber" pattern as the queue-as-buffer lesson (Day 9), applied one layer deeper.

---

## Map to Rare.lab's stack

**What you already have:**
Supabase Realtime IS WAL-based CDC. When you enable Realtime on a table, Supabase provisions a Postgres logical replication slot, reads WAL, and pushes structured events to connected WebSocket clients. You are already on this pattern.

**What needs attention right now:**
- Set `max_slot_wal_keep_size` in your Supabase Postgres config before your first production spike. The default is unlimited retention, which means a stalled WebSocket consumer can fill the disk.
- Add slot lag monitoring. The `pg_replication_slots` view gives you `wal_status` and `confirmed_flush_lsn`. Alert if a slot falls more than 500 MB behind.

**The next ceiling:**
At 10k scene writes per minute, your Supabase Realtime slot will accumulate lag if your WebSocket handler is slow (thumbnail generation, search index rebuild, email notifications triggered by a write). The fix: decouple. Write events from Supabase Realtime into Cloudflare Queues. Let the slow handlers drain from the queue. The Supabase slot stays current at all times. The slow work drains at its own pace. You pay zero extra load on the Postgres primary for each new consumer you add.

**R2 immutable scene JSON + manifest:**
Every new scene write could fire a Realtime CDC event that triggers a search index update. Instead of the manifest service polling Supabase directly, subscribe to Realtime on the manifest table and fan-out to your search tier in under 200 ms. No poll interval, no extra DB connections.

---

## References and summaries

### Core documentation

**PostgreSQL: Write-Ahead Logging (WAL) Introduction**
https://www.postgresql.org/docs/current/wal-intro.html
The official source. Explains the core guarantee: WAL records are flushed to disk before any data page changes. Sequential writes to the WAL are faster than random writes to data pages, which is why WAL is a performance win and not just a safety net. Covers checkpoints, recovery, and why `wal_level = logical` is needed for CDC. Start here to understand what a WAL record actually contains.

**Debezium Documentation: PostgreSQL Connector**
https://debezium.io/documentation/reference/stable/connectors/postgresql.html
The practical reference for setting up WAL-based CDC on Postgres. Covers required config (`wal_level = logical`), replication slot creation, the difference between `wal2json` and `pgoutput` plugins (pgoutput ships with Postgres 10+, no extension needed), and `REPLICA IDENTITY FULL` for getting before-values on UPDATE. The production checklist is here. Read before writing any CDC code.

### Engineering blogs from real companies

**LinkedIn Engineering: Open-sourcing Databus**
https://engineering.linkedin.com/data-replication/open-sourcing-databus-linkedins-low-latency-change-data-capture-system
The 2013 post that introduced Databus to the world. LinkedIn built it in 2005 because 50 downstream systems polling one Oracle primary was destroying it. The relay buffer (an in-memory ring buffer per source) lets consumers catch up without hammering the primary. Delivery latency under 1 second. This is the original motivation for log-based CDC in a production system.

**Netflix TechBlog: DBLog - A Generic Change-Data-Capture Framework**
https://netflixtechblog.com/dblog-a-generic-change-data-capture-framework-69351fb9099b
Netflix needed to bootstrap new CDC consumers on live tables without locking them. The watermark trick: write sentinel rows to the table, take snapshot chunks between watermarks, and interleave snapshot rows into the live event stream at the correct WAL position. No lock, no gap, no paused consumers. Used in production by tens of microservices at Netflix. The academic paper (arxiv.org/abs/2010.12597) formalizes the algorithm.

**GitHub: LinkedIn Databus source code and wiki**
https://github.com/linkedin/databus
The open-source implementation. The MySQL adapter wiki at `https://github.com/linkedin/databus/wiki/Databus-for-MySQL` explains how the MySQL fetcher connects as a replication slave and uses OpenReplicator to parse binlog events. Good for understanding the low-level replication protocol before using a higher-level tool like Debezium.

**Notion's Data Lake Architecture (Quastor deep-dive)**
https://blog.quastor.org/p/architecture-notions-data-lake
How Notion migrated from batch Postgres exports to a real-time CDC pipeline in 2022: Debezium reads WAL, publishes to Confluent Kafka, Hudi Deltastreamer writes to S3 in lakehouse format. End-to-end ingestion time dropped from hours (batch) to minutes (CDC). The lesson relevant to Rare.lab: CDC makes audit logs accurate because it captures the exact before/after of every write including intermediate states that polling misses. Notion's sharding story (480 shards on 96 nodes) is also worth reading because CDC was what made that migration verifiable.

### Articles and explainers

**Algomaster.io: What is Change Data Capture (CDC)?**
https://blog.algomaster.io/p/change-data-capture-cdc
A clean written explainer covering all three CDC patterns side by side: timestamp-based polling (cheap, misses deletes, races on intermediate states), trigger-based (correct but doubles write load because triggers fire synchronously on every write), and log-based (best latency and correctness, most operational complexity). Good for interview prep because it frames each pattern in terms of trade-offs: exactly the conversation you want to have in a system design interview when someone asks "how would you fan out writes to 20 downstream systems?"

**Medium: Understanding CDC in MySQL and PostgreSQL: BinLog vs WAL + Logical Decoding**
https://medium.com/data-science/understanding-change-data-capture-cdc-in-mysql-and-postgresql-binlog-vs-wal-logical-decoding-ac76adb0861f
A side-by-side comparison of how MySQL and Postgres implement the same concept differently. MySQL uses a binary log (binlog) that logs logical changes (row format shows before/after column values). Postgres uses a WAL with a logical decoding layer that translates physical WAL records into logical events. The difference matters: MySQL's binlog is always available; Postgres requires `wal_level = logical` and a decoder plugin. Practical for teams running mixed DB fleets.

**Confluent: How Change Data Capture Works**
https://www.confluent.io/blog/how-change-data-capture-works-patterns-solutions-implementation/
Confluent's deep-dive covering the Kafka side: how Kafka Connect hosts Debezium as a source connector, how topic partitioning preserves per-row order (all events for the same primary key go to the same partition), and how schema evolution is handled via Schema Registry. Covers the `exactly-once` semantics setting and the trade-off (it adds 2x write amplification). Read this after understanding basic CDC to see how all the pieces fit at production scale.

**Redpanda: CDC Approaches, Architectures, and Best Practices**
https://www.redpanda.com/guides/fundamentals-of-data-engineering-cdc-change-data-capture
A comprehensive guide comparing the four CDC patterns with real latency numbers. Key stat: log-based CDC via Debezium delivers events in under 100 ms end-to-end; polling-based delivers in poll-interval seconds to minutes. Covers the outbox pattern in depth and when NOT to use full CDC (small tables, low write volume, under 3 downstream consumers, no need for delete capture).

**DEV.to: Summary of Chapter 9 (WAL) from "The Internals of PostgreSQL"**
https://dev.to/vkt1271/summary-of-chapter-9-write-ahead-logging-wal-from-the-book-the-internals-of-postgresql-56kn
A readable summary of Hironobu Suzuki's free online book "The Internals of PostgreSQL," specifically the WAL chapter. Covers LSN (Log Sequence Number: a 64-bit byte offset into the WAL that uniquely identifies every record), REDO points, checkpointing mechanics, and WAL recycling. If you want to understand what `pg_current_wal_lsn()` returns and why it matters for monitoring slot lag, this is the right 15-minute read.

### YouTube videos

**Change Data Capture (CDC) | Why & How | Use Case | System Design**
https://www.youtube.com/watch?v=dN_11nBcv_A
A visual architecture walkthrough showing Debezium capturing MySQL changes and streaming to Kafka, with a live demo. Covers the architecture diagram (MySQL -> Debezium -> Kafka -> consumers) and explains the "connector offset" concept: how Debezium stores its last consumed binlog position so it picks up exactly where it left off after a restart. Good for visual learners; watch at 1.5x speed. Pay attention to the connector offset segment as it is the key to at-least-once delivery.

**CDC Explained | Change Data Capture**
https://www.youtube.com/watch?v=jlh8G8Txly4
A shorter (~15 min) explanation of all three CDC patterns with clean animations comparing their latency, correctness, and operational cost. Framed explicitly for system design interviews: the video asks "what happens on delete?" and "what happens if the consumer crashes?" for each pattern, which are exactly the follow-up questions interviewers ask. Watch this one first if you are preparing for an interview.

**Change Data Capture (CDC) Explained (with examples)**
https://m.youtube.com/watch?v=5KN_feUhtTM
A longer deep-dive with concrete code examples of the outbox pattern and log-based CDC. Shows the before/after JSON structure of a real Debezium event, which helps you understand what your downstream consumer actually receives. Covers idempotency: why you must handle the same event arriving twice without corrupting your data.

### Research paper

**DBLog: A Watermark Based Change-Data-Capture Framework (Netflix, 2020)**
https://arxiv.org/abs/2010.12597
The academic paper behind Netflix's DBLog. Formalizes the watermark algorithm for lock-free snapshot bootstrapping, proves correctness (no event is missed, no event is duplicated, snapshot and stream are interleaved in commit order), and benchmarks against naive approaches. A 20-page paper worth reading once you understand the operational problem it solves. The key theorem: a watermark pair (lo, hi) in the WAL bounds the snapshot chunk such that any concurrent write to a snapshotted row will appear in the stream after the chunk completes, never before, and never be missing.
