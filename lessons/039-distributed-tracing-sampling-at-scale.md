# Day 39 — How do you debug one slow request when it crossed 40 services and you can't afford to record all of them?

*2026-07-23*

---

## 1. The company and the number that breaks a naive design

**Google, Dapper, and Uber, Jaeger.** Dapper is the internal tracing system Google described in its 2010 paper "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure" (Sigelman et al.), built because a single user-facing request, a web search, an ad click, could fan out across hundreds of internal services and thousands of machines, and no engineer could hold that call graph in their head or grep it out of separate log files. Jaeger is the open-source tracing system Uber built starting in 2015 (open-sourced 2017) to solve the exact same problem inside its own service-oriented architecture, which by 2019 had grown past 2,200 microservices.

The number that breaks a naive design is a storage-and-overhead number, not a machine count. Dapper's own paper reports that its production sampling rate, roughly **1 trace kept for every 1,024 candidates (about 0.1%)**, still produces **more than one terabyte of sampled trace data every single day** on Google's production clusters. Run that arithmetic backward: capturing every request, unsampled, from just one cluster, would be on the order of a petabyte a day. And Uber's own experience with a *fixed* sampling rate exposed a second, sharper number: Uber's ride volume has strong daily seasonality, a fixed sampling probability that is reasonable at 3am is either far too sparse to catch anything interesting or far too aggressive (and expensive) during rush hour, on the same service, a few hours apart. **A single fixed knob cannot be correct at both ends of a traffic curve that swings by an order of magnitude within one day.**

## 2. Why the naive design dies

The naive version: every service writes its own local log file, one line per request, with a timestamp and whatever the developer thought to print. When a request is slow, an engineer manually greps each service's logs and tries to match them up by eyeball, using timestamps and maybe a user ID, to reconstruct what happened across the chain of calls. This collapses in three concrete ways.

**a. Timestamps alone cannot disambiguate concurrent requests.** At any real traffic volume, dozens or hundreds of unrelated requests are in flight on the same service within the same millisecond. Two log lines that happen to be adjacent in time might belong to completely different user requests. Without a value that is shared across every hop of one specific request, and only that request, correlation across services is a guess, not a lookup.

**b. Full, unsampled capture does not survive the traffic it is meant to explain.** If every request writes its full call-chain data everywhere, storage and network cost scale linearly with request volume, exactly the volume that is largest, and therefore the tracing system's own ingest cost is highest, precisely when traffic (and incident risk) is highest. Dapper's own number makes this concrete: even at roughly 1-in-1,024 sampling, a single Google cluster still produces over a terabyte a day. Capturing everything scales the tracing system's own storage bill in lockstep with your success, and that is the one bill you cannot let outgrow the thing paying for it.

**c. Manual correlation does not survive fan-out.** At Uber's 2,200+ microservices, or Google's search stack touching many hundreds of internal RPCs per user request, a human manually grepping and eyeballing timestamps across dozens of separate log streams during a live incident takes minutes to hours. The incident is often over, or has escalated, before the root cause is found this way.

The analogy: imagine trying to figure out why a package arrived three days late by calling the warehouse, the regional hub, and the local courier separately, each of whom has their own private notebook with times but no shared tracking number, and hoping their handwriting and clocks line up well enough to reconstruct one package's actual journey. What you actually want is a single tracking number stamped on the package at the first warehouse, carried untouched through every hand-off, so every notebook entry for that number can be pulled and stitched into one timeline on demand.

## 3. The architecture, top to bottom

```
Clients (a user's request: a search, a ride request, a checkout)
   |  the FIRST service to touch the request mints a new trace ID
   |  (a random 128-bit value) if none was already propagated in
   v
Instrumented app tier (every microservice in the call chain)
   |  each service, on entry, reads an incoming trace ID + parent
   |  span ID from a request header (W3C Trace Context's
   |  `traceparent` header: version-traceid-parentid-flags),
   |  creates its OWN new span ID for the work it's about to do,
   |  and propagates trace-id + its-own-span-id-as-new-parent-id
   |  to every downstream call it makes
   |  analogy: a single tracking number stamped on the package,
   |  carried through every hand-off, each hub adds its own
   |  scan event but never invents a new tracking number
   v
Local agent / sidecar (Jaeger: jaeger-agent, a UDP-listening
                        daemon deployed on every host)
   |  spans are buffered locally and shipped OFF the request's
   |  own critical path, in batches, over the network, never
   |  blocking the user-facing response on tracing I/O
   |  analogy: the delivery driver's phone silently logging scan
   |  events in the background, never making the customer wait
   |  while it uploads
   v
Collector tier (stateless, horizontally scaled; Jaeger collectors
                 or an OpenTelemetry Collector)
   |  validates spans, applies the sampling decision (see below),
   |  and for TAIL-based sampling specifically, must route every
   |  span for the SAME trace ID to the SAME collector instance
   |  so a keep/drop decision can see the whole trace at once,
   |  the exact same "same key, same shard" problem consistent
   |  hashing solves (Day 10), just applied to trace IDs instead
   |  of cache keys or database rows
   v
Storage (Cassandra for fast trace-ID lookup, or Elasticsearch for
          richer search across span tags; Jaeger supports both)
   |  spans are appended, indexed primarily by trace ID, plus
   |  service name / operation name / duration for search
   v
Query / UI service (Jaeger Query, stateless, reads from storage)
   |  reconstructs the full span tree for one trace ID on demand,
   |  renders it as a waterfall: which hop took how long, where
   |  the critical path actually was
   v
Async aggregation (span data streamed into metrics: per-service
                    p50/p99 latency, error rate, dependency maps)
      this is the layer that turns millions of individual traces
      into the small number of dashboards an on-call engineer
      actually watches, without keeping every raw trace around
      forever to compute them
```

The split that matters: the trace data path is deliberately **out-of-band and best-effort**, decoupled from the user-facing request path at the very first hop (the local agent, not the request thread, ships spans). A dropped span, a collector restart, a storage hiccup, none of it should ever be able to slow down or fail an actual user request. That decoupling is the single most important architectural decision in this whole system, more important than which storage engine or which sampling algorithm you pick.

## 4. The transferable mechanisms

**a. Context propagation via a correlation ID.** A trace ID is minted once, at the first hop, and threaded through every subsequent call as a header (W3C Trace Context's `traceparent`, or the older B3 headers Zipkin popularized). Every span that shares a trace ID is provably part of the same causal chain, no timestamp-matching guesswork required. This is the same primitive as a request ID, an idempotency key (Day 12), or an order ID that follows a package through a supply chain: mint once, carry everywhere, never regenerate mid-flight.

**b. Sampling as a spectrum, not a binary.** Head-based sampling decides at the very first hop, before anything is known about how the request turns out (Dapper's original approach: a uniform probability, later made adaptive, parameterized by a target rate of *traces per unit time* rather than a fixed percentage, so a quiet service auto-raises its rate and a hot one auto-lowers it, without a human retuning a knob per service per time-of-day). Tail-based sampling waits until an entire trace has been collected, then decides, which means you can deliberately keep 100% of traces that errored or ran slow and sample down the boring, fast, successful ones far more aggressively. The trade is precision (tail-based) against simplicity and lower buffering cost (head-based).

**c. Never let the observability path block the observed path.** Spans are buffered locally and shipped asynchronously; if the collector tier is overloaded or down, spans queue or get dropped, but the user's actual request completes on time regardless. This is the exact same principle as Day 9's queue-as-shock-absorber and Day 13's backpressure, applied specifically to telemetry: the thing watching the system must never become a dependency the system needs to succeed.

**d. Route by the same key you need to reassemble by.** Tail-based sampling needs every span for one trace ID to land on one collector instance to make a single keep/drop decision, which means the collector tier is sharded (consistently hashed) on trace ID, exactly the "same key, same node" guarantee Day 10 covers for cache and database sharding, just repurposed here for stitching, not storage.

**e. Separate high-cardinality raw events from low-cardinality aggregates.** A trace, keyed by a unique 128-bit ID, is about as high-cardinality as data gets, which is exactly why it can't live forever at 100% and why it's stored differently (indexed lookup by ID) than a metric like "p99 latency for service X," which is deliberately pre-aggregated down to a handful of numbers per service per minute. Honeycomb's Charity Majors frames this as the difference between metrics (answering *known* questions you thought to pre-aggregate for) and high-cardinality traces/events (answering the *unknown-unknown*, the question you didn't know you'd need to ask until an incident forced it). Keep both, for different jobs, at different retention.

**f. Structured spans, not free-text logs.** A span is a typed record (service name, operation name, start time, duration, parent span ID, key-value tags), not a printed string. That structure is what makes "show me every trace where service=payments and duration>2s" a fast indexed query instead of a regex sweep over gigabytes of unstructured text.

## 5. The trade-offs

**Consistency vs. availability, split by data type, same shape as every other lesson in this series but applied to the observability layer itself.** The trace data itself is allowed to be, and is deliberately, only best-effort: spans can arrive late, out of order, or not at all, and that is an acceptable, expected cost, because tracing is a diagnostic side channel, not the source of truth for the request's correctness. The actual request path (the payment, the ride match, the search result) must remain correct and available regardless of whether a single span made it to the collector. Never conflate the two. A tracing outage should degrade your ability to *see*, never your ability to *serve*.

**Cost vs. latency (and completeness), paid at every sampling decision.** Aggressive sampling (Dapper's 1-in-1,024) keeps storage and CPU overhead near-invisible (Dapper reports under 0.3% of one core, under 0.01% of network traffic, on the machines it instruments) but risks the one genuinely rare, genuinely important trace simply not being sampled. Tail-based sampling buys back precision on the traces that matter most (errors, outliers) at the cost of buffering every span in a decision window before deciding, real memory and coordination cost at the collector tier, and the hard requirement that same-trace spans land on the same collector. There is no free option here, only where you choose to spend: memory at the collector, or the chance of missing the trace you needed.

## 6. The systems-thinking lens

**The feedback loop here is that the tracing system's load is directly correlated with the incidents it exists to diagnose, and if it isn't decoupled, it becomes the second outage.** Trace the failure path: traffic spikes or an upstream dependency starts erroring → request volume and error rate both rise at once → if spans are captured synchronously, or if the app tier ever blocks waiting on the tracing collector (a slow flush, a full queue with no drop policy), then the very telemetry meant to explain the incident starts adding its own latency to every request, on top of the original problem → slower requests hold connections and threads open longer → more requests queue up behind them → the system looks even more overloaded than the original incident alone would explain, and now nobody can tell how much of the slowness is the real problem versus the tracing pipeline itself. This is the same shape as a retry storm or a thundering herd: a mechanism meant to help (observing, retrying, warming a cache) instead amplifies the exact failure it was supposed to catch, once it stops being decoupled from the thing it depends on.

**The senior fix is architectural decoupling and load-shedding on the tracing pipeline itself, not adding more collector capacity after the fact.** Ship spans asynchronously from a local buffer that drops the oldest data under pressure rather than blocking the caller (never backpressure onto the request path). Make the sampling rate itself adaptive to load, not just to a static target, so the tracing pipeline sheds its own volume first when overloaded, the same instinct as Day 13's load shedding, aimed inward at the observability system instead of outward at user traffic. And keep the collector tier stateless and independently scalable from the app tier it's watching, so a spike in trace volume competes for its own dedicated capacity, never for the same threads and connections the actual user requests need to complete.

---

## References and summaries

**Sigelman, B. H., Barroso, L. A., Burrows, M., Stephenson, P., Plakal, M., Beaver, I., Jaspan, S., Shanbhag, C. "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure." Google Technical Report, 2010.**
https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/ (listing) and https://research.google.com/archive/papers/dapper-2010-1.pdf (PDF)
The primary source for this lesson's core numbers: uniform sampling at roughly 1-in-1,024 candidate traces in Dapper's first production version, later replaced by an adaptive scheme parameterized by a target rate of sampled traces per unit time rather than a fixed probability (so low-traffic processes sample up and high-traffic processes sample down automatically); overhead reported at under 0.3% of one CPU core and under 0.01% of network traffic on instrumented production machines; and Google's production clusters generating over one terabyte of sampled trace data per day even at that sampling rate. Cross-corroborated via the paper listing and via Adrian Colyer's "the morning paper" summary (https://blog.acolyer.org/2015/10/06/dapper-a-large-scale-distributed-systems-tracing-infrastructure/), a well-known, widely-cited academic-paper-summary source used here as secondary confirmation, consistent with the primary abstract.

**Uber Engineering Blog. "Evolving Distributed Tracing at Uber Engineering."**
https://www.uber.com/blog/distributed-tracing/
Primary source for Uber's Jaeger motivation and its critique of fixed-probability sampling: Uber's business traffic shows strong daily seasonality (more rides during peak hours), so a single fixed sampling probability is simultaneously too low for off-peak traffic and too high (and costly) for peak traffic, and any one service has little visibility into how its own sampling choice affects total downstream data volume when it calls into services with high fan-out. This is the direct source for the adaptive, rate-based (not probability-based) sampling approach described in this lesson.

**Jaeger documentation. "Architecture."**
https://www.jaegertracing.io/docs/1.18/architecture/ and https://www.jaegertracing.io/docs/1.16/architecture/
Primary source for the jaeger-agent (UDP-listening host daemon that batches and forwards spans, abstracting collector discovery away from client libraries) and jaeger-collector (stateless, horizontally scalable, runs the validate/index/store pipeline) components described in the architecture section above.

**CNCF Blog / Logz.io. "Jaeger Persistent Storage with Elasticsearch, Cassandra & Kafka."**
https://www.cncf.io/blog/2021/03/12/jaeger-persistent-storage-with-elasticsearch-cassandra-kafka/ and https://logz.io/blog/jaeger-persistence/
Source for the storage-backend trade-off described here: Cassandra as a key-value store is efficient for direct trace-ID lookups but offers weaker search capability than Elasticsearch, which supports richer queries across span tags at the cost of a heavier operational footprint, the reason Jaeger supports both and lets operators choose based on their actual query patterns.

**W3C. "Trace Context" specification.**
https://github.com/w3c/trace-context/blob/main/spec/20-http_request_header_format.md
Primary source for the `traceparent` header format (version, trace ID, parent ID, trace flags) used in the context-propagation mechanism described in section 4a, now the standard most tracing systems (OpenTelemetry included) implement for cross-service header propagation, superseding earlier vendor-specific formats like Zipkin's B3 headers.

**OpenTelemetry Blog. "Tail Sampling with OpenTelemetry: Why it's useful, how to do it, and what to consider."**
https://opentelemetry.io/blog/2022/tail-sampling/
Primary source for the tail-based sampling mechanism described in section 4b and 4d: the decision is deferred until a full (or windowed) trace is available, which lets a system keep all error and high-latency traces while sampling routine successful ones far more aggressively, at the structural cost that every span for one trace must be routed to the same collector instance to make one coherent decision.

**Charity Majors / Honeycomb. Various (interviews and blog, e.g. "The Truth About 'Meh-trics'").**
https://charity.wtf/p/the-truth-about-meh-trics and https://www.honeycomb.io/blog/honeycomb-10-year-manifesto-part-1
Secondary source (industry commentary from Honeycomb's co-founder, a widely cited voice in the observability space) for the framing used in section 4e: metrics answer known-unknowns you pre-aggregated for in advance, while high-cardinality traces and events are what let you answer unknown-unknowns, the question you didn't know to ask until an incident demanded it, which is why the two are stored and retained differently rather than treated as the same data at different resolutions.

---

## Map to Rare.lab's stack

Rare.lab's compile pipeline, node graph in, parse, validate, code-generate, compile the shader, load it into the shared WebGL runtime, is already a multi-hop chain, just compressed into one process instead of spread across a data center. The exact failure mode this lesson describes already exists in miniature: when a specific user's shader fails to compile, or renders wrong, or runs slow on a specific GPU, today that's debugged by manually correlating client-side console output with whatever server-side compile logs exist, the same timestamp-matching guesswork Dapper and Jaeger were built to eliminate.

The concrete, actionable move, before the node-graph library or the user base grows past what one person can debug by eye: mint a trace ID per compile job at the moment a user hits "compile," and propagate it through every stage, parse, validate, codegen, shader compile, and critically, into the WebGL error and shader-compile-log output on the client, so a single ID stitches the whole journey of one specific compile attempt back together, tagged with which node in the graph the failure actually traces to. Given Rare.lab's current volume, 100% capture is fine, exactly the way Google's naive single-server Dapper predecessor would have been fine at low scale. The moment that stops being true, adopt the tail-based instinct directly: always keep full traces for anything that failed to compile or ran slow, and sample the fast, successful compiles far more aggressively, rather than paying full storage cost for the 99% of compiles that were never in question. And because the shared WebGL context is itself the scarce, oversubscribed resource this lesson's Day 38 counterpart already flagged, per-node timing spans through the compile-and-render path are also the concrete way to eventually answer "which node types are actually expensive across thousands of graphs," rather than guessing from a handful of manually reported slow cases.
