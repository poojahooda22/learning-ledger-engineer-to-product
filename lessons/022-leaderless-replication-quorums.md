# Day 22 — How does a database stay writable when there is no leader and a data center just went dark?

*2026-07-03*

## 1. Named company + the breaking number

Amazon built Dynamo because the shopping cart kept falling over. The team's own account (the 2007 Dynamo paper, DeCandia et al.) is blunt about the requirement: a customer must **always** be able to add an item to their cart. Not "usually." Always. A cart that occasionally refuses a write during a Tuesday-afternoon network blip is a cart that loses a sale.

Here is the number that breaks a single-leader design. In July 2014, Netflix published a benchmark titled ["Revisiting 1 Million Writes per Second"](http://techblog.netflix.com/2014/07/revisiting-1-million-writes-per-second.html): a single logical Cassandra cluster, 300 nodes, spread across 3 AWS regions, sustaining **1,000,000 writes per second**. Today Netflix runs Cassandra across hundreds of clusters globally and reports roughly 1 trillion requests a day on the platform.

Route that same load through one elected leader, whether it is a single Postgres primary or a Raft-elected leader like the one built in Lesson 011, and the ceiling is not 1,000,000 writes/sec. It is whatever one machine's disks, NIC, and WAL fsync can absorb before it falls over, commonly somewhere in the 10,000-50,000 writes/sec range for a well-tuned single primary. And every one of those writes, if it originates in Europe or Asia, first has to physically travel to wherever that one leader lives before it is even acknowledged.

## 2. Why the naive (demo) design dies

The naive version: one primary database. Every write, from every region, goes to it. Replicas exist only as read copies or as failover standbys.

Three concrete ways this collapses at Netflix/Amazon scale:

- **Leader death stalls everything, everywhere.** Even a fast Raft election (Lesson 011) takes hundreds of milliseconds to a few seconds. During that window, all writes system-wide are blocked, not just writes to the record that was "affected." One receptionist has to personally approve every hotel check-in across a hundred branches worldwide; you call her desk phone for each one. If her line goes dead, nobody anywhere can check in, even at branches where every other clerk is standing around idle.
- **Geography tax on every write.** If the leader sits in `us-east-1`, a write from a shopper in Mumbai pays a ~200ms round trip before the leader even sees it, every single time, forever. There is no way to make that faster without moving the leader, and you only have one.
- **A network partition makes healthy nodes lie idle.** If a fiber cut isolates `eu-west` from `us-east` and the leader is in `us-east`, the European side cannot write at all, even to update completely unrelated data, even though every European node is fully healthy. This is CAP in its rawest form: the system chose consistency (one authoritative copy) and paid for it with availability during the exact moment it was needed most.

## 3. The architecture, top to bottom

This is Dynamo/Cassandra-style **leaderless replication**. No node is special. Any node can act as coordinator for any request.

```
CLIENT (e.g. "add running shoes, size 9, to cart")
   |
   v
COORDINATOR NODE  -- any node in the cluster, picked by the client driver
   |  job: routes this one request; not a fixed leader, just whoever picks up the phone
   |  analogy: a rotating on-call clerk, not a dedicated receptionist
   v
CONSISTENT HASHING RING  -- partitions the whole keyspace across all nodes
   |  job: maps "cart:pooja123" to a fixed point on a ring; the next N nodes clockwise
   |       from that point are this key's home addresses (its "preference list")
   |  analogy: a rolodex where every possible key has a fixed page, and that page's
   |       tab always points to the same 3 physical drawers, wherever they live today
   |  (this is the same ring mechanism as Lesson 010, reused for replica placement)
   v
N REPLICAS (say N=3)  -- three nodes hold a copy of this exact record
   |  job: durability through redundancy, not through one machine's disk
   |  analogy: three separate filing cabinets, not one cabinet with a spare key
   v
WRITE QUORUM (W=2 of 3)  -- coordinator waits for W acks, not all N
   |  job: "good enough" durability without waiting for the slowest replica
   |  analogy: two of three shelves confirm the note is filed; don't wait on the third
   v
SLOPPY QUORUM + HINTED HANDOFF  -- if a home node is down, write lands on the next
   |  healthy node on the ring instead, tagged "this belongs to node X"
   |  job: a write is never blocked by one temporarily-dead home node
   |  analogy: substitute teacher holds the absent teacher's homework pile, hands it
   |       back the moment the regular teacher returns
   v
READ QUORUM (R=2 of 3) + READ REPAIR  -- coordinator asks R nodes, compares versions
   |  job: overlap between what you wrote and what you read (R+W > N guarantees this)
   |  if replicas disagree, the newer version is pushed to the stale one, right then
   |  analogy: ask two shelves for the file; if they disagree, quietly correct the
   |       older one on the spot instead of waiting for a scheduled audit
   v
ANTI-ENTROPY / MERKLE TREE SYNC  -- background job, runs whether or not anyone reads
   |  job: diffs replicas hash-tree by hash-tree, catches drift that no read ever
   |       happened to touch (a cold key nobody queried in weeks)
   |  analogy: an auditor comparing shelf checksums overnight, never trusting live
   |       reads alone to fully heal the system
   v
VECTOR CLOCK / CRDT CONFLICT RESOLUTION  -- when two writes happen concurrently, with
   |  neither replica able to see the other's write yet, merge instead of guessing
   |  job: distinguish "A happened before B" from "A and B happened at the same time"
   |  analogy: the shopping cart takes the union of both versions; a deleted item
   |       might briefly reappear, but nothing a customer added is ever silently lost
```

## 4. The transferable mechanisms

- **Quorum reads/writes (R + W > N).** As long as the set of replicas you read from and the set you wrote to are guaranteed to overlap by at least one node, you never need every replica to be up to make progress. This is the core trick that lets you trade "all N must respond" for "any majority-ish subset will do."
- **Consistent hashing for partitioning and replica placement.** Same ring from Lesson 010, now doing double duty: it decides which node owns a key AND which N nodes hold its replicas (the "preference list"). Add or remove a node and only its neighbors on the ring reshuffle, not the whole cluster.
- **Vector clocks for causality without a global clock.** A per-key, per-replica counter that lets the system say "version B is a descendant of version A" (safe to discard A) versus "A and B are concurrent siblings" (must merge, cannot pick one and throw the other away). This is what makes the Dynamo paper's shopping-cart merge possible.
- **Sloppy quorum + hinted handoff.** Never let a write block because the "correct" home node is briefly unreachable. Borrow the next healthy node on the ring, stash a hint, hand the write back later. Cassandra bounds this: hints are held for up to `max_hint_window` (3 hours by default) and then dropped, because an unbounded promise to "deliver eventually" is itself a liability.
- **Read repair vs. anti-entropy: two healing loops, not one.** Read repair fixes drift only on keys that happen to get read, cheaply, as a side effect of normal traffic. Merkle-tree anti-entropy is the slower, thorough backstop that fixes cold keys nobody queried. You need both; read repair alone leaves silent gaps.
- **Merge policy is a per-data-type decision, not a global one.** A cart quantity can safely be a CRDT-style union (Lesson 015). A bank balance or an oversell-preventing inventory counter (Lesson 016's single-owner atomic decrement) cannot: those need a single point of arbitration. Leaderless replication is the right tool for one and the wrong tool for the other, on purpose.

## 5. The trade-offs

CAP made concrete, per data type:

- **Cart contents: choose AP.** Availability wins. The failure mode of choosing consistency here (blocking writes during a partition) is worse than the failure mode of choosing availability (occasionally showing two near-duplicate line items that a background merge quietly reconciles). The merge is cheap, visible, and reversible; a blocked "add to cart" during a holiday sale is a lost sale that never comes back.
- **Money and inventory counts: this model is a bad fit, intentionally.** This is exactly why Lesson 020 exists (2PC/sagas for cross-shard transactions) and why Lesson 016's oversell-prevention counter routes through a single owning node rather than a leaderless quorum. Choosing AP for a ledger balance means two replicas can each honestly believe they hold the "true" balance after a partition, and there is no clean CRDT merge for "who actually gets the last unit of stock."
- **Cost vs. latency.** N=3 replication triples storage cost versus a single-copy system, and anti-entropy repair runs continuously in the background, consuming bandwidth around the clock. In exchange, R=2/W=2 out of N=3 means the coordinator only waits for the two fastest of three replicas, masking the one slow straggler from client-facing p99 latency. You are paying storage and bandwidth to buy both availability and a better tail latency.

## 6. The systems-thinking lens

The feedback loop here is **hinted-handoff pile-up**, a recovering-node overload loop that is specific to this architecture, distinct from the generic backpressure story in Lesson 013.

- A node goes down not for seconds but for tens of minutes (a deploy, an AZ blip, a slow GC pause). Every neighbor on the ring keeps accepting sloppy-quorum writes on its behalf and stockpiling hints, because that is exactly what the mechanism is designed to do.
- The moment the dead node comes back online, cold JVM, cold page cache, no warmup, every one of those neighbors tries to replay its backlog of hints onto it at once. Cassandra's own documentation calls this out by name: "one of the drawbacks of the hinted handoff pattern is the potential thundering herd problem when the backup node tries to quickly replay hints on a newly returned target node." A node can accumulate gigabytes of hints during an outage, and a naive full-speed replay can take the node right back down, at the exact moment it was supposed to be recovering.
- That is a self-reinforcing flap loop: node down, hints pile up elsewhere, node returns, hint flood knocks it back down, more hints pile up somewhere else, repeat.
- The senior fix does not add capacity. It breaks the loop with two levers that already exist in production Cassandra: a bounded hint window (`max_hint_window`, 3 hours default, after which the coordinator gives up on hints and falls back to full Merkle-tree repair instead of an unbounded promise) and a throttled replay rate (`hinted_handoff_throttle`, historically defaulting to a conservative ~1024 kbps per node) so the recovering node gets its backlog back gradually instead of all at once. The trade is a slower, paced convergence in exchange for never re-crashing the node you just brought back.

## Sources

- [Dynamo: Amazon's Highly Available Key-value Store (DeCandia et al., 2007)](https://www.cs.cornell.edu/courses/cs5414/2017fa/papers/dynamo.pdf) — the original paper: N/W/R quorums, sloppy quorum, hinted handoff, vector clocks, the shopping-cart merge example.
- [Amazon's Dynamo — All Things Distributed (Werner Vogels, 2007)](https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html) — plain-language walkthrough from Amazon's own CTO.
- [Netflix Tech Blog: Revisiting 1 Million Writes per Second (2014)](http://techblog.netflix.com/2014/07/revisiting-1-million-writes-per-second.html) — the 300-node, 3-region, 1M writes/sec Cassandra benchmark used as this lesson's breaking number.
- [Netflix Tech Blog: Benchmarking Cassandra Scalability on AWS (2011)](http://techblog.netflix.com/2011/11/benchmarking-cassandra-scalability-on.html) — earlier scale-out benchmark showing near-linear throughput growth by adding nodes.
- [Apache Cassandra Documentation: Hints](https://cassandra.apache.org/doc/4.0/cassandra/operating/hints.html) — `max_hint_window` default and hint mechanics.
- [The Last Pickle: Hinted Handoff and GC Grace Demystified (2018)](https://thelastpickle.com/blog/2018/03/21/hinted-handoff-gc-grace-demystified.html) — operational detail on hint replay and the thundering-herd risk on node recovery.
- [Bandwidth Required for Cassandra Hinted Handoff](http://www.uberobert.com/bandwidth-cassandra-hinted-handoff/) — the hint replay throttle default and why gigabytes of accumulated hints can take hours to drain.
