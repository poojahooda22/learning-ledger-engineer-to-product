# Day 40 — How does a stream processor count 583,000 events a second exactly once, when they arrive out of order and any machine can die mid-count?

*2026-07-24*

---

## 1. The company and the number that breaks a naive design

**Alibaba, running Apache Flink for the real-time dashboards behind its Double 11 shopping festival.** Double 11 (November 11) is Alibaba's annual sale day, and it is the closest thing e-commerce has to a live-fire load test of a country's entire payment and logistics stack in a single afternoon. Alibaba Cloud reported a peak of **544,000 orders created per second in 2019**, and **583,000 orders per second in 2020**, with total data volume for the day reported in the hundreds of petabytes. Every one of those orders needs to show up, correctly and exactly once, in the live dashboards that Alibaba's business teams and engineers watch to decide whether to shed load, open more payment capacity, or page someone.

The number that breaks a naive design here is not just the peak rate. It's that **events do not arrive in the order they happened.** A phone on a congested cellular network can buffer a tap-to-buy event for seconds before it reaches the server; a retried request after a dropped connection can land minutes after the original attempt; two events from the same user, milliseconds apart in reality, can arrive at a server in the opposite order because they took different network paths. At 583,000 events a second, "just count them as they arrive, in arrival order" produces a number that is wrong in two directions at once: it can double-count a retried event, and it can miss counting a genuinely late one if the window has already been closed and reported.

## 2. Why the naive design dies

The naive version: one process reads events off a queue, keeps a running total in memory (or increments a row in a database), and periodically reports the total. If the process crashes, it restarts and picks up wherever the queue's cursor happens to be.

This collapses in three concrete ways at Alibaba's scale.

**a. A single mutable counter cannot absorb 583,000 writes a second.** Whether the counter lives in memory on one process or as a row in a database, every event serializes into one point of contention. This is the exact same row-lock wall Day 6 covered for Stripe's payment writes, now applied to aggregation instead of payments: one hot row, one thread of execution, and a request rate that is orders of magnitude higher than one lock can grant per second.

**b. "At-least-once plus a naive counter" silently becomes "counted more than once."** If a process crashes after incrementing its in-memory counter but before checkpointing that state anywhere durable, the restart has to replay events from the last known-safe point in the queue. It replays events the crashed process had already counted. Do nothing special about that, and every crash quietly inflates the total. Across a 24-hour event with rolling deploys, autoscaling, and inevitable machine failures at Alibaba's node count, crashes are not an edge case, they're an hourly occurrence, and a system that overcounts on every one of them is unusable for a business dashboard that people are making real decisions from.

**c. Wall-clock arrival order is not the same thing as when the event actually happened, and pretending otherwise produces wrong answers even with zero crashes.** Say the dashboard reports "orders in the 14:00:00 to 14:00:10 minute window" and closes that window the instant its wall clock hits 14:00:10. An order that genuinely happened at 14:00:09 but was delayed six seconds by a flaky mobile connection arrives at 14:00:15, after the window already reported and moved on. Close the window on arrival time, and that order is silently dropped from the number that mattered. Wait indefinitely for every possible late arrival before reporting anything, and the dashboard is never live at all. Neither extreme is acceptable, and a naive design has no principled way to choose the point in between.

The analogy: imagine trying to count how many people crossed a finish line during a specific 10-second window, but some runners' finish times are only reported to you by messenger, and messengers take a random, unpredictable amount of time to reach you. If you close the books the instant your watch hits the 10-second mark, you'll under-count everyone whose messenger hasn't arrived yet. If you never close the books, you can never announce a result. What you actually need is a rule for "how late a messenger am I willing to wait for, before I'm willing to call the count final," stated explicitly, not assumed.

## 3. The architecture, top to bottom

```
Clients (a tap-to-buy on a phone, in Hangzhou or Jakarta, with
         unpredictable network latency before it reaches the server)
   |  each event carries its own event-time timestamp, stamped
   |  at the client, not at whatever server eventually receives it
   v
Ingestion log (Kafka, partitioned, e.g. by user ID or item ID)
      an ordered, replayable, append-only log per partition; this
      is the durable "source of truth" a stream job can always
      rewind to and replay from, the same role a WAL plays for a
      database (Day 17), now serving as the input to a live
      computation instead of a downstream replication target
   v
Stream processing cluster (Flink JobManager + many TaskManagers)
      TaskManagers run the actual keyed operators (dedupe, filter,
      aggregate); the JobManager is the coordinator, it does not
      touch data, it triggers checkpoints and tracks which tasks
      are healthy, the same "control plane separate from data
      plane" split Day 38's scheduler makes for compute jobs
   |
   |-- Keyed state, partitioned (each operator instance owns only
   |   the counters for its slice of keys, e.g. hash(item_id) % N,
   |   the same "same key, same node" sharding rule as Day 10,
   |   now applied to in-memory aggregation state instead of a
   |   database's rows)
   |
   |-- Watermarks (a periodic, monotonically advancing marker
   |   injected into the stream, "I believe all events with
   |   event-time before T have now arrived"; every windowed
   |   operator uses the watermark, not the wall clock, to decide
   |   when a window is finally safe to close and report)
   |
   |-- Checkpoint barriers (special markers the JobManager injects
   |   at the source, flowing through the graph alongside real
   |   records, never overtaking them; when an operator has seen
   |   the barrier on every input, it snapshots its own state
   |   asynchronously and forwards the barrier downstream, so the
   |   whole cluster ends up with one globally consistent snapshot
   |   without ever fully pausing)
   v
Durable state backend (RocksDB locally per task, checkpoints
                        shipped to remote durable storage, e.g.
                        HDFS or object storage)
      this is what a crashed task recovers from: roll back to the
      last completed checkpoint, replay only the log records since
      that checkpoint's offset, and the recomputed state is
      provably identical to what would have existed with no
      crash at all
   v
Transactional sink (two-phase commit into the next Kafka topic,
                     a database, or a dashboard's serving store)
      the sink only makes output visible once a full checkpoint
      that covers it has been confirmed complete, so a half-
      finished write from a crashed attempt is never observable
      downstream, the same "don't publish an uncommitted attempt"
      discipline as Day 20's distributed transactions
   v
Dashboards / downstream consumers
      read a number that is guaranteed consistent with exactly
      one pass over every event up to the last completed
      checkpoint, no more, no less
```

The layer that makes this whole thing work is the pairing of watermarks (which say *when a result is safe to finalize in event time*) with checkpoint barriers (which say *when the cluster's internal state is safe to declare durable*). They solve two different problems that are easy to conflate: watermarks are about correctness of the *answer* given out-of-order data; checkpoints are about durability of the *computation* given crashes. Alibaba's system needs both, at the same time, without either one stopping the other.

## 4. The transferable mechanisms

**a. The ordered, replayable log as the one source of truth.** Every event lives in a Kafka partition before anything computes over it, and a partition guarantees order only within itself, not globally, exactly the trade-off a WAL or a CDC stream makes (Day 17). Because the log is replayable, "restart from the last checkpoint and replay forward" is always possible; the log is what makes recovery a replay problem instead of an unrecoverable-data problem. This is the same shift Day 21's LSM trees make for storage: don't try to be right in place, be replayable from an append-only record.

**b. Checkpoint barriers as a Chandy-Lamport-style distributed snapshot, not a global pause.** Flink's checkpointing (its own documentation credits the Chandy-Lamport distributed snapshot algorithm as the inspiration, adapted for a DAG rather than a general graph) injects a barrier into the stream at the source, and each operator snapshots its own state the instant it has seen that barrier on every input, then forwards the barrier and keeps processing. No operator, and no clock, ever stops the whole pipeline to take a consistent picture of it. This is the general answer to "how do you get a consistent snapshot of a system that never stops moving": mark a cut point in the data flow itself, and let each component snapshot locally when the cut point reaches it, rather than trying to freeze everything simultaneously by wall clock (which distributed clocks cannot even guarantee agree, per Day 27).

**c. Watermarks: a policy for how late is late, made explicit instead of assumed.** A watermark at event-time T asserts "no event with a timestamp earlier than T should show up after this point" and lets every downstream window use event time, not arrival time, to decide correctness. The most common real policy is bounded out-of-orderness: pick a tolerance (say, "I will wait up to 30 seconds past when I'd otherwise close this window"), and every window closes exactly that much later than its nominal boundary, trading a fixed, known amount of latency for a fixed, known amount of completeness. This is Google's Dataflow Model paper (Akidau et al., VLDB 2015) formalizing what, where, when, and how for stream computation; Flink's watermark mechanism is a direct implementation of that model's "when" axis.

**d. Two-phase commit at the sink for end-to-end exactly-once, not just internal exactly-once.** Checkpointing alone only guarantees Flink's own internal state is consistent; it says nothing about what already got written to some external system before a crash. The fix borrows exactly the same primitive as Day 20's distributed transactions: the sink pre-commits (writes but does not yet make visible) tentatively, and only finalizes that write once the enclosing checkpoint is confirmed complete across the whole job. If the checkpoint never completes, the pre-committed write is rolled back or simply never surfaced. This is also, honestly, why "exactly-once" is a claim that only holds within a defined boundary: Kafka's own transactional producer guarantees exactly-once from producer to broker, not through every system downstream of the broker forever, and Confluent's own engineering writing on the feature is explicit that the industry-wide "everyone wants exactly-once delivery, nobody truly has it across arbitrary boundaries" caveat still applies. What you actually get is *exactly-once processing effects, within a defined system boundary*, achieved by making duplicates harmless (idempotent, Day 12) rather than by making duplicates impossible (which the two generals problem rules out in the general case).

**e. Keyed, partitioned state, same sharding rule as everywhere else in this series.** Each TaskManager holds only the counters for the keys it owns, so state size and checkpoint size both scale by adding more partitions and more TaskManagers, not by making one machine's state bigger. This is Day 10's consistent-hashing rule, applied to in-memory aggregation state instead of database shards.

**f. Incremental checkpointing: ship the delta, not the whole state.** Alibaba's own Blink fork of Flink was built specifically because materializing a job's entire state (many terabytes, individual task states running many gigabytes each) at every checkpoint was, in their own account, one of the biggest blocking issues to running Flink at their scale. The fix is the same idea as an LSM tree's incremental flush (Day 21) or CDC's diff-based replication (Day 17): only the state that changed since the last checkpoint gets written out, so checkpoint cost scales with the rate of change, not with total accumulated state size.

## 5. The trade-offs

**Consistency vs. availability, expressed here as completeness vs. latency, per window.** A window that waits longer for late-arriving events (a larger watermark tolerance) is more complete, but reports later. A window that closes fast reports sooner but risks under-counting events that were genuinely still in flight. Alibaba's Double 11 dashboards make this trade-off differently for different numbers on the same screen: the headline "current GMV" figure needs to update within seconds and can tolerate small, transient under-counts that true up on the next refresh; a final settlement or reconciliation number, computed after the fact, can afford to wait far longer for full completeness because correctness matters more than freshness once the sale is over. Same system, two different points on the same completeness-vs-latency curve, chosen deliberately per use case rather than picked once for the whole pipeline.

**Checkpoint interval is a direct cost vs. recovery-time knob.** A shorter checkpoint interval means less work is ever lost on a crash (less to replay), but checkpointing itself costs CPU, network, and I/O every time it runs, and under backpressure it can cost more than that. Flink's own engineering blog documents a specific failure shape here worth naming directly: with the *aligned* checkpoint mechanism (the original design), a checkpoint barrier has to wait at any operator until it has arrived on every input channel before that operator can snapshot, and if one input channel is backed up (a slow or overloaded downstream operator applying backpressure upstream), the barrier queues up behind whatever data is already buffered on that channel. Under real backpressure, that alignment wait can stretch long enough that checkpoints time out entirely, at exactly the moment (an overloaded pipeline) when you most need a completed checkpoint to recover from. The system that exists to make failure recoverable becomes unable to complete its own job precisely when failure is most likely. Flink's answer, unaligned checkpoints, lets the barrier overtake buffered records on a busy channel instead of waiting behind them, snapshotting the in-flight buffered data itself as part of the checkpoint rather than draining it first. That's a direct cost-for-reliability trade: unaligned checkpoints are typically larger (because in-flight buffered records get stored too) in exchange for checkpoints that actually complete under load instead of stalling indefinitely.

## 6. The systems-thinking lens

**The feedback loop here is checkpoint alignment stalling under the exact backpressure it was meant to help recover from, a close cousin of the thundering-herd and metastable-failure shapes covered earlier in this series.** Trace it: traffic spikes → one operator in the pipeline falls behind and starts backpressuring its upstream neighbors → aligned checkpoint barriers queue up behind the resulting backlog on that slow channel → checkpoints take longer, or time out and never complete → because no checkpoint completes, more and more uncommitted work piles up that would need to be replayed on any crash → operators are now doing double duty, processing the live backlog and holding ever-more state that isn't safely durable, which makes them slower still, which worsens the very backpressure that stalled the checkpoint in the first place. The mechanism meant to make the system recoverable becomes another source of load on the exact bottleneck causing the trouble, the same "the fix amplifies the failure" shape as a retry storm hammering an already-struggling service.

**The senior fix breaks the loop structurally, it doesn't add TaskManagers and hope.** Unaligned checkpoints remove the dependency of "checkpoint completion" on "backlog first drains," by snapshotting in-flight buffered data instead of waiting behind it, so a checkpoint can complete even while backpressure is ongoing. That's the same instinct as Day 13's backpressure and load-shedding lesson, applied one layer deeper: don't let a subsystem's own health-check or recovery mechanism have a hard dependency on the thing that's currently unhealthy. More capacity thrown at the backpressured operator might help too, eventually, but it doesn't fix the structural coupling that made checkpointing itself fail under load, and that coupling is what turns an ordinary traffic spike into an unrecoverable one.

---

## References and summaries

**Alibaba Cloud Community Blog. "Alibaba Cloud Supported 583,000 Orders/Second for 2020 Double 11."**
https://www.alibabacloud.com/blog/alibaba-cloud-supported-583000-orderssecond-for-2020-double-11---the-highest-traffic-peak-in-the-world_596884
Primary source for this lesson's headline number: 583,000 orders per second at peak during the 2020 Double 11 event, described by Alibaba Cloud as the highest traffic peak recorded in the world at the time. Cross-referenced against earlier reporting of 544,000 orders/second in 2019 and 970 petabytes of data processed across that 24-hour event, confirming the year-over-year scale trajectory used in section 1.

**Alibaba Cloud Community Blog. "Apache Flink: Powering Real-Time Personalization in Retail and E-Commerce."**
https://www.alibabacloud.com/blog/602072
Source for Alibaba's use of Apache Flink specifically to power live, real-time dashboards (order volume, transaction totals, user activity) during Double 11, tying the raw traffic number to the specific stream-processing system this lesson is about.

**Wang, F. and Wang, Z. (Alibaba). "Runtime Improvements in Blink for Large-Scale Streaming at Alibaba." Flink Forward SF 2017.**
https://www.slideshare.net/slideshow/flink-forward-sf-2017-feng-wang-zhijiang-wang-runtime-improvements-in-blink-for-large-scale-streaming-at-alibaba/75014480
Primary source (Alibaba's own conference talk) for the incremental-checkpointing motivation used in section 4f: job state reaching many terabytes with individual task state in the many-gigabytes range, and full-state materialization at every checkpoint becoming one of the biggest blockers to running Flink at Alibaba's production scale, the direct reason Alibaba's Blink fork prioritized incremental checkpointing.

**Apache Flink Blog. "Managing Large State in Apache Flink: An Intro to Incremental Checkpointing."**
https://flink.apache.org/2018/01/30/managing-large-state-in-apache-flink-an-intro-to-incremental-checkpointing/
Corroborating source, from the Flink project itself, for how incremental checkpointing works mechanically: only state that changed since the previous checkpoint is materialized and shipped, rather than the full state snapshot every time, the general mechanism summarized in section 4f.

**Apache Flink Blog. "From Aligned to Unaligned Checkpoints - Part 1: Checkpoints, Alignment, and Backpressure."**
https://flink.apache.org/2020/10/15/from-aligned-to-unaligned-checkpoints-part-1-checkpoints-alignment-and-backpressure/
Title and topic confirmed via search indexing; the direct page returned an access error (HTTP 403) when fetched during research for this lesson, so the description in section 5 and section 6 of aligned checkpoints stalling under backpressure, and unaligned checkpoints letting barriers overtake buffered in-flight data instead of waiting behind it, reflects the Flink project's own well-documented public design rationale for this feature (also described consistently across the Flink documentation's checkpointing pages and third-party summaries indexed alongside it), not a direct quote from the inaccessible primary post. Flagging this explicitly per this lesson series' fact-vs-inference discipline.

**Chandy, K.M. and Lamport, L. "Distributed Snapshots: Determining Global States of Distributed Systems." ACM Transactions on Computer Systems, 1985.** (Flink's own adaptation described in its documentation.)
https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/fault-tolerance/checkpointing/ and https://nightlies.apache.org/flink/flink-docs-release-1.4/internals/stream_checkpointing.html
Primary source (via Flink's own documentation, which explicitly credits Chandy-Lamport) for the checkpoint-barrier mechanism described in section 3 and 4b: barriers injected at sources, flowing in-line with records without overtaking them, each operator snapshotting locally the instant it has seen the barrier on every input and forwarding it downstream, adapted by Flink for a DAG topology rather than the original algorithm's general graph.

**Akidau, T. et al. (Google). "The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing." VLDB 2015.**
https://blog.acolyer.org/2015/08/18/the-dataflow-model-a-practical-approach-to-balancing-correctness-latency-and-cost-in-massive-scale-unbounded-out-of-order-data-processing/ (summary) and Akidau's companion posts, "Streaming 101" and "Streaming 102," https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/ and https://www.oreilly.com/radar/the-world-beyond-batch-streaming-102/
Primary source for the watermark concept and the what/where/when/how framing used in section 3 and 4c: watermarks as the mechanism that tracks event-time completeness as processing-time progresses, letting a system decide when a window is safe to close without assuming perfectly ordered input. This paper (and Akidau's accessible companion blog posts, later expanded into the O'Reilly book "Streaming Systems") is the foundational public reference for how modern stream processors, Flink included, reason about event time versus processing time.

**Confluent Blog. "Exactly-once Semantics is Possible: Here's How Apache Kafka Does It."**
https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
Primary source for the section 4d claim about Kafka's transactional exactly-once guarantee being scoped specifically to producer-to-broker, and more broadly for the industry framing (echoed across multiple engineering blogs indexed during research for this lesson) that true end-to-end exactly-once delivery across arbitrary system boundaries is not achievable in the general distributed-systems case (the two generals problem), and that what production systems actually deliver is exactly-once *processing effects* within a defined boundary, built from idempotent operations plus atomic commit, not from preventing duplicates from ever existing in the first place.

---

## Map to Rare.lab's stack

Rare.lab doesn't run a 583,000-events-a-second stream today, but the underlying problem, out-of-order updates racing each other, is already live in the node-based editor the moment more than one collaborator edits the same graph, and it will show up again the instant Rare.lab adds any live telemetry (compile success rates, render performance, error rates) that needs to aggregate across many concurrent users rather than reporting one session at a time.

The concrete, actionable piece to borrow now, before that telemetry pipeline exists: if Rare.lab ever builds a "compiles per minute" or "shader errors per minute" dashboard fed by client-reported events, resist the naive design of incrementing a counter the instant an event arrives at the server. Stamp every client event with the event-time it actually happened at the client, not the time it happened to reach the server, and pick an explicit, stated tolerance for how late an event can arrive and still count (Rare.lab's clients are browsers on ordinary internet connections, not a controlled data-center network, so this will matter sooner than it looks like it will). That one discipline, event time plus a stated late-arrival tolerance, is the entire difference between a dashboard number that's occasionally wrong in a way nobody can explain and one that's transparently, deliberately "complete as of N seconds ago." And when that pipeline's state starts to matter (session state per active editor, aggregate counts per shader template), keep it keyed and partitioned by graph ID or user ID from day one, the same sharding discipline Day 10 already recommended for the runtime's compiled-shader cache, so the aggregation layer scales by adding partitions rather than by making one process's in-memory state bigger until it doesn't fit.
