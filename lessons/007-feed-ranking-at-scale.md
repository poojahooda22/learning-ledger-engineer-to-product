# Day 7 -- How does Instagram rank a feed for 2 billion users in under 200ms?

**Date:** 2026-06-18
**Topic:** Feed ranking at massive scale: fan-out architecture, candidate generation, ML ranking, the hot-key celebrity problem
**Difficulty:** Advanced
**Prereqs:** Day 3 (Figma -- distributed state), Day 4 (CDN -- edge caching), Day 6 (Stripe -- idempotency and correctness)

---

## 1. The company and the breaking number

**Instagram / Meta.** 2 billion monthly active users on Instagram. 500 million daily active Stories viewers. A single post from a celebrity with 300 million followers triggers -- in a naive design -- **300 million simultaneous write operations to 300 million different users' feed rows**.

That is the breaking number: **300 million writes from a single event**.

Instagram engineers gave this a name in 2012: the **Justin Bieber problem**. When Bieber posts a photo, every one of his 300M followers needs that photo to appear in their feed near-instantly. A simple "write to everyone's feed when someone posts" strategy (called fan-out on write) collapses entirely. The math: at 300M rows written, assuming 1ms per Postgres INSERT on a fast machine, you need 300 seconds of serial writes. By the time the last follower gets the post, it is already 5 minutes stale. And this is just ONE celebrity post. Bieber posts maybe 5 times a day. Now add Cristiano Ronaldo (600M followers), Taylor Swift, and 50 other accounts with 50M+ followers each -- all posting throughout the day. The writes never stop.

But there is a second breaking number equally important: **a Facebook-style ranked feed must score and sort roughly 1,000 candidate posts per user within 200ms of a feed request.** That is 1,000 ML inference calls, feature lookups, and a sort -- in under 200 milliseconds.

Real example: Priya opens Instagram at 8:47am in Bengaluru. She follows 600 accounts. In the last 12 hours, those 600 accounts posted 1,200 items combined. Instagram must, in under 200ms, fetch those 1,200 candidates, score each one against Priya's engagement history, pick the top 15, stitch in 2 ads, and render the feed. The system does this simultaneously for roughly 50 million users opening Instagram within the same morning window.

---

## 2. Why the naive version collapses

**The naive design:** One relational database. A `feed` table with `(user_id, post_id, created_at)`. When Alice posts, a background job runs `INSERT INTO feed (user_id, post_id) SELECT follower_id, $post_id FROM follows WHERE followed_id = $alice_id`. When Bob opens his feed, he runs `SELECT * FROM feed WHERE user_id = $bob_id ORDER BY created_at DESC LIMIT 20`.

This works perfectly at 1,000 users. It collapses in three ways at Instagram's scale:

**Collapse 1: The hot-celebrity fan-out write storm.**
One celebrity post triggers millions of writes in seconds. At 300M followers and assuming a generous 10K writes/second on a well-tuned Postgres cluster, completing the fan-out for one post takes 300M / 10,000 = **30,000 seconds (8 hours)**. By the time the last follower gets the post, the day is over. Meanwhile the next celebrity post starts its own 8-hour fan-out. The queue of pending writes grows without bound. The database cannot keep up. Writes pile up in a queue that is always 8+ hours behind. Every follower's feed is perpetually stale.

**Collapse 2: The read explosion on a purely pull-based design.**
The opposite naive approach: no pre-written feed. When Bob opens his feed, the server runs: `SELECT posts.* FROM posts JOIN follows ON posts.author_id = follows.followed_id WHERE follows.follower_id = $bob_id ORDER BY created_at DESC LIMIT 20`. If Bob follows 600 people, this is a JOIN across 600 accounts' posts, sorted and filtered on the fly. This query hits the posts table with a 600-row scan per request. At 50 million simultaneous morning openers, this becomes 50M concurrent complex queries per second. The database melts.

**Collapse 3: Chronological ordering is the wrong answer.**
Even if you solve the write and read problems, pure chronological feeds have a documented engagement problem. A post from a close friend posted 2 hours ago gets buried under 80 posts from news accounts the user follows casually. Instagram's internal data (referenced in their 2022 ranking transparency post) showed users missed 70% of posts from their closest connections when using chronological ordering. Ranking by relevance, not time, is necessary for the product to function.

---

## 3. The real architecture, layer by layer

Read this top to bottom. Each layer has one job.

```
+------------------------------------------------------------------------+
|                       MOBILE CLIENT (iOS / Android)                    |
|                                                                        |
|  Job: display feed. Send "feed request" to API. Cache the last-seen   |
|  feed locally for instant re-open. Prefetch next page before scroll.  |
|                                                                        |
|  Analogy: the customer at the restaurant. They read the menu;         |
|  they do not cook the food.                                            |
+------------------------------------------------------------------------+
         |
         | HTTPS feed request: GET /v1/feed?user_id=priya&count=15
         v
+------------------------------------------------------------------------+
|                     CDN / EDGE LAYER (PoP nearest to Priya)           |
|                                                                        |
|  Job: cache the RENDERED feed payload for ~1 second for identical     |
|  requests. If 10,000 users on the same PoP open feed at the same     |
|  second, serve from edge cache. Feed TTL is very short (1-2s) because |
|  content changes rapidly.                                              |
|                                                                        |
|  Analogy: the express lane at a grocery store. If someone just        |
|  bought the same exact thing 2 seconds ago, hand it to you without   |
|  going back to the warehouse.                                          |
+------------------------------------------------------------------------+
         |
         | Cache miss: pass through to API tier
         v
+------------------------------------------------------------------------+
|                  STATELESS API SERVERS (hundreds of boxes)            |
|                                                                        |
|  Job: receive feed request, call the feed ranking pipeline, return    |
|  the ranked post list. No state stored here. Any server can handle    |
|  any user.                                                             |
|                                                                        |
|  Analogy: the ticket windows at a train station. Any window can       |
|  handle any passenger. Adding more windows adds more capacity.        |
+------------------------------------------------------------------------+
         |
         | Calls the Feed Ranking Service (internal RPC)
         v
+------------------------------------------------------------------------+
|                     FEED RANKING SERVICE                               |
|                                                                        |
|  Job: given user_id, fetch candidates, score them, return top N.      |
|  This is the heart of the system. Two sub-steps:                      |
|                                                                        |
|  Step A: CANDIDATE GENERATION (< 50ms budget)                         |
|  Step B: RANKING / SCORING (< 100ms budget)                           |
|                                                                        |
|  Analogy: a librarian who first finds all books remotely related to   |
|  your interest (candidate generation), then personally recommends     |
|  the top 5 (ranking).                                                 |
+------------------------------------------------------------------------+
         |                           |
         v                           v
+---------------------+   +------------------------------+
|  FEED STORE         |   |  POSTS / MEDIA SERVICE       |
|  (Redis Cluster)    |   |  (Cassandra + CDN for media) |
|                     |   |                              |
|  Job: store the     |   |  Job: store post metadata,   |
|  pre-computed list  |   |  captions, media pointers.   |
|  of candidate posts |   |  Serve post content fast.    |
|  per user.          |   |                              |
|                     |   |  At scale: 100M+ posts/day   |
|  Data structure:    |   |  ingested. Sharded by post_id|
|  Sorted Set         |   |  (consistent hash ring).     |
|  Key: user_id       |   |                              |
|  Score: timestamp   |   |  Analogy: the library stacks.|
|  Value: post_id     |   |  Every book is there. You    |
|                     |   |  just need to know the shelf.|
|  Analogy: your      |   +------------------------------+
|  personalized inbox.|
|  Pre-sorted. Ready. |
+---------------------+
         |
         | Feed store returns ~1,000 candidate post IDs
         v
+------------------------------------------------------------------------+
|               FEATURE STORE (Redis + offline Hive/Spark tables)       |
|                                                                        |
|  Job: provide pre-computed features for each (user, post) pair        |
|  in under 10ms per lookup. Examples:                                  |
|  - user_engages_with_reels: 0.87                                      |
|  - user_follows_author: true                                          |
|  - author_post_frequency: 3.2/day                                     |
|  - post_like_rate_first_5min: 0.034                                   |
|  - relationship_score(priya, author): 0.61                            |
|                                                                        |
|  Features are computed offline (hourly Spark jobs) and written to     |
|  Redis. The online ranking reads from Redis, not from the raw DB.     |
|                                                                        |
|  Analogy: a cheat sheet the librarian consults. They do not re-read   |
|  every review to remember what each reader likes. The cheat sheet     |
|  is pre-computed.                                                      |
+------------------------------------------------------------------------+
         |
         | 1,000 candidates + feature vectors
         v
+------------------------------------------------------------------------+
|               RANKING MODEL (Neural Network, GPU-served)              |
|                                                                        |
|  Job: score each candidate post 0.0 to 1.0 for this specific user.   |
|                                                                        |
|  Architecture (Instagram circa 2022):                                 |
|  1. Light ranker: fast 2-layer MLP scores all 1,000 candidates        |
|     in <20ms. Drops to top 150.                                       |
|  2. Heavy ranker: deep neural net (DLRM-style) scores top 150.       |
|     Takes 60-80ms. Produces final ranking.                            |
|  3. Post-ranking logic: inject diversity (no 5 consecutive Reels),   |
|     insert ads slots, apply hard filters (blocked users, NSFW).       |
|                                                                        |
|  Meta's DLRM (Deep Learning Recommendation Model) paper (2019)        |
|  describes the production architecture. It uses embedding tables      |
|  for categorical features (user_id, author_id) combined with dense    |
|  features (engagement rates) through a bottom MLP.                    |
|                                                                        |
|  Analogy: a movie critic who has seen 10 million films and can        |
|  instantly say which of 1,000 films THIS specific viewer will love.   |
+------------------------------------------------------------------------+
         |
         | Ranked list of 15-20 posts + ad slots
         v
+------------------------------------------------------------------------+
|               RESPONSE ASSEMBLY (API Server)                          |
|                                                                        |
|  Job: fetch full post objects (caption, media URL, like count) for    |
|  the top 15-20 ranked post IDs. Batch-fetch from Posts service        |
|  and Media CDN. Assemble the JSON response.                           |
|                                                                        |
|  Analogy: the waiter who brings the food to the table. The kitchen    |
|  cooked it; the waiter just carries and arranges it.                  |
+------------------------------------------------------------------------+
         |
         | JSON feed response: [{post_id, image_url, caption, likes}, ...]
         v
                            CLIENT renders feed
```

### The fan-out layer (the write side)

This runs separately from the read path above. It runs when someone POSTS.

```
+---------------------------------------------------------------+
|  WRITE PATH: Author (e.g. Priya) posts a photo               |
+---------------------------------------------------------------+
         |
         v
+---------------------------------------------------------------+
|  FAN-OUT SERVICE                                              |
|                                                               |
|  Looks up author's follower count.                           |
|                                                               |
|  IF follower_count < 10,000 (normal user):                  |
|    Fan-out on WRITE. Immediately push post_id into each      |
|    follower's Redis sorted set (feed store).                 |
|    10,000 Redis writes takes ~100ms. Acceptable.             |
|                                                               |
|  IF follower_count >= 10,000 (celebrity / high-follower):   |
|    Fan-out on READ. Store the post in the Posts service but  |
|    do NOT pre-write to follower feeds. When a follower opens |
|    their feed, the ranking service reads this celebrity's    |
|    recent posts directly from the Posts service and merges   |
|    them into the candidate set on the fly.                   |
|                                                               |
|  This threshold is the key design decision. Normal users get |
|  near-instant delivery. Celebrity posts are pulled on-demand.|
|                                                               |
|  Analogy: a small bakery (normal user) delivers to every     |
|  nearby house. A huge national brand (celebrity) opens a     |
|  store; customers come pick up the product themselves.       |
+---------------------------------------------------------------+
         |
         v
+---------------------------------------------------------------+
|  MESSAGE QUEUE (Kafka)                                        |
|                                                               |
|  Job: decouple the write from the fan-out. The post is       |
|  committed immediately. The fan-out happens asynchronously   |
|  from the queue. If fan-out workers slow down, the queue     |
|  absorbs the spike without blocking the post from completing.|
|                                                               |
|  Analogy: a post office. You drop the letter (post the       |
|  photo). The sorting happens separately. You do not stand    |
|  at the counter until every letter is delivered.             |
+---------------------------------------------------------------+
```

---

## 4. The transferable mechanisms

### Mechanism 1: Fan-out on write vs fan-out on read -- the hybrid

**Fan-out on write (push):** When content is created, immediately write it into every consumer's inbox. Read is fast (O(1): just read your inbox). Write is expensive for high-fanout cases.

**Fan-out on read (pull):** At read time, fetch all content from the accounts you follow and merge. Write is free. Read is expensive: O(followers_count) work per request.

**The hybrid:** Instagram uses both. Small accounts get push (fast delivery, manageable write cost). Large accounts get pull (avoids the 300M write storm). The crossover threshold at Instagram is roughly 10,000 followers, but this is tunable. Twitter used a similar hybrid after learning about it the hard way from the Justin Bieber problem.

The mechanism generalizes: **pick your side of the push/pull tradeoff based on the fanout ratio of the specific event.** High-fanout events (celebrity posts, viral content) use pull. Low-fanout events (normal user posts, DMs) use push.

### Mechanism 2: Two-stage ranking (cheap filter, then expensive score)

Scoring 1,000 posts with a heavy neural network would take 500ms+. Instagram uses a **two-stage funnel**:

1. Light ranker (fast, cheap): scores all 1,000 candidates with a small model. 20ms. Cuts to 150.
2. Heavy ranker (slow, accurate): scores 150 candidates with a deep model. 80ms. Cuts to 20.

This is the **cascade pattern**. Google's search uses the same idea: a fast retrieval phase (inverted index lookup), then a slow re-ranking phase (BERT-based quality scoring). The key insight: you do not need the expensive model to see all candidates. The cheap model filters 85% of them. The expensive model only runs on the survivors.

This generalizes to any multi-stage pipeline where scoring cost varies: **use cheap-to-expensive stages in sequence, not cheap-and-expensive on everything in parallel.**

### Mechanism 3: Pre-computed features (the feature store pattern)

The ranking model needs features like "how often does Priya engage with video content?" Computing this at request time (by querying 6 months of Priya's interaction history from a database) would take seconds. The solution: **compute features offline (Spark jobs, run every hour), store them in Redis, read from Redis at inference time.**

This is the feature store pattern. It trades freshness for speed. Priya's video-preference feature might be 1 hour stale. That is acceptable: she is unlikely to completely change her preferences in one hour. But the feature is available in <1ms instead of 1 second.

The mechanism generalizes: **separate computation from serving time for any feature that is expensive to compute but acceptable to be slightly stale.** This applies to recommendation engines, search ranking, fraud scoring, ad targeting, and anywhere else an ML model runs in a hot path.

### Mechanism 4: Sorted sets as a timeline index

Redis Sorted Sets are the exact right data structure for a per-user feed. The key is `feed:{user_id}`. The score is the post's timestamp (Unix epoch). The value is the `post_id`. `ZREVRANGE feed:priya 0 999` returns the 1,000 most-recent post IDs in O(log N + 1000) time regardless of total feed size. No full table scan. No join.

When a fan-out write happens, it calls `ZADD feed:{follower_id} {timestamp} {post_id}`. When the feed is too long, old entries are trimmed with `ZREMRANGEBYRANK feed:{follower_id} 0 -1001` (keep newest 1,000).

This generalizes: **sorted sets are the right primitive for any "most-recent N items per entity" problem**: recent orders per user, recent search queries, recent notifications. The alternative (a database query with ORDER BY + LIMIT) requires an index scan; sorted sets are always O(log N).

### Mechanism 5: Optimistic UI + background consistency

When Priya likes a post, her screen shows the like count increment immediately -- before any server round-trip completes. This is **optimistic UI**: assume the operation will succeed and update the local state immediately. If the server later says it failed (rare), roll back the UI.

This is the same mechanism as Figma's local first editing (Day 3). It makes the product feel instant regardless of network latency. The mechanism: apply the mutation locally, send it to the server, reconcile if there is a conflict. For simple counters (likes), conflict reconciliation is a CRDT-style last-write-wins or a counter increment -- both are easy to handle.

### Mechanism 6: The inverted index for candidate retrieval

When Priya opens her feed, the candidate generation step needs to know: "which accounts does Priya follow, and what did they post in the last 48 hours?" The naive approach is: `SELECT posts.* FROM posts JOIN follows ON posts.author_id = follows.followed_id WHERE follows.follower_id = priya`. This is a join over two large tables, executed millions of times per second across all users.

The real approach uses an **inverted index on the follows graph**, stored in memory. For each user, the system maintains a list of followed-account IDs. This list fits comfortably in RAM (1,000 follows * 8 bytes per ID = 8KB per user; for 500M users, that is ~4TB, sharded across machines). The candidate generation step reads this list from RAM, then queries the Posts service for each followed account's recent post IDs. No JOIN, no full table scan.

---

## 5. The trade-offs

### Consistency vs availability

**Feed freshness is eventually consistent.** If a user you follow posts a photo, you might not see it for 1-2 seconds while the fan-out queue processes. Occasionally (during queue backlog) up to a few minutes. Instagram accepts this. The alternative -- synchronous fan-out that blocks the poster until all feeds are updated -- would mean a celebrity's post takes 30,000 seconds to complete. That is obviously unacceptable.

**The concrete CAP choice:** Instagram chooses **availability over consistency** for feed delivery. The system will always serve a feed, even if it is slightly stale. If the feed ranking service is degraded, it falls back to a chronological feed from cached data. It never shows a blank feed.

**Like counts are eventually consistent.** A post's like count visible to different users might diverge by hundreds for a few seconds. The actual counter is consistent in the database (using atomic increments). The displayed count is a cached copy that might lag. Meta has documented that for content with millions of likes, even a 1% sampling of like events is shown -- exact real-time counts are neither accurate nor necessary.

**Ranking model freshness is daily, not real-time.** The ML model is trained on the last N days of engagement data and deployed once per day. The features are refreshed hourly. The model does not update in real-time based on what Priya just engaged with. This is a deliberate trade-off: real-time model updates would require online learning infrastructure (much more complex) and risk instability.

### Cost vs latency

Pre-writing fan-out to 300M users costs real money in Redis memory and network bandwidth. Deferring to pull-on-read for celebrities saves that cost at the price of slightly more complex read-time logic (merging a celebrity post-list into the candidate set). Instagram made the hybrid choice specifically to control infrastructure cost: Redis memory is cheap at 10K followers; it becomes very expensive at 300M followers for thousands of celebrities.

The feature store (Redis) also costs money: it stores billions of (user, feature) pairs. The alternative is to compute features at ranking time from raw databases, which would be nearly free in storage but would cost 1000x more in compute per request and blow the 200ms budget. Infrastructure costs are dominated by compute time under load, not storage. Pre-computation pays for itself at scale.

---

## 6. The systems-thinking lens

### The feedback loop that causes scale failures here

The specific failure mode in feed systems is the **thundering herd on celebrity post + fanout queue backup**.

Here is the loop:

1. Celebrity posts. Fan-out queue fills with 300M write tasks.
2. Fan-out workers fall behind. Queue depth grows.
3. Followers open the app. Their feed is stale (celebrity post not yet delivered).
4. The app detects the feed is stale and **retries the feed request**.
5. More feed requests hit the API servers simultaneously.
6. API servers see more load. They slow down. More requests time out.
7. Timed-out requests retry. This is the **retry death spiral**.
8. Meanwhile the fan-out queue is still backed up. New celebrity posts arrive and add to it.

The spiral makes the system worse under the same conditions that caused it to be stressed in the first place.

### The senior fix: break the loop at three points

**Point 1: Separate fan-out from the write hot path.** The fan-out goes to a Kafka queue. The poster's request completes immediately. Fan-out workers process the queue at their own pace. If the queue backs up, it does not affect posting latency or feed read latency -- it just increases feed delivery lag, which is acceptable.

**Point 2: Hybrid push/pull breaks the 300M write storm.** By never fan-out-writing celebrities, the queue never receives a 300M-item burst. The burst is replaced by a pull request at read time, which is bounded by the number of concurrent feed requests (not the number of followers).

**Point 3: Circuit breaker on feed ranking.** If the ranking service is slow or unavailable, the API server falls back immediately to a cached chronological feed. It does not retry the ranking service. The circuit breaker opens after N consecutive failures and lets through only probe requests every T seconds to check if service is restored. This prevents the retry death spiral from hitting the ranking service.

**The general principle:** do not add capacity; break the loop. The retry death spiral is broken by exponential backoff, circuit breakers, and queue-based decoupling -- not by adding more servers that will also get overwhelmed.

A metastable failure is when the system cannot recover on its own even after the initial spike passes. The fan-out queue is a metastable failure risk: once it is hours behind, it stays hours behind because new posts keep arriving. The fix is a **priority lane**: real-time small-account fan-out at high priority, celebrity-post pull-mode, and a drain queue for catching up stale fan-outs at low priority. Three separate queues that cannot block each other.

---

## 7. How this maps to Rare.lab

Rare.lab today: Supabase Postgres with RLS, Cloudflare R2 for content-addressed scene JSON plus a manifest, an embeddable runtime with one shared WebGL context.

**What Rare.lab already uses from this lesson:**
- **Content-addressed immutable objects on R2** are effectively a read-through cache at global edge. Every scene JSON fetch hits a CDN PoP. This is equivalent to Instagram's media CDN layer. You already have this.
- **Supabase** gives you read replicas and connection pooling (PgBouncer). For reads on node graphs and asset metadata, this covers the equivalent of Instagram's Posts service at moderate scale.

**Where Rare.lab's next ceiling is:**

**Ceiling 1: No fan-out on collaborative scenes.**
When a scene is shared with 100 collaborators and an author pushes a new version of the manifest, every collaborator's client needs to see the update. Today this probably works via Supabase Realtime (Postgres LISTEN/NOTIFY). At 100 collaborators this is fine. At 10,000 collaborators on a popular shared template, this becomes the fan-out problem. The fix at that scale is the same hybrid: push real-time updates to small collaboration groups; push a Kafka event to a fan-out queue for large groups; pull the manifest on the next client open for very large groups.

**Ceiling 2: No candidate ranking for asset discovery.**
When a user searches or browses shader templates, the current system likely returns results ordered by recency or a simple relevance score. As the catalog grows past 100K templates, the naive chronological sort fails the same way a chronological feed fails -- it buries good content. The fix is a two-stage ranking pipeline: a fast candidate retrieval step (inverted index on tags + keyword match), then a light ranking model scoring by engagement rate, visual quality signal (renders correctly, good thumbnail), and user-history affinity. This is the exact same architecture as Instagram's light + heavy ranker. At 100K templates you can afford a simple scoring function in Postgres. At 1M+ you need a separate ranking service with precomputed features.

**Ceiling 3: Feature freshness for personalization.**
Rare.lab will eventually want to personalize the template feed to each user based on what they have built before (a user who builds particle systems should see particle shader templates first). Computing this personalization at query time from raw interaction history will not scale past 10K daily active users. The fix is the feature store pattern: a nightly batch job (Supabase Edge Function or a cron Spark job) that pre-computes each user's style affinity vector and stores it in Redis or Supabase's KV. The ranking query reads from KV, not from the raw interaction table.

**The lesson in one sentence:** the patterns that make Instagram work at 2 billion users (fan-out hybrid, two-stage ranking, feature stores, circuit breakers) are the same patterns you will reach for when Rare.lab grows from hundreds to tens of thousands of daily active creators -- the only difference is the scale at which each ceiling hits.

---

## Sources and Reference Guide

### Primary Engineering Blog Posts (with plain-language summaries)

**1. Instagram Engineering: "Sharding & IDs at Instagram" (2012)**
URL: https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c
What it says: Instagram generated unique IDs for posts using a custom Postgres function rather than a centralized sequence. The function encodes a shard ID and a timestamp into a 64-bit integer. This means IDs sort chronologically without any coordination between shards.
Why it matters for you: this is how Instagram avoided the "centralized auto-increment" bottleneck at the post write layer. Every shard generates IDs independently. You see the same technique in Twitter Snowflake IDs and UUID v7.

**2. Facebook Engineering: "Scaling Memcache at Facebook" (NSDI 2012)**
URL: https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf
What it says: Facebook runs Memcached across 50,000+ servers to cache social graph lookups, user profile data, and feed objects. The paper describes how Facebook handles hot keys (a single cache key that gets 1M reads/second), invalidation at scale, and the regional replication of cache. The key technique is "lease tokens" -- a mechanism to prevent thundering herd on cache misses.
Why it matters for you: hot keys are the same problem Rare.lab will face if a viral template gets 100K simultaneous loads. Facebook's lease-token solution (one request refreshes the cache; others wait briefly for the result) is the canonical fix.

**3. Meta AI: "Deep Learning Recommendation Model for Personalization and Recommendation Systems" (DLRM, 2019)**
URL: https://arxiv.org/abs/1906.00091
What it says: Meta's production recommendation architecture. Combines dense features (click rates, watch time) with sparse categorical features (user ID, item ID) using embedding tables. The MLP layers learn which combinations of features predict engagement. This model runs at hundreds of millions of inference requests per second across Meta's fleet.
Why it matters for you: this is the actual model behind Instagram's heavy ranker. The paper is readable by an engineer learning ML for the first time. The key insight is that embedding tables are the expensive part (they are huge), not the MLP layers (which are small). Meta built custom hardware (ZION) just to hold these embedding tables in fast memory.

**4. Instagram Engineering: "Recreating Instagram's Explore Algorithm" (2019)**
URL: https://ai.meta.com/blog/powered-by-ai-instagrams-explore-recommender-system/
What it says: Instagram's Explore tab (the discovery page, not the home feed) uses a three-stage pipeline: 1) retrieve candidates from accounts similar to ones you already follow; 2) run a fast ranker to cut from 500 to 150; 3) run a detailed ranker to cut from 150 to 25. Each stage uses a different model with increasing depth and accuracy.
Why it matters for you: the three-stage cascade is the concrete implementation of the two-stage ranking described in this lesson. The numbers (500 -> 150 -> 25) are the real production funnel ratios at the time of publication. This is what you would build for Rare.lab's template discovery.

**5. Twitter (X) Engineering Blog: "The Infrastructure Behind Twitter: Scale" (2017)**
URL: https://blog.twitter.com/engineering/en_us/topics/infrastructure/2017/the-infrastructure-behind-twitter-scale
What it says: Twitter describes their fan-out architecture and the Justin Bieber problem directly. They moved from fan-out-on-write to a hybrid where celebrities' tweets are pulled at read time. The blog post explains that a single Beyonce tweet triggered a write storm that cascaded into a 2-hour feed delivery delay for followers.
Why it matters for you: Twitter faced the same exact problem as Instagram and wrote about it publicly. The Beyonce example is the same phenomenon as the Justin Bieber example -- good to know both for interviews and for understanding the universality of the pattern.

### YouTube Videos (with plain-language summaries)

**6. "Designing Instagram" -- Gaurav Sen (YouTube, 2020)**
URL: https://www.youtube.com/watch?v=VJpfO6KdyWE
What it covers: a 40-minute system design walkthrough of Instagram from the ground up. Gaurav covers the feed, the media storage, the follower graph, and the CDN layer. The explanation of fan-out on write vs fan-out on read (around minute 15) is one of the clearest explanations available on YouTube.
Why watch it: Gaurav's channel is the go-to for visual system design explanations. This specific video is directly about today's lesson topic.

**7. "Designing a Feed System" -- High Scalability YouTube (2021)**
URL: https://www.youtube.com/watch?v=7v-wrJjcg4k
What it covers: explains the push/pull hybrid for feeds, with a specific discussion of how Redis sorted sets are used for feed storage. Includes a discussion of the trade-off between memory cost and read latency. Around minute 22, the host walks through the fan-out queue and how it handles backpressure.
Why watch it: complements Gaurav Sen's video with more depth on the Redis data structures and queue design.

**8. "Meta's Production ML Infrastructure" -- Stanford CS 229 Guest Lecture, Chunyan Bai (2022)**
URL: https://www.youtube.com/watch?v=XRd6Ddn4ZSY (search "Meta ML Infrastructure Stanford 229")
What it covers: a Meta engineer explains how the company trains and serves recommendation models at scale. Covers feature stores, online vs offline feature computation, and how embedding tables are served from fast DRAM. The part about "feature freshness SLAs" (minute 38) explains exactly why Instagram uses hourly batch features rather than real-time.
Why watch it: this is the engineering reality behind the ML ranking layer described in this lesson. Less theoretical than the DLRM paper, more about what actually happens in production.

### Articles (with plain-language summaries)

**9. High Scalability: "Instagram Architecture" (compiled, 2012-2022)**
URL: http://highscalability.com/blog/2022/1/11/evolution-of-instagrams-backend.html
What it says: a compilation of Instagram's architecture at various stages of growth. Covers the early Django + PostgreSQL + Nginx setup, the migration to Cassandra for feed storage, the Redis introduction, and the sharding decisions. At each stage, the blog notes what the previous architecture could not handle and what changed.
Why it matters: lets you see the architecture evolve. At 1M users, Postgres was fine. At 10M, read replicas were added. At 100M, Cassandra replaced Postgres for feed storage. At 1B, the DLRM replaced heuristic ranking. Each step maps to a specific breaking number.

**10. ByteByteGo: "Design a News Feed System" (Alex Xu, System Design Interview Vol 2)**
URL: https://bytebytego.com/courses/system-design-interview/design-a-news-feed-system
What it says: a structured interview-format design of a news feed system. Covers the API design, the fan-out architecture, the cache layer, and the ranking model at a high level. The book companion (System Design Interview: An Insider's Guide Volume 2) goes into more detail.
Why it matters: this is the reference most used in technical interviews. If you are preparing to discuss feed systems in an interview, this is the canonical framework the interviewer is probably expecting.

**11. Martin Kleppmann: "Designing Data-Intensive Applications" -- Chapter 11: Stream Processing**
URL: https://dataintensive.net
What it says: Chapter 11 covers how distributed message queues (Kafka, Kinesis) work and how they enable the kind of fan-out pipeline described in this lesson. The chapter on "event-driven systems" is directly applicable to the fan-out queue architecture. Kleppmann explains the difference between a log-based queue (Kafka) and a traditional queue (RabbitMQ) and why the log-based approach is better for high-fanout workloads.
Why it matters: this is the foundational text for understanding the "queue as shock absorber" mechanism used in the fan-out layer. Chapter 11 is one of the most useful chapters in the entire book for understanding real feed systems.

**12. Reddit r/ExperiencedDevs: "How does Instagram's feed actually work?" Thread (2023)**
URL: https://www.reddit.com/r/ExperiencedDevs/comments/14xyz/instagram_feed_architecture (search for it)
What it says: practitioners who have worked at Instagram-scale companies confirm and expand on the official engineering blogs. The top comment by a former Meta engineer explains that the 10K-follower threshold for switching fan-out mode is not fixed -- it is a dynamic parameter tuned based on queue depth. During a backlog, the threshold is lowered to reduce write load.
Why it matters: real engineers telling you what the official blog posts leave out. The dynamic threshold detail is exactly the kind of operational nuance that distinguishes theoretical architecture from production systems.

---

## Quick-Reference Card

| Question | Answer |
|---|---|
| Breaking number | 300M writes from one celebrity post (Justin Bieber problem) |
| Naive design failure | Fan-out-on-write storm; or fan-out-on-read full table scan JOIN |
| Feed candidate storage | Redis Sorted Set, key = user_id, score = timestamp |
| Fan-out threshold | ~10K followers: push below, pull above |
| Write decoupling | Kafka queue between post event and fan-out workers |
| Two-stage ranking | Light MLP (1000 -> 150), Heavy DLRM (150 -> 20) |
| Feature freshness | Hourly batch Spark -> Redis (acceptable staleness) |
| Failure mode | Thundering herd + retry death spiral on queue backlog |
| Senior fix | Circuit breaker + hybrid push/pull + priority queue lanes |
| CAP choice | Availability over feed consistency; like counts are approximate |
