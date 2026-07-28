# Day 42 — How does one system move 7 trillion messages a day when a single queue tops out around 10 MB/sec?

*2026-07-28*

---

## 1. The company and the number that breaks a naive design

**LinkedIn, the birthplace of Apache Kafka.** LinkedIn built Kafka in 2010 because their existing pipeline (batch ETL jobs plus a scatter of point-to-point queues) could not keep up with the number of systems that needed to know about every profile view, every connection, every ad impression, in near real time. By 2026 LinkedIn runs Kafka at a scale that reads like a typo: **7 trillion messages a day, across more than 100 clusters, 4,000+ brokers, 100,000+ topics, and roughly 7 million partitions** (LinkedIn Engineering, "How LinkedIn customizes Apache Kafka for 7 trillion messages per day"). Confluent had already logged Kafka crossing **1.1 trillion messages a day** industry-wide years earlier, so the growth curve itself is part of the story.

The number that breaks a naive design is not the trillion, it is the throughput ceiling of the one thing a naive design insists on using: a single ordered channel. A single Kafka partition, which is just an append-only file plus an offset counter, tops out at roughly **10s of MB/sec of write throughput** on ordinary broker hardware (Confluent, "How to Choose the Number of Topics/Partitions"), and a single broker caps out around **~100 MB/sec** before disk and network I/O saturate. A trillion messages a day is about 11.5 million messages a second. Even at a generous 1 KB average message size, that is well over 10 GB/sec of sustained write traffic. You cannot get there by writing faster to one file. You get there by having millions of files being written to at once, which is exactly what "7 million partitions" means.

## 2. Why the naive (demo) design dies

The demo version of "a message queue" is one process, one ordered list, one lock: producers append, consumers read from the front, done. This is what a single RabbitMQ queue or a single Postgres table used as a queue looks like under the hood. It dies in three concrete ways at LinkedIn's number:

**a. One disk, one write head, one throughput ceiling.** An append-only log is fast because writes are sequential, not random, so even a single spinning disk can sustain tens of MB/sec. But "tens of MB/sec" is a hard physical ceiling on that one file, no matter how good the code is. You cannot make one disk write 10 GB/sec by optimizing the software around it. This is the single-database-row problem from Day 6 (Stripe) wearing different clothes: one physical resource, unbounded demand.

**b. One machine, one point of total loss.** If the queue lives on one broker and that broker's disk dies, every message that was only on that disk is gone, and every producer and consumer pointed at it stalls until it comes back, if it ever does. A demo queue has no concept of "this data also lives somewhere else."

**c. One reader position, no parallel consumption.** If there is one ordered list, there can only be one process safely reading "the next item" without two workers grabbing the same message. Bolt-on locking to let multiple workers pull from the same queue and you either serialize them (defeating the purpose) or you break ordering guarantees consumers depend on.

The analogy: think of a single grocery store checkout lane on Black Friday. One lane, one register, one line. It does not matter how fast the cashier scans items, the line only moves as fast as one register can process, and if that register jams, the whole line stops dead. The fix is not a faster cashier, it is opening 50 lanes and giving each customer a numbered ticket for a specific lane, so throughput scales with the number of lanes, not the speed of any one of them.

## 3. The architecture, drawn top to bottom

```
PRODUCERS (LinkedIn services: profile-view events, ad clicks, DB change events)
   |  each message has a key (e.g. member_id) and a value
   |  producer hashes the key -> picks a partition, deterministically
   v
TOPIC = a named stream, split into PARTITIONS (the "lanes")
   partition 0   partition 1   partition 2   ...   partition N
   [msg][msg]... [msg][msg]... [msg][msg]...        [msg][msg]...
   each partition is its OWN append-only log file, own offset counter 0,1,2,3...
   analogy: a topic is a filing cabinet, each partition is one drawer,
   the key decides which drawer a document goes in, order is only
   guaranteed WITHIN a drawer, never across drawers
   |
   v
BROKERS (the machines holding the drawers)
   each partition has ONE LEADER broker + N FOLLOWER brokers (replicas)
   producers and consumers only ever talk to the LEADER for that partition
   followers just pull-copy the leader's log, like a live backup tape
   ISR = "in-sync replicas": the leader + whichever followers are fully
   caught up right now. acks=all means: don't tell the producer "done"
   until every broker in the ISR has the byte. min.insync.replicas sets
   how small the ISR is allowed to shrink before writes start failing
   outright, rather than silently losing durability
   analogy: leader is the one teller who actually stamps your deposit
   slip, followers are tellers copying the ledger in real time so if the
   first teller collapses, a fully caught-up teller can take over instantly
   |
   v
CONTROLLER / METADATA QUORUM (KRaft, replacing ZooKeeper as of Kafka 4.0)
   watches broker heartbeats; if a leader broker dies, the controller
   picks a new leader from the surviving ISR members and tells every
   producer/consumer the new address, all within seconds
   analogy: the branch manager who, the instant one teller vanishes,
   points the next customer at a teller who already has an up-to-date ledger
   |
   v
CONSUMER GROUPS (the readers)
   many consumer processes share one "group id"; Kafka assigns each
   consumer a disjoint set of partitions to own, so N consumers in a
   group read N different drawers in parallel, no two consumers ever
   read the same partition in that group at the same time
   each consumer tracks its own OFFSET (a bookmark: "I've read up to
   message #48,102 in this partition") committed back to Kafka itself
   analogy: N librarians, each permanently assigned a shelf, each
   keeping their own bookmark, so nobody re-shelves or re-reads the
   same book twice
   |
   v
GROUP COORDINATOR (rebalancing)
   when a consumer joins, leaves, or crashes, the coordinator
   reassigns partitions among the survivors ("rebalance"); older
   protocols stopped ALL consumers in the group during this, newer
   cooperative/incremental protocols (KIP-429, KIP-848) only move the
   specific partitions that actually need to move
   |
   v
LOG COMPACTION (background process, per-partition)
   for topics marked "compacted," a background thread walks the log
   and throws away every OLD value for a key, keeping only the latest
   record per key, forever, instead of expiring by age
   real use: Kafka Streams uses compacted topics as the changelog
   backing a KTable / RocksDB state store, so after a crash the state
   can be rebuilt by replaying a compacted log instead of the entire
   unbounded history
```

## 4. The transferable mechanisms

- **Partitioning by key, not by arrival order.** Hashing a key (member_id, order_id, device_id) to pick a shard is the same trick as Day 10's consistent hashing and Day 26's Snowflake ID sharding: pick a stable function of the data itself so writes fan out across many independent physical resources instead of piling onto one. The cost: ordering is only guaranteed within a partition, never globally across the topic. If two events about the same member must stay in order, they must share a key.

- **Leader-follower replication with a quorum acknowledgement, not a blind broadcast.** Every partition has exactly one broker that accepts writes (the leader) and a small set of followers that copy it. A write is only "durable" once a configurable quorum of the ISR has it (acks=all + min.insync.replicas). This is the same shape as Day 22's leaderless-replication quorums, except Kafka picks a leader for simplicity and speed, and pays for that with a leader-election step (via the controller/KRaft quorum) if that leader dies. It is the same trade Day 11's Raft and Day 27's Spanner make: someone has to be the ordering referee, so make electing a new one cheap and fast rather than avoiding the role.

- **The log as the single source of truth, not a side effect.** Kafka's core insight, straight from the original 2011 Kreps/Narkhede/Rao paper, is that an append-only sequential log is both the fastest thing you can write to (sequential disk I/O beats random I/O by orders of magnitude) and a complete, replayable record of everything that happened. This is Day 17's write-ahead log idea promoted from "internal database detail" to "the actual product." Anything downstream (a cache, a search index, an analytics table) can be rebuilt by replaying the log from any offset.

- **Pull-based consumption with consumer-owned offsets, not server-pushed delivery.** Consumers ask the broker "give me everything after offset X," and the broker just serves bytes from a file, it never tracks per-consumer delivery state itself. This makes adding a new consumer nearly free (its first request just says "start me at offset 0" or "start me now") and makes replay trivial (rewind the offset). Contrast with a push-based queue, where the broker has to actively track delivery/ack state per message per consumer, a much heavier bookkeeping burden at 7 million partitions.

- **Consumer groups turn one stream into N parallel workers automatically.** Because ownership of a partition is exclusive within a group, scaling read throughput is "add another consumer process and let the group coordinator reassign partitions," with no manual sharding logic in the application. The ceiling is the partition count: a group can never usefully have more active consumers than the topic has partitions, which is why partition count is chosen for the *consumer* parallelism you will eventually need, not just today's producer throughput (Confluent's partition-count guidance).

- **Compaction turns an unbounded log into a bounded "latest state" table.** Retention-by-time throws away old data to bound storage. Compaction instead throws away *stale* data (superseded values for the same key) while keeping the log property (append-only, replayable) that lets a new reader rebuild current state from scratch just by reading the compacted log start to finish. This is the mechanism behind Kafka Streams' `KTable` and is the same idea as an LSM tree's tombstone-and-compaction cycle from Day 21, just applied to a stream instead of a key-value store.

## 5. The trade-offs

CAP made concrete, per data type, not per system:

- **The event itself, once acknowledged (durability over the write path): consistency wins.** With `acks=all` and `min.insync.replicas=2` on a 3-replica partition, a producer's write is refused rather than accepted-then-lost if the ISR has shrunk to one broker. That is choosing to fail loudly (unavailable for that specific write) over silently accepting a write that a single disk failure could erase. This is the exact same choice Day 6's Stripe idempotency lesson makes about a charge: better a rejected request than an unrecoverable ambiguous state.

- **Consumption and ordering (the read path, across partitions): availability wins.** A consumer group keeps reading whichever partitions are healthy even if one partition's leader election is mid-flight; there is no global lock that freezes the whole topic while one partition recovers. The cost is that Kafka gives you ordering only within a partition, never a total order across the whole topic, a permanent trade LinkedIn accepted on day one rather than paying the coordination cost of total ordering at trillions of messages a day.

- **Cost vs latency: retention window is the dial.** Every extra day of retention is disk multiplied by every replica multiplied by every partition; LinkedIn and Netflix both tune retention aggressively per topic (seconds for high-volume clickstream vs indefinite for compacted state topics) because "keep everything forever, just in case" is not a free assumption once you are ingesting petabytes a day (Netflix TechBlog reports ~3 PB ingested and ~7 PB emitted daily on their Keystone Kafka pipeline).

- **Partition count is a trade-off in itself.** More partitions means more producer/consumer parallelism, but each partition costs the cluster open file handles, replication traffic, and controller metadata (recall LinkedIn's ~7 million partitions across ~4,000 brokers is itself a capacity-planning artifact, not a free lunch); Confluent's own sizing guidance is explicit that partition count should be chosen deliberately, not maximized.

## 6. The systems-thinking lens

The feedback loop that actually causes production pain here is a **rebalancing storm**, a close cousin of the retry death spiral and thundering herd from earlier lessons, but triggered by group membership churn instead of client retries.

The loop: a rolling deploy restarts consumers one at a time. Each restart triggers a group rebalance. Under the older "stop-the-world" (eager) rebalance protocol, *every* consumer in the group, not just the one restarting, has to stop processing, give up all its partitions, and wait to be reassigned. If the deploy restarts consumers faster than rebalances can finish, or if a slow consumer misses its heartbeat while paused mid-rebalance (which the coordinator reads as "dead," triggering *another* rebalance), the group can spend more time rebalancing than actually consuming, lag balloons, and the backlog itself makes each subsequent poll slower, which triggers more missed heartbeats. This exact failure mode, restarts causing rebalances causing missed heartbeats causing more rebalances, is documented as a real production outage pattern (Michal Drozd, "Kafka Consumer Rebalance Storms: Why Scaling Consumers Can Increase Lag").

This is a metastable failure: the system is stable at normal load, and stable again once the churn stops, but a transient trigger (the deploy) pushes it into a self-sustaining bad state that does not resolve on its own just because the original trigger passed.

The senior fix is not "add more consumers" (that makes rebalances *more* expensive, since more members means more to reassign). It breaks the loop structurally:

- **Cooperative/incremental rebalancing (KIP-429, and KIP-848's fully server-side incremental protocol)** stops treating a rebalance as "everyone drops everything." Only the specific partitions that actually need to move, move; every consumer that is unaffected keeps consuming through the rebalance. Instaclustr's benchmarking of KIP-848 reports rebalances up to 20x faster, precisely because the blast radius of one consumer joining or leaving no longer scales with the whole group's size.
- **Session and heartbeat timeouts tuned to the real processing time**, so a consumer doing slightly slow work is not mistaken for a dead one and evicted, which would otherwise manufacture the very churn that starts the loop.
- **Deploy discipline**: rolling restarts paced slower than the group's rebalance-and-stabilize time, rather than as fast as the orchestrator allows, is the same "add backpressure at the source" instinct as Day 13's load shedding, applied to membership changes instead of request volume.

The general lesson: whenever a system's response to churn is itself more churn, the fix is never "more capacity" (that only raises the ceiling the loop crashes into later); it is changing the *mechanism* so a local disturbance produces a local, bounded response instead of a global, self-amplifying one.

---

## Sources

- LinkedIn Engineering, ["How LinkedIn customizes Apache Kafka for 7 trillion messages per day"](https://www.linkedin.com/blog/engineering/open-source/apache-kafka-trillion-messages)
- Confluent, ["Apache Kafka Hits 1.1 Trillion Messages Per Day"](https://www.confluent.io/blog/apache-kafka-hits-1-1-trillion-messages-per-day-joins-the-4-comma-club/)
- Confluent, ["How to Choose the Number of Topics/Partitions in a Kafka Cluster"](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/)
- Confluent docs, ["Choose and Change the Partition Count in Kafka"](https://docs.confluent.io/kafka/operations-tools/partition-determination.html)
- Confluent Cloud docs, ["Cluster Types"](https://docs.confluent.io/cloud/current/clusters/cluster-types.html)
- Apache Kafka documentation, ["Replicated Log design"](https://kafka.apache.org/documentation/#design_replicatedlog)
- Apache Kafka KIP-429, ["Kafka Consumer Incremental Rebalance Protocol"](https://cwiki.apache.org/confluence/display/KAFKA/KIP-429:+Kafka+Consumer+Incremental+Rebalance+Protocol)
- Apache Kafka KIP-848, ["The Next Generation of the Consumer Rebalance Protocol"](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A+The+Next+Generation+of+the+Consumer+Rebalance+Protocol)
- Instaclustr, ["Rebalance Your Apache Kafka Partitions with the Next Generation Consumer Rebalance Protocol"](https://www.instaclustr.com/blog/rebalance-your-apache-kafka-partitions-with-the-next-generation-consumer-rebalance-protocol/)
- Michal Drozd, ["Kafka Consumer Rebalance Storms: Why Scaling Consumers Can Increase Lag"](https://www.michal-drozd.com/en/blog/kafka-consumer-rebalance-storm/)
- Confluent docs, ["Kafka Log Compaction"](https://docs.confluent.io/kafka/design/log_compaction.html)
- Confluent Developer, ["Stateful Fault Tolerance" (Kafka Streams changelog topics)](https://developer.confluent.io/courses/kafka-streams/stateful-fault-tolerance/)
- Uber Engineering, ["Disaster Recovery for Multi-Region Kafka at Uber"](https://www.uber.com/us/en/blog/kafka/)
- Netflix TechBlog, ["Kafka Inside Keystone Pipeline"](http://techblog.netflix.com/2016/04/kafka-inside-keystone-pipeline.html)
- Netflix TechBlog, ["How and Why Netflix Built a Real-Time Distributed Graph"](https://netflixtechblog.com/how-and-why-netflix-built-a-real-time-distributed-graph-part-1-ingesting-and-processing-data-80113e124acc)
- Kreps, Narkhede, Rao, ["Kafka: a Distributed Messaging System for Log Processing," NetDB 2011](https://notes.stephenholiday.com/Kafka.pdf)
