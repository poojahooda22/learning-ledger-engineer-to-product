# Day 10 — How do you add a 13th database node without reshuffling 75% of your data?

**Date:** 2026-06-21
**Difficulty:** Advanced (first topic beyond the base curriculum)
**Stack relevance:** Supabase Postgres, Cloudflare R2, Redis, distributed caching, Cassandra, ScyllaDB

---

## 1. Named Company + The Breaking Number

**Discord. 1 billion messages per day. 177 Cassandra nodes by 2022.**

The naive formula for distributing data across N database nodes is:

```
shard = hash(key) % N
```

You have 12 nodes in 2017. You add a 13th. Now the formula is `hash(key) % 13` instead of `hash(key) % 12`. For almost every key, the answer changes. You need to move roughly **92%** of all your data between nodes, while the system is live, under user load.

With Discord storing billions of messages across 12 nodes, adding one node triggers roughly 920 million rows of live data transfer. Engineers were manually rebooting Cassandra nodes to survive the GC pauses and migration-induced overload. That is the cost of the naive design.

**The breaking number: 92% of data moves when you add just one node to a 12-node cluster using modular hashing.**

---

## 2. Why the Naive Design Dies

The naive design is `shard = hash(key) % N`.

### It collapses in three concrete ways:

**a) Every scaling step reshuffles almost everything.**

Change N from 12 to 13. The formula `hash(key) % 12` and `hash(key) % 13` give different answers for 11 out of 12 keys. Those 11 need to move. Live. Under traffic. While the nodes doing the migration are the same nodes serving real users.

**b) Migration causes a thundering herd.**

The data migration is CPU-heavy, network-heavy, and disk-heavy. The nodes migrating data are also serving live reads and writes. Latency spikes. Clients retry. Retries add more load. Load causes more latency. This is a death spiral. Discord's on-call engineers were dealing with GC pauses and migration overload simultaneously. They were physically rebooting nodes.

**c) One hot key takes down one shard.**

Consistent hash-based sharding distributes keys evenly. It does NOT distribute access load evenly. When a popular Discord server with 500,000 members all talk in the same channel simultaneously, every message request hashes to the same `channel_id`, which maps to one node. That node handles 100x the load of every other node. Modular sharding has no mechanism to say "this key needs to live on multiple nodes." One shard becomes a bottleneck and drags cluster-wide latency with it.

---

## 3. The Real Architecture: The Hash Ring

### Step 1 — Replace the line with a ring

Instead of a number line `0 to N-1`, picture a circle. The full circumference represents a hash space of `0` to `2^32` (about 4 billion positions). This is called the **hash ring** or **consistent hash ring**.

```
                      0 / 2^32
                       |
              Node A ---+--- Node B
             /                      \
            /                        \
    Node D --------         ---------- (empty)
            \                        /
             \                      /
              Node C ---------------
                       |
                     2^31
```

**Nodes are placed on the ring.** Hash each node's identifier (name or IP) to a position. Node A at 100M. Node B at 1.2B. Node C at 2.4B. Node D at 3.6B.

**Keys are also placed on the ring.** Hash each key (e.g., `channel_id=12345`) to a position. To find which node owns a key: start at the key's position, walk clockwise, and stop at the first node you hit. That node is responsible for this key.

**The critical property: adding a node only disturbs its immediate clockwise neighbor.**

If you insert Node E at position 700M (between A at 100M and B at 1.2B), only the keys between 100M and 700M move from Node B to Node E. Node A, C, and D keep all their existing keys untouched.

**Adding 1 node to a 100-node ring moves only about 1% of keys, not 99%.**

### Step 2 — Full architecture, top to bottom

```
[Clients]
    |
    | GET/POST channel_id=12345
    v
[API Routing Layer]
    |  Knows the current ring state.
    |  Computes: hash("12345") -> position 847M on ring
    |  Looks up: which node owns position 847M?  -> Node 7
    |
    v
[Node 7 — Primary Shard]
    |  Owns the key range containing position 847M.
    |  Accepts the write first.
    |
    +---> [Node 8 — Replica 1]   next node clockwise
    |
    +---> [Node 9 — Replica 2]   next-next node clockwise
```

**What each layer does in plain language:**

- **Routing layer** — The phone switchboard. Knows which desk (shard) handles your call (key). Stateless, fast, just runs the ring formula.
- **Primary shard** — The desk that owns your account. First to accept writes.
- **Replicas** — Filing cabinets with photocopies. If the primary dies, promote a replica. Your data survived.

### Step 3 — Virtual nodes fix the imbalance

With 4 physical nodes placed once each on the ring, you get 4 arcs of unequal size. Node A might own 40% of the ring. Node C might own 10%. One node carries 4x the load of another.

The fix: each physical node gets placed 100 to 200 times on the ring. Each copy is a **virtual node** (vnode).

With 150 vnodes per physical node:
- 4 physical nodes produce 600 positions on the ring.
- Each physical node statistically ends up owning about 25% of the ring.
- When Node A goes down, its 150 virtual positions scatter the freed-up load across all 3 remaining nodes evenly. Not all onto one unlucky neighbor.

**The analogy:** vnodes are like splitting one busy cashier into 150 short shifts scattered across all checkout lanes. The queue distributes itself.

Cassandra production deployments use 256 vnodes per node by default. Discord ran with the Cassandra defaults and tuned from there.

### Step 4 — What Discord actually built on top of this

Discord does not put clients on the Cassandra ring directly. They added a "Data Service" tier in Rust between the API and Cassandra. That tier does two things:

1. **Consistent hash routing** — All requests for a given `channel_id` go to the same Data Service instance. That instance is the only one querying Cassandra for that channel.
2. **Request coalescing** — If 10,000 concurrent requests arrive for `channel_id=12345` within the same 50ms window, the Data Service issues ONE query to Cassandra and fans the response out to all 10,000 waiters. The DB sees 1 read, not 10,000.

This is why their p99 read latency dropped from 40-125ms (Cassandra with GC pressure) to 15ms (ScyllaDB with the Data Service in front). The ring did not change. The engine and the routing tier changed.

---

## 4. The Transferable Mechanisms

These are the reusable patterns regardless of whether you use Cassandra, Redis, or Postgres.

**a) Hash ring instead of modulo**

Formula change: `shard = first_node_clockwise(hash(key))` instead of `hash(key) % N`. This one change reduces data movement from O(K) per resize to O(K/N) per resize. At 1 billion keys across 100 nodes, adding 1 node moves 10 million keys instead of 990 million.

**b) Virtual nodes for statistical balance**

V vnodes per physical node. V = 100 to 200 in production. Higher V = more even balance = more metadata overhead. Most teams start with 100 and raise if they see skew. Never go below 50. The imbalance at V=1 is too severe.

**c) Replication factor for durability**

Every key is stored on the next RF nodes clockwise from its hash position. RF=3 is the industry default: tolerate 2 simultaneous node deaths. RF is about survival. Sharding is about capacity. They are different levers. Discord ran RF=3 on Cassandra.

**d) Quorum reads/writes for consistency**

With RF=3, a write is durable when 2 of 3 replicas confirm it. A read queries 2 of 3 replicas and picks the one with the newer timestamp if they disagree. This is "quorum" (W=2, R=2, W+R > RF). You get durability without waiting for all 3 replicas on every write. If one replica is slow, you still complete in under 5ms.

**e) Hotkey isolation — ring is not enough**

The ring distributes keys evenly. It cannot distribute access load evenly. Solutions for hot keys, in order of complexity:

- **Read replicas** — Route all reads to replicas, writes to primary only. RF=3 already gives you 3x read throughput for free if you use them.
- **Request coalescing** — Discord's approach. One in-flight DB request per hot key. All concurrent requests wait and share the answer.
- **Dedicated shard** — Move the celebrity key to a shard that handles nothing else. Instagram does this for accounts with 100M+ followers.
- **Compound shard key** — Hash on `user_id + date` instead of just `user_id`. Spreads one user's data across multiple shards over time at the cost of cross-shard queries for "show me everything."

**f) Gossip protocol for ring membership**

Nodes must agree on who is on the ring. Production systems use gossip: every node tells 2-3 random neighbors its view of ring membership every 1-2 seconds. Within a few rounds, all nodes converge. No central coordinator. No single point of failure. Cassandra uses this. So does Dynamo.

---

## 5. The Trade-offs

### CAP made concrete for Discord's data types

| Data type | AP or CP | Why they chose this |
|---|---|---|
| Message content | AP (availability preferred) | A message appearing 200ms late is fine. Service outage is not. |
| Message order within a channel | Tunable (W=2, R=2) | Strong enough for ordering, still survives 1 node death |
| User presence / online status | AP (eventual) | Wrong "online" label for 2 seconds is tolerable |
| Payment record (Stripe, not Discord) | CP | Wrong ledger entry is catastrophic. Latency is acceptable. |

### Cost vs latency

More replicas = more read throughput but more write overhead. RF=3 doubles your storage cost vs RF=1 and adds 2x write network overhead. That is the explicit trade: pay more to survive failures without downtime.

More vnodes = more even balance but more metadata. A 1,000-node cluster with 256 vnodes per node tracks 256,000 ring positions. That fits easily in memory (a few MB). Not a real concern until you hit tens of thousands of nodes.

---

## 6. The Systems-Thinking Lens

### The failure loop: rebalance thundering herd

1. Node 7 slows down. Its p99 latency climbs to 500ms.
2. The load balancer sees it as "slow" and marks it unhealthy.
3. Node 7's 150 vnodes become unowned. The routing layer shifts all those requests to Node 7's clockwise neighbors on the ring.
4. Those neighbors are now handling 2x their normal load. Their latency climbs.
5. The load balancer marks them slow too.
6. Cascade. You lose the cluster one node at a time. The failure originated in one slow node.

**This is a metastable failure.** The system reached a state where healing itself makes things worse. Sending traffic away from the sick node overloads the healthy nodes. Adding more nodes starts more migrations that add more load.

### The senior fix: break the loop, not just add capacity

Adding more nodes under cascade adds more rebalancing work. You make it worse.

**Fix 1 — Backpressure before the ring.**
Put a queue (SQS, Kafka, Cloudflare Queue) between the API tier and shard writes. The queue absorbs the spike. Workers pull from the queue at the rate shards can handle. The ring never sees the burst.

**Fix 2 — Circuit breaker per shard.**
If shard 7 fails 5 requests in 10 seconds, open the circuit: return a cached or stale response instead of forwarding more requests. The sick node is quarantined. Neighbors are protected. Netflix's Hystrix and Resilience4j implement this.

**Fix 3 — Shed load at the router, not the node.**
The routing layer knows per-shard capacity. If shard 7 is at 85% utilization, the router starts rejecting or deferring new requests at the routing tier — before they hit the shard. The node never becomes overwhelmed.

**Fix 4 — Read from any replica, not just the primary.**
Hot read keys are the most common cause of single-shard overload. If your ring sends all reads to the primary, you are wasting RF-1 replicas. Route reads to any replica. Discord does this in the Data Service layer.

**The principle:** Fix the feedback mechanism, not the symptom. The retry storm is a loop. A circuit breaker breaks the loop. Adding nodes does not.

---

## Rare.lab Stack Mapping

**What you already do right:**

Cloudflare R2 stores content-addressed immutable scene JSON. Content-addressing IS consistent hashing's core idea applied to objects: `address = hash(content)`. The address is deterministic, globally unique, and inherently distributable. R2 handles the sharding of those objects invisibly. You already benefit from this without thinking about it.

**Where your next ceiling is:**

Supabase Postgres is a single primary writer. At your current scale this is fine. The ceiling appears when:
- Concurrent shader compile jobs hit thousands per minute
- Scene manifests grow to millions of rows with heavy concurrent reads

**What to do now (before you need sharding):**

Add a Supabase read replica. Route all scene manifest reads, asset lookups, and user profile queries to the replica. Write operations (saving a project, updating a manifest) go to the primary only. This gets you 5-10x more read headroom without any schema changes or application sharding logic.

**If you hit the write ceiling after replicas:**

Shard by `user_id` or `project_id` range. Do NOT use `project_id % N`. Use a hash ring at the application layer or route through a proxy (PgBouncer or Pgpool with shard awareness) that knows the ring layout. The lesson from Discord: design the routing layer to be separate from the storage layer. Then you can change either independently.

**R2 does not need this work.** Object storage is content-sharded by design and scales to exabytes. Your content-addressed design is the correct architectural call.

---

## References and Summaries

Every resource below is worth your time. Summary is included so you know what to expect before you open it.

---

### Research Papers

**1. "Consistent Hashing and Random Trees" — Karger et al., MIT, 1997**
The original paper. Proves mathematically that consistent hashing reduces key movement during a resize from O(K) in naive modular hashing to O(K/N). Dense mathematics but the Abstract and Section 2 are readable by any engineer. The key insight: arrange the hash space as a circle, not a line.
URL: https://dl.acm.org/doi/10.1145/258533.258660

**2. "Dynamo: Amazon's Highly Available Key-value Store" — DeCandia et al., Amazon / SOSP 2007**
The paper that made consistent hashing mainstream outside academia. Amazon built Dynamo so the shopping cart would always be writable, even during partial outages. Section 4 covers the ring and virtual nodes step by step. Section 6 is the most valuable: it covers what they observed in production and why they added vnodes. Read Section 6 first if you are short on time.
URL: https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html

**3. "Sharding Distributed Databases: A Critical Review" — arXiv, 2024**
Academic review of every sharding strategy in use today: range-based, hash-based, consistent hash-based, and emerging learned-hash approaches (where ML predicts access patterns to pre-place hot keys on their own shards). Useful for understanding where the field is heading beyond the hash ring.
URL: https://arxiv.org/pdf/2404.04384

---

### Engineering Blogs

**4. "How Discord Stores Trillions of Messages" — Discord Engineering Blog**
The canonical case study for this lesson. Covers the full arc: 12 Cassandra nodes in 2017, 177 nodes in 2022, migration to ScyllaDB. Explains the hot partition problem directly, the Data Service layer they built (consistent hash routing + request coalescing), and the before/after latency numbers. The p99 read latency chart alone is worth the read. This is primary source material.
URL: https://discord.com/blog/how-discord-stores-trillions-of-messages

**5. "How Discord Solved the Hot Partition Problem" — Engineering at Scale (Substack)**
Focused deep-dive on the celebrity problem inside Discord. When a 500k-member server all hit the same channel, one node received all the traffic. This post explains request coalescing in detail: if 10,000 requests for `channel_id=12345` arrive in 50ms, only 1 goes to Cassandra. The other 9,999 wait and share the result. Concrete, short, and directly applicable to any system with read-heavy hot keys.
URL: https://engineeringatscale.substack.com/p/how-discord-solved-hot-partition-problem

**6. "Amazon's Dynamo" — Werner Vogels, All Things Distributed**
Werner Vogels (Amazon CTO) explains the business motivation behind Dynamo: the shopping cart must always accept writes, even when nodes fail. The tradeoff he accepted was eventual consistency for cart data. Three paragraphs of clear thinking from the architect. Read this before the full paper.
URL: https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html

**7. "You Sharded Your Database. Now One Shard Is on Fire" — DEV Community**
Honest post about what actually goes wrong after sharding is set up. Covers the celebrity problem with a concrete example (Taylor Swift's account generating 100x normal traffic). Solutions: dedicated celebrity shards, compound shard keys, automatic shard splitting. Written by an engineer who hit these problems, not a textbook author describing them abstractly.
URL: https://dev.to/jinpyo181/you-sharded-your-database-now-one-shard-is-on-fire-1p7h

**8. "Mastering Consistent Hashing: Dynamo, Cassandra, Redis Cluster" — Medium**
Side-by-side comparison of how three major systems implement consistent hashing differently. Cassandra uses a continuous ring with 256 vnodes per node. Redis Cluster uses 16,384 fixed hash slots with CRC16(key) mod 16384 assigned as slot ranges to nodes. Dynamo uses a ring with preference lists for replication ordering. Reading all three implementations in one article saves significant time compared to reading three separate blog posts.
URL: https://medium.com/@krnandhakumar/from-chaos-to-consistency-how-consistent-hashing-keeps-distributed-systems-scalable-61b150635003

---

### Video Resources

**9. "Consistent Hashing | Algorithms You Should Know #1" — ByteByteGo, YouTube**
Alex Xu animates the hash ring being built from scratch. Covers why modular hashing breaks, how to draw the ring, how virtual nodes are added, and how to look up a key. Under 10 minutes. Best starting point if you prefer to see the ring move before reading about it.
URL: https://www.youtube.com/watch?v=UF9Iqmg94tk

**10. "Consistent Hashing: Easy Explanation for System Design Interviews" — YouTube**
Interview-focused walkthrough. Covers how to draw the ring in an interview, explain vnodes to a skeptical interviewer, and answer the follow-up question about hot keys. Good for solidifying the mental model after reading the Discord blog post.
URL: https://www.youtube.com/watch?v=vccwdhfqIrI

**11. "Basics of Consistent Hashing in Plain English" — YouTube**
Starts from zero. Builds from "why hash anything" to "what breaks with modular hashing" to "the ring." Roughly 15 minutes. Best option if the Dynamo paper feels too dense and you want a spoken explanation before you read.
URL: https://www.youtube.com/watch?v=oKAU6LaYFhw

---

### Articles

**12. "Consistent Hashing for System Design Interviews" — Hello Interview**
The most practically useful web explainer. Covers the ring, vnodes, replication factor, and has a dedicated section on why Redis Cluster chose fixed slots instead of a continuous ring (operational simplicity: every admin knows which slot holds a key without running a hash function). Also covers when consistent hashing is NOT the right tool. Clear diagrams throughout.
URL: https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing

**13. "Consistent Hashing 101: How Modern Systems Handle Growth and Failure" — ByteByteGo Blog**
Written companion to video #9 above. Covers what happens step by step during a node failure: which vnodes go unowned, how the ring reroutes, how replication ensures data is not lost. Good reference to have open alongside the Discord blog post.
URL: https://blog.bytebytego.com/p/consistent-hashing-101-how-modern

**14. "Sharding in System Design Interviews" — Hello Interview**
Broader than consistent hashing. Covers range-based sharding, hash-based sharding, and directory-based sharding side by side. Explains when to use each and the celebrity/hotkey problem for each approach. Includes the compound shard key pattern and how databases like Vitess (YouTube's MySQL sharding layer) implement it in practice.
URL: https://www.hellointerview.com/learn/system-design/core-concepts/sharding
