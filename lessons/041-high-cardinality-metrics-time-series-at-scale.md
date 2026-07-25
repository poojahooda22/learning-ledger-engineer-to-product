# Day 41 — How do you store 12 million metric points a second without the monitoring system becoming the outage?

*2026-07-25*

---

## 1. The company and the number that breaks a naive design

**Facebook, 2015, the Gorilla in-memory time series database.** By the spring of that year Facebook's internal monitoring generated more than **2 billion unique time series**, each identified by a string key such as `service.host.metric`, taking in an insertion rate of roughly **700 million data points a minute, about 12 million a second**. Every one of those points is a timestamp plus a floating-point value, 16 bytes raw. Engineers needed to query the last 26 hours of any of those series and get an answer in low milliseconds, specifically because the moment they need that dashboard most is the middle of an active incident, not a quiet Tuesday afternoon.

The number that breaks a naive design is not only the 12 million points a second of steady-state traffic. It is that **the number of distinct time series can multiply overnight for reasons that have nothing to do with user growth.** A single engineer adding one new label to an existing counter, for example tagging a per-request counter with a user ID or a request ID to make debugging easier, does not add traffic. It multiplies the counter into one new time series per distinct label value. One counter with a million distinct users behind it becomes a million physical time series, each needing its own place to live and its own entry in whatever index finds it. That growth is invisible in dashboards that only chart total request volume, and it keeps growing quietly for days or weeks before it crosses a wall.

## 2. Why the naive design dies

The naive version: write every incoming point straight into the durable store you already have, one row per point, keyed by series name and timestamp, and build whatever index you need on top to make queries fast.

This collapses in three concrete ways at Facebook's and any similar company's scale.

**a. Direct-to-disk random writes cannot absorb 12 million points a second and stay queryable.** Sixteen bytes times 12 million is roughly 192MB a second of new raw data, before replication and before any index maintenance, and that write pattern lands across an enormous, effectively random key space of series identifiers. A disk-backed store tuned for durable long-term storage (Facebook's chosen backing store was HBase) is good at sequential, batched writes and range scans over a small number of keys. It is bad at exactly the read pattern an incident dashboard needs: a fast range scan over the last few minutes, across thousands of series, answered in milliseconds while someone is actively debugging.

**b. Cardinality growth is invisible until it is catastrophic.** A widely documented, generic version of this failure runs like this, illustrative rather than tied to one named outage: a deploy quietly adds a request-scoped label to a counter, each new distinct label value creates a new time series, and over ninety days and a million users that one label alone can generate millions of new series. Nothing alerts on this because request volume looks flat. Then the in-memory index and working set needed to track all those series finally exceeds available memory, the collector gets killed by the operating system's out-of-memory killer, and it re-enters a long replay of its write-ahead log on restart. Dashboards go blank. Alerts go silent. Nobody gets paged that the paging system is down.

**c. The tool built to tell you production is broken becomes the thing that is broken, at the worst possible moment.** Incidents and cardinality spikes are correlated, not independent: a bad deploy that adds a label is often the same deploy that also destabilizes the service it instruments, so the monitoring system tends to fail exactly when it is needed most, not at some unrelated quiet moment.

The analogy: a filing room that keeps one index card per event, filed by exact arrival time, on ever-growing shelves. That works at a hundred cards a day. At 12 million cards an hour it is already straining. Now imagine a clerk decides, with good intentions, to also start filing a separate card for every visitor's ID number. The number of shelves needed did not grow with the events anymore. It grew with something else entirely, silently, until the room ran out of floor with no warning.

## 3. The architecture, top to bottom

```
Clients (a counter increment or a histogram observation inside
         one running process, e.g. one web server handling requests)
   |  each metric is a (name, label set, value, timestamp) tuple
   v
Local pre-aggregation (a statsd-style agent on the same host, or
                        Uber's M3 Aggregator tier)
      combine many raw increments on the same host into one rolled-
      up point per interval before anything crosses the network,
      the same combiner idea a MapReduce job uses locally before
      shuffling data across the network (Day 18's PageRank lesson)
   v
Stateless ingestion / routing tier
      shard incoming series by a hash of (metric name + sorted
      label set) using consistent hashing, so the same series
      always lands on the same downstream shard, Day 10's rule
      applied to metric keys instead of database rows
   |
   |-- Cardinality admission control (a per-tenant quota on
   |   distinct series; a label set that would push a tenant over
   |   its budget is rejected or the offending label is dropped
   |   at the door, not accepted and dealt with later)
   v
In-memory compressed write-back tier (Gorilla-style: recent data,
                                       e.g. the last 26 hours, held
                                       fully in RAM across a fleet
                                       of machines)
      values compressed with XOR encoding, timestamps compressed
      with delta-of-delta encoding, both described in section 4;
      data is chunked into blocks aligned to a fixed interval
      (Gorilla used 2-hour blocks) so a query only has to open the
      blocks that actually cover its time range
   |  write-through: every point is also shipped onward
   v
Durable long-term store, tiered by age (HBase behind Gorilla;
                                          M3DB at Uber)
      recent data kept at full resolution, older data downsampled
      to coarser intervals as it ages (Uber's M3 lets a team
      declare a retention policy like "10 second resolution for 2
      days, 1 minute resolution for 6 months"), the same offline-
      think/online-lookup shape as Day 19's caching discipline
      applied to time instead of to reads
   v
Horizontally sharded query / federation tier (Cortex, Grafana
                                                Mimir, or Thanos in
                                                the open-source
                                                Prometheus world)
      fans a query out across many storage shards and time blocks
      in parallel and merges partial results, the same scatter-
      gather shape Day 18's search ranking lesson used for
      inverted index posting lists, now over compressed time
      series blocks instead of documents
   v
Dashboards and alerting
      read only the compressed, already-aggregated form; nobody
      ever re-derives a rollup from 12 million raw points at query
      time
```

The layer that actually makes this survive is the in-memory compressed tier sitting in front of the durable store. It exists because the two things a monitoring system needs, fast recent queries and cheap long-term storage, want opposite physical media, and no single tier is good at both at this rate.

## 4. The transferable mechanisms

**a. Delta-of-delta timestamp compression.** Most metrics scrape or report at a fixed interval, so the gap between consecutive timestamps is nearly constant. Gorilla stores the delta of that gap (this interval's delta minus the previous interval's delta) rather than the raw timestamp, and uses four control-bit prefixes depending on how much that second-order delta drifted: a single `0` bit when it is exactly zero (the common case, since intervals rarely change), then `10` plus 7 bits for a small drift in [-63, 64], `110` plus 9 bits for [-255, 256], `1110` plus 12 bits for [-2047, 2048], and `1111` plus a full 32-bit value for anything larger. Because real scrape intervals are almost always regular, about 96% of timestamps in Facebook's production data compressed down to that single bit.

**b. XOR compression for slowly changing floating-point values.** Many metrics (CPU percent, queue depth, latency) change gradually between consecutive samples, so their binary representations share most of their leading and trailing bits. Gorilla XORs each new value against the previous one and stores only what changed: a `0` bit if the XOR is zero (identical value), `10` plus just the meaningful bits if the nonzero region fits inside the same leading-zero and trailing-zero window as the previous point (so the window itself doesn't need re-encoding), or `11` plus a fresh 5-bit leading-zero count, a 6-bit length field, and the meaningful bits themselves when the window shifts. Combined, these two encodings took Gorilla's real 16-byte-per-point data down to an average of **1.37 bytes per point, a roughly 12x reduction**, small enough that Facebook's entire 26-hour window fit in **1.3TB of RAM spread across 20 machines**, and because RAM is that much faster than the disk-backed HBase store it replaced, Gorilla's own reported benchmarks showed roughly **73x lower query latency and 14x higher query throughput** for the same dashboard queries.

**c. Pre-aggregate at the edge, before data crosses the network.** Combining many raw events into one rolled-up point on the same host, before it is ever transmitted, cuts both network volume and the number of distinct series a central system has to track. This is the identical instinct behind a MapReduce combiner or a Pregel message combiner (Day 18): do the cheap, local reduction close to the source, and only ship the reduced result across the expensive network hop.

**d. Tiered retention as time-based caching.** Keep the last few hours or days at full precision in the expensive, fast tier, and downsample everything older into coarser, cheaper rollups. Uber's M3 makes this an explicit, declarable policy per metric (full resolution for two days, one-minute resolution for six months, and so on) rather than a single blanket retention setting, which is the same expensive-recent, cheap-durable split behind Day 19's caching lesson, applied along the time axis instead of the request axis.

**e. Shard by series key with consistent hashing.** Route each (metric name, label set) combination to the same shard every time via a hash, so adding capacity means adding shards, not buying a bigger single machine to hold an ever-growing key space. This is Day 10's rule, unchanged, just applied to metric identity instead of a user ID or a database row.

**f. Treat cardinality itself as a rate-limited resource.** The failure mode in section 2 is not a traffic problem, it is an unbounded-growth-in-dimensionality problem, and the fix is to budget it explicitly: cap the number of distinct series a single metric or tenant may create, and reject or drop the offending label at admission time rather than silently accepting it and hoping storage keeps up. This is Day 13's backpressure and Day 8's rate limiting, both applied to a dimension (how many distinct things exist) instead of a rate (how many requests per second arrive).

## 5. The trade-offs

**Consistency vs. availability, expressed as completeness vs. survivability.** A monitoring pipeline should almost always choose to drop a sample under load rather than block, retry indefinitely, or crash trying to accept it. A slightly incomplete dashboard for the last few seconds is a minor inconvenience; a monitoring system that falls over trying to guarantee it never loses a single data point takes down the very visibility an incident needs. This is metrics data leaning hard toward availability, the opposite lean from a payments ledger (Day 6), and the right choice specifically because a metric is a statistical signal, not a record of an individual transaction that must never be lost.

**Cost vs. latency, resolved by tiering on age rather than picking once.** RAM fast enough to answer "what happened 30 seconds ago" during an incident costs far more per byte than object storage or a disk-backed store, and keeping months of raw, full-resolution data in RAM for every metric is not affordable at any real company's scale. The resolution is not choosing one tier for everything, it is accepting that recent data is worth the expensive tier and old data is not, and moving data between them automatically as it ages.

**Cardinality vs. debuggability.** The exact labels that make debugging one specific user's problem easiest, a user ID, a request ID, a session token, are the most expensive possible dimensions to add to a time series metric, because each distinct value multiplies the series count. That tension is precisely why traces and logs exist as systems separate from metrics (Day 39's distributed tracing lesson covers the trace side of this split): high-cardinality, per-request detail belongs in a system built to index and sample individual events, not in a time series database built to compress and aggregate a bounded, known set of dimensions.

## 6. The systems-thinking lens

**Cardinality explosion is a metastable failure that grows silently before it fails all at once.** Unlike a traffic spike, which shows up immediately in an obvious QPS graph, an unbounded label change produces no visible symptom in the metrics that teams normally watch (request volume, error rate) while the series count climbs underneath, sometimes for weeks. It only becomes visible the instant memory or an index structure crosses a hard limit, and because that failure tends to correlate with the same deploy or feature rollout that introduced the label in the first place, the collapse often lands during exactly the period of change and risk when observability matters most. The system fails quietly for a long time, then fails loudly and completely, with no gradual warning in between.

**The senior fix enforces a budget at admission time, it doesn't add more memory and hope.** More RAM, a bigger cluster, or a beefier index only delays the same wall, because the underlying growth is unbounded and nothing about adding capacity changes that. The structural fix is the same instinct as Day 13's backpressure lesson, applied one layer earlier: refuse to accept unbounded growth in the first place, by capping distinct series per tenant or per metric and rejecting or dropping what exceeds that cap at the door, and by routing genuinely high-cardinality data to a system designed for it (traces, logs) instead of trying to make the metrics store big enough to survive anything anyone ever sends it. Breaking the loop means making the failure mode "a rejected label, logged once" instead of "an unbounded resource that eventually takes the whole system down."

---

## References and summaries

**Pelkonen, T. et al. (Facebook). "Gorilla: A Fast, Scalable, In-Memory Time Series Database." Proceedings of the VLDB Endowment, Vol. 8, No. 12, 2015.**
https://www.vldb.org/pvldb/vol8/p1816-teller.pdf
Primary source for this lesson's headline numbers: 2 billion unique time series as of spring 2015, an insertion rate of roughly 700 million data points a minute (about 12 million a second), a 26-hour in-memory retention window, 1.3TB of RAM spread across 20 machines for that window, an average compressed size of 1.37 bytes per point (a roughly 12x reduction from 16 bytes raw), and reported query performance of roughly 73x lower latency and 14x higher throughput compared to the prior HBase-backed system. Also the source for the two-tier design, Gorilla as an in-memory write-through cache in front of HBase as the durable long-term store, used throughout section 3.

**ghilesmeddour. "gorilla-time-series-compression" README, GitHub.**
https://github.com/ghilesmeddour/gorilla-time-series-compression/blob/main/README.md
Source for the exact bit-level encoding schemes described in section 4a and 4b: the four control-bit prefixes and bit-length buckets for delta-of-delta timestamp compression (`0`, `10`+7 bits, `110`+9 bits, `1110`+12 bits, `1111`+32 bits), and the three control-bit cases for XOR value compression (`0` for an unchanged value, `10` for a nonzero XOR reusing the previous leading/trailing-zero window, `11` for a nonzero XOR with a new 5-bit leading-zero count and 6-bit meaningful-bit-length field). This independent implementation writeup was used to confirm the exact bit counts because the original paper's PDF and several third-party summaries of it returned access errors when fetched directly during research for this lesson.

**Uber Engineering Blog. "M3: Uber's Open Source, Large-scale Metrics Platform for Prometheus."**
https://eng.uber.com/m3/
Primary source for the architecture described in section 3's M3 references: the M3 Aggregator tier that pre-aggregates and routes metrics, M3DB as the horizontally sharded, replicated storage layer, and per-metric retention and resolution policies (for example, full resolution for a short window and coarser resolution for months). Also the source for Uber's reported scale figures, well over 6 billion time series tracked and hundreds of millions of metrics per second aggregated across the fleet, showing this pattern operating roughly three orders of magnitude past Facebook's original 2015 numbers within the same architectural shape.

**Uber Engineering Blog. "The Billion Data Point Challenge: Building a Query Engine for High Cardinality Time Series Data."**
https://eng.uber.com/billion-data-point-challenge/
Corroborating source for the query-side scale numbers referenced in section 3 and section 1's framing, describing the engineering work needed for M3's query engine to remain fast when a single query can span very high cardinality label sets, reinforcing that cardinality, not raw point volume alone, is the harder scaling axis for a metrics query engine.

**OpenObserve Blog. "The Prometheus Cardinality Bomb: Causes, Impact & How to Fix It."**
https://openobserve.ai/blog/prometheus-data-cardinality/
Source for the illustrative cardinality-growth incident narrative summarized in section 2b (a request-scoped label added to a counter, silently generating millions of new time series over roughly ninety days as users grow, eventually causing an out-of-memory kill and a long write-ahead-log replay with blank dashboards and silent alerting). This is presented in the lesson as a widely-documented, generic failure pattern used across the observability industry to teach this exact risk, not as a report of one specific named company's outage. The direct page returned an access error (HTTP 403) when fetched during research; the description here reflects the page's content as surfaced through search indexing, flagged explicitly per this lesson series' fact-vs-inference discipline.

**Grafana Labs. "Grafana Mimir" project documentation and repository.**
https://github.com/grafana/mimir and https://grafana.com/oss/mimir/
Source for the horizontally sharded query and long-term storage tier described in section 3: Mimir (and the related Cortex and Thanos projects in the same ecosystem) extend a single Prometheus server's storage into a distributed, multi-tenant backend reported to scale to a billion or more active series, by sharding both ingestion and querying rather than relying on one machine's memory and disk.

---

## Map to Rare.lab's stack

Rare.lab is nowhere near 12 million metric points a second today, but the underlying risk, an engineer adding one well-intentioned label that silently multiplies a metric's cardinality, does not require Facebook's traffic to bite. It only requires one label with an unbounded number of distinct values, and Rare.lab already has good candidates for that the moment any runtime telemetry ships: a device ID, a session ID, a specific node ID inside a user's shader graph, or a scene ID from the content-addressed R2 store.

The concrete, actionable piece to adopt before that telemetry pipeline exists, not after: when Rare.lab instruments its embeddable runtime for things like shader compile time, frames-per-second, or compile error rate, keep metric labels restricted to a small, bounded, enumerable set (GPU vendor, device tier, browser engine, shader template ID from a known catalog), and explicitly forbid unbounded identifiers (user ID, session ID, individual scene hash) as metric labels. Anything that needs a specific user's or a specific scene's history for debugging one incident belongs in a trace or a log line keyed by that identifier, not in a time series label, the same split section 5 draws between metrics and traces. Given that Rare.lab's scenes are already content-addressed and immutable in R2, a scene's content hash is exactly the kind of high-cardinality key that is cheap to log once per event but ruinous to use as a metric label repeated on every frame. And because the runtime shares one WebGL context across everything it renders, per-frame timing data from that single context is precisely the kind of high-frequency, regular-interval signal that Gorilla's delta-of-delta and XOR tricks were built for: if Rare.lab ever needs to keep more than a few minutes of raw per-frame timing history client-side or in a lightweight edge collector before shipping a rollup to Supabase, the same two encodings would compress that stream by an order of magnitude for close to no engineering cost, well before reaching for anything as heavy as a dedicated time series database.
