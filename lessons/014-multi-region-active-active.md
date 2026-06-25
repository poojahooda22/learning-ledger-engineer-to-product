# Day 14 — How does a global app accept writes from every continent without a 300ms penalty?

**Date:** 2026-06-25
**Difficulty:** Advanced
**Stack relevance:** Supabase Postgres, Cloudflare Workers, Cloudflare D1, distributed databases, CRDTs

---

## 1. Named Company + The Breaking Number

**Company: Discord**

Discord serves 560 million registered users and peaks at over 10 million concurrent connections during major game launches and esports events. The product promise is simple: send a message and your friend sees it in under 100ms, wherever on the planet they are.

The naive setup breaks at one number: **190ms.**

That is the one-way network latency from Tokyo to AWS us-east-1 (Virginia). It is not a server problem. It is a physics problem. The Pacific Ocean is 10,000 km wide. Light in fiber travels at roughly 200,000 km/s. Minimum transit: 50ms. Real internet routing (waypoints, BGP hops, cables that do not run in straight lines) pushes it to 190ms one-way, 380ms round-trip.

A user in Tokyo hits Enter to send a message. With a single Postgres primary in Virginia:
- Network out to Virginia: 190ms
- DB write + processing: 10ms
- Network back: 190ms
- **Total felt delay: ~390ms**

Discord's write latency target for message delivery is under 100ms end-to-end. A single-region primary forces 40% of their global users to start at 3-4x over budget before a single line of application code runs.

**The breaking number: 190ms. The one-way speed-of-light penalty from Tokyo to us-east-1 is 3.8x Discord's 50ms write-budget for the database tier. No hardware upgrade fixes an ocean.**

---

## 2. Why the Naive Design Dies

**The naive version: one Postgres primary in us-east-1, read replicas in Tokyo, London, and Sydney.**

Read replicas help reads. Reads are fast. But every write still crosses an ocean. This collapses in three ways.

**Collapse 1: Every write is a cross-ocean round trip.**
Discord's write QPS at peak is roughly 250,000 messages per second. Each one requires a synchronous round-trip to Virginia to confirm durability before the server can tell the client "delivered." You cannot cache your way around writes. The message must hit durable storage before the double-tick appears. Multiply 250,000 writes/sec by 390ms of round-trip and you have 97,500 writes "in flight" at any moment, each one burning a connection slot and a thread.

**Collapse 2: Read replicas lag at exactly the wrong moment.**
A Tokyo read replica might be 200-800ms behind the primary during network congestion. A user in Tokyo sends "ready?" and their friend's reply ("yes!") is already in Virginia, but the Tokyo replica has not caught up yet. The sender sees stale state for nearly a second after the reply was written. For presence, reactions, and typing indicators, 800ms of visible lag breaks the sense of real-time. Users think the product is broken. They are right.

**Collapse 3: One region down means the whole world is down.**
AWS us-east-1 had a DNS-triggered outage in October 2025. The failure started small: a DynamoDB endpoint became unreachable via DNS. AWS's own internal services (EC2 control plane, Lambda) depended on DynamoDB. They started failing. Customer SDKs retried. The retries overwhelmed the already-degraded DNS resolver, turning a local failure into a global retry storm that took down services across 60 countries for hours. Every company whose only primary was in us-east-1 went dark globally. Discord, which had already migrated to a multi-region architecture, continued serving Europe and Asia without interruption.

The core problem is not throughput. A well-tuned Postgres can handle 250k writes per second on beefy hardware. The problem is the speed of light, and no database on the market runs faster than physics.

---

## 3. The Architecture — Drawn Top to Bottom

**Active-Active means: every region can accept reads AND writes, independently, simultaneously.**

This is the opposite of Active-Passive (one primary, everyone else is a read-only standby). In Active-Active, every node is a branch office with full authority, not a photocopy machine.

```
[Users — Tokyo, London, Sao Paulo, New York, Sydney]
                          |
                          v
              [GeoDNS / Anycast Layer]
     Route each user to the nearest region by IP geolocation
     plus live latency probes. No user-side config needed.
     Analogy: a switchboard that knows where everyone lives
     and connects you to the nearest branch office,
     not the global headquarters.
                          |
       +------------------+------------------+
       |                  |                  |
 [Region: Tokyo]  [Region: Virginia]  [Region: Frankfurt]  ... [Sydney]
       |                  |                  |
 [Stateless App Tier] [Stateless App Tier] [Stateless App Tier]
  Accepts reads + writes  Accepts reads + writes  Accepts reads + writes
  from local users.       from local users.       from local users.
  Analogy: a full-service  Analogy: HQ. Same      Analogy: EU branch.
  branch bank. You can    powers. Different       Full authority.
  deposit AND withdraw.   address book.
       |                  |                  |
 [Regional DB Primary]  [Regional DB Primary]  [Regional DB Primary]
  Writes land here first. Writes land here first. Writes land here first.
  Committed immediately.  Committed immediately.  Committed immediately.
  Acknowledged to client. Acknowledged to client. Acknowledged to client.
       |                  |                  |
       +--[Cross-Region Async Replication]---+
          Writes propagate to other regions
          within 100ms to 2s.
          Uses a dedicated private backbone,
          NOT the public internet.
          Each write carries an HLC timestamp
          (Hybrid Logical Clock, explained below).
          Analogy: pneumatic tubes between bank branches.
          The deposit is committed. The ledger
          update travels in the background.
                          |
              [Conflict Resolution Layer]
       When two regions write the same row "simultaneously":
       - Last-Write-Wins (LWW) using HLC timestamps
       - CRDTs for counters, sets, and presence data
       - Regional leases for financially-sensitive records
       Analogy: two cashiers accidentally assign the same
       seat number. One rule settles the dispute, calmly
       and automatically, without asking the customer.
                          |
        [Circuit Breaker + Gradual Traffic Shifter]
       Monitors per-region error rate and latency.
       Shifts traffic 5% -> 10% -> 25% -> 50% on degradation.
       Rejects traffic early (503 + Retry-After) not late (500).
       Analogy: a water valve that turns slowly,
       not a light switch that flips instantly.
```

**Each layer in one line:**

| Layer | Single job | Analogy |
|-------|-----------|---------|
| GeoDNS / Anycast | Route user to nearest region | Switchboard operator |
| Stateless app tier | Accept writes locally, no coordination required | Branch bank teller |
| Regional DB primary | Commit write durably before acknowledging | Branch vault |
| Cross-region replication | Propagate committed writes to every other region | Pneumatic tubes between branches |
| Conflict resolution | Reconcile writes that arrived "simultaneously" | Branch manager rule book |
| Circuit breaker | Protect surviving regions during a neighbour's failure | Fuse box, not flood |

---

## 4. The Transferable Mechanisms

### 4a. GeoDNS and Anycast Routing

**Rule: Route the user to the nearest region, not the "correct" one.**

Cloudflare's Anycast gives 300+ cities the same IP address. BGP routing ensures a packet from Tokyo arrives at Tokyo's data center, not Virginia. Amazon Route 53 latency-based routing periodically probes real round-trip times from each AWS region to each IP prefix and updates routing tables. A user in São Paulo gets connected to sa-east-1. A user in Tokyo gets ap-northeast-1.

This step alone cuts felt latency from 300ms to under 30ms for the network hop, because now the app tier is physically close to the user.

Cost: essentially free beyond the base DNS pricing. This is the first thing you implement because it pays the biggest dividend.

### 4b. Async Cross-Region Replication (Not Synchronous)

**Rule: Commit locally first. Replicate after. Async means fast writes with eventual consistency.**

Synchronous replication means: before acknowledging a write to the client, wait for every other region to confirm they received it. This re-introduces the cross-ocean round-trip you just eliminated with GeoDNS. A Tokyo write that must synchronously confirm to Virginia is back to 380ms.

Async means: commit in the local DB immediately, acknowledge to the client in ~5ms, then ship the write to other regions in the background. Replication lag is typically 100ms to 2s depending on write volume and backbone congestion.

The trade-off: other regions briefly have stale data. For a chat message, "stale for 200ms" is invisible to humans. For a bank balance, it is not (handled separately with leases, section 4f).

CockroachDB's Raft replication between regions targets under 1s for typical inter-region replication. DynamoDB Global Tables targets under 1s for replication across regions with a 2s SLA. Cloudflare D1's read replication (March 2026) provides sequentially consistent reads against local replicas with writes still committed at the D1 primary.

### 4c. Hybrid Logical Clocks for Conflict Resolution

**Rule: When two regions write the same row "simultaneously," the one with the higher HLC timestamp wins.**

Wall clocks lie in distributed systems. Two servers calling `now()` at the "same moment" can disagree by up to 250ms even with NTP synchronization. If you use `updated_at TIMESTAMP` as a last-write-wins tiebreaker, a server with a fast clock silently wins every conflict, regardless of which write was actually more recent.

Hybrid Logical Clocks (HLC) fix this. Each HLC timestamp is a 64-bit value: the high bits are the physical wall-clock millisecond, the low bits are a logical counter. The logical counter increments whenever two events share the same physical millisecond. This preserves causal ordering (if event A happened before event B in the logical sense, A's HLC is lower than B's HLC) while staying close to wall-clock time so timestamps are human-readable.

Every write carries its HLC. Every replication message carries it. When Tokyo and Virginia both update the same user preference within 10ms of each other, the HLC on each write is compared and the higher one wins -- deterministically, without needing to ask any coordinator. CockroachDB, MongoDB (cluster time), Aurora DSQL, and Cloudflare D1's Sessions API all use HLC-based ordering.

### 4d. CRDTs for Naturally Conflict-Free Data

**Rule: For data that can be mathematically merged, use a CRDT so conflict resolution is a free operation.**

Some data types cannot conflict because their merge rule is part of their definition:

- **G-Counter (Grow-Only Counter):** Each region tracks its own increment count. The total is the sum of all regions' counts. Two regions can increment independently and the merge is just addition. Online user count, emoji reaction tallies, play counts -- all G-Counters.
- **OR-Set (Observed-Remove Set):** Each element has a unique ID. Adding an element adds the ID; removing it tombstones the ID. Two regions can add the same emoji reaction concurrently and the merged set contains both. This is how Figma handles concurrent node additions.
- **LWW-Register:** Each field in a record is a separate last-write-wins value with its own HLC. Two regions editing different fields of the same user profile can both win. Only simultaneous edits to the same field need tiebreaking. This is how Figma does property-level conflict resolution (see lesson 003).

CRDTs are why you can open the same Figma file in Tokyo and Berlin, move objects simultaneously, and have a mathematically guaranteed merge with no data loss.

### 4e. Gradual Traffic Shifting on Regional Failure

**Rule: When a region degrades, shift traffic 5% at a time. Never flip all traffic instantly.**

If Tokyo starts failing (DB OOM, disk full, network partition), the instinct is to immediately redirect all Tokyo traffic to Singapore. This is the mistake that turns a single-region outage into a multi-region cascade.

The correct pattern:
1. Detect degradation at 60% error rate (not 100%). At 100% it is already too late.
2. Shift 5% of Tokyo traffic to Singapore. Wait 30 seconds. Measure Singapore's error rate.
3. If Singapore is stable, shift 10%. Wait 60 seconds. Measure.
4. Continue: 25%, then 50%, then 100% -- each step gated on stability metrics.
5. Singapore must be running at 60% normal capacity (not 95%) so each 5% step lands in slack.

This is a traffic valve, not a light switch. The key is pre-warmed headroom. Every region runs at a maximum of 60-65% of its provisioned capacity in steady state. This costs 40-67% more compute, but it is the cost of true availability. A region that is always at 95% capacity cannot absorb anything.

### 4f. Fencing Tokens for Single-Owner Records

**Rule: For records that cannot tolerate any ambiguity (money, inventory), assign one region as the owner and use a fencing token to stop zombie writes.**

Active-Active with LWW conflict resolution is correct for most user-facing data. But for a bank debit or an inventory decrement, "last write wins" means double-charging a user or overselling a product. These records need a single authoritative writer.

The pattern: each record (or shard of records) is leased to one "owner" region for a TTL (say, 30 seconds). Other regions forward writes for that record to the owner rather than writing locally. When the owner region fails, the lease expires and a new owner is elected via Raft consensus (lesson 011). The fencing token is a monotonically-increasing epoch number that travels with every write. If a "zombie" message from the old owner arrives after the new owner is elected, the new owner checks: "is this token's epoch less than the current epoch?" If yes, the write is rejected. The token is the proof that a write is still valid.

This is how DynamoDB handles conditional writes (via `ConditionExpression: attribute_not_exists(pk)`) and how Stripe handles idempotency keys (lesson 012): the record is owned by one shard, every write goes through that shard, and the epoch fences out stale retries.

---

## 5. The Trade-offs

**CAP theorem made concrete, per data type:**

| Data type | Consistency need | Multi-region-write safe? | How |
|-----------|-----------------|--------------------------|-----|
| Chat messages | Eventual OK | Yes | Async replication + LWW with HLC |
| User presence (online/offline) | Eventual OK | Yes | CRDT G-Counter with TTL expiry |
| Emoji reactions | Eventual OK | Yes | OR-Set CRDT |
| User settings / profile | Eventual OK | Yes | LWW per field with HLC |
| Bank balance, subscription | Strongly consistent | No -- single-owner | Regional lease + fencing token |
| Auth tokens / sessions | Strongly consistent | No -- single-owner | One authoritative region or conditional writes |
| Inventory count at sale peak | Strongly consistent | No -- single-owner | Atomic decrement on lease owner (lesson 012) |

Discord's choice: **AP (Available + Partition-tolerant)** for all user-generated content. **CP (Consistent + Partition-tolerant)** for authentication and payments. The split is deliberate. Most of the product is in the AP bucket. Users do not notice if a reaction appears 300ms late on another continent. They do notice if they are double-charged.

**Cost vs. latency:**

| Setup | Write latency (Tokyo) | Availability | Monthly cost multiplier |
|-------|----------------------|-------------|------------------------|
| Single-region (us-east-1) | 380ms | 99.9% (one region's uptime) | 1x baseline |
| Active-Passive + read replicas | 380ms writes, 20ms reads | 99.95% | 1.4x |
| Multi-region active-active | 10-20ms writes | 99.99% | 2.5-4x |

Cross-region replication bandwidth costs $0.02-0.09/GB depending on cloud provider and regions. A chat app generating 10TB/day of write data pays $200-900/day just in replication egress. Multi-region is not always the answer. It is justified when write latency above 100ms produces visible user-facing degradation or when a regional outage is a business-critical event.

---

## 6. The Systems-Thinking Lens

**The feedback loop: Thundering Herd on Regional Failover**

The failure you are building multi-region to survive is not one region dying. It is the recovery attempt killing the surviving regions.

The loop, step by step:

1. **Tokyo fails at 3:00 AM.** DB OOM, or an upstream router, or a datacenter power event.
2. **GeoDNS detects failed health checks.** Propagates new routing within 60 seconds. All Tokyo traffic now points to Singapore.
3. **Singapore was at 55% load. It is now at 110% load instantaneously.** The traffic volume doubles in under 60 seconds.
4. **Singapore's DB connection pool (1,000 connections) is exhausted.** New queries queue behind the pool limit.
5. **App server threads fill up waiting for DB connections.** Requests time out. App servers return 500.
6. **Clients see 500s and retry.** Discord's mobile clients retry in 1-2 seconds. This is correct behaviour for a client. It is lethal for an overloaded server.
7. **Singapore is now handling 150% of normal traffic:** original load, plus Tokyo overflow, plus retries from both.
8. **Singapore falls over.** Its health checks start failing.
9. **Seoul and Sydney receive Singapore's traffic.** Now three regions are trying to serve four regions' worth of traffic.
10. **A single-region failure has become a global outage.** Recovery from here takes 30-120 minutes.

This exact cascade is what happened to AWS's internal infrastructure in October 2025. A DNS issue in us-east-1 triggered SDK retries. The retries DDoS'd AWS's own internal DNS resolver. Services that depended on that resolver started failing. Their dependencies started failing. 60+ countries went down.

**The name of this loop:** Retry Storm feeding a Thundering Herd. It is the same metastable failure from lesson 013, applied at geographic routing scale.

**Three places the senior engineer breaks the loop:**

**Break point 1: Run every region at 60% capacity, not 90%.**
Pre-wired headroom. If Tokyo (55% load) fails and Singapore (55% load) absorbs it, Singapore is at 82.5% -- painful but survivable. If both regions are at 90%, any failover is instantly fatal. The cost of always running at 60% is real (40% compute overhead) but it is what you are buying with multi-region: not just geographic redundancy, but overhead to absorb the shock of a failover.

**Break point 2: Shift traffic at 5% intervals, not all at once.**
GeoDNS should not be a binary flip. Implement a traffic-shifting layer (AWS Global Accelerator, Cloudflare Traffic Manager, or a custom weighted DNS) that can move traffic in increments. Gate each step on real-time error rate and p99 latency from the receiving region. If Singapore's error rate exceeds 5% at the 25% step, stop. Alert. Do not proceed to 50%. The gradual shift gives the receiving region time to warm caches, expand connection pools, and autoscale compute.

**Break point 3: Return 503 with `Retry-After: 15s` at the load balancer, not 500 inside the application.**
A 503 returned at the load balancer costs zero DB resources. A 500 returned after the application started processing the request already consumed a thread, a connection, and potentially a partial DB transaction. Under overload, fail at the outermost layer possible. The `Retry-After` header tells well-behaved clients to wait 15 seconds, cutting retry volume by 90% compared to immediate retry on 500.

---

## Rare.lab Stack Mapping

Rare.lab's current setup: Supabase Postgres (single-region), Cloudflare R2 for scene JSON and the shader asset manifest (globally distributed object storage, CDN-backed), embeddable runtime with one shared WebGL context per embed.

**What is already right:**

Scene JSON on R2 is immutable and content-addressed. Cloudflare serves it from the nearest POP, typically under 30ms anywhere in the world. This is the optimal architecture for read-heavy, write-once content. No multi-region DB changes will improve it.

Cloudflare Workers already route compute to the nearest edge POP. The Worker itself executes in under 5ms near the user.

**Where the next ceiling is:**

The project manifest (which asset versions belong to a project, access permissions, publish state) lives in Supabase Postgres. It is a mutable record. Any write to it -- publishing a project, updating a shader node, sharing a link -- travels from the user's nearest Worker to Supabase's single-region Postgres and back. A user in Tokyo updating a manifest hosted on us-east-1 Supabase gets approximately 380ms write latency. For a creative tool where "publish" or "save" feels slow, this is the ceiling.

**Practical sequencing:**

Step 1 (free, do now): Add Supabase read replicas in the regions where your users are concentrated. Reads of the manifest get fast everywhere. Writes are still slow but publish/save is less frequent than preview/view in a creative tool. This covers 80% of the latency problem at zero architecture change.

Step 2 (at 50,000 users): Evaluate Cloudflare D1 for manifest storage. D1 now ships global read replication (March 2026) with sequential consistency via the Sessions API. A read to a D1 replica in the local region takes under 10ms. Writes still go to the D1 primary but writes are the minority operation. D1 sits natively inside Cloudflare Workers, eliminating the Worker-to-Supabase round trip for read operations.

Step 3 (at real-time collaboration, 500,000+ users): Move collaborative operations -- changing a shader node's properties, connecting nodes, repositioning graph elements -- to a CRDT model. Each node in the shader graph is a CRDT-safe object: properties are LWW-registers with HLC timestamps, child-order uses fractional indexing (same as Figma, lesson 003). Store these in Cloudflare Durable Objects (one DO per project document, located in the region nearest to the session owner). Keep the authoritative project manifest in Postgres for billing and ACL; it can afford to be slightly slower because it is not on the render-critical path. The WebGL runtime already compiles from the manifest on load; incremental CRDT updates feed a live recompile loop.

---

## References and Further Reading

### Engineering Blogs

**Cloudflare: "Sequential consistency without borders: how D1 implements global read replication"**
https://blog.cloudflare.com/d1-read-replication-beta/
Published March 2026. The best first-hand account of building read replication on a SQLite-backed edge database. Explains why Cloudflare chose sequential consistency (not eventual) and how they implemented it using the Sessions API and Lamport timestamps. Directly relevant to Rare.lab because D1 is the most natural next-step database from Workers. Read this if you use Cloudflare.

**Cloudflare: "Building D1: a Global Database"**
https://blog.cloudflare.com/building-d1-a-global-database/
April 2024. The origin story of D1. Covers the three-layer architecture (Workers, Durable Objects, Storage Relay Service), how reads are routed to the fastest available copy, and how Lamport timestamps underpin write ordering. Essential background for the above.

**CockroachDB: "Multi-Active Availability"**
https://www.cockroachlabs.com/docs/stable/multi-active-availability
The definitive explanation of how CockroachDB differs from traditional active-active. Key insight: CockroachDB uses consensus replication (Raft) so a write is only committed when a majority of replicas acknowledge it. This means you get strong consistency across regions at the cost of inter-region round-trip on every write. Contrast this with async replication's eventual consistency. Neither is universally better; they suit different data types.

**CockroachDB: "How to build a highly available database for a multi-region architecture"**
https://www.cockroachlabs.com/blog/build-a-highly-available-multi-region-database/
Step-by-step guide. Shows the SQL primitives (`ALTER DATABASE SET PRIMARY REGION`, `LOCALITY REGIONAL BY ROW`) that pin rows to the region nearest their data. A user's profile row lives in the Tokyo region. A query from Tokyo never crosses an ocean. An equivalent feature exists in Neon (Supabase's competitor) and is coming to Supabase itself.

**Redis: "Active-Active vs Active-Passive"**
https://redis.io/blog/active-active-vs-active-passive/
Short and precise comparison from the Redis team, who implemented active-active in Redis Enterprise using CRDTs. Explains the conflict types (write-write, read-write), which data structures are CRDT-safe, and why a distributed counter in a naive active-active setup loses increments.

**AWS: "Latency-based routing with Amazon CloudFront for a multi-region active-active architecture"**
https://aws.amazon.com/blogs/networking-and-content-delivery/latency-based-routing-leveraging-amazon-cloudfront-for-a-multi-region-active-active-architecture/
Practical implementation guide using Route 53 latency routing + CloudFront + DynamoDB Global Tables. Shows real latency numbers measured between AWS regions (us-east-1 to ap-northeast-1: 183ms RTT; us-east-1 to eu-west-1: 91ms RTT). These are the real physics numbers, not estimates.

**Microsoft Learn: "Retry Storm Antipattern"**
https://learn.microsoft.com/en-us/azure/architecture/antipatterns/retry-storm/
The clearest explanation of why retry storms happen and why they feel like "the recovery kills the system." Includes diagrams of the feedback loop and the three mitigations: exponential backoff, circuit breaker, and server-side 503 with Retry-After. Relevant to every distributed system, not just Azure.

### Hybrid Logical Clocks Deep Dive

**Kevin Sookocheff: "Hybrid Logical Clocks"**
https://sookocheff.com/post/time/hybrid-logical-clocks/
The clearest written explanation of HLC available on the open web. Walks through why physical clocks fail, why Lamport clocks are too abstract for production use, and how HLC combines both into a practical 64-bit timestamp. After this post you will understand why CockroachDB, MongoDB, and Aurora DSQL all converged on the same design.

**AWS Database Blog: "How Aurora DSQL uses clocks"**
https://aws.amazon.com/blogs/database/everything-you-dont-need-to-know-about-amazon-aurora-dsql-part-5-how-the-service-uses-clocks/
Goes one level deeper: shows how AWS uses TrueTime (GPS + atomic clocks, same as Google Spanner) to bound clock uncertainty to under 1ms, versus HLC's 250ms NTP-bound uncertainty. The trade-off is cost and complexity. Most systems cannot afford Spanner-level clock infrastructure. HLC with 250ms uncertainty is the pragmatic choice.

### YouTube Videos

**"Distributed Systems in One Lesson" by Tim Berglund (O'Reilly)**
Search: "Tim Berglund distributed systems one lesson" on YouTube
50 minutes. Covers partitioning, replication, CAP theorem, and consistency models in plain language with live demos. The best one-shot tour of the concepts underlying this lesson. Watch this before diving into CRDTs or consensus papers.

**"CRDT: Conflict-free Replicated Data Types" by Martin Kleppmann**
Search: "Martin Kleppmann CRDT" on YouTube (Cambridge lecture, 2020)
1 hour. The author of "Designing Data-Intensive Applications" explains CRDTs from first principles. He starts with the collaboration problem (why you cannot just timestamp everything), builds up G-Counters, OR-Sets, and sequence CRDTs step by step. Figma, Google Docs, and every real-time collaboration product is built on these ideas. After this lecture, lesson 003 (Figma) and this lesson click into place together.

**"Patterns of Distributed Systems" by Unmesh Joshi**
Search: "Unmesh Joshi patterns distributed systems ThoughtWorks" on YouTube
45 minutes. Covers fencing tokens, Raft leader election, write-ahead logs, and heartbeat patterns with concrete pseudocode. This is the "how does the code actually work" companion to the theory from Kleppmann.

### Books

**"Designing Data-Intensive Applications" by Martin Kleppmann (O'Reilly, 2017)**
The single most important book for this curriculum. Chapters 5 (Replication), 6 (Partitioning), and 9 (Consistency and Consensus) directly cover everything in this lesson. Chapter 5 alone covers single-leader, multi-leader, and leaderless replication with concrete trade-offs for each. If you read one book from this entire lesson series, read this one.

### Research Papers

**"Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases" (Kulkarni et al., 2014)**
The original HLC paper. Available free via Google Scholar search for "hybrid logical clocks Kulkarni 2014." Dense but worth reading the abstract and section 3. This is where CockroachDB's timestamp logic comes from.

**"Spanner: Google's Globally-Distributed Database" (Corbett et al., 2012)**
https://research.google/pubs/pub39966/
The paper that showed the industry it was possible to build a globally-consistent database across datacenters. Introduces TrueTime (GPS + atomic clocks for bounded clock uncertainty) and external consistency (every transaction reads a consistent snapshot of the global state). CockroachDB is essentially "Spanner but on commodity hardware using HLC instead of TrueTime." Reading this paper explains every design decision in CockroachDB and Aurora DSQL.

### Quick Reference

| Situation | Pattern to reach for |
|-----------|---------------------|
| Want fast reads globally, writes can stay slow | Read replicas + GeoDNS |
| Want fast reads AND fast writes globally | Multi-region active-active with async replication |
| Data can be freely merged (counters, sets, presence) | CRDT |
| Data cannot tolerate any ambiguity (money, inventory) | Regional lease + fencing token |
| Need to compare writes across nodes without trusting wall clocks | Hybrid Logical Clock (HLC) |
| One region is degrading, prevent cascade | Circuit breaker + gradual traffic shift at 5% intervals |
| Overloaded, retries making it worse | 503 + Retry-After at the load balancer, not 500 inside the app |
