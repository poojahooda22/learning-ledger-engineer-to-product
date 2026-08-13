# Day 53 — How do you store a metric with a million different label combinations without one bad label taking down the whole monitoring system?

**Date:** 2026-08-13
**Difficulty:** Expert
**Topic:** High-cardinality time-series storage: why "just add a label" is the most dangerous line of code in an observability stack, why a single Prometheus process cannot hold a company's metrics past a certain point no matter how big the box is, and how Uber's M3, Facebook's Gorilla, and Grafana Mimir each split "write fast, in memory" from "store cheap, forever" to survive both normal traffic and the moment someone accidentally turns one metric into five million.
**Stack relevance:** Rare.lab's embeddable runtime shares one WebGL context per session and will need telemetry: per-node compile time, per-shader frame cost, per-session render health, across every concurrent session. The exact mistake this lesson opens with, labeling a metric by something with unbounded cardinality, has a direct Rare.lab shape: tagging a frame-time metric by `node_id` or `session_id` instead of `node_type` or `shader_hash` turns one gauge into millions of independent series the moment usage grows, and it will not fail loudly the day it's added. It fails quietly, for weeks, until an ingester falls over.

---

## 1. The company and the breaking number

**Uber**, in its own engineering blog posts introducing **M3**, the open-source metrics platform it built after outgrowing single-node Prometheus and Graphite. Uber's own published numbers: M3 aggregates **500 million metrics per second** company-wide and persists **20 million resulting metrics-per-second** to its storage layer, M3DB, globally, using a **quorum write that copies each metric to three replicas per region**. A later post on M3's query engine, "The Billion Data Point Challenge," describes the read side returning up to **8.5 billion data points per second** across the fleet, and the platform as a whole holding on the order of **6.6 billion active time series**. These are Uber's own reported figures, not a third-party estimate.

**Facebook**, in Pelkonen et al.'s 2015 VLDB paper, *"Gorilla: A Fast, Scalable, In-Memory Time Series Database,"* gave the field its founding numbers on the same problem a decade earlier at a smaller but still enormous scale: as of spring 2015, Facebook's monitoring generated **more than 2 billion unique time series** with **about 12 million data points added per second**, and the team found that a widely used disk-backed store (HBase) simply could not serve dashboard queries fast enough at that volume, motivating an in-memory design with a custom compression scheme that cut per-point storage by roughly **10 to 12x**.

Both of those are "we built for this on purpose" numbers. The number that actually breaks a naive design is smaller, sneakier, and shows up in almost every postmortem written about this problem rather than in a keynote: **each active time series costs roughly 3 to 4 KB of RAM** in a standard in-memory time-series index (this is the widely documented cost for Prometheus's in-memory head block, reported consistently across multiple production postmortems and vendor writeups). A million active series is already 4 to 6 GB of RAM just to hold the index, before a single query runs. And cardinality, the count of distinct label combinations a metric can take, multiplies fast and silently: a single engineer adding one label with unbounded values, a `user_id`, a `request_id`, a `trace_id`, to an existing counter turns that one metric into as many time series as there are distinct values of that label. One widely cited real-world account describes exactly this: a request counter labeled with `user_id` grew to **over five million time series after 90 days and a million users**, from a single line of instrumentation code that looked completely reasonable in code review. Nothing crashes on day one. The process just gets a little bigger every day until, weeks later, it doesn't come back up.

## 2. Why the naive (demo) design dies

**Version one: one Prometheus (or one time-series database instance) scrapes every target and holds every series in memory.** This is the version every monitoring setup starts as, and for a genuinely small system it is correct and sufficient. It fails on three separate axes once traffic and label variety grow.

**Memory is proportional to active series, not to write volume, and series count is attacker-shaped, not traffic-shaped.** A well-behaved metric (say, `http_requests_total` labeled only by `route` and `status_code`) might have a few hundred distinct combinations, forever. A poorly labeled metric (the same counter, but also labeled by `user_id`) has as many combinations as there are users, and that number only grows. Prometheus's own documentation and multiple production postmortems describe exactly this failure: the in-memory head block allocates a new resident series the instant a new label combination is scraped, at roughly 3 to 4 KB apiece, and there is no natural ceiling. One process can be scraping the same total request rate on day one and day ninety, and still go from a healthy 1 million series to an OOM-killed 40+ million series in between, purely because of what the labels happened to contain. Traffic volume does not predict this failure; label design does, and label design is not something a fixed-size box can defend itself against.

**A single writer cannot both ingest at write-time speed and stay queryable at read-time speed.** Every sample landing on one process has to be indexed, in the same process, at the same time queries are trying to scan that index for a dashboard or an alert rule. Under load, the write path and the read path compete for the same CPU, the same locks, the same memory bandwidth, on the same machine. There is no room to isolate "keep ingesting" from "keep answering," so a spike in either one degrades the other. This is the same shape as this ledger's earlier feed-ranking and search-ranking lessons: a single machine trying to be authoritative for both writes and reads runs out of headroom long before either workload alone would.

**A crashed single instance is a total, synchronous loss of the most recent window, not a graceful degradation.** Prometheus by design keeps the last chunk of data (the head block, typically the last couple of hours) unflushed to disk in memory for write efficiency. If that one process OOMs, the most recent, most operationally relevant window, exactly the data an on-call engineer needs to diagnose the incident that is probably happening right now, is gone or unavailable until the process restarts and rebuilds. The naive design's single point of storage is also a single point of failure for the data you need most urgently.

## 3. The architecture

```
[Clients: application processes, exposing counters, gauges, and
 histograms via a metrics client library, e.g. one that tags every
 emitted sample with a small, BOUNDED set of labels]
   analogy: a factory floor where every machine reports its own
   gauge readings on a fixed dial, not a free-text field
   |
   v
[Edge / collection tier: a local scrape agent or push-based collector
 (Prometheus server per shard, or an OpenTelemetry Collector) pulls or
 receives raw samples close to the source, and does the FIRST cardinality
 check here, before anything travels further]
   analogy: a mail sorting office at the source town, rejecting
   obviously malformed envelopes before they ever enter the network
   |
   v
[Stateless distributor tier: validates each sample, checks it against a
 per-tenant / per-metric series limit, and hashes each series' label
 set to decide which shard owns it — this tier holds NO state itself
 and can scale out horizontally by just adding more of it]
   analogy: a ticket window that checks your form is filled in
   correctly and tells you which counter to queue at, but keeps
   no record itself
   |
   v
[Async ingest queue (e.g. Kafka, in Grafana Mimir's ingest-storage
 architecture): decouples "accepted the write" from "durably indexed
 the write," so a slow or restarting ingester does not block acceptance]
   analogy: a loading dock with a conveyor belt — trucks (writes) drop
   off crates the instant they arrive, whether or not the warehouse
   worker at the other end is ready that exact second
   |
   v
[In-memory write cache / ingester tier: each series is owned by one
 shard (consistent-hash of its label set), held in a compressed
 in-memory structure (Gorilla-style delta-of-delta timestamps + XOR'd
 float deltas, cutting per-point cost roughly 10x), and replicated
 with a quorum write to 3 replicas before being acknowledged]
   analogy: a whiteboard per department that three people can see and
   update at once, so losing any one person doesn't lose the board
   |
   v
[Compaction / flush to immutable blocks: every couple of hours, each
 ingester's in-memory series are compacted into an immutable on-disk
 (or object-storage) block, and a background compactor later merges
 overlapping blocks from replicas into fewer, larger, deduplicated ones]
   analogy: closing the books at the end of each shift and filing a
   permanent, unchangeable ledger page instead of leaving the day's
   whiteboard as the only copy
   |
   v
[Cold tier: cheap, durable object storage (S3-class), holding the vast
 majority of historical data as immutable blocks, queried directly by
 a stateless query tier rather than kept hot in memory]
   analogy: a warehouse archive, cheap per square foot, slower to
   retrieve from, used for anything older than "right now"
   |
   v
[Query tier + inverted index: queries resolve label matchers (exact,
 regex, AND/OR/NOT) through a per-shard inverted index mapping label
 values to series IDs, fetch only the matching series' blocks, and
 evaluate functions lazily, only materializing the data actually needed
 for the requested time range]
   analogy: a library card catalog that tells you exactly which shelf
   and volume to pull, instead of reading every book in the building
   |
   v
[Load shedding / admission control: per-tenant rate limits and hard
 series-count ceilings enforced back at the distributor tier, so one
 team's cardinality mistake is rejected or truncated at the edge
 instead of being allowed to consume shared ingester memory]
   analogy: a bouncer checking capacity at the door, not a fire
   marshal discovering overcrowding after the room is already full
```

Two structural choices are doing the real work here, and they are worth separating from the box diagram.

**Hot and cold are different systems with different guarantees, not one system with two speeds.** The last few hours live in replicated, compressed, in-memory series because that is what write and alerting latency need. Everything older lives as immutable blocks on cheap object storage because that is what a company's retention budget can actually afford at billions of series. M3DB's own documentation is explicit that it treats flushed data as immutable specifically so it does not have to pay the ongoing compaction-rewrite tax that systems like Cassandra pay for mutable storage; once a block is written, it is written, and merging only ever produces new blocks, never rewrites old ones in place.

**The inverted index, not a full scan, is what makes "give me every series where `route="checkout"` and `status_code=~"5.."`" fast at billions of series.** M3DB documents an inverted index per shard, mapping each distinct label value to the set of series IDs that have it, supporting exact-match and regex queries on any label, combinable with AND, OR, and NOT. Without that index, answering a two-label query over 6.6 billion series means scanning 6.6 billion series; with it, the answer is the intersection of two much smaller posting lists, the same underlying idea as candidate fetch through an inverted index in the search-ranking lesson earlier in this ledger, just applied to labels instead of query terms.

## 4. The transferable mechanisms

**Bounded write cost via delta encoding.** Gorilla's compression, storing each timestamp as the delta-of-the-delta from the previous two points (usually a handful of bits, since scrape intervals are regular) and each value as an XOR against the previous value (usually mostly zero bits, since metric values change slowly sample to sample), cut Facebook's per-point storage roughly 10 to 12x and was the difference between fitting a day's data in memory or not. The general lesson: when consecutive values in a stream are usually similar, encode the *difference*, not the value, and the compression follows almost for free.

**Quorum replication for durability without a single point of failure.** Uber's write path acknowledges a metric only after it has reached a quorum of 3 replicas, the same primitive covered in this ledger's leaderless-replication lesson, applied here to metrics instead of a shopping cart. It buys the ability to lose one replica mid-write without losing the sample or blocking the writer.

**Consistent hashing to shard ownership, not a lookup table.** Each series' label set hashes to a shard; the distributor tier computes that hash statelessly rather than consulting a directory service on every write. This is the same sharding primitive from Day 10 of this ledger, applied to "which ingester owns this specific combination of labels" instead of "which database node owns this user."

**Immutable blocks plus background compaction instead of in-place rewrites.** Flushing to append-only, immutable blocks and merging duplicates in the background (rather than updating records in place) avoids the write-amplification tax that mutable storage engines pay, the same LSM-style trade-off from Day 21 of this ledger, here applied to time-series blocks instead of key-value rows.

**Cardinality limits enforced at the edge, as admission control, not as cleanup.** The fix for the runaway `user_id` label is not "buy the ingester more memory," it is rejecting or truncating a series the instant it would push a metric past its declared cardinality budget, at the distributor, before it ever reaches shared ingester memory. This is the same shape as rate limiting from Day 8: the cheapest place to stop a problem is the earliest point in the pipeline that has enough information to recognize it.

**Downsampling as a retention-cost lever.** Keeping full-resolution data for a bounded recent window and rolling older data up into coarser resolution (1-minute becomes 1-hour, becomes 1-day) is the metrics-specific version of Day 51's bounded-retention lesson: an unbounded promise ("keep every raw point forever") is a fundamentally more expensive system than a bounded one ("keep raw for 30 days, rollups after that"), and almost nobody actually needs the unbounded version for anything older than a season.

## 5. The trade-offs

**Write acceptance: availability over consistency, per sample.** A monitoring system that refuses to accept a metric because it cannot immediately guarantee every replica has it yet is worse than one that accepts fast and resolves replication in the background; a slightly-stale dashboard number costs nothing, a rejected write during an incident (exactly when observability matters most) costs the ability to see the incident. This mirrors the same choice this ledger's Amazon-cart lesson describes for "add to cart": accept now, reconcile durability asynchronously.

**Query freshness: near-real-time is good enough, exact-to-the-millisecond is not required.** Dashboards and alert rules tolerate the few seconds of lag between a sample landing on an ingester and it becoming queryable, because the decisions built on top of them (paging someone, scaling a fleet) operate on a timescale of minutes, not milliseconds. This is a deliberate choice to spend consistency budget elsewhere.

**Cardinality budget: never traded away for convenience.** Unlike write availability, cardinality limits are not softened under pressure. Letting one team's mislabeled metric exceed its budget "just this once" is exactly the failure mode from section 2; the system holds this line even when it means a dropped or truncated series rather than a silently unbounded one. This is the one place strictness beats flexibility.

**Cost vs. latency: a two-tier storage split, priced deliberately.** In-memory replicated storage for the last few hours is expensive per byte but necessary for write and alert latency; object storage for everything older is roughly one to two orders of magnitude cheaper per byte but slower to query and immutable once written. Uber, Facebook, and every Prometheus-at-scale operator all converge on the same split because the alternative, keeping everything hot, does not survive the arithmetic at billions of series; and the alternative, keeping everything cold, does not survive the alerting latency requirement.

## 6. The systems-thinking lens

The failure mode this architecture has to defend against by design, not just by adding capacity, is a **cascading OOM avalanche**, a specific shape of **metastable failure**: the system runs fine well below a threshold, a slow-building trigger pushes one component over that threshold, and the failure of that one component pushes its neighbors over the same threshold in turn, without ever needing a new external cause.

Picture the mechanism concretely. A `user_id` label quietly grows one metric's cardinality for 90 days. Nothing alerts on this, because "series count trending up slowly" does not look like an outage in progress, it looks like normal growth. Eventually one ingester holding a disproportionate share of that metric's shards crosses its memory ceiling and gets OOM-killed. The shards it owned do not vanish, they get rebalanced onto the surviving ingesters (the standard, correct response to a node loss in a sharded system). But those surviving ingesters were already carrying their own share of the same slow-growing cardinality problem, closer to their own ceiling than anyone realized, because the growth was happening company-wide, not just on the node that died first. The rebalance pushes them over their own thresholds too. Now two nodes are down, their shards rebalance onto fewer remaining healthy nodes, each of which absorbs an even larger share of an already-oversized dataset. This is the loop: **node failure causes rebalancing, rebalancing increases load on survivors, increased load causes more node failure.** It is self-reinforcing, and unlike a thundering herd, it does not need a retry storm or a stampede of clients to sustain itself, it needs nothing more than the cluster's own standard, correct failure-recovery behavior.

The naive response, provisioning bigger ingesters or more of them, does not break this loop, it only raises the cardinality threshold at which the avalanche starts, and the same silent, unbounded growth pattern will eventually find the new ceiling too. The senior fix breaks the loop's mechanics directly:

- **Hard per-tenant and per-metric cardinality limits, enforced at the distributor, before a write ever reaches an ingester.** A write that would push a metric over its declared series budget is rejected or the offending label is dropped, at the edge, so no single mislabeled metric can ever consume unbounded shared memory in the first place.
- **Circuit breakers on rebalance itself.** If a shard rebalance would push a target ingester over its own safe memory threshold, the right move is to refuse or throttle that rebalance and shed load elsewhere (or reject new writes for that shard) rather than accepting the transfer and guaranteeing the next OOM.
- **Alerting on the leading indicator, not the failure.** `prometheus_tsdb_head_series` (or the ingester-tier equivalent) climbing monotonically, rather than plateauing, is the signal to catch weeks before the OOM, not the OOM itself. A metastable failure's whole danger is that its buildup looks like nothing until the threshold is crossed; the fix is watching the slope, not just the current value.

The general principle: whenever a system's normal, correct recovery behavior (rebalance load off a failed node) is also the exact mechanism that spreads the failure to the next node, adding capacity only delays the collapse. Breaking the loop, capping the input that caused it and refusing to let recovery itself become a load amplifier, is what actually stops it from happening again.

---

## Sources

- [M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus, Uber Engineering Blog](https://www.uber.com/en-IN/blog/m3/)
- [The Billion Data Point Challenge: Building a Query Engine for High Cardinality Time Series Data, Uber Engineering Blog](https://eng.uber.com/billion-data-point-challenge/)
- [Gorilla: A Fast, Scalable, In-Memory Time Series Database, Pelkonen et al., VLDB 2015 (PDF)](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf)
- [Gorilla: A fast, scalable, in-memory time series database, summary, the morning paper](https://blog.acolyer.org/2016/05/03/gorilla-a-fast-scalable-in-memory-time-series-database/)
- [M3DB, a distributed time series database, official docs](https://m3db.io/docs/architecture/m3db/)
- [Storage Engine, M3 Documentation](https://m3db.io/docs/architecture/m3db/engine/)
- [About ingest storage architecture, Grafana Mimir documentation](https://grafana.com/docs/mimir/latest/get-started/about-grafana-mimir-architecture/about-ingest-storage-architecture/)
- [How Cloudflare runs Prometheus at scale, Cloudflare Blog](https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/)
- [The Prometheus Cardinality Bomb: Causes, Impact & How to Fix It, OpenObserve](https://openobserve.ai/blog/prometheus-data-cardinality/)
- [Troubleshooting Common Prometheus Issues: Cardinality & More, Last9](https://last9.io/blog/troubleshooting-common-prometheus-pitfalls-cardinality-resource-utilization-and-storage-challenges/)

---

*Inference vs. fact, stated plainly: Uber's M3 numbers (500M metrics/sec aggregated, 20M metrics/sec persisted, quorum writes to 3 replicas, 6.6 billion series, 8.5 billion data points/sec read) and Facebook's Gorilla numbers (2 billion series, 12M points/sec, roughly 10 to 12x compression, HBase as the rejected baseline) are drawn directly from Uber's and Facebook's own published engineering blog posts and the peer-reviewed VLDB paper. The per-series memory cost (roughly 3 to 4 KB) and the "user_id label grew to 5 million series in 90 days" account are drawn from third-party production postmortems and vendor writeups documenting the widely observed pattern, not from a single named company's own incident report, and are labeled here as a well-documented pattern rather than a specific company's disclosed outage. The cascading-OOM-avalanche mechanism and its framing as a metastable failure are this lesson's reasoned inference about how a sharded ingester tier with standard rebalance-on-failure behavior would have to fail under slow cardinality growth, built from how these systems are documented to rebalance, not a quoted incident report from Uber, Meta, or Cloudflare specifically.*
