# Day 43 — How do you answer any of thousands of possible slice-and-dice questions, over tens of billions of rows, in single-digit milliseconds, 250,000 times a second?

*2026-07-29*

---

## 1. The company and the number that breaks a naive design

**LinkedIn, 2013, building "Who's Viewed Your Profile."** In early 2014 LinkedIn shipped a redesigned version of that feature: not just a count, but the ability to slice your profile-view history by viewer's company, title, school, industry, and time window, all at once, refreshed in near real time. The underlying event stream, a single "someone viewed a profile" fact, needed to answer an open-ended set of GROUP BY questions ("how many of my viewers work at Google, are VPs, and viewed me in the last week?") over a dataset spanning what is now 1.2 billion+ LinkedIn members. Nobody could enumerate in advance which combination of dimensions a member would ask for next. That project became Apache Pinot (LinkedIn Engineering, "Open Sourcing Pinot"; the original design is documented in "Pinot: Realtime OLAP for 530 Million Users").

The number that breaks a naive design: LinkedIn's Pinot deployment today answers **250,000+ queries per second at millisecond latency across hundreds of billions of records**, spread over 50+ user-facing applications (LinkedIn Engineering, "Real-time Analytics at Massive Scale with Pinot"). Uber runs the same engine at a different kind of scale: its Neutrino query layer serves **500 million+ Pinot queries a day** over tables with **10 billion+ rows**, with **300,000 analytics events a second** flowing into Pinot's real-time tables, and Uber's own benchmarks report **sub-100ms p99.5 query latency** even at that volume (Uber Engineering, "Serving Millions of Apache Pinot Queries with Neutrino"; StarTree, "Uber Serving Real-Time App Crash Analytics"). The breaking number in one line: a query engine that must answer an arbitrary, unpredictable combination of GROUP BY dimensions, over tens of billions of rows, in single-digit-to-tens-of-milliseconds, hundreds of thousands of times a second.

## 2. Why the naive (demo) design dies

The demo version of "let people slice and dice their data" is one of two things: a single relational database table, or a pre-built OLAP cube. Both die at LinkedIn's number, in different ways.

**a. The relational table dies on the scan.** `SELECT company, title, COUNT(*) FROM profile_views WHERE viewed_id = X GROUP BY company, title` over a table with tens of billions of rows either needs an index that happens to match this exact query shape, or it scans. An index on `(viewed_id, company)` does not help a query grouped by `(viewed_id, title, date)`. You cannot pre-build an index for every dimension combination a member might ask for tomorrow, and a full scan of a billion-row partition takes seconds, not milliseconds. Even if it took 50ms, one Postgres instance is nowhere near 250,000 queries/sec.

**b. The pre-built cube dies on combinatorial explosion.** The traditional OLAP answer, materialize a cube for every dimension combination in advance, works until you count the combinations. Five dimensions (company, title, school, industry, date) each with realistic cardinality already produces thousands of distinct GROUP BY combinations, and every one needs its own precomputed rollup, refreshed continuously as new views stream in. Storage and refresh cost grow exponentially with dimension count. This is why classical MOLAP cubes (that materialize every combination) simply do not fit a use case with several high-cardinality dimensions.

**c. Batch OLAP (Hadoop/Hive) dies on freshness.** MapReduce or Hive can crunch the full history overnight and produce beautiful aggregate tables, but "who viewed you in the last hour" cannot wait for tomorrow's batch job. A pure batch pipeline is structurally incapable of a real-time feature, no matter how much hardware you throw at it, because its unit of work is "yesterday's file," not "the event that just happened."

The analogy: imagine a library where the only catalog is sorted by the author's last name. If a patron asks "how many mystery novels published after 2015 are on the third floor," the librarian has no choice but to walk every shelf, because the one catalog they have answers a different question than the one being asked. Building a new catalog for every conceivable question in advance would need more shelf space than the books themselves.

## 3. The architecture, drawn top to bottom

```
CLIENTS / APPS (LinkedIn's "Who Viewed Your Profile," Uber's Neutrino/dashboards)
   send a SQL-like query: GROUP BY whatever dimensions the user picked, right now
   |
   v
BROKER (the query's traffic cop)
   knows which servers hold which SEGMENTS of a table (via the controller)
   scatters the query only to the servers holding relevant segments, gathers
   partial results, merges them, returns one answer
   analogy: a dispatcher who only rings the specific warehouses that could
   possibly hold the item, not every warehouse in the country
   |
   v
CONTROLLER (Helix + Apache ZooKeeper underneath)
   tracks "ideal state": which segment lives on which server, right now
   when a new segment is built, updates that state; brokers and servers
   just watch it
   analogy: the shipping manifest office that always knows which
   warehouse has which crate, so nobody has to guess
   |
   v
   +-----------------------------+-----------------------------+
   |                             |                             |
REAL-TIME SERVERS             OFFLINE SERVERS               DEEP STORE
consume directly from a       hold SEGMENTS built by a       (S3 / HDFS / blob
Kafka topic (Day 42's log)    batch job (Spark/Hadoop)       storage): the
partition-by-partition,       over yesterday's full          durable backup
building an in-memory,        history, immutable and         copy of every
mutable segment as events     already columnar               sealed segment,
arrive; once a segment                                       so a server
fills up (row count or                                       that dies can be
time threshold) it is                                        rebuilt from here
sealed: flushed to columnar,
made immutable, uploaded to
deep store
   |                             |
   +--------------+--------------+
                  v
     REAL-TIME TABLE + OFFLINE TABLE, queried as ONE logical table
     recent (unsealed / just-sealed) data comes from real-time servers,
     history comes from offline segments; the broker stitches both
     halves of the answer together per query, invisibly to the caller
                  |
                  v
        STAR-TREE INDEX (built per segment)
        pre-aggregates metrics for combinations of LOW-CARDINALITY
        dimensions (Pinot's default cutoff: ~10,000 distinct values),
        so a query matching a pre-aggregated node reads one summarized
        row instead of scanning and summing raw rows
        analogy: instead of re-counting a warehouse's inventory shelf
        by shelf every time someone asks, keep a running tally sheet
        at the end of each aisle, at the cost of maintaining the sheet
```

## 4. The transferable mechanisms

- **Immutable columnar segments as the unit of storage and replication.** A segment is a chunk of rows, stored column-by-column (not row-by-row), dictionary-encoded, and sealed once built. Immutability is the same trick as Day 21's LSM-tree SSTables and Day 23's content-addressed blobs: once written, a segment never changes, so replicating it, caching it, or rebuilding a dead server from deep store is just a copy, never a synchronization problem.

- **The star-tree index: a tunable space-for-latency trade.** Instead of scanning raw rows and aggregating on the fly, the star-tree pre-computes and stores the aggregate for chosen low-cardinality dimension combinations, so a matching query reads one row instead of millions. You choose which dimensions get this treatment and how deep, which is explicitly a dial between disk space and guaranteed query latency, not a free win (Apache Pinot docs, "Star-tree index"; StarTree, "What Makes Apache Pinot Fast? Indexing!").

- **Real-time and offline tables unified behind one query, instead of two pipelines.** Classic Lambda Architecture keeps a slow, correct batch path and a fast, approximate speed path as two separate systems that an application has to reconcile itself. Pinot collapses that into one broker-side merge: real-time servers hold what Kafka has produced in roughly the last few hours to days, offline servers hold everything batch-processed before that, and a single query transparently reads both. The mechanism generalizes: whenever "fresh but maybe-incomplete" and "complete but stale" data must both be queried, merge them at the read boundary rather than forcing every caller to do it.

- **Pruning before fan-out, not after.** A broker does not ask every server "do you have this data"; it prunes by time range and partition first (segments are tagged with the time range and partition key they cover), so a query for "last hour" never touches a server holding only last year's segments. This is the same instinct as Day 10's consistent hashing: know which physical location holds the answer before you go looking, instead of asking everyone.

- **Replica groups bound the scatter-gather fan-out.** Instead of every query touching every server that could theoretically have a piece of the answer, a replica group is a full copy of a table's segments held by one subset of servers; a query is routed to just one replica group rather than the whole cluster (Apache Pinot docs, "Routing"). This directly controls how many machines a single query depends on, which matters enormously for the failure mode in section 6.

- **Pull-based ingestion straight from Kafka partitions.** Real-time servers consume Kafka the same way Day 42 described: each server owns a subset of partitions and pulls at its own pace, so ingestion parallelism scales with partition count, with no separate "ingestion service" bottleneck in between.

## 5. The trade-offs

CAP made concrete, per data type:

- **Real-time segment data: availability and freshness win over strict consistency.** Two replicas of the same real-time segment can consume their Kafka partition at slightly different offsets at any instant, so two "identical" queries hitting different replicas within the same second can return marginally different counts. Pinot accepts this because a "who viewed you in the last hour" counter that is occasionally off by a handful of events, but always available and always fast, is a better product than a counter that blocks until every replica agrees.

- **Sealed segments and deep store: consistency wins.** Once a segment is sealed and pushed to deep store, it is immutable and identical everywhere it is replicated. There is nothing to reconcile because nothing about it can change; this is the same "make the disagreement structurally impossible" trade Day 23 makes with content-addressed storage.

- **The star-tree index is a space-for-latency trade, chosen per table.** Pre-aggregating more dimension combinations gives a lower, more predictable query latency ceiling at the cost of more disk per segment; Pinot deliberately caps which dimensions qualify (by cardinality) rather than pre-aggregating everything, because pre-aggregating a high-cardinality dimension like user ID would make the index bigger than the raw data it is indexing.

- **Replica groups are a cost-for-latency trade.** Each replica group is a full, separate copy of a table's segments. More replica groups means the cluster can bound each query's fan-out to fewer machines (better tail latency), but it also means paying for that many full copies of the data in hardware, explicitly, rather than hoping fewer, larger replicas will be fast enough.

## 6. The systems-thinking lens

The feedback loop here is **scatter-gather tail-latency amplification**, the phenomenon Dean and Barroso named "the tail at scale" (Dean & Barroso, "The Tail at Scale," CACM 2013): a query that must fan out to N servers and wait for all N to respond completes only as fast as the *slowest* of those N, and as N grows, the probability that at least one of them is having a bad moment (a GC pause, a disk hiccup, a network blip) grows with it. Pinot's own operational guidance says this explicitly: as query throughput or server count rises, the odds of hitting a slow server climb steeply, and the more servers a single query touches, the worse its tail latency gets, because the scatter-gather cannot finish until the last straggler reports in.

This is a genuine feedback trap, not just a fact of life: the naive response to "queries are getting slower as data grows" is to add more servers and split data across more of them, which is exactly the change that increases how many machines each query has to wait on, which makes the tail *worse*, not better. Scaling out the obvious way feeds the very problem it was meant to solve.

The senior fix does not add capacity, it changes which machines a query is allowed to depend on:

- **Replica groups** bound the fan-out structurally: a query is routed to one full-copy group of servers, never to the whole cluster, so adding more data and more servers to the overall system does not automatically mean any single query has to wait on more of them.
- **Time and partition pruning** shrink the candidate server set before the query ever leaves the broker, so a query for "the last hour" simply never asks a server that holds only last year's segments.
- **Segment-size tuning** keeps individual servers from holding so much data that a single slow disk operation stalls a disproportionate share of all queries.

The general lesson, the same one Day 34's cell-based architecture teaches from the failure-isolation angle: whenever a system's answer to "we're getting bigger" is "so every request touches more of the fleet," growth itself becomes the thing degrading latency. The fix is to keep the blast radius of any single request bounded and roughly constant, no matter how large the whole cluster gets, by routing narrowly instead of broadcasting widely.

---

## Sources

- LinkedIn Engineering, ["Open Sourcing Pinot: Scaling the Wall of Real-Time Analytics"](https://engineering.linkedin.com/pinot/open-sourcing-pinot-scaling-wall-real-time-analytics)
- LinkedIn Engineering, ["Real-time Analytics at Massive Scale with Pinot"](https://engineering.linkedin.com/analytics/real-time-analytics-massive-scale-pinot)
- Apache Software Foundation (cwiki), ["Pinot: Realtime OLAP for 530 Million Users"](https://cwiki.apache.org/confluence/download/attachments/103092375/Pinot.pdf)
- StarTree, ["Launching At LinkedIn: The Story Of Apache Pinot"](https://startree.ai/resources/launching-at-linkedin-the-story-of-apache-pinot/)
- Uber Engineering, ["Serving Millions of Apache Pinot Queries with Neutrino"](https://www.uber.com/us/en/blog/serving-millions-of-apache-pinot-queries-with-neutrino/)
- Uber Engineering, ["Operating Apache Pinot @ Uber Scale"](https://www.uber.com/at/en/blog/operating-apache-pinot/)
- Uber Engineering, ["Real-Time Analytics for Mobile App Crashes using Apache Pinot"](https://www.uber.com/us/en/blog/real-time-analytics-for-mobile-app-crashes/)
- StarTree, ["Uber Serving Real-Time App Crash Analytics While Saving $2M+ with Apache Pinot"](https://startree.ai/user-stories/uber-serving-real-time-app-crash-analytics-while-saving-2m-with-apache-pinot/)
- Apache Pinot documentation, ["Star-tree index"](https://docs.pinot.apache.org/basics/indexing/star-tree-index)
- StarTree, ["What Makes Apache Pinot Fast? Indexing!"](https://startree.ai/resources/what-makes-apache-pinot-fast-chapter-ii/)
- Apache Pinot documentation, ["Routing"](https://docs.pinot.apache.org/operators/operating-pinot/tuning/routing)
- Apache Pinot documentation, ["Real-time" tuning](https://docs.pinot.apache.org/operators/operating-pinot/tuning/realtime)
- Jeffrey Dean and Luiz André Barroso, ["The Tail at Scale," Communications of the ACM, 2013](https://cacm.acm.org/research/the-tail-at-scale/)
