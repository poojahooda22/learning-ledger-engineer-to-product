# Day 65 — How does Facebook keep 2 billion live time series queryable at memory speed, when its own monitoring systems write 700 million new data points every single minute?

**Date:** 2026-08-26
**Difficulty:** Expert
**Topic:** Time series metrics ingestion and storage at scale: the specific shape of database that has to accept a firehose of small, append-only, timestamped numbers forever, and serve read queries that overwhelmingly care about the last few hours, not the whole history. The forcing example is Facebook's own Gorilla paper (Pelkonen et al., VLDB 2015), which discloses that as of spring 2015 Facebook's monitoring systems were adding roughly 12 million data points a second, more than 700 million a minute, over 1 trillion a day, across 2 billion distinct time series, and that at least 85% of all queries against that data asked only for the last 26 hours of it. Why a plain row-per-point table in a general-purpose relational database dies twice on that workload, once on raw sustained write throughput and once on the index bloat caused by high-cardinality tags, long before it dies on total storage volume. How Gorilla's answer (an in-memory, compressed, 26-hour write-through cache in front of a durable HBase store) and two independently built, differently shaped systems that converge on the same underlying moves, Uber's M3 (quorum-replicated storage plus configurable downsampling tiers) and Datadog's rewritten timeseries index (a universal inverted tag index, sharded within each node), show this is a real, load-bearing pattern rather than one company's idiosyncratic fix. And why the single most common way these systems actually fall over in production is not raw volume at all, it is cardinality explosion, one careless label turning one metric into millions of new, permanently tracked time series overnight, the same failure Prometheus's own documentation and a wide, repeated body of operator experience across its ecosystem all warn about by name.
**Stack relevance:** Rare.lab does not run a metrics or observability platform at anything like this scale today, and that is worth saying plainly before claiming a transfer. But the moment Rare.lab instruments its embeddable runtime, the one shared WebGL context serving many embedded shader players at once, with performance telemetry (frame time, shader compile time, draw calls per scene), this lesson's exact failure mode is one careless tag away: a scene ID, a node ID, or an embed/customer ID dropped into a metric label is precisely the kind of near-unbounded identifier this lesson's Section 6 names as the trigger for cardinality explosion, and Rare.lab's actual Supabase Postgres, used naively as a row-per-point metrics table, is precisely this lesson's naive design in Section 2.

---

## 1. The company and the breaking number

**Facebook, spring 2015.** The Gorilla paper, published by Facebook engineers Tuomas Pelkonen and colleagues at VLDB 2015, opens with the operating numbers behind Facebook's internal monitoring system, ODS (Operational Data Store). At that point, Facebook's monitoring pipeline was generating more than 2 billion unique time series, each identified by a string key such as a host name plus a metric name plus a set of tags, and writing into that population of series at a sustained rate the paper's own design target states plainly: an insertion rate of 700 million data points, each one a timestamp plus a floating-point value, every minute. Restated per second, that is roughly 12 million individual writes a second, and the paper separately reports that the real, observed spring-2015 write rate was in fact close to that figure, around 12 million points added per second, tallying to more than 1 trillion data points written in a single day. Those two figures, the 700-million-a-minute design target and the roughly-12-million-a-second observed rate, agree with each other to within rounding, which is the strongest signal available that this is a real, sustained production number rather than a one-time peak.

The second number in the paper is the one that actually explains Gorilla's entire architecture, and it is not a write number, it is a read number: at least 85% of all queries issued against ODS asked only for data collected in the past 26 hours. Almost nobody was querying last month's CPU graph. Almost everybody was querying the last few hours, because that is what "is this incident still happening" and "did that deploy just break something" actually require. A monitoring system's read pattern, in other words, is heavily, predictably skewed toward the newest slice of an otherwise unbounded, ever-growing dataset, and any design that treats "store everything" and "serve the last few hours fast" as the same problem is going to pay for that conflation somewhere.

Put the two numbers together and the shape of the actual engineering problem appears: build something that can absorb roughly 12 million small writes a second, forever, without falling behind, while making the read path for the last 26 hours of that stream fast enough that a human staring at a dashboard during an active incident does not notice the system underneath it. Everything else, including where the data eventually lives once it is more than a day old, is a secondary concern next to those two numbers.

---

## 2. Why the naive (demo) design dies

**The obvious version:** one relational table, one row per data point. Something like `metric_points(metric_name text, tags jsonb, ts timestamptz, value double precision)`, with a B-tree index on `(metric_name, ts)` so a dashboard query can ask for "this metric, ordered by time, in this range" and get an index range scan. For a small deployment, a handful of hosts each emitting a few dozen metrics every ten seconds, this is not just adequate, it is the right answer, the same way Day 61 pointed out that a small catalog's autocomplete belongs in a client-side trie rather than a distributed system nobody needs yet. The design only becomes naive once it is asked to survive something close to Facebook's actual numbers.

**Death one: raw sustained write throughput.** A single, well-tuned relational primary, one that durably commits every write (fsyncing its write-ahead log so a crash cannot silently lose an already-acknowledged point) and maintains a B-tree index on every insert, comfortably sustains individual row writes on the order of tens of thousands a second on strong modern hardware, and batching many points into one multi-row insert or a bulk COPY can push that meaningfully higher, but not by three orders of magnitude. 12 million individual points a second is not an optimization problem for a single relational writer, it is a different category of system entirely, the same gap Day 21's LSM-tree lesson described between a B-tree's random-write cost and a log-structured merge tree's sequential-append cost, except here the gap shows up before the table even has enough rows to make a B-tree's page splits the dominant cost.

**Death two: cardinality turns the index itself into the bottleneck, independent of raw volume.** Every distinct combination of `metric_name` and `tags` is, functionally, a distinct time series, and a distinct time series is a distinct thing the index has to be able to look up quickly. A metric like `request_latency` tagged by `host`, `endpoint`, and `status_code` across a fleet of 10,000 hosts with 200 distinct endpoints and a handful of status codes is already millions of unique combinations. If a well-meaning engineer adds one more tag, `user_id` or `request_id` or a Kubernetes pod name that changes on every deploy, the number of distinct series that single metric name maps to can jump from thousands to millions overnight, because the tag space is now effectively unbounded rather than a small, fixed set of values. Prometheus, the open-source time series system whose own documentation and a wide, repeated body of operator writeups across its ecosystem call this exact failure "cardinality explosion" or the "cardinality bomb," recommends keeping a single instance's total active series count under the tens of millions for this reason: past that point, the in-memory index mapping every unique label combination to its own series simply does not fit comfortably in RAM anymore, and everything downstream, ingestion, indexing, and query, degrades together. A naive relational table with `tags` folded into the index key inherits this problem directly. Its B-tree does not distinguish "a metric with a genuinely bounded, small tag space" from "a metric someone accidentally tagged with a request ID." Both make the index bigger, and only one of them is a mistake anyone notices before it is already in production.

**Death three: the query pattern the naive design has to serve does not match the storage layout it actually has.** Given the 85%-of-queries-want-the-last-26-hours pattern from Section 1, the ideal storage layout keeps recent data physically close together and fast to scan, and lets old data sit somewhere cheaper and slower without anyone caring. A single growing table with one index ordered by timestamp gets closer to this than a table indexed some other way, but it still means every query for "the last hour of this metric" is a scan through a B-tree that also contains, and has to skip past the structural overhead of, months of history it will never touch for that query. There is no notion in the naive design of "this data is hot, keep it somewhere that answers in microseconds" versus "this data is cold, it is fine if it takes longer," and building that distinction in after the fact, once a table already holds a trillion rows, is a substantially harder migration than designing for it from day one.

---

## 3. The architecture

```
Host / application (the source of the numbers)
  - a local agent or client library counts, times, and gauges
    events in-process and pre-aggregates before anything leaves
    the machine (a counter increment becomes one number per flush
    interval, not one network call per event)
  - job: cut volume at the source, before it becomes anyone else's
    problem
  - analogy: a store clerk tallying the day's sales on a notepad
    instead of phoning head office after every single transaction

        |  batched flush, every few seconds
        v
Aggregation / rollup tier (Uber M3's Aggregator)
  - deduplicates points that arrived from redundant sources,
    combines points across a flush window into the metric's
    declared resolution, and applies each metric's retention
    policy (for example: full resolution for 2 days, 1-minute
    rollups for 1 month, 10-minute rollups for a year or more)
  - job: shrink write volume by resolution tier, so old data costs
    less to keep without deleting it outright
  - analogy: a photo album that keeps every frame from today's
    shoot but files away only one souvenir print per week once a
    year has passed

        |
        v
In-memory write-through cache (Gorilla's TSmap)
  - holds the most recent window (26 hours, in Facebook's design)
    fully in RAM, compressed on arrival using delta-of-delta
    timestamps and XOR'd floating-point values, so 85% of all
    queries are answered without a single disk read
  - job: make the slice of data almost every query actually wants
    effectively free to read
  - analogy: a whiteboard on the wall showing today's live numbers,
    erased and refiled only once the day is genuinely over

        |  periodic compressed snapshot, for crash recovery
        v
Durable, sharded, replicated cold store (HBase behind Gorilla;
M3DB nodes behind M3, written via quorum to 3 replicas)
  - the long-term system of record; a node dying does not lose
    data because a majority of replicas still hold it
  - job: survive individual machine failure without losing history
  - analogy: three separate filing cabinets in three separate
    buildings, each holding a full copy of the same folder, so a
    fire in one building does not destroy the record

        |
        v
Sharded inverted tag index (Datadog's rewritten indexing layer)
  - every tag maps directly to the list of series IDs that carry
    it (`env:prod` maps to every series tagged with it), rather
    than every query needing to scan every series; each storage
    node further splits its own index into shards (8 per 32-core
    node in Datadog's design) so one node's index work runs in
    parallel across its own cores
  - job: turn "find every series matching these tags" into a
    handful of key lookups and a set intersection, not a scan
  - analogy: a library card catalog cross-indexed by both author
    and subject, so a request for "science fiction by this author"
    is two lookups and an overlap, not a walk down every shelf

        |
        v
Cardinality guard (a limiter at the point of ingestion)
  - tracks how many distinct series each metric name is generating
    and refuses, quarantines, or alerts on a metric whose tag space
    is growing without bound, before that growth reaches the index
  - job: stop an unbounded label from ever becoming millions of
    permanent index entries
  - analogy: a guest-book at the door capping how many genuinely
    new names get added tonight, so the sign-in sheet does not
    grow into an unusable phone book by morning

        |
        v
Query / aggregation layer
  - fans a dashboard's query out to every relevant shard, merges
    partial results, and applies any further rollup the query asks
    for (a max, a percentile, a rate) without ever moving the raw
    underlying points anywhere
  - job: turn billions of stored points into one number per
    dashboard cell, computed close to where the data already lives
  - analogy: asking every branch office for its own subtotal and
    adding the subtotals centrally, instead of shipping every
    individual receipt to headquarters to be added up there
```

Gorilla itself sits only in the top two boxes of this picture, the in-memory 26-hour cache in front of HBase's durable storage, and its own reported numbers justify the design: the paper states that Gorilla's compression scheme shrinks each 16-byte raw point (an 8-byte timestamp, an 8-byte double) down to an average of about 1.37 bytes, a roughly 12x reduction, which is what makes holding 26 hours of 2 billion time series in RAM at all affordable. The compression itself exploits two very ordinary properties of real monitoring data rather than using a general-purpose compressor: timestamps almost always arrive at fixed intervals, so encoding each one as the delta of the delta from the previous two points collapses to a single bit about 96% of the time (a point that landed exactly on schedule needs no correction at all); and a metric's value at one moment is usually close to, or identical to, its value a moment before, so XOR-ing the current value's bit pattern against the previous one leaves mostly zero bits, which the paper reports compresses to a single bit about 51% of the time (value unchanged), to roughly 26.6 bits on average for another 30% of points, and to roughly 36.9 bits for the remaining 19% where the value moved by more. The paper reports this combination reduces production query latency by more than 70x compared to the prior HBase-only design, precisely because the vast majority of queries, the 85% asking for the last 26 hours, never touch disk at all anymore.

Uber's M3 and Datadog's rewritten indexer were both built independently of Gorilla and of each other, in different languages, for different companies, and neither is a copy of Facebook's design. What makes their convergence meaningful is that both land on the same underlying moves from opposite directions: M3 (per Uber's own "M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus" post) aggregates roughly 500 million metrics a second and persists roughly 20 million metrics a second into M3DB, written to 3 replicas by quorum for durability, houses more than 6.6 billion time series, and lets an operator declare per-metric retention and resolution tiers, the same downsample-the-old-stuff idea Gorilla's HBase tier and this lesson's aggregation box both rely on. Datadog (per its own engineering blog, fetched directly for this lesson) rebuilt its indexing layer specifically because its original approach, three separate RocksDB stores per node with only selectively indexed queries, required 7 or more key lookups for any query that was not one of the specific patterns it had chosen to index ahead of time, and replaced it with the universal inverted tag index and per-node sharding described above, cutting worst-case query lookups to a consistent couple of steps, supporting 20x higher cardinality on the same hardware, and cutting indexing cost by half. Two companies, two completely different codebases, arriving at "index by the actual query predicate, shard the index itself, and treat recent data specially" independently is the same kind of convergent evidence this ledger's Day 62 and Day 63 lessons both leaned on.

---

## 4. The transferable mechanisms

- **A write-through cache scoped by recency, not by a fixed size.** Gorilla does not try to keep "the most important" data in memory, it keeps the most recent 26 hours, full stop, because that is what the read pattern actually demands. The mechanism worth taking elsewhere is narrower than "add a cache": scope the cache by the dimension your own read pattern is actually skewed along, and let anything the cache does not cover fall through to slower, durable storage without treating that fallback as a failure.

- **Compress against the real, predictable structure of your data, not with a general-purpose algorithm.** Delta-of-delta timestamps and XOR'd floats work because monitoring data has specific, exploitable regularities (fixed sampling intervals, slowly changing values), and a compressor built around those regularities beats gzip by a wide margin at a fraction of the CPU cost. The transferable idea is to ask what is actually predictable about your own data before reaching for a generic compression library.

- **Pre-aggregate and downsample at the edge, before volume becomes someone else's problem.** A client-side agent that turns a burst of raw events into one number per flush interval, and a rollup tier that turns full-resolution history into coarser summaries once it ages past its usefulness window, both shrink the data before it reaches the expensive, shared part of the system. This is the same instinct Day 39's distributed-tracing lesson applied to sampling, do the volume reduction as early and as cheaply as possible, not after the data has already cost you a network hop and a disk write.

- **Index by what you will actually filter on, not by how you happen to store the data.** Datadog's move from selectively-indexed materialized views to a universal inverted tag index is exactly this: the index's shape now matches the query's shape (find series with these tags) instead of forcing every query to work around an index that was optimized for a different, earlier guess about what people would ask for.

- **Shard the index itself, not just the data.** Datadog's 8-shards-per-node split spreads both the write and the lookup cost of maintaining the index across a node's own cores, the same consistent-hashing intuition Day 10 applied to spreading data across machines, applied one level down, to spreading the bookkeeping about the data across a single machine's own parallelism.

- **Treat cardinality as a resource with a budget, not an emergent property you discover in an incident.** A metric's tag space is, structurally, a promise about how many distinct time series it can ever produce. Guarding that promise at the point of ingestion, refusing or quarantining a metric whose distinct-series count is growing without bound, is cheaper by orders of magnitude than discovering the same growth after it has already doubled an index's memory footprint in production.

---

## 5. The trade-offs

**Consistency vs. availability, and it splits cleanly by which tier of the data you are talking about.** The hot, in-memory 26-hour window is explicitly a cache, not a system of record, Gorilla's own paper frames it as a write-through cache precisely because losing a few seconds of the most recent points during a crash, before the next snapshot, is an acceptable cost for a system whose entire purpose is fast answers about recent state, not a legal or financial record. The durable cold tier behind it, HBase for Gorilla, quorum-replicated M3DB nodes for M3, makes the opposite choice deliberately, because that tier is the actual long-term record, and losing history there is not an acceptable cost the way losing the last few seconds of the hot cache is. The same system runs both postures at once, on purpose, because a single metric point is not one kind of data, it is two: a value that is only interesting while it is fresh, and a value that becomes part of a permanent record once it ages past the hot window.

**Cost vs. latency, paid explicitly as RAM.** Holding 26 hours of 2 billion time series in memory, even compressed 12x, is a genuinely expensive amount of RAM across a large fleet of machines, and Facebook chose to pay that cost because the alternative, serving 85% of all queries from disk, was measured to be more than 70x slower. The trade-off is not free, it is a deliberate decision that the operational cost of that RAM is worth paying for the latency it buys, and it is a decision that only makes sense once you know, as Facebook's own query logs told them, exactly how skewed the read pattern actually is toward recent data. A system that guessed wrong about that skew, keeping a shorter or longer hot window than its actual query pattern warranted, would be paying for RAM it does not need or missing latency it could have bought.

**Storage cost vs. CPU cost, and Datadog's own migration shows a team consciously choosing which one to spend.** The universal inverted index stores each time series once per tag it carries, which Datadog's own post states plainly costs 10x or more in raw storage for a series carrying 10 or more tags, a real, accepted increase in disk usage. That trade was worth making specifically because Datadog's original bottleneck was CPU, index-matching work during ingestion and worst-case multi-lookup queries, while disk capacity was comparatively underutilized. The lesson is not "inverted indexes are always worth 10x the storage," it is that the right trade-off depends on which resource is actually scarce in your own system, and that answer can flip a rearchitecture from wasteful to obviously correct.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is a **cardinality bomb**, a specific, well-documented instance of the same class of failure as a retry storm or a thundering herd, except the trigger is a single bad decision made once, upstream, by someone who never sees the consequence land. It plays out like this: an engineer adds a label to a widely emitted metric, `user_id` on a request-latency counter, a Kubernetes pod name on a per-container metric, a raw URL path instead of a normalized route template. Each of those looks, locally, like a harmless, even helpful, addition, more detail is usually good. But because the label's value space is effectively unbounded, every new unique value the label takes on creates a brand-new time series that the system now has to track permanently: a new entry in the in-memory index, a new set of chunks to allocate and eventually compact, a new row in every downstream aggregation that has ever grouped by that label. The index that used to hold thousands of series for that metric can hold millions within hours, and because the index for every metric usually lives in shared memory or a shared storage tier, this one metric's runaway growth degrades ingestion and query latency for every other, well-behaved metric sharing that same node or shard, exactly the way one hot key degrades a shared cache cluster for every other key riding along with it.

The naive instinct, give the ingestion and index tier more memory so it can absorb a bigger cardinality spike, does not break this loop, it only raises the number of unique series at which the same underlying failure eventually recurs, because the underlying cause is not insufficient capacity, it is that nothing in the system asks whether a given label ought to exist as a metric dimension at all before accepting it. The senior fix, the one this lesson's cardinality-guard box in Section 3 names directly, is to budget and enforce cardinality at the point of ingestion: track how many distinct series each metric is generating, and refuse, quarantine, or loudly alert on a metric whose distinct-series count crosses a threshold, before that growth ever reaches the shared index. The second half of the same fix is a routing decision, not a capacity decision: identifiers that are inherently high-cardinality by nature, a user ID, a request ID, an ephemeral container name, belong in logs or distributed traces, systems explicitly built to index and query high-cardinality, individual-event data, not in a metric's label set, a data model built around the opposite assumption, that the space of distinct series for a given metric stays small and roughly fixed. Breaking the loop means refusing to let unbounded cardinality enter the metrics system in the first place, not building a bigger index to absorb it once it already has.

---

## Map to Rare.lab's stack

Rare.lab does not run anything resembling a metrics or observability platform at this lesson's scale today, and it is worth being precise about why the comparison is not yet apt: there is no fleet of hosts emitting millions of tagged counters a second, no dashboard querying the last 26 hours of anything, and building the machinery this lesson describes, an in-memory recency-scoped cache, a sharded inverted tag index, a cardinality guard, ahead of any actual telemetry volume would be exactly the premature-infrastructure mistake Day 62 and Day 63 both warned against for problems Rare.lab does not yet have either.

The gap that would open this lesson's exact failure mode is specific and near-term rather than hypothetical: if Rare.lab instruments its embeddable runtime, the one shared WebGL context serving many embedded shader players across many customer sites at once, with performance telemetry, frame time, shader compile time, draw calls, memory used per scene, the natural first instinct is to tag each of those measurements by the thing that produced them: a scene ID, a node ID inside the shader graph, an embed or customer ID. Every one of those is structurally the same unbounded label this lesson's Section 6 names as the trigger for a cardinality bomb, because the number of distinct scenes, nodes, and embeds a successful product accumulates grows without an obvious ceiling, exactly the way `user_id` does on a metric label. The concrete, actionable lesson worth banking now, before Rare.lab ships that telemetry, is a design constraint rather than new infrastructure: keep metric labels to a small, bounded set of dimensions that describe the fixed shape of the pipeline itself, which rendering stage a stat is about, which GPU tier, which runtime version, and route the high-cardinality identifiers, the specific scene, node, or embed ID, into structured logs or traces instead, where they belong and where they are cheap to store per event rather than expensive to track forever as a distinct time series. The same discipline applies one layer down to Rare.lab's actual Supabase Postgres: a naive `INSERT INTO events` table logging one row per telemetry point, indexed by whatever columns are convenient today, is this lesson's Section 2 naive design in miniature, survivable at low volume, and worth remembering is not the design to still be running once that volume stops being low.

---

## Sources

- [Gorilla: A Fast, Scalable, In-Memory Time Series Database, Pelkonen et al., VLDB 2015](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf): primary source for Facebook's spring-2015 write rate (roughly 12 million points a second, more than 700 million a minute, over 1 trillion a day, across 2 billion unique time series), the 26-hour write-through cache design and the 85%-of-queries-want-recent-data observation that justifies it, the delta-of-delta timestamp and XOR value compression scheme, the reported 16-byte-to-1.37-byte average compression ratio, the reported compression breakdowns (96% of timestamps to 1 bit, 51%/30%/19% split for value compression), and the reported 70x-plus query latency improvement over the prior HBase-only design. Direct fetch to vldb.org was blocked by this session's network egress policy; every figure above is relayed via search-indexed summaries and secondary sources that themselves cite the paper's own reported numbers (including a survey paper hosted on arxiv.org and a technical blog summary), not a first-hand read of the PDF, and worth re-verifying directly.
- [M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus, Uber Engineering Blog](https://www.uber.com/en-us/blog/m3/): primary source for M3's aggregation rate (roughly 500 million metrics a second aggregated, roughly 20 million metrics a second persisted via quorum write to 3 replicas), the more-than-6.6-billion time series figure, and M3's configurable per-metric retention and resolution tiers, referenced in Section 3. Direct fetch to uber.com was blocked; relayed via search-indexed summary, worth re-verifying directly.
- [The Billion Data Point Challenge: Building a Query Engine for High Cardinality Time Series Data, Uber Engineering Blog](https://www.uber.com/en-us/blog/billion-data-point-challenge/): secondary corroborating source for M3's query-engine scale (roughly 2,500 queries a second, roughly 8.5 billion data points a second scanned, as of the post's November 2018 disclosure), used only to establish that M3 operates at a comparable order of magnitude to Gorilla's own numbers, not quoted as a precise figure in this lesson's body. Direct fetch was blocked; relayed via search-indexed summary.
- [Timeseries indexing at scale, Datadog Engineering Blog](https://www.datadoghq.com/blog/engineering/timeseries-indexing-at-scale/): primary source, fetched directly in this session, for Datadog's original three-RocksDB-store indexing design and its 7-or-more-lookup cost for unindexed queries, the rewritten universal inverted tag index and its consistent low-lookup-count query cost, the 8-shards-per-32-core-node intranode sharding scheme and its roughly 8x throughput improvement, the Go-to-Rust migration and its benchmarked performance gains, the 30x data volume growth between 2017 and 2022, and the reported 20x cardinality improvement, 99% reduction in query timeouts, and 50% indexing cost reduction, all referenced in Section 3, Section 4, and Section 5.
- Prometheus documentation and the broader Prometheus operator ecosystem's repeated, independently written coverage of "cardinality explosion" and the "cardinality bomb" (including operator-facing writeups from multiple independent monitoring vendors covering the same failure mode in their own words): secondary, aggregated source for the specific claim that a single new high-cardinality label can multiply a metric's active series count by millions overnight, and for the general guidance to keep a single instance's total active series count in the tens of millions, referenced in Section 2 and used as this lesson's forcing example for Section 6. No single primary Prometheus documentation page was fetched directly in this session, direct fetch to prometheus.io was blocked; this is relayed via search-indexed summary of multiple independent, converging secondary sources rather than one authoritative citation, and is flagged as lower authority than the three primary engineering sources above, the same caveat this ledger's Day 62 and Day 63 lessons applied to their own lower-authority sources.

---

*Inference vs. fact, stated plainly: Facebook's write-rate figures, the 26-hour cache window and the 85% recent-query observation, the compression scheme and its reported ratios, and the reported query-latency improvement are all documented claims from Gorilla's own published VLDB paper, but that paper was relayed through this session's web search and secondary summaries of it rather than a first-hand read of the PDF, because direct fetch to vldb.org was blocked by this session's network egress policy; these figures were not independently re-verified against the primary text and should be treated as worth confirming directly. Uber's M3 aggregation and persistence rates, its time-series count, and its query-engine scale figures come from Uber's own named engineering blog posts as relayed through search, not a first-hand read, for the same network-policy reason. Datadog's original and rewritten indexing architecture, its sharding scheme, its Go-to-Rust migration benchmarks, and its reported performance improvements were fetched directly from Datadog's own engineering blog in this session and are treated as a confirmed primary source. The cardinality-explosion failure mode and the tens-of-millions active-series guidance are drawn from Prometheus's documentation and a wide, converging body of independent operator and vendor writeups relayed through search rather than one single authoritative source fetched directly, and are flagged as lower authority accordingly. The naive relational-table design in Section 2, its specific throughput reasoning (the tens-of-thousands-of-writes-a-second order-of-magnitude estimate for a single durable relational primary), the architecture diagram's exact layering and analogies, the cardinality-bomb feedback-loop framing in Section 6, and the entire Rare.lab mapping in the final section are this lesson's own synthesis built on top of the documented mechanics above, not a claim that Facebook, Uber, Datadog, or Prometheus describe their systems or the general failure mode in these exact terms.*
