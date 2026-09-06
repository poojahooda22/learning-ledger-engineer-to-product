# Day 69 — How does Netflix plan a query over a data lake table scattered across hundreds of thousands of files in S3, in 10 seconds instead of 10 minutes, with no database sitting in front of it?

**Date:** 2026-09-06
**Difficulty:** Expert
**Topic:** Lakehouse table formats — Apache Iceberg, Delta Lake, and Apache Hudi. This is the layer that gives a folder of immutable Parquet files in an object store the properties normally reserved for a database: atomic commits, snapshot isolation, schema evolution, and query planning that does not require walking the whole table. This ledger has already built every piece this lesson assembles, just aimed at rows instead of files: Day 17 covered an append-only log of every change (WAL/CDC), Day 21 covered the write-amplification-vs-read-amplification trade of rewriting data now versus merging it later (LSM-trees), Day 23 covered content-addressed, immutable objects linked into a tree so nothing needs locking to be read safely (Merkle DAGs, via Git), and Day 24 covered winning a single atomic compare-and-swap as the one piece of the system that must never be eventually consistent (fencing tokens). A table format is those four ideas, combined, and pointed at the specific, very ordinary failure of "a directory listing became the slowest part of the query."
**Stack relevance:** Rare.lab already stores scene state as content-addressed, immutable JSON in Cloudflare R2, plus a manifest, which is structurally the same shape as a table format's data-files-plus-manifest pattern; the hard part, immutability, is already done. The open question is what plays the role of Iceberg's catalog: the one place that says, with a single atomic operation, "the current version of this scene is manifest X." If that answer today is "list R2 objects under this project's prefix and take the newest one," that is exactly the O(n) object-store listing that broke Netflix's Hive tables once file counts climbed into the hundreds of thousands, and Rare.lab's ceiling for hitting it is a project with a deep autosave or undo history, not a hypothetical.

---

## 1. The company and the breaking number

**Netflix, and the Atlas dataset that took longer to plan than to read.** Netflix's big data platform team, presenting at Strata Data Conference NY 2018 ("The Evolution of Netflix's S3 Data Warehouse," Ryan Blue and Daniel Weeks), described a Hive table over their Atlas monitoring data that had grown, the ordinary way any actively-written table grows, into hundreds of thousands of small files in S3. Planning a single query against that table, before a single byte of actual data was read, involved listing every partition directory to discover which files existed: over 400,000 splits, taking 9.6 minutes of wall-clock time just to figure out what to read. After moving the same table to Iceberg, the table format Netflix built specifically to fix this, the same query planned against 15,218 combined splits in about 10 seconds. The data did not get smaller. The way of finding out what data existed did.

**The number behind why this keeps happening: nobody plans for 400,000 files, they arrive one write at a time.** A table does not start at 400,000 files. It starts at a few hundred, from a few weeks of daily batch jobs, each job dropping new Parquet files into a partition directory. Nothing about that process ever asks "how many files exist in this table now," so nothing forces anyone to compact them, until the day someone runs a query and it takes ten minutes to start.

**The Uber number for the other half of the problem: updates, not just appends.** Uber Engineering's account of running Apache Hudi at what they describe as trillion-record scale reports roughly 19,500 Hudi datasets, about 6 trillion rows ingested per day, on the order of 3 million new data files added per day, and 350 logical petabytes of data under management across HDFS and Google Cloud Storage, with on the order of 10 petabytes ingested and multiple petabytes written to the lake daily. Uber's core workload is not "append new rows and never touch them again," it is trip records, driver states, and fare calculations that get corrected and re-written constantly. A naive Hive table has exactly one way to apply an update to a row that already landed in a Parquet file in S3: rewrite the entire file, or in practice the entire partition, that row lives in. At Uber's volume, that is not a slow query, it is an infeasible one.

---

## 2. Why the naive (demo) design dies

**The obvious version:** a Hive-style table on top of an object store. Partitions are just directory prefixes (`s3://bucket/table/dt=2026-09-06/`), the Hive Metastore stores a mapping from partition key to that S3 prefix, and to find the actual files inside a partition, the query engine issues a `LIST` call against S3 at query time. This is exactly how the data warehouse ecosystem worked for years, and for a table with a few hundred files it works fine.

**Death one: query planning is an O(n) directory listing, and S3 listing has never been instant.** To plan a query, the engine lists every relevant partition directory to enumerate its files, one `LIST` API call per prefix, sometimes many if a prefix has more objects than one page returns. Do this across tens of thousands of partitions and query planning alone, before any actual reading, becomes the dominant cost. This is exactly Netflix's 9.6 minutes: not I/O reading data, I/O discovering that the data exists at all.

**Death two: shared prefixes hit S3's own request-rate limits.** Hive's directory-per-partition layout means many files sharing a common key prefix. High-concurrency listing or writing against that shared prefix runs into S3's per-prefix request-rate limits, returning `503 SlowDown` throttling errors under exactly the load a growing, actively-queried table produces. The naive design's own popularity is what triggers its failure mode.

**Death three: no snapshot isolation, so a reader can see a table that is being written to.** The Hive Metastore only tracks the partition-to-path mapping, not a versioned, atomic view of "everything that exists in this table as of one consistent moment." A reader listing a directory while a writer is mid-append can see a partially-written partition, count a file twice if a rename is still in flight, or simply get a different answer than a query that ran five seconds earlier for reasons that have nothing to do with the actual data changing. Concurrent readers and writers were never given a consistent point to agree on.

**The real-world version:** a Netflix analyst runs the same monitoring query that ran fine last quarter. Six months of daily jobs have quietly grown the underlying table from a few thousand files to over 400,000. The query itself hasn't changed. The answer takes ten times longer to even start, and nobody touched a single line of SQL to cause it.

---

## 3. The architecture

```
Writers (Spark, Flink, Trino jobs; in Rare.lab's case, the editor or
runtime writing new scene state)
  - job: produce new immutable data (Parquet files, or Rare.lab's
    content-addressed scene JSON), never edit an existing file in place
  - analogy: a reporter filing a new dated article, never editing
    yesterday's already-printed paper

        |
        v
Table format client library (Iceberg / Delta / Hudi SDK)
  - job: wrap the new data files in a manifest (a list of exactly which
    files this write added, plus per-file stats: row count, min/max
    per column), then build a new snapshot that points at the old
    snapshot's manifests plus this new one
  - analogy: a librarian who, instead of re-cataloguing the whole
    library on every new arrival, writes one new index card listing
    just today's new books, and staples it to last week's stack of cards

        |
        v
Object store (S3 / R2 / HDFS): data files + manifest files +
manifest lists + metadata/snapshot log, all immutable once written
  - job: hold every version of every piece cheaply and durably; nothing
    here is ever mutated, only added, so nothing here ever needs a lock
  - analogy: a shipping dock's stacked, dated crates; nobody
    reorganizes yesterday's crates to make room for today's

        |
        v
Catalog (Hive Metastore / AWS Glue / Iceberg REST catalog / Nessie /
Delta's transaction log; for Rare.lab, one row in Supabase Postgres)
  - job: hold the one thing in this whole system that must be strongly
    consistent, a single pointer: "the current snapshot ID for table T
    is X." Updated with one atomic compare-and-swap (S3 conditional
    PUT, an atomic rename, or a Postgres `UPDATE ... WHERE version = X`)
  - analogy: a single sign at the library entrance reading "current
    edition of the catalog is #482," swapped for a new sign only when
    the new edition is completely ready, never a partial swap

        |
        v
Query engines (Spark, Trino, Flink, DuckDB)
  - job: read the catalog pointer (one RPC), load that snapshot's
    manifest list, load only the manifests whose partition stats could
    possibly match the query's filters, and only then touch the actual
    data files those manifests named
  - analogy: checking a phone directory instead of walking every street
    in the city to find one phone number

        |
        v
Table maintenance / compaction service (background job, not optional)
  - job: rewrite many small data files into fewer large ones, expire
    snapshots nobody needs anymore, and delete data files no live
    snapshot still references
  - analogy: a library's overnight reshelving crew, merging today's
    hundred single-book index cards back into one clean shelf list
    before tomorrow's readers show up
```

---

## 4. The transferable mechanisms

- **Metadata as a tree with stats at every level, not a directory listing.** Snapshot → manifest list → manifests → data files, and each layer carries enough summary statistics (partition values, per-column min/max) that a query engine can prune entire branches without ever touching the layer below. This turns "list every file" (O(n) in table size) into "one RPC for the catalog pointer, plus a bounded number of manifest reads" (roughly O(matching partitions), independent of total table size). It is the same shape as an inverted index skipping documents that can't match, aimed at files instead of terms.

- **Optimistic concurrency control via one atomic pointer swap.** A writer builds its entire new snapshot, files, manifests, manifest list, out of immutable pieces that touch nothing anyone else is reading, then performs exactly one atomic compare-and-swap on the catalog pointer. If another writer's swap landed first, the loser retries against the new current state instead of corrupting it. This is Day 24's fencing-token idea, generalized: the lock isn't held for the duration of the write, it's a single, instantaneous, all-or-nothing handoff at the very end.

- **Copy-on-write versus merge-on-read is the same trade as Day 21's LSM-trees, one layer up.** Copy-on-write rewrites the data files an update touches immediately, so reads stay simple and fast but every write costs more. Merge-on-read appends a small delta/log file and defers the merge to read time or to a background compaction job, so writes stay cheap but reads (until compaction catches up) must merge multiple files to answer one query. Uber's trip-record corrections lean toward merge-on-read for exactly the reason Day 21's write-heavy workloads lean toward LSM-trees: the update rate is too high to pay the rewrite cost synchronously.

- **Snapshot isolation and time travel, for free, because nothing is ever deleted until you say so.** Every commit produces a new, immutable, numbered snapshot. A query pins one snapshot ID for its entire execution, so it can never observe a half-written state, and because old snapshots aren't deleted until explicitly expired, "query the table as it looked yesterday" is just "point a reader at yesterday's snapshot ID," no separate backup system required. This is Day 17's append-only WAL idea, but the log entries are whole-table states instead of individual row changes.

- **Immutability is what makes concurrent, lock-free reads safe.** No data file, manifest, or manifest list is ever edited in place, only added, and only ever referenced by a new snapshot once it's completely written. A reader mid-query can never see a file change underneath it, because nothing already committed ever changes. This is Day 23's Merkle-DAG, content-addressed-object idea, doing for a data warehouse table exactly what it does for a Git repository.

- **Hidden partitioning removes the "user filtered on the wrong column" failure class.** Iceberg specifically lets the table format compute partition values from a transform of a raw column (say, `day(event_time)`) and track that transform in the metadata itself, so a query filtering on the raw timestamp still prunes correctly, without every query author needing to know or remember the underlying physical partitioning scheme.

---

## 5. The trade-offs

**Consistency is concentrated into one tiny pointer; everything else only needs eventual visibility.** The catalog pointer is the single place this system cannot tolerate being eventually consistent, two writers racing to update it must produce exactly one winner, or two readers could each believe they hold the current, complete state while actually looking at two different, inconsistent versions. Every other piece, the actual data files, the manifests, only needs to become visible eventually once written, because nothing ever reads them until a snapshot that already references them has been committed. The whole design is built around shrinking "must be strongly consistent" down to the smallest possible surface area, one pointer, so everything else can be cheap, dumb, highly-available object storage.

**Cost versus latency: a small, constant metadata write per commit, to avoid an O(n) listing per query.** Every commit writes a few kilobytes to tens of kilobytes of manifest and snapshot metadata, real, recurring storage and write cost. The trade is Netflix's own number: paying that small constant cost on the write side is what turned a 9.6-minute, size-of-the-table planning cost on the read side into a 10-second, size-of-the-catalog-lookup cost instead.

**Copy-on-write versus merge-on-read is a read-latency-versus-write-latency dial, not a free win.** Copy-on-write suits read-heavy, update-light workloads (BI dashboards querying yesterday's finalized data). Merge-on-read suits write-heavy, correction-heavy workloads (Uber's constantly-revised trip and fare data), at the cost of slower or compaction-dependent reads until the background merge catches up.

**The small-file problem is the same failure mode this design exists to fix, waiting to come back.** Every write, especially frequent, small, streaming writes, can create new small data files. Left uncompacted, file count climbs the same way it did under Hive, and query planning slows down again, not because the table format failed, but because the mandatory maintenance step (compaction, snapshot expiration, orphan-file removal) was skipped. A table format buys you the tools to keep file count bounded; it does not enforce that you use them.

---

## 6. The systems-thinking lens

The feedback loop here is a slow-building **metastable failure hiding inside ordinary, unremarkable growth**: a table starts small, every individual write is fast and unremarkable, file count creeps upward one batch job at a time, and nothing in the system ever alarms on "file count" as a metric worth watching, because no single write ever looked expensive. The cost is entirely deferred to query planning, which degrades gradually and non-obviously, exactly the shape of failure that is invisible until a threshold is crossed and then suddenly very visible: Netflix's 9.6 minutes wasn't a spike, it was the slow accumulation of six months of ordinary jobs finally showing up in one number. Left unaddressed, this becomes a genuine retry-death-spiral: slow planning causes query timeouts, timeouts cause client retries, retries add more concurrent listing load against the same S3 prefixes and the same metastore, which are already the bottleneck, making the next query's planning slower still.

The naive fix, throwing capacity at it, a bigger Hive Metastore, requesting a higher S3 request-rate limit, buys headroom without changing the shape of the problem: query planning is still O(n) in file count, it just tolerates a slightly larger n before failing the same way again. The senior fix is structural, and it's the same move Day 13's backpressure lesson makes for retries: don't add capacity to the loop, remove the step that scales with the thing that's growing. Iceberg's manifest hierarchy turns planning from "list every file" into "read a bounded, catalog-anchored metadata tree," so query planning cost stops scaling with total table size at all, it scales with how many partitions could possibly match the query. Mandatory compaction closes the second half of the loop, keeping file count itself bounded so the metadata tree never grows deep enough to matter, cutting off the failure at its source instead of coping with a bigger version of it later.

---

## Sources

- [The Evolution of Netflix's S3 Data Warehouse, Ryan Blue and Daniel Weeks, Strata Data Conference NY 2018](https://conferences.oreilly.com/strata/strata-ny-2018/public/schedule/detail/69503.html): the primary source for the Atlas dataset comparison (400,000+ splits and 9.6 minutes of planning time under Hive-on-S3, versus 15,218 combined splits and roughly 10 seconds of planning time under Iceberg) and for Iceberg's own origin as an internal Netflix project to fix exactly this. Direct fetch of the slides and the O'Reilly/SlideShare hosting of them was blocked by this session's network egress policy, so these figures are drawn from search-indexed summaries of the talk rather than a direct read of the original slide deck, and are flagged as such rather than presented as independently verified quotes.
- [Apache Iceberg, GitHub (originally Netflix/iceberg)](https://github.com/apache/iceberg): the project itself, now an Apache Software Foundation project, originally released by Netflix's big data platform team.
- [Apache Iceberg documentation, Reliability](https://iceberg.apache.org/docs/1.4.1/reliability/): source for the general architectural claim that reading a snapshot requires O(1) RPC calls rather than O(n) directory listing, and for Iceberg's snapshot-isolation and atomic-commit model.
- [Apache Hudi at Uber: Engineering for Trillion-Record-Scale Data Lake Operations, Uber Engineering blog](https://www.uber.com/en-us/blog/apache-hudi-at-uber/): source for the roughly 19,500 Hudi datasets, approximately 6 trillion rows ingested per day, on the order of 3 million new data files added per day, and 350 logical petabytes under management across HDFS and Google Cloud Storage. Direct fetch was blocked by this session's network egress policy; figures are drawn from search-indexed excerpts of the post.
- [Building a Large-scale Transactional Data Lake at Uber Using Apache Hudi, Uber Engineering blog](https://www.uber.com/en-us/blog/apache-hudi-graduation/): secondary corroborating source for Hudi's motivating problem at Uber, incremental upserts replacing full-partition rewrites, and the resulting reduction in ETL pipeline runtime.
- [Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores, Armbrust et al., Proceedings of the VLDB Endowment, Vol. 13 (2020)](https://www.vldb.org/pvldb/vol13/p3411-armbrust.pdf): the primary academic source for Delta Lake's transaction-log design and its optimistic-concurrency-control commit protocol (read the latest snapshot, stage new files, check for conflicts against anything committed since, then attempt one atomic commit).
- [Understanding the Delta Lake Transaction Log, Databricks Engineering blog](https://www.databricks.com/blog/2019/08/21/diving-into-delta-lake-unpacking-the-transaction-log.html): source for how the `_delta_log` ordered record of every commit is compacted into Parquet checkpoints and used to reconstruct table state without replaying the full history on every read.
- [Concurrency control, Delta Lake documentation](https://docs.delta.io/concurrency-control/): source for the mechanics of atomic commits across storage backends, atomic rename on HDFS/ADLS-style filesystems versus conditional-write or external-coordination-service based locking historically required on S3 before S3 added native conditional writes.
- Day 17 (this ledger, WAL and change data capture), Day 21 (LSM-trees and write-optimized storage), Day 23 (content-addressed storage and Merkle DAGs), Day 24 (distributed locks and fencing tokens): the ledger's own prior lessons this one directly combines, generalized here from single-row or single-object mechanics up to whole-table mechanics.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of the original Strata 2018 slide deck and both cited Uber Engineering blog posts, so the specific numbers drawn from them (the 400,000-split/9.6-minute figure, the 15,218-split/10-second figure, and Uber's trillion-record-scale statistics) are summarized here from search-indexed excerpts rather than quoted from a full direct read of the source pages, consistent with how this ledger has flagged network-blocked sources before. The Apache Iceberg documentation and the Delta Lake VLDB paper and Databricks blog post were used for the mechanism-level claims (manifest hierarchies, transaction logs, optimistic concurrency) and are treated as solid, well-established public documentation of these systems' actual designs.
