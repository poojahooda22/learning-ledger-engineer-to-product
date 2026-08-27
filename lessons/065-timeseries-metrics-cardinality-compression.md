# Day 65 — How does Facebook ingest 12 million monitoring data points a second, when one careless label on one counter can multiply that counter into 12 million distinct time series overnight?

**Date:** 2026-08-27
**Difficulty:** Expert
**Topic:** Time-series metrics at scale: the two-sided breaking problem of monitoring infrastructure, where the system has to survive both a volume axis (how many points arrive per second) and a cardinality axis (how many distinct series those points belong to), and where the second axis is the one that actually takes production monitoring down, because it is invisible until it isn't. The forcing example is Facebook's own disclosed numbers from the Gorilla paper (Pelkonen et al., VLDB 2015): as of spring 2015, Facebook's monitoring systems held more than 2 billion unique time series and ingested roughly 12 million data points per second, over 1 trillion points a day, and the HBase-backed system Gorilla replaced could not answer dashboard queries fast enough during the exact moments, live incidents, when speed mattered most. Layered on top of Facebook's volume number is a second, independently documented number from the Prometheus and observability engineering world: a single HTTP request counter carrying three ordinary labels (method, status code, a few dozen endpoint paths) sits at roughly 1,200 distinct series, and adding one more label most engineers reach for instinctively, a customer or user ID with 10,000 distinct values, turns that same one counter into 12 million series, from one line of instrumentation code, with no error, no warning, and no gate stopping it. Why a naive row-per-point relational design dies on both axes independently, and why the real fix is not one mechanism but three working together: compress the data itself so volume stops being the bottleneck, cap cardinality at the door so it never becomes unbounded, and downsample aging data so cost stays flat while retention grows.
**Stack relevance:** Rare.lab does not run a metrics pipeline at this scale today, and there is no dashboard anywhere in the stack currently ingesting millions of points a second, so this lesson's headline numbers do not transfer directly, the same discipline Day 63 and Day 64 applied to their own not-yet-true problems. What transfers ahead of need is the shape of the mistake: the moment Rare.lab instruments its embeddable runtime, the one shared WebGL context running inside other people's pages, to report client-side performance (frame time, dropped frames, shader compile time) back for monitoring, the natural first instinct is to tag every point with the identifying IDs at hand: a raw shader ID, a session ID, an embed's origin domain. That is precisely this lesson's cardinality mistake, one counter turned into millions of series by the IDs that feel most useful to keep. The fix has to be decided before the first line of instrumentation code ships, not after a shader goes viral on a public gallery page and its ID starts appearing in ten thousand concurrent runtime sessions at once.

---

## 1. The company and the breaking number

**Facebook, Gorilla, spring 2015.** The VLDB paper "Gorilla: A Fast, Scalable, In-Memory Time Series Database" (Pelkonen, Franklin, Teller, Cavallaro, Huang, Meza, Veeraraghavan, Proceedings of the VLDB Endowment, Vol 8, No 12, 2015) states plainly what Facebook's monitoring systems looked like at the time: more than 2 billion unique time series of counters, roughly 12 million data points added per second, which works out to over 1 trillion data points a day. That is the volume-axis number, and it is a real, disclosed figure, not an estimate.

Do the naive arithmetic on just that one number, ignoring cardinality for a moment. A time series point is, at minimum, a timestamp and a value, and a completely naive encoding, an 8-byte timestamp plus an 8-byte double, costs 16 bytes per point before any row, index, or replication overhead is added. At 12 million points a second, that is 192 MB/sec of raw values, roughly 16.6 TB/day. Add the row and index overhead a general-purpose store actually charges (a relational row header, a B-tree index entry, column-family metadata in something like HBase, commonly pushing effective cost well past 40-100 bytes/point in practice) and a standard 3x replication factor for durability, and the daily footprint for one day's writes alone lands in the hundreds of terabytes, for data that, in Gorilla's own stated access pattern, is queried almost entirely for the most recent 26 hours. The paper's own comparison makes the real failure concrete: Facebook already had monitoring data durably stored in HBase before Gorilla existed, and the problem was never that HBase lost data, it was that an HBase-backed read path could not serve monitoring dashboard queries fast enough during a live incident, exactly the moment an engineer staring at a graph needs an answer in under a second, not several. Gorilla's in-memory engine, sitting in front of that same HBase store as a write-through cache, cut query latency by a reported 73x and improved query throughput by 14x over the HBase-backed system it fronted.

**The second, sharper number: cardinality, not volume, is what actually detonates in production.** Independent, widely cited engineering explainers of Prometheus-style metrics systems describe the same arithmetic that plays out at any company running counters in production, not just Facebook's scale. A single HTTP request counter with three ordinary labels, an HTTP method (4 values), a status code (6 values), and roughly 50 distinct endpoint paths, already sits at about 1,200 distinct time series, one per unique label combination, from one counter. Add a fourth label that feels completely reasonable to an engineer trying to debug a specific customer's traffic, a `customer_id` with 10,000 distinct values, and that same single counter becomes 12 million distinct time series. Nobody approved that 10,000x jump. Nobody had to. It happened the moment the label was added to the code and the first request from a new customer arrived, because in a Prometheus-style system, a brand-new label combination allocates a brand-new independent in-memory time series immediately and unconditionally, with no quota check in the default path. This is the number this lesson is actually built around: the breaking point for time-series metrics is rarely "too many requests," it is one instrumentation decision away, on any given day, from turning one counter into a number that looks suspiciously like Facebook's entire fleet-wide count of unique series, in a system built to hold at most a small fraction of that.

---

## 2. Why the naive (demo) design dies

**The obvious version:** one relational table, one row per point, columns for `metric_name`, a JSON or delimited blob of label key-value pairs, `timestamp`, and `value`, with a compound index on `(metric_name, labels, timestamp)`. An application increments a counter or records a gauge, a client library issues an `INSERT`, and a dashboard queries with something like `SELECT timestamp, value FROM points WHERE metric_name = 'http_requests_total' AND labels @> '{"customer_id": "..."}' ORDER BY timestamp`. This is exactly how a metrics feature starts inside almost every application that eventually needs a real one: a Postgres table, because Postgres is already there, and it works fine for the first few weeks, because early on nobody has added a high-cardinality label yet and traffic is nowhere near the point where write throughput matters.

**Death one: write throughput on a single row-store collapses three orders of magnitude short of the target.** A single Postgres primary handling ordinary row inserts, with index maintenance on every write, tops out realistically in the tens of thousands of writes per second on capable hardware before latency and lock contention start climbing. Facebook's disclosed 12 million points a second is roughly 300-1,000x past that ceiling, and the gap does not close by adding a bigger box, because the bottleneck is B-tree index churn under constant insert pressure, the same fundamental limit this ledger's LSM-tree lesson (Day 21) named for any write-optimized workload forced through a structure tuned for point lookups instead of sequential appends.

**Death two: storage and memory cost blow up by roughly the same factor Gorilla's own compression closes.** The Gorilla paper reports its compression algorithm, delta-of-delta encoding for timestamps and XOR encoding for values, achieves an average 12x reduction versus a naive fixed-width encoding, landing at roughly 1.37 bytes per point instead of 16. The compression's own internal breakdown, as reported in the paper: about 51% of all values compress to a single bit, because a metric's current value is identical to its previous value more often than intuition suggests; about 30% compress to an average of 26.6 bits using an XOR-with-meaningful-bits scheme; the remaining roughly 19% of values, where the XOR result does not fall inside the previous value's leading/trailing zero window, cost an average of 36.9 bits. A naive design that stores full-width timestamps and values gets none of this, which means at Facebook's scale the difference between the naive encoding and Gorilla's is the difference between an in-memory hot tier that fits in a reasonable fleet of machines and one that does not fit in memory at all, forcing every query back onto the slow, disk-backed path this system exists to avoid.

**Death three: cardinality explosion turns a bounded schema into an unbounded one, and it takes down more than the offending metric.** In the naive table, and in every real time-series system built the same way, a new label combination is not a capacity-planning decision, it is a side effect of a line of application code. The Prometheus-world example from Section 1 is the general case: nothing in the naive write path asks "should this counter be allowed to have 12 million distinct series," it simply allocates memory for series number 12,000,001 the instant a request from a new `customer_id` arrives. On a shared ingestion cluster, independent, widely documented operational post-mortems of Prometheus-style deployments describe this exact failure: one team's careless label (a raw customer ID, a pod's ephemeral hostname suffix, a full request path with embedded IDs) exhausts memory on the shared ingestion tier, and every other team's dashboards and alerts go dark with it, a blast radius that has nothing to do with how much traffic the offending team's own service actually receives.

**Death four: even a design that survives ingestion is laid out wrong for the read pattern that matters.** The query a monitoring dashboard actually issues during an incident is not "fetch one point by primary key," it is "fetch the last few hours of 50 related series, at a coarsened resolution, ranked and rendered, in well under a second, while an engineer is staring at a blank graph waiting." A row-store indexed for point lookups answers that with either a large, unbounded range scan across an index that was never laid out contiguously by time, or a query planner falling back to something close to a full scan, precisely the 73x latency gap Gorilla's own benchmark against its HBase-backed predecessor measured.

---

## 3. The architecture

```
Instrumented hosts and services (the client)
  - application code increments counters, records gauges and
    histograms, using a metrics client library, not a raw database
    write
  - job: emit a measurement, cheaply, without blocking the request
    path it is instrumenting
  - analogy: a factory worker jotting a tally mark on a clipboard,
    not stopping the line to phone head office every time

        |  in-process
        v
Local pre-aggregation buffer (client-side, per host)
  - batches and locally aggregates points over a short window
    (commonly seconds to a minute) before anything leaves the host:
    sums counters, computes histogram buckets, keeps only the
    aggregate, not every individual event
  - job: turn "one network call per event" into "one network call
    per window," cutting the volume that ever reaches the network by
    orders of magnitude before compression even enters the picture
  - analogy: a cashier ringing up a whole basket once at checkout,
    instead of phoning the warehouse after scanning every single item

        |  batched, once per window
        v
Edge collector / ingestion tier (stateless, horizontally scaled)
  - receives batched points from every host, stateless so any
    instance can take any host's traffic and the tier scales out
    under load the same way this ledger's stateless booking tier
    (Day 64) or aggregator tier (Day 61) does
  - job: the first point in the pipeline that can see a metric's full
    label set and decide whether to accept it
  - analogy: a loading dock that weighs every truck before it's
    allowed onto the warehouse floor

        |  accepted points only
        v
Cardinality-limiting gate
  - tracks, per metric, an approximate count of how many distinct
    label combinations (series) currently exist, using a cheap
    probabilistic estimator (this ledger's Day 28 HyperLogLog is the
    load-bearing primitive here) rather than an exact, expensive count
  - rejects or drops new series once a metric crosses its quota,
    and, critically, alerts the owning team instead of silently
    degrading everyone sharing the ingestion cluster
  - job: make cardinality a bounded, enforced resource, the same way
    a rate limiter (Day 8) makes request volume a bounded, enforced
    resource, instead of an unbounded side effect of application code
  - analogy: a fire marshal capping how many people a room can hold,
    checked at the door, not discovered when the floor gives way

        |  bounded cardinality
        v
Durable ingestion log (short-lived buffer, not the store of record)
  - a write-ahead-style durable queue absorbing bursts and giving the
    compression/write tier a chance to fall behind briefly without
    losing data, the same shock-absorber role this ledger's Day 9
    queue lesson and Day 17 WAL lesson both named
  - job: decouple "how fast points arrive" from "how fast the
    compression engine can commit them"
  - analogy: an inbox that lets mail pile up for an hour without
    losing a single letter, even if nobody's reading it that hour

        |  drained by the write engine
        v
In-memory compression and write engine (Gorilla-shaped)
  - encodes timestamps with delta-of-delta compression (most
    intervals between points are identical to the last interval,
    so most of the cost is near zero) and values with XOR compression
    (most consecutive values differ in only a few bits), landing
    near the paper's reported 1.37 bytes/point average, a 12x
    reduction over a naive 16-byte encoding
  - holds the most recent window (26 hours, in Gorilla's own
    disclosed design) fully in memory, because that is where nearly
    all monitoring queries land
  - job: make the hot, most-queried window of data cheap enough in
    memory that dashboard queries never have to touch a disk-backed
    store during the exact moments, live incidents, when speed is
    worth the most
  - analogy: keeping this week's paperwork in a desk drawer, not the
    filing warehouse across town, because that's the paperwork
    someone actually asks for

        |  aged out of the hot window
        v
Sharded durable storage (partitioned by metric/series key)
  - the store of record once data ages out of the in-memory tier,
    partitioned across nodes by a hash of the series key, the same
    consistent-hashing mechanism this ledger's Day 10 lesson named,
    so no single node becomes the write-hot target for one popular
    metric
  - job: hold everything durably, at a cost profile that makes sense
    for data queried far less often than the last 26 hours' worth
  - analogy: the filing warehouse itself, organized so no one filing
    cabinet has to hold every record in the building

        |  continuous background process
        v
Downsampling / rollup compactor
  - as raw points age past their hot retention window, a background
    process rewrites them at coarser resolution (1-minute averages,
    then hourly, then daily), the same aging-driven compaction this
    ledger's Day 21 LSM-tree lesson described for write-optimized
    storage generally, applied here to time resolution instead of
    key overwrites
  - job: keep storage cost roughly flat as retention grows, by
    spending precision, not correctness, as the currency
  - analogy: a photo library that keeps this month's shots at full
    resolution and automatically compresses last year's into a
    smaller thumbnail archive

        |  queried by
        v
Query engine + result cache, then dashboards and alerting
  - answers a query by reading from whichever tier the requested time
    range actually lives in (hot in-memory, cold durable, or a
    rollup), and caches hot dashboard queries the same way this
    ledger's Day 19 caching lesson described for any expensive,
    frequently repeated read
  - job: the one interface engineers and alerting rules actually talk
    to, hiding which storage tier answered the query
  - analogy: a single reference desk that knows which shelf, or which
    warehouse, holds the answer, so the visitor never has to know
```

---

## 4. The transferable mechanisms

- **Compress the domain, not just the bytes.** Gorilla's delta-of-delta timestamp encoding and XOR value encoding both exploit a structural fact about the specific data type, monitoring points arrive at roughly regular intervals and consecutive values are usually close to each other, rather than applying a generic compressor after the fact. The transferable idea is to look for the structural regularity in whatever domain you're storing, the same way this ledger's Day 23 content-addressed storage lesson exploited the fact that most of a large repository's content is unchanged between commits, and let that regularity, not a bigger machine, do the compression's work.

- **Cardinality limiting as an enforced quota, checked with an approximate structure.** Treating "how many distinct series does this metric have" as a bounded, gated resource, the same way a rate limiter treats request volume, is what stops one instrumentation mistake from becoming a fleet-wide outage. Doing the counting cheaply, with a HyperLogLog-style estimator (Day 28) instead of an exact count that itself becomes expensive at the scale it's protecting, is what makes the gate affordable to run on every single write.

- **Client-side pre-aggregation, batch before you ever transmit.** Reducing "one network call per event" to "one network call per aggregation window" cuts the volume the rest of the pipeline ever has to handle, before compression, sharding, or downsampling get involved at all. This is the same principle as batching writes anywhere a network hop is the expensive part of the operation.

- **Tiered storage aged by a compaction process, spending precision as the currency, not correctness.** Full resolution for a short hot window, progressively coarser rollups for everything older, is the same aging-driven trade-off this ledger's LSM-tree lesson (Day 21) named for compaction generally, applied to time resolution instead of key versions: the data that's cheap to answer with reduced fidelity gets reduced fidelity, and the data that's actually queried at full fidelity, the last day or so, is the only data that has to pay full storage cost.

- **Shard the hot resource by key, the same primitive as every prior sharding lesson.** Partitioning series across nodes by a hash of the series key (Day 10's consistent hashing) means one popular metric's write volume doesn't serialize against every other metric's writes on the same node, the identical mechanism this ledger has now applied to seat inventory (Day 64), usernames (Day 62), and now time-series keys.

- **In-memory hot tier as a write-through cache in front of durable storage, not a replacement for it.** Gorilla itself is explicitly described in its own paper as a cache: the durable HBase-backed store still exists underneath it. The lesson generalizes past monitoring: keep the expensive, low-latency tier only as large as the genuinely hot working set, and let a slower, cheaper, durable tier hold everything else, the same shape this ledger's Day 19 caching lesson argued for reads generally.

---

## 5. The trade-offs

**Consistency vs. availability, and the answer is different for the hot window than for the durable record, exactly the split this ledger has now drawn for a green dot (Day 63) and a queue position (Day 64).** Gorilla's own design explicitly accepts that a crash can lose the small window of the most recent, not-yet-durably-flushed in-memory data, and states this trade-off outright: losing a few seconds of data for a handful of hosts is an acceptable cost, because the alternative, refusing to serve queries at all while waiting for a fully durable write path on every point, would make the monitoring system less available at precisely the moments, active incidents, when its availability matters most. A monitoring system that is occasionally missing the last few seconds of one host's data is still useful. A monitoring system that is down is not.

**Cost vs. latency, paid as memory for the hot tier and disk for everything else.** Keeping the most recent roughly-26-hour window fully in memory, uncompressed relative to disk but heavily compressed relative to a naive encoding, is a deliberate, expensive choice, paid in RAM, specifically because that window is where the overwhelming majority of real queries land, live dashboards, active alerts, an engineer debugging the last hour. Data older than that window is downsampled and moved to cheaper, slower storage precisely because the query volume against it is a small fraction of the query volume against the hot window, so paying disk latency for it costs little in practice.

**Precision vs. storage cost, spent by the downsampling compactor on a schedule, not a per-query basis.** Rolling a day of raw points into an hourly average is a one-way trade: nobody can later ask "what was the exact value at 3:14:07am six months ago" once that resolution is gone. This is accepted because the overwhelming majority of long-range monitoring queries, "what did this look like over the last month," genuinely do not need second-level precision, and the storage saved by not keeping trillion-point-a-day resolution forever is the difference between a monitoring system's storage cost being flat over time and it being unbounded.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is **cardinality explosion as a metastable, shared-blast-radius failure**, structurally close to this ledger's hot-key lesson (Day 16) but running on a different axis: Day 16's hot key was one resource getting too many reads from one popular item; this loop is one metric getting too many distinct series from one careless label, and the damage lands on every other metric sharing the same ingestion memory, not just the offending one. It runs like this: an engineer adds a label to an existing counter because it looks genuinely useful for debugging, a customer ID, a pod name, a full URL path with an embedded identifier. Nothing in the naive write path objects. The first request carrying a new label value allocates a new, independent time series, in memory, on whichever ingestion node happens to receive it. Traffic for that service continues normally, nothing looks wrong from the outside, until enough distinct label values have appeared that the ingestion node's memory is dominated by series nobody is actually querying, at which point that node runs out of memory, and every metric it was holding, for every team, becomes unavailable at once, the exact shared-tenancy collapse independent operational write-ups of Prometheus-style deployments describe as the standard cardinality-explosion failure mode.

The naive fix, give the ingestion tier more memory, does not break this loop, it only raises the threshold at which the same unbounded-growth mechanism detonates again, the same way this ledger has now said about a retry storm (Day 13, Day 63, Day 64) that adding servers does not fix a loop whose root cause is unbounded demand on a shared resource. The senior fix is the cardinality-limiting gate from Section 3: enforce a hard, per-metric quota on distinct series, checked cheaply with an approximate cardinality estimator so the check itself stays affordable at ingestion volume, and when a metric crosses its quota, reject the new series and alert the team that owns it, rather than silently degrading the shared cluster for everyone. That is the general shape this ledger keeps rediscovering under a new name each time: the failure that actually takes a shared system down is rarely raw volume on its own, it is an unbounded, ungated dimension of growth, whether that dimension is retries (Day 13), reconnects (Day 63), or, here, the distinct label combinations one team's code is allowed to create without anyone's approval.

---

## Sources

- [Gorilla: A Fast, Scalable, In-Memory Time Series Database, Pelkonen et al., Proceedings of the VLDB Endowment, Vol 8, No 12 (2015), PDF](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf): the primary source for Facebook's 2015 scale numbers (2 billion+ unique time series, ~12 million points/sec, over 1 trillion points/day), the compression algorithm (delta-of-delta timestamps, XOR-encoded values, ~1.37 bytes/point average, ~12x reduction), the compression's internal bit-cost breakdown, the 26-hour in-memory hot window, the explicit trade-off of accepting a small window of data loss on crash in exchange for availability, and the 73x query latency / 14x throughput improvement over the HBase-backed predecessor.
- [Gorilla: a fast, scalable, in-memory time series database, summarized on "the morning paper"](https://blog.acolyer.org/2016/05/03/gorilla-a-fast-scalable-in-memory-time-series-database/): a secondary, accessible summary of the same paper, used to cross-check the plain-language framing of the architecture and its design goals.
- [M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus, Uber Engineering Blog](https://www.uber.com/us/en/blog/m3/): source for M3's disclosed scale, over 6.6 billion time series held, aggregating roughly 500 million metrics/sec and persisting roughly 20 million metrics/sec to storage, used as a second real company's numbers corroborating that Facebook's order of magnitude is a recurring one at this class of company, not a one-off.
- [The Billion Data Point Challenge: Building a Query Engine for High Cardinality Time Series Data, Uber Engineering Blog](https://eng.uber.com/billion-data-point-challenge/): source for Uber's own account of cardinality, not raw storage volume, being the practical limiting factor on how much of a metrics store can be queried at once, corroborating this lesson's central claim that cardinality, not volume, is the axis that actually breaks systems in production.
- [Timeseries indexing at scale, Datadog Engineering Blog](https://www.datadoghq.com/blog/engineering/timeseries-indexing-at-scale/): used for context on how a comparably-scaled commercial metrics platform (Datadog, architected toward quadrillions of points/day) approaches indexing and storage for time-series data at a volume beyond Facebook's 2015 disclosed figures.
- [How to Manage High Cardinality Metrics in Prometheus, Last9](https://last9.io/blog/how-to-manage-high-cardinality-metrics-in-prometheus/): the source for the worked cardinality-explosion example used throughout this lesson (a 1,200-series HTTP counter becoming 12 million series after one `customer_id` label is added), and for the description of Prometheus's default behavior of allocating a new in-memory series unconditionally on first appearance of a new label combination. This is an engineering explainer, not a primary Prometheus source document, and is flagged here as the clearly labeled industry-pattern version of the claim, not a company-disclosed figure the way the Facebook and Uber numbers above are.

---
