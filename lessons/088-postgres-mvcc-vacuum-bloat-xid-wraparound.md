# Day 88: How can Postgres let thousands of readers and writers touch the same rows at once, and why does one forgotten open transaction slowly kill it? MVCC, vacuum, bloat and XID wraparound

**Date:** 2026-10-12
**Difficulty:** Advanced (storage-engine internals under your own database: row versions, visibility rules, garbage collection as a hidden feedback loop)
**Topic:** Earlier lessons used Postgres as a black box (Day 78 connection pooling, Day 58 online migrations, Day 29 write skew). This lesson opens the box: what an UPDATE physically does, why old rows pile up, and how that becomes an outage.
**Stack relevance:** Rare.lab runs on Supabase Postgres. Everything below is the engine under your tables, including the autosave and "last opened" columns that quietly generate the most dead rows.

---

## 0. The framework (same recipe as Days 74, 86, 87)

The LeetCode mock interview in your prompt follows one recipe. Reuse it on any product:

1. Functional requirements, plus an explicit out-of-scope list.
2. Non-functional requirements (availability vs consistency, latency, scale, isolation).
3. Core entities, then API routes.
4. The simplest high-level design, one requirement at a time.
5. Break it, then optimize (API server runs code = bad, VMs = heavy, containers = light, then queue, cache, replicas).
6. Name the trade-off out loud every time.

The full walk-through of that video is in [Day 74](074-leetcode-online-judge-end-to-end.md). One line in it matters today: the interviewee picks DynamoDB because "we don't need joins" and says a primary database with replicas removes the single point of failure. Both choices have a hidden cost at the storage-engine level. Whatever you pick (Postgres, MySQL, Dynamo), someone must answer: **when a row is updated while another query is reading it, what happens to the old row, and who cleans it up?** That is today's lesson.

Today's recipe:
- Functional: many sessions read and write the same rows concurrently, each seeing a consistent snapshot, with no reader blocking a writer.
- Non-functional: reads never wait on writes, space is reclaimed without downtime, the database never refuses writes because of housekeeping.
- Naive design: lock the row while anyone touches it.

---

## 1. The company and the breaking number

**Two real stories.**

- **Uber (2016).** Evan Klitzke's engineering post "Why Uber Engineering Switched from Postgres to MySQL" lists Postgres limits for their workload: inefficient write path, inefficient replication, table corruption after a 9.2 upgrade bug, and poor replica MVCC support. The core write complaint: every row version has a new physical location, so indexes pointing at the old location must also be updated (write amplification), and Postgres's physical replication ships all of that to replicas.
- **Sentry (a postmortem on a Monday, July 20).** Sentry's blog "Transaction ID Wraparound in Postgres" describes being down for most of a US working day because Postgres hit its transaction ID (XID) safety limit. (I saw only the opening of the post via search; I did not read the full root-cause section, and the year is unverified.) A secondhand Bytebase summary also cites a Mailchimp Mandrill outage of roughly 40 hours with the same shape: autovacuum fell behind on a busy shard, wraparound protection halted writes. I could not verify Mailchimp's own text.

**The breaking number (my arithmetic, labeled as a worked example, not a company figure):**

- A table of 1,000,000 rows, each about 1 KB, takes **10,000 UPDATEs per second**.
- Every UPDATE writes a NEW row version. That is 10,000 x 1 KB = **10 MB/s of new heap data**, even though the live table stays 1 GB.
- One analyst opens a transaction and forgets to close it. Vacuum may not delete any version that transaction could still see. After one hour: 10,000 x 3,600 = **36,000,000 dead versions, about 36 GB** that cannot be reclaimed, in a 1 GB table.
- Same load, second clock: 10,000 write transactions per second burn **864 million transaction IDs per day**. The default freeze trigger (`autovacuum_freeze_max_age`) is 200 million, reached in 200,000,000 / 10,000 = **20,000 seconds, about 5.6 hours**. The hard wall is about 2.1 billion (2^31): roughly **2.5 days** if vacuum never freezes anything.

Analogy: a library that never throws away old editions. Every correction prints a new copy of the page. A janitor (vacuum) removes old copies only when no reader still holds that edition. One reader who falls asleep with an old edition in hand stops the janitor in the whole building.

---

## 2. Why the naive design dies

Naive design: a row lock. Readers take a shared lock, writers an exclusive lock.

- **Readers block writers and writers block readers.** A 10-second report scanning the orders table stalls every checkout UPDATE for 10 seconds. At 10,000 writes/sec that is 100,000 blocked writes queued behind one SELECT.
- **Deadlock risk grows with concurrency.** Two sessions each hold one row and want the other's.
- **No consistent snapshot without holding locks for the whole query.** A report that reads accounts, then reads transfers, can see money leave one table before it arrives in the other.

Real example: LeetCode contest leaderboard (Day 74). The leaderboard query reads 100,000 submissions rows while submissions keep arriving. With locking, the leaderboard stalls submission inserts, or inserts stall the leaderboard.

Postgres's answer (and Oracle's, MySQL InnoDB's, and others') is **MVCC, multi-version concurrency control: never overwrite, keep versions, let each reader see the version that was current when its snapshot began.** The price is the rest of this lesson.

---

## 3. The architecture, drawn top to bottom (inside one Postgres)

```
Client sessions  (Supavisor / PgBouncer pool, Day 78)
        |  BEGIN; UPDATE accounts SET balance = balance - 50 WHERE id = 7; COMMIT;
        v
Transaction manager
   - hands each writing transaction a 32-bit XID (e.g. 5001)
   - takes a SNAPSHOT for each reader: (xmin_horizon, xmax, in-progress list)
        |
        v
Heap (table file, 8 KB pages)
   Page holds TUPLES (row versions). Each tuple header carries:
     t_xmin = XID that created this version
     t_xmax = XID that deleted/replaced it (0 = still live)
     t_ctid = pointer to the NEWER version, forming a chain
        |
        +--> Indexes: entries point at heap locations (page, slot)
        |
        v
Visibility check (on every tuple read):
   "Was xmin committed before my snapshot, and xmax not yet?"  -> visible, else skip
        |
        v
Cleanup layer
   - Page pruning (opportunistic, on read) and HOT chains
   - Autovacuum workers: remove dead versions, update the free-space map,
     FREEZE old tuples so XIDs can be reused
        |
        v
WAL (write-ahead log, Day 17) --> physical replicas (read replicas)
```

| Layer | Single job | Analogy |
|---|---|---|
| **XID counter** | Stamp every change with a number so versions are ordered | Ticket number at a deli counter |
| **Tuple header (xmin, xmax)** | Say who created and who retired this version | "Printed by clerk 5001, withdrawn by clerk 5120" on a page's footer |
| **Snapshot** | Freeze a reader's view of "what was committed when I started" | A photo of the notice board |
| **Visibility check** | Decide per row version whether your photo includes it | Checking each footer against your photo |
| **Page pruning / HOT** | Tidy a single page cheaply during normal reads | Wiping your own desk |
| **Autovacuum** | Delete versions nobody can still see, and freeze old ones | The janitor |
| **Freeze** | Mark very old rows "visible to everyone forever" so their XID no longer matters | Stamping a document "permanent" so its ticket number can be recycled |

Vocabulary:
- **Tuple**: one physical version of a row.
- **Dead tuple**: a version no current or future snapshot can see.
- **Bloat**: disk and cache space occupied by dead tuples and free-but-unreused space.
- **Snapshot**: the set of transactions a reader treats as committed.
- **Vacuum horizon**: the oldest XID any open transaction might still need; nothing newer than that may be removed.
- **Wraparound**: XIDs are 32-bit and compared in a circle; if a row's xmin gets more than about 2.1 billion XIDs old without being frozen, it would suddenly look like it is in the future.

---

## 4. The transferable mechanisms

### 4.1 Never overwrite, append a version (copy-on-write rows)

An UPDATE in Postgres is "insert a new tuple, set xmax on the old one." Walk the example:

1. Account 7 has balance 100, tuple A with `xmin=4800, xmax=0`.
2. Reader R starts at snapshot "everything up to XID 5000 committed."
3. Writer W (XID 5001) runs `UPDATE accounts SET balance = 50 WHERE id = 7`. Tuple A gets `xmax=5001`. A new tuple B is written: `xmin=5001, balance=50`. A's ctid points to B.
4. R reads account 7: A is visible (xmin 4800 is in its photo, xmax 5001 is not committed in its photo). R sees 100. No lock, no waiting.
5. After W commits, a new reader sees B (50).

Reader and writer never blocked each other. The same idea shows up in Day 23 (content-addressed immutable objects), Day 21 (LSM trees append), and Day 59 (event sourcing): **cheap reads and concurrency by never mutating in place, paid for with a garbage-collection duty.**

### 4.2 Snapshot isolation by comparing numbers, not holding locks

Visibility is arithmetic on (xmin, xmax, snapshot). No lock table lookup per row. This is why a 10-second analytic scan does not delay writers. The catch from Day 29: snapshot isolation allows write skew, and you need SERIALIZABLE (SSI) or explicit `SELECT ... FOR UPDATE` where it matters.

### 4.3 Garbage collection with a horizon: vacuum

Vacuum can delete a dead tuple only if **no open snapshot could still see it.** The oldest open transaction sets the horizon for the whole database. Things that pin the horizon (secondary sources, consistent with Postgres docs):
- A long-running query or an idle-in-transaction session (a connection that ran `BEGIN` and then sat there).
- A forgotten logical replication slot.
- A forgotten prepared transaction (two-phase commit, Day 20).
- A read replica with `hot_standby_feedback` on, reporting its old snapshots upstream.

This is the single most common cause of mysterious bloat. Dead rows are produced at the write rate and removed at the vacuum rate, but **only below the horizon.**

Autovacuum triggers per table when dead tuples exceed `50 + 0.2 x rows` by default (Snowflake engineering post, see sources). On a 100 million row table that is 20 million dead rows before vacuum even starts. Large hot tables need per-table tuning (`autovacuum_vacuum_scale_factor` near 0.01 or a fixed threshold).

### 4.4 HOT updates and fillfactor: avoid touching indexes

The Uber complaint is real: a new tuple at a new location means every index needs a new entry. Postgres's mitigation is the **Heap-Only Tuple (HOT)** update. It applies when (a) the new version fits on the same 8 KB page and (b) no indexed column changed. Then no index entry is added; the old tuple points to the new one inside the page, and page pruning can clean up chain members without a full vacuum. Source: the Postgres source tree's README.HOT and Adyen/Egor Rogov write-ups.

How you help it: `ALTER TABLE ... SET (fillfactor = 80)` leaves 20 percent of each page empty so updated versions can land on the same page. Rule from this: **do not index columns that change constantly** (`last_seen_at`, `updated_at`, counters), or every update becomes a non-HOT update touching every index. With 10 indexes, one update means 11 writes (table plus 10 indexes); HOT cuts it to 1 (Cybrosys/levelup summaries).

### 4.5 Freezing and the 32-bit clock

XIDs are 32 bits and compared modulo 2^32, so any XID sees 2^31 (about 2.1 billion) XIDs "in the past" and 2^31 "in the future." A very old tuple's xmin would flip to "future," making old rows invisible. The fix is **freezing**: vacuum rewrites old tuples as frozen, meaning "older than everything." Once a table's oldest unfrozen XID (`relfrozenxid`) is `autovacuum_freeze_max_age` (200 million default) behind, Postgres launches an aggressive anti-wraparound vacuum even if autovacuum is off. If that still fails (horizon pinned, vacuum starved), Postgres stops accepting commands that need a new XID when about 3 million remain, leaving reads working and writes failing (Fastware, Netdata, malisper.me summaries). That is the Sentry-shaped outage. Recovery means a manual vacuum, often in single-user mode, with downtime proportional to table size.

Cloud providers add guardrails. Google's Cloud SQL docs have a page on "TXID wraparound protection" for this exact state. (Supabase has similar docs; I did not fetch them.)

### 4.6 Replicas inherit the cost (and the conflicts)

Physical replicas replay the primary's WAL, byte for byte. So (a) write amplification crosses the wire (Uber's point), and (b) a long query on a replica can need a row version that the primary's vacuum has already removed. Postgres must either cancel the replica query ("canceling statement due to conflict with recovery") or delay replay, or the replica tells the primary to hold back vacuum (`hot_standby_feedback`), which moves the bloat problem back to the primary. There is no free option, only a choice of which pain.

---

## 5. The trade-offs

**Consistency vs availability, per data type:**

| Data | Choice | Why |
|---|---|---|
| Money / entitlement rows | Snapshot isolation plus `FOR UPDATE` or SERIALIZABLE; accept retries on conflict | Wrong answer costs more than a retry |
| Dashboards, leaderboards | Plain snapshot reads, may be slightly stale; run on a replica | Never block writers; nobody notices 2 seconds |
| Hot counters (views, last_seen) | Batch into a separate table or cache, flush every few seconds | Each update is a full new row version; the cheapest update is the one you do not make |

**Cost vs latency:**
- MVCC with in-table versions (Postgres) makes reads and writes non-blocking but pays disk, cache and vacuum CPU. Undo-log designs (MySQL InnoDB, Oracle) keep old versions in a separate rollback area, so the main table stays compact and secondary indexes are updated differently. This is why Uber's workload looked better on InnoDB. The price there is long-transaction pain in a different place (undo history growing), so it is a trade, not a win. (Comparison is my summary from the Uber post plus general knowledge; I did not benchmark either engine.)
- More aggressive autovacuum = less bloat, more I/O competing with traffic. Less aggressive = cheaper now, a larger bill later.
- Lower `fillfactor` = more HOT updates but a bigger table on disk (80 percent fill means about 25 percent more pages for the same live rows).

**The choice Postgres makes:** never block readers, accept dead-row garbage and a janitor you must watch.

---

## 6. The systems-thinking lens

**The feedback loop: the vacuum falls behind and slows the thing it is supposed to save (bloat spiral).**

1. Update rate rises (a sale, a contest, an autosave storm).
2. Dead tuples accumulate faster than vacuum removes them (or one old transaction pins the horizon).
3. Tables and indexes grow. Queries scan more pages. Cache hit rate falls. Each query is slower.
4. Slower queries hold transactions open longer. The horizon advances more slowly. Vacuum removes even less.
5. Vacuum itself also runs slower on a bigger table. Loop closed. Meanwhile the XID age climbs toward the wall.

Add the application-side loop from Day 76/87: slower queries cause timeouts, timeouts cause retries, retries cause more updates and more open transactions.

How the senior fix **breaks** the loop (it does not just buy a bigger instance):
- **Cap how long anything can hold the horizon.** `idle_in_transaction_session_timeout` (for example 30 s), `statement_timeout`, and `transaction_timeout` (Postgres 17 and later) so no session can pin vacuum for hours. Drop or monitor stale replication slots.
- **Reduce garbage at the source.** Make updates HOT (do not index volatile columns, lower fillfactor), batch counters, and prefer insert-only versioning for data that changes often.
- **Tune per table, not globally.** Lower `autovacuum_vacuum_scale_factor` on hot tables so vacuum starts at 1 percent dead, not 20 percent.
- **Alert on leading indicators**, not on the outage: oldest transaction age, `n_dead_tup / n_live_tup`, `age(datfrozenxid)` against the 200 million and 2.1 billion marks, replication slot lag.
- **Isolate analytics.** Long reports go to a replica or an export, not the primary, with the `hot_standby_feedback` trade-off decided deliberately.

---

## 7. Mapping to Rare.lab

Rare.lab: Supabase Postgres with RLS, Cloudflare R2 (immutable content-addressed scene JSON plus manifest), embeddable runtime with one shared WebGL context.

**Already doing, whether named or not:**
- **Immutable scene JSON in R2.** You are already using the "never overwrite, append a version" idea (4.1) at the object layer. A scene edit writes a NEW hash; nothing is mutated. That is MVCC without a vacuum bill, apart from garbage-collecting unreferenced hashes later (Day 23).
- **Supavisor/pooled connections** (Day 78). In transaction mode a connection only holds a transaction while it is actually in one, which helps; but code that opens `BEGIN`, awaits an HTTP call, then commits will still pin the horizon.

**Where this bites next (my inference from your stack description, not from reading your schema):**

| Risk | Why it matters | Action |
|---|---|---|
| **Autosave updating one manifest or project row in place** | At 1 save per 2 s per active editor, 5,000 editors = 2,500 full row versions per second on one small table. Small table, huge churn, fast bloat | Do not index columns that change on every save. Set `fillfactor` 70 to 80 on the manifest/projects table so HOT works. Better: write the new manifest pointer into R2, keep only a tiny `current_hash` row |
| **`last_opened_at` / view counters in Postgres** | Each increment is a new row version of a wide row | Buffer in memory or Workers KV/Durable Object, flush in batches every 10 to 30 s |
| **RLS policies using subqueries** | Policy checks run per row inside the snapshot; slow scans keep transactions open longer | Index policy columns; keep policy predicates cheap (Day 35 authorization) |
| **Long dashboard queries or admin SQL on the primary** | One forgotten `BEGIN` in the Supabase SQL editor pins the horizon | Set `idle_in_transaction_session_timeout` at the role level; run reports on a read replica or export |
| **Logical replication / CDC slots** (Day 17, Supabase Realtime) | An inactive slot pins WAL and the horizon | Alert on slot lag; drop dead slots |
| **Migrations** (Day 58) | `ALTER TABLE ... ADD COLUMN DEFAULT` rewrites are fine on recent Postgres, but backfills that update every row double the table | Backfill in small batches with vacuum between, or write to a new column and swap |

**Next ceiling:** the first thing to degrade will not be CPU, it will be a small hot table that bloats and a forgotten transaction that stops cleanup. Add three alerts before you need them: oldest open transaction over 5 minutes, dead-tuple ratio over 20 percent on any table over 100 MB, and `age(datfrozenxid)` over 500 million.

**One-line lesson for Rare.lab:** every UPDATE is a new row plus a future cleanup job, so keep frequently changing data out of wide indexed Postgres rows (append immutable versions to R2, flush counters in batches), and cap how long any session can hold a transaction open so the janitor can always work.

---

## 8. Sources and what is actually in them

**Honest note on access:** this session I could use web search summaries only. I did not open the full pages, so claims below are from those summaries plus my own knowledge of Postgres. Numbers in section 1 are my own arithmetic. The comparison of undo-log vs in-table versions is my synthesis. Items flagged "unverified" were not checked.

- [Uber Engineering: Why Uber Engineering Switched from Postgres to MySQL (Evan Klitzke, July 2016)](https://www.uber.com/blog/postgres-to-mysql-migration/). Plain language: Uber lists what hurt at their scale. Updating a row creates a new physical copy, so every index must be rewritten (write amplification). Because replicas copy the physical changes, that extra work is shipped over the network too. Replicas also had trouble serving long reads while applying changes. They moved many workloads to Schemaless on MySQL. Read it as one workload's story, not a verdict on Postgres.
- [InfoQ summary of the Uber post](https://www.infoq.com/news/2016/08/Uber-Engineering-Postgres-MySQL). Plain language: a shorter news write-up; notes HOT updates can relieve some of the write amplification.
- [Sentry: Transaction ID Wraparound in Postgres](https://blog.sentry.io/transaction-id-wraparound-in-postgres). Plain language: a real outage postmortem. Sentry was down most of a US workday when the database ran out of transaction IDs. Good example of why housekeeping is a production risk. (I only saw the opening via search.)
- [Bytebase: Postgres Transaction ID Wraparound](https://www.bytebase.com/blog/postgres-transaction-id-wraparound.md). Plain language: explains how wraparound happens, what blocks vacuum from freezing, and cites a Mailchimp Mandrill outage (about 40 hours). The Mailchimp details are secondhand and unverified.
- [malisper.me: Transaction ID Wraparound in Postgres](https://malisper.me/transaction-id-wraparound-in-postgres/). Plain language: a clear walkthrough of why 32-bit XIDs wrap and how freezing replaces old XIDs with a special "frozen" marker so comparisons stay valid.
- [Fastware: How to fix transaction wraparound in PostgreSQL](https://www.postgresql.fastware.com/blog/how-to-fix-transaction-wraparound-in-postgresql). Plain language: what to do when the database goes read-only because it is close to the limit.
- [Netdata: PostgreSQL transaction ID wraparound, detection and emergency recovery](https://www.netdata.cloud/guides/postgres/postgres-transaction-id-wraparound/). Plain language: which queries show how close you are, and what pins the horizon (old transactions, replication slots, prepared transactions).
- [Google Cloud SQL: Overcome TXID wraparound protection](https://docs.cloud.google.com/sql/docs/postgres/txid-wraparound). Plain language: a managed-database vendor's own page, showing this is a standard, named failure state.
- [Snowflake engineering: Do You Need to Tune Postgres Vacuum?](https://www.snowflake.com/en/blog/engineering/tuning-postgres-vacuum/). Plain language: dead tuples and hidden transaction ID fields; default autovacuum trigger of 50 rows plus 20 percent of the table; per-table `relfrozenxid` check as the wraparound safety valve.
- [Percona: An illustration of PostgreSQL bloat](https://www.percona.com/blog/an-illustration-of-postgresql-bloat/). Plain language: a hands-on picture of old row versions staying on disk while a transaction might need them.
- [Google Cloud: AlloyDB adaptive autovacuum under the hood](https://cloud.google.com/blog/products/databases/alloydb-for-postgresql-under-the-hood-adaptive-autovacuum). Plain language: one vendor's answer is to tune vacuum automatically, evidence that fixed defaults do not fit every table.
- [Adyen: PostgreSQL HOT updates](https://www.adyen.com/knowledge-hub/postgresql-hot-updates), [boringSQL: HOT updates](https://boringsql.com/posts/hot-updates/), [Postgres Pro (Egor Rogov) on HOT](https://postgrespro.com/blog/pgsql/5967910), and the Postgres tree's [README.HOT](https://www.postgresql.org/message-id/attachment/19791/README.hot). Plain language: an update avoids touching indexes when it fits on the same page and changes no indexed column; fillfactor leaves room for that; page pruning cleans chains without a full vacuum.
- Video context: the LeetCode mock interview in your prompt (Day 74 has the full summary). Takeaway for today: it chooses DynamoDB for no-joins and a primary with promotable replicas; today's lesson says whichever store you pick, ask how it handles old versions, and what pins its cleanup.

**Confirmed (from search summaries):** Uber's four listed limits and the write amplification and replication reasoning; the Sentry outage existing and being XID-related; XIDs are 32-bit and wrap; freezing replaces old XIDs; Postgres goes read-only for writes near the limit (about 2 billion unvacuumed, around 3 million left); long transactions, replication slots and prepared transactions pin the horizon; autovacuum default 50 + 0.2 x rows; HOT needs same-page room and no indexed column change. **Mine or from memory:** section 1 arithmetic, the 200 million freeze default and 5.6 hour calculation, `transaction_timeout` in Postgres 17, undo-log vs in-table comparison, all Rare.lab risk rows (inference about your schema).

**Related lessons:** Day 17 (WAL and CDC), Day 21 (LSM trees), Day 23 (content-addressed storage), Day 29 (write skew), Day 58 (online migrations), Day 78 (connection pooling), Day 87 (timers and retry storms).
