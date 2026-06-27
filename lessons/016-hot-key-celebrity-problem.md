# Day 16 — How does one celebrity's tweet bring down your cache cluster?

**Date:** 2026-06-27
**Difficulty:** Advanced
**Stack relevance:** Redis sharding, distributed caching, fan-out architecture, Cloudflare edge caching, Supabase read replicas

---

## 1. Named Company + The Breaking Number

**Company: Twitter (now X), circa 2012**

**The breaking number: 31 million.**

Lady Gaga had 31 million followers in 2012. Barack Obama's election-night tweet hit 327,000 retweets per hour. Each of those events woke up tens of millions of people who refreshed their timelines at roughly the same moment.

Here is the arithmetic that kills the naive system.

Twitter's timeline service stores each user's precomputed timeline in Redis. One Redis node handles roughly 100,000 to 300,000 operations per second at safe utilization (70%).

31 million followers each refresh their feed every 30 seconds. That is 1,033,333 reads per second hitting one cache key: `timeline:Lady_Gaga`. One Redis node can handle 100,000 ops/sec comfortably. This single key requires 10x the capacity of a full node. The node saturates. It stops responding. Other nodes in the cluster are sitting at 8% utilization wondering what went wrong.

This is the hot-key problem. One key. One dead node. The rest idle.

The identical pattern appears everywhere:
- Amazon: a product goes viral on a TV show. 80,000 requests per second hit one item record.
- YouTube: the Super Bowl halftime performance. One video. One CDN shard overwhelmed.
- GitHub: a repository trends on Hacker News. Every CI runner fetches the same 500MB artifact.
- Stripe: a single merchant's webhook endpoint backlog key grows until the queue partition saturates.

The abstraction is always the same: a power-law distribution (a few items get most of the traffic) collides with a hash-based routing scheme (one key lands on one node). The node drowns. Your system does not know how to move the water.

---

## 2. Why the Naive Design Dies

The demo design looks like this:

```
Client --> App Server --> Redis Cluster (consistent hashing) --> PostgreSQL
```

Consistent hashing maps `timeline:Lady_Gaga` to Node 3. Every read goes to Node 3. Node 3 is overwhelmed. You cannot move the key without rehashing. Adding a 4th node does not help because Lady Gaga's key still hashes to Node 3.

Three specific failures:

**Failure 1: The node saturates and starts dropping connections.** Redis is single-threaded per core. When the network socket queue fills, new connections get refused. Clients see timeouts. App servers, seeing cache misses, fall through to the database. 1 million requests/second now hit PostgreSQL. PostgreSQL dies.

**Failure 2: The thundering herd on cache expiry.** Suppose you have a smarter setup with a TTL. The TTL fires at 14:00:00.000. At that exact millisecond, 5,000 app server threads each detect a cache miss and each independently decide to query the database. 5,000 simultaneous SELECT queries for the same row. Database connection pool exhausted in 200ms. Every subsequent request gets "too many connections." The system is now in a collapse from which it cannot recover on its own -- recovery requires more load, which causes more collapse.

**Failure 3: The retry death spiral.** Clients that got timeouts retry. Retries arrive during the exact window when the system is already overwhelmed. The load does not decrease; it increases. Each retry buys 3 seconds of delay then adds another request. This is a feedback loop that self-amplifies. Senior engineers call this a **metastable failure**: once in the bad state, the system cannot exit it without external intervention (traffic shutoff, manual cache warm-up, circuit break).

---

## 3. The Architecture

```
                         [Clients]
                         31M browsers / mobile apps
                              |
                    [Edge Cache / CDN]
                    Cloudflare, Fastly
                    Caches public timelines at 110+ PoPs
                    Job: absorb read volume before it touches your network
                    Analogy: a library branch near your house so you never drive downtown
                              |
                    [Load Balancer]
                    Distributes across app server fleet
                    Job: spread connections; terminate TLS
                    Analogy: a building receptionist routing calls to the right floor
                              |
                    [Stateless App Tier]
                    10,000 app server instances
                    Each carries an L1 in-process LRU cache
                    ~1,000 hot keys per instance, ~1MB RAM, TTL 5 seconds
                    Job: absorb repeated reads within one process in zero network hops
                    Analogy: your desk has a sticky note with the answer; you don't call IT every time
                              |
                         [L1 MISS]
                              |
                    [Hot Key Detection Layer]
                    Background thread samples 1% of all Redis commands
                    Maintains a key-count sliding window (last 10 seconds)
                    If key crosses N reads/sec threshold --> promotes to "HOT" status
                    Hot keys are replicated across M Redis nodes automatically
                    Job: dynamically identify and distribute the hot spots
                    Analogy: a traffic cop who watches for jams and opens extra lanes in real time
                              |
                    [Redis Cluster -- L2 Distributed Cache]
                    Normal keys: consistent hashing, 1 copy per key
                    HOT keys: replicated as timeline:Lady_Gaga:0 through :15
                              reads pick replica = hash(request_id) mod 16
                              writes update all 16 replicas (async, 10ms lag acceptable)
                    Job: serve cache hits at 0.3ms without touching the database
                    Analogy: a photocopy room where the most-requested document
                              gets 16 copies so 16 people can read simultaneously
                              |
                         [L2 MISS]
                              |
                    [Request Coalescing Layer (singleflight)]
                    If 1,000 threads all miss the same key simultaneously:
                      - Thread 1 fires the database query
                      - Threads 2-999 wait on Thread 1's result
                      - One database hit. 1,000 threads served.
                    Job: convert correlated cache misses from N database hits to 1
                    Analogy: one person goes to the kitchen to make 20 coffees;
                              everyone else waits rather than all crowding the espresso machine
                              |
                    [Database -- PostgreSQL with Read Replicas]
                    Primary: writes only
                    5-10 read replicas: serve cache-miss queries
                    Queries are rare -- L1 + L2 together absorb 99.9%+ of reads
                    Job: durable source of truth
                    Analogy: the original filed document; photocopies handle most requests,
                              the original is only pulled for genuine research
                              |
                    [Async Fan-out Queue -- Kafka]
                    When celebrity posts: event published to Kafka topic
                    Fan-out workers consume and write to follower timelines
                    For megacelebrities (>1M followers): HYBRID model
                      - Followers get a "you have new items" marker, not full fan-out
                      - Timeline is assembled at read time from the celebrity's feed + personal feed
                    Job: decouple write speed from follower count
                    Analogy: instead of hand-delivering 31M letters, post a notice board
                              that says "new letter available; come read it yourself"
```

---

## 4. The Transferable Mechanisms

### Mechanism 1: L1 In-Process Cache (the desk sticky-note)

Each app server keeps a small, bounded LRU (Least Recently Used) cache in its own process memory. No network call. No Redis. Just RAM.

The tradeoff: staleness. With 10,000 app servers each caching for 5 seconds, the same data can be 5 seconds old on each machine. For social timelines, that is fine. For inventory counts, it is not.

**When to use it:** Read-heavy data with tolerable staleness. Hot user profiles, product descriptions, feature flags, rate-limit configs.

**When not to use it:** Stock counts, financial balances, anything where two servers seeing different values causes real harm.

### Mechanism 2: Key Replication (the photocopier)

Instead of one copy of a hot key on one Redis node, store M copies distributed across M nodes: `item:viral_product:0` through `item:viral_product:15`. Reads are spread by picking `index = hash(request_id) mod M` or simply `random int mod M`.

Writes must update all M copies. If M=16 and you update once per second, that is 16 writes/second -- cheap. If you update 1,000 times/second with M=16, that is 16,000 writes/second -- evaluate the tradeoff.

Twitter open-sourced their "Twemcache" which implemented this pattern. Redis 7.x Cluster does not do this natively; you implement it at the client or in a middleware layer.

### Mechanism 3: Probabilistic Early Expiration (the early-bird refresh)

Standard TTL: all copies expire at the same instant. Thundering herd guaranteed.

**XFetch algorithm** (from a 2015 academic paper, now widely implemented):

```
delta = time to recompute the cached value (measure this)
beta = tuning constant (1.0 default)
expiry = time when key expires

if current_time - delta * beta * ln(random()) >= expiry:
    refresh the cache
```

A small random fraction of reads will refresh the cache slightly before expiry. By the time the TTL actually hits, the cache is already fresh. The thundering herd never forms because the key never actually expires under load.

This looks like magic but is just statistics. A few requests pay the refresh cost early; the rest get a warm cache.

### Mechanism 4: Request Coalescing / Singleflight

When a cache key is missing, exactly one request goes to the origin. All concurrent requests for the same key wait for that one result and share it.

In Go, the `golang.org/x/sync/singleflight` package does this in 12 lines of code. Nginx does it with `proxy_cache_lock on`. Varnish calls it "request coalescing."

The before state: 1,000 threads see a cache miss. 1,000 database queries fire. Database collapses.

The after state: 1,000 threads see a cache miss. Thread 1 fires. Threads 2-999 wait on a channel. Thread 1 returns. All 1,000 are served. 1 database query fired.

This is the most leverage per line of code in distributed caching.

### Mechanism 5: Hybrid Push/Pull Fan-out (the notice board vs. the postal service)

Fan-out on write: when a user posts, immediately write to every follower's timeline inbox. Read is O(1). Write is O(followers). For normal users (100 followers), this is fast. For Lady Gaga (31M followers), this means 31M writes per tweet. A tweet storm can queue for hours.

Fan-out on read: do not pre-compute timelines. When a user opens their feed, query the people they follow in real time. Read is O(following count). Write is O(1). This collapses at read time when a user follows 500 people.

**The hybrid:** Twitter uses push for all users with <a certain follower threshold, and pull-merge for celebrities above that threshold. At read time, the app fetches the precomputed personal feed (all non-celebrity followings, precomputed on write) and then fetches and merges the raw feeds of the handful of celebrity accounts the user follows. This keeps write fan-out bounded while keeping read time fast.

### Mechanism 6: Circuit Breaker on Cache Miss Spike

If the cache miss rate spikes above a threshold (say, 5% of reads are misses instead of the normal 0.1%), a circuit breaker trips. App servers stop trying to fall through to the database and instead return stale data or a graceful fallback (empty timeline, last-seen data, a "try again" response). This prevents the stampede from reaching the database at all.

The circuit breaker is the emergency brake. It accepts a temporary user-visible degradation in exchange for not taking the entire system down.

---

## 5. The Trade-offs

### Consistency vs. Availability

**For social timelines:** Twitter explicitly chose availability over consistency. Lady Gaga's followers might see her tweet 200ms to 5 seconds late depending on which L1 cache their app server happens to hold. This is acceptable. Nobody is harmed by seeing a tweet 5 seconds late.

**For inventory:** Amazon does NOT accept this trade-off for the "Add to Cart" confirmation. The item count is eventually consistent on the product listing page (you might see "Only 3 left" when there are 2), but the actual reservation is a strongly consistent atomic operation. Two separate data flows, two different consistency contracts, on the same page.

**The CAP breakdown for this topic:**

| Data | Consistency choice | Why |
|---|---|---|
| Social timeline | Eventual | 5s stale is invisible to humans |
| User profile (name, photo) | Eventual (minutes) | Changes rarely |
| Item listing price | Eventual (seconds) | Display only; buy flow re-checks |
| Inventory reservation | Strong | Double-selling causes real damage |
| Payment amount | Linearizable | Wrong number = fraud |

### Cost vs. Latency

| Layer | Latency | RAM cost | When to add |
|---|---|---|---|
| L1 in-process | 0ms | Low (1MB per server) | Always, for hot read paths |
| L2 Redis | 0.3ms | Medium ($0.10/GB/month) | When L1 is insufficient |
| Hot key replication (16x) | 0.3ms | 16x the key size | When a key exceeds N reads/sec |
| Origin database | 5-50ms | N/A (already paid for) | Last resort |

Replicating a hot key across 16 nodes costs 16x the RAM for that key. For a 10KB timeline payload, that is 160KB. For a cluster with 1,000 hot keys, that is 160MB. Trivially affordable. For 10MB video metadata, think harder.

---

## 6. The Systems-Thinking Lens

**The feedback loop:** Cache stampede / thundering herd

```
Cache key expires
        |
        v
N concurrent requests detect miss (all at same instant)
        |
        v
N requests fire to database simultaneously
        |
        v
Database connection pool exhausted
        |
        v
Requests queue behind the pool, latency climbs
        |
        v
Client timeouts fire. Clients retry.
        |
        v
MORE requests arrive on top of existing overload
        |
        v
System cannot serve enough requests to drain the queue
        |
        v
Metastable failure: system cannot exit bad state on its own
```

This is a **positive feedback loop**. Each retry increases load, which increases failures, which increases retries. The system is in a trap.

**Why adding capacity does not help (the key insight):** If you have 100 database nodes and every single one gets simultaneously hit by 1,000 requests for the same key, you still collapse. The problem is **correlation**, not capacity. 100,000 correlated requests will overwhelm 100 nodes just as easily as they overwhelm 1, because all 100,000 target the same shard. Horizontal scaling helps with average load. It does not help with perfectly correlated spikes.

**The senior engineer's two fixes break the loop at different points:**

Fix 1 (XFetch probabilistic expiration): prevents the correlated miss from forming. The key never reaches zero simultaneously across all requesters. No miss spike, no stampede. Breaks the loop at the root.

Fix 2 (Request coalescing / singleflight): if a miss happens, collapses N origin requests to 1. Breaks the loop at the amplification step. Even if 10,000 threads all miss simultaneously, only 1 database query fires.

Fix 3 (Circuit breaker): if the database is already overwhelmed, stops new requests from making it worse. Breaks the loop at the retry amplification step.

**The pattern name:** This class of failure is called a **metastable failure** (Bronson et al., "Metastable Failures in Distributed Systems," HOTOS 2021). The system has two stable states: "cache warm, all is well" and "cache cold, everything on fire." The dangerous property is that the bad state is self-reinforcing. Getting from bad-state back to good-state requires the system to serve traffic well, but it cannot serve traffic because it is in bad state.

Metastable failures are not caused by the original trigger (the cache expiry). They are maintained by the system's own response to the trigger (retries, amplification). The fix is always structural: break the feedback loop, not the trigger.

---

## 7. Map to Rare.lab's Stack

Rare.lab's architecture has specific hot-key exposure points worth naming explicitly.

**Where hot keys hit you today:**

- **Cloudflare R2 content-addressed scene files.** A popular template or a scene shared in a tutorial video could generate 50,000 downloads in an hour. R2 handles this well because Cloudflare automatically caches at edge PoPs. Your cost is low. Your latency is low. This is the CDN fan-out working correctly.

- **Supabase Postgres + RLS: the shared shader template row.** If a particularly popular shader template row gets queried on every page load by every user, and that row lives in Postgres, you will feel the hot-key problem as Postgres connection pressure. Supabase does not automatically cache your data. A read replica helps (spreads the query load) but does not collapse N identical queries to 1. The fix here is explicit caching: Supabase Edge Functions can maintain an in-memory cache of hot template rows, or you push the data to a Cloudflare D1 / KV instance closer to the user.

- **The manifest file.** Your runtime fetches a manifest to know which scene files to load. If millions of users fetch the same project's manifest simultaneously (say, a widely-embedded Rare.lab component on a high-traffic site), the manifest becomes a hot key. Since it is content-addressed and immutable, it is perfectly safe to cache aggressively at the CDN layer with a long TTL (hours or days). Zero origin egress, zero hot-key pressure.

- **Your next ceiling:** The WebGL runtime's initialization likely fetches some config or shared node-library payload on startup. As embedding grows, this becomes a hot key at your origin. Audit what your runtime fetches on init, make every byte content-addressed and immutable, and push it behind a CDN with a multi-hour TTL. The XFetch pattern is not applicable at the CDN layer directly (CDNs use their own expiry logic), but you can achieve the same effect by using `stale-while-revalidate` HTTP cache headers: the CDN serves stale while asynchronously refreshing, preventing the miss window entirely.

**Patterns you already have working:**
- Content-addressed R2 immutable storage (perfect cache key hygiene -- no invalidation needed, no stale risk)
- Cloudflare edge distribution (your own L1 cache at 110+ PoPs)

**Pattern to add next:** An explicit in-memory or KV cache for the top K most-read Supabase rows (popular templates, shared project manifests), with request coalescing so simultaneous cache misses hit Postgres exactly once.

---

## References (with Summaries)

### Primary Sources

**1. Twitter Engineering: "Timelines at Scale" (2013)**
https://www.infoq.com/presentations/Twitter-Timeline-Scalability/

A presentation by Twitter engineers describing exactly how they solved the celebrity fan-out problem. They walk through why pure fan-out-on-write fails for accounts with millions of followers, how they moved to a hybrid push/pull model, and how their Flock graph store and Caching layers interact. The most honest account of this failure mode from someone who lived it. Summary in plain language: Twitter tried to precompute timelines for everyone. Works fine for 500 followers. Completely breaks for Lady Gaga. The fix was to treat celebrities as a special case: their content is fetched and merged at read time rather than pushed to 31 million mailboxes at write time.

**2. "Metastable Failures in Distributed Systems" (Bronson et al., HOTOS 2021)**
https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf

The academic paper that named the "metastable failure" pattern. Written by Facebook/Meta engineers who observed that large-scale distributed system failures are rarely caused by the original trigger but are maintained by the system's own feedback mechanisms (retries, queuing). Extremely readable for an academic paper. The key insight: your system can be in a perfectly sustainable state at 80% load, but a brief spike to 110% can push it into a different stable state from which it cannot recover without external intervention. Summary in plain language: systems can get stuck in a "everything is on fire" mode that they cannot exit on their own, because the very act of trying to recover (retrying requests) makes the fire worse. The fix is to design systems that break their own feedback loops.

**3. "Optimal Probabilistic Cache Stampede Prevention" (Vattani, Chierichetti, Lowenstein, 2015)**
https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf

The academic paper behind XFetch. Proves mathematically that adding random jitter to cache refresh decisions (refreshing slightly before expiry, with probability proportional to how close you are to expiry) eliminates the thundering herd without requiring coordination between servers. Each server independently makes a probabilistic decision. Collectively, they prevent correlated expiry. Summary in plain language: instead of setting a hard expiry time for your cache and having 10,000 servers all miss at once, you add randomness so that some servers refresh a bit early. By the time the official expiry happens, the cache is already warm. It is the difference between a crowd all leaving a concert at the same moment versus people filtering out gradually over 20 minutes.

**4. Redis Documentation: Finding Hot Keys**
https://redis.io/docs/management/optimization/memory-optimization/#finding-hot-keys

Official Redis documentation on using `redis-cli --hotkeys` (Redis 4.0+) and the `MONITOR` command (dangerous at scale but useful in dev) to identify which keys are receiving disproportionate traffic. Also covers the `lfu-log-factor` config for LFU-based eviction which naturally protects hot keys from eviction. Summary in plain language: Redis has a built-in way to tell you which keys are being hit the most. Run it on a staging environment under realistic load to find your hot keys before production finds them for you.

**5. Pinterest Engineering: "Sharding Pinterest: How We Scaled Our MySQL Fleet"**
https://medium.com/pinterest-engineering/sharding-pinterest-how-we-scaled-our-mysql-fleet-3f341e96ca6f

Pinterest's detailed write-up on moving from a single MySQL to a sharded fleet. Particularly relevant is their discussion of how they identified and handled celebrity boards (boards with millions of followers) differently from regular boards. Also covers consistent hashing, virtual nodes, and the rebalancing problem. Summary in plain language: Pinterest split their database into many smaller shards using consistent hashing. But some boards (think a famous designer with a million followers) were so popular they overwhelmed their assigned shard. The solution was to identify these hot shards proactively and apply different caching strategies rather than trying to re-shard them (re-sharding is expensive and risky).

### Secondary Resources (Broader Context)

**6. "Go: Singleflight package documentation"**
https://pkg.go.dev/golang.org/x/sync/singleflight

The Go standard library's implementation of request coalescing. The source code is 150 lines and extremely readable. Worth reading even if you do not use Go -- the pattern itself is universal and the implementation shows exactly how to use a mutex and a wait channel to collapse concurrent requests. Summary in plain language: when 1,000 goroutines all ask for the same thing at the same moment, this package makes only one of them actually do the work. The other 999 wait and receive the result when the first one finishes. Three function calls to implement in any language.

**7. Cloudflare Blog: "How we made the Cloudflare cache even faster"**
https://blog.cloudflare.com/argo-smartrouting/

Cloudflare's description of how they handle hot keys at the edge, including their implementation of request coalescing (they call it "Request Collapsing") and how they route around overloaded PoPs using smart routing. Useful for understanding what CDNs do for you automatically versus what you still need to handle yourself at the origin. Summary in plain language: Cloudflare collapses duplicate requests for the same URL at each PoP so that your origin only ever sees one request per unique cache miss per location. If 10,000 users in Frankfurt all request the same image at the same moment and it is not cached there, Frankfurt sends one request to your origin, not 10,000. Understanding this lets you reason about what load your origin actually sees.

**8. YouTube Engineering Blog: "Handling Video Upload Surges"**
https://developers.google.com/youtube/

YouTube's approach to the viral video problem -- specifically how they handle a video going from 100 views/hour to 5 million views/hour in minutes. Their answer involves a multi-tier CDN with per-region popularity detection, where newly-viral content is quickly replicated to more edge nodes before the surge overwhelms any single one. Summary in plain language: YouTube watches how fast a video's view count is growing, not just how many views it has. A video that doubles its views every 10 minutes gets special treatment: it gets pushed to more CDN locations proactively before demand outstrips supply in any one region. This is predictive scaling rather than reactive scaling.

**9. Facebook TAO: "TAO: Facebook's Distributed Data Store for the Social Graph" (USENIX ATC 2013)**
https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf

Facebook's paper on TAO, their distributed graph store that powers the social graph (who follows whom, what likes what). TAO handles the hot-key problem at the social graph level: popular entities (famous people, viral posts) generate massive read traffic. TAO's solution is a two-tier caching hierarchy with leader and follower caches, where reads are served from followers and writes propagate asynchronously. The paper is dense but the architecture section is worth reading. Summary in plain language: Facebook built a special-purpose distributed database just for "who is connected to whom." The insight was that the social graph has extreme hot spots (famous people are connected to millions of others) and extreme cold spots (most users have a few hundred connections). A general-purpose database treats every row the same. TAO treats hot rows differently -- they are cached in more places and served from nearby servers -- while cold rows live only in the primary store.

**10. High Scalability Blog: "Cache is King: A Cache at Every Layer"**
http://highscalability.com/blog/2019/2/4/its-time-to-move-on-from-two-tier-to-multitier-caches.html

A practitioner-written post (not academic, written by a CTO) that surveys how sophisticated companies build multi-tier caches and why two tiers (L1 + L2) are often not enough. Covers L0 (browser/client cache), L1 (in-process), L2 (distributed), L3 (CDN), and when to use each. Useful as a mental model for thinking about where in the stack to add caching versus where the problem cannot be solved with more caching. Summary in plain language: there is no single right place to put a cache. The best systems have caches at every level, with each level solving a different problem. Browser caches save bandwidth. Process-level caches save network calls. Distributed caches save database queries. CDN caches save origin capacity. If you only have one cache layer and your system is struggling, the answer is usually to add a cache at a different layer, not to make the existing cache bigger.

---

*Next lesson:* Day 17 -- Write-Ahead Logs and Change Data Capture: how databases never lose a committed write, and how you stream those changes to other systems.
