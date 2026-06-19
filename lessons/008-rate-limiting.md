# Day 8 — How does rate limiting protect a system at 100,000 requests/second?

**Date:** 2026-06-19
**Difficulty:** Intermediate-Advanced
**Stack relevance:** Cloudflare Workers, Durable Objects, Redis, Supabase Postgres, API design

---

## 1. Named company + the breaking number

**Company: Stripe**

Stripe processes roughly 250 million API calls per day. That is ~3,000 req/sec on average. On peak days (Cyber Monday, product launches by large merchants) it spikes past 50,000 req/sec across all customer API keys.

**The breaking number: ~10,000 atomic counter writes per second to one Postgres table.**

Each Stripe API key has a rate limit (default: 100 requests/second per key). The naive way to enforce this is to write to a `rate_limits` row on every single API request. At 10,000 concurrent customers each hammering their limit, you need roughly 1 million atomic counter updates per second. A tuned Postgres instance on good hardware handles maybe 20,000-50,000 simple writes per second before lock contention and WAL throughput become the bottleneck. The math does not close. The system breaks before it can defend itself.

**Why this specific number matters:** The rate limiter is the first line of defense. If the defense itself cannot handle the load it is supposed to limit, the attacker (or a traffic spike) has already won by making the guard collapse. The rate limiter must be cheaper to update than the thing it is protecting.

---

## 2. Why the naive (demo) design dies

The demo version every engineer writes on day one:

```sql
-- One row per API key, per minute
CREATE TABLE rate_limits (
  key_id TEXT,
  window_start BIGINT,  -- floor(now()/60)
  count INTEGER,
  PRIMARY KEY (key_id, window_start)
);

-- On every API request:
INSERT INTO rate_limits (key_id, window_start, count)
VALUES ($1, $2, 1)
ON CONFLICT (key_id, window_start)
DO UPDATE SET count = rate_limits.count + 1
RETURNING count;
-- Then check: if count > limit, return 429
```

This works fine for 10 req/sec in a demo. At scale, it collapses in three ways:

**A. Row-level lock contention.**
Every request for the same API key tries to write to the same row at the same time. Postgres serializes those writes with a row lock. With 100 concurrent requests to one key, 99 of them are waiting in a queue behind a lock. The latency to increment a counter goes from 0.1ms to 50ms. Your rate limiter is now slower than the thing it is supposed to protect. Tail latency on your API explodes.

**B. The boundary burst (fixed window's fatal flaw).**
A clever client -- or just a normal burst -- sends 100 requests at 11:59:59.9 and 100 more at 12:00:00.1. Both windows see exactly 100 requests, within the limit. But your server received 200 requests in 200 milliseconds, double the intended rate. The naive fixed window cannot see this. Real systems at Stripe get hammered this way by legitimate retry logic during rolling deployments.

**C. No defense at depth.**
Your database is at the bottom of your stack, behind app servers, behind a load balancer, behind your edge. A DDoS sending 1 million requests/sec still burns through your app server connection pools and your load balancer before the rate limiter can reject anything. By the time the DB row gets updated, you have already paid the compute cost of every request you wanted to block. The guard is inside the vault.

---

## 3. The architecture (top to bottom)

```
[Clients: browsers, mobile apps, third-party services]
         |
         | HTTPS
         v
[Edge Layer: Cloudflare / Fastly]
  Single job: block obvious abuse before it touches your origin.
  IP-based rate limits (e.g. 1,000 req/min per IP).
  DDoS absorption. TLS termination.
  Costs ~0.5ms. Blocked requests never reach your servers.
  Analogy: the airport security checkpoint. Most threats stop here.
         |
         | Only clean traffic passes
         v
[Load Balancer]
  Single job: spread traffic across app pods.
  Layer 4 or Layer 7. No rate limit logic here.
  Analogy: the hotel front desk routing guests to rooms.
         |
         v
[Rate Limiter Middleware]  <-- lives inside every app pod
  Single job: decide yes/no for this specific API key.
  Runs BEFORE any business logic. Adds 1-2ms latency.
  Reads and writes to Redis (not Postgres).
  Returns 429 + Retry-After header if limit exceeded.
  Analogy: the ticket checker at the museum turnstile.
         |
         +--------> [Redis Cluster]
         |            Single job: store counters in RAM.
         |            Sub-millisecond reads and writes.
         |            Sharded across nodes by key_id hash.
         |            Each node handles ~200,000 ops/sec.
         |            Token buckets, sliding windows, sorted sets.
         |            Analogy: the scoreboard visible to all ticket checkers.
         |
         | (only requests that pass the limit check)
         v
[Stateless App Tier]
  Single job: actual business logic.
  Talks to Postgres only for real work (reads, writes, joins).
  Never touches rate limiting infrastructure.
         |
         v
[Postgres Primary + Read Replicas]
  Single job: durable data (API responses, billing records, audit logs).
  Rate limiting logic never touches this in the hot path.
  Reads fan out to replicas. Writes go to primary.
```

**The core insight this diagram teaches:** move the enforcement as far up the stack (toward the client) as possible, and make each enforcement layer so cheap that it cannot itself become the bottleneck. Redis at 200k ops/sec per node is the right guard for a 100k req/sec API. Postgres at 50k writes/sec is not.

---

## 4. The transferable mechanisms

### A. Token Bucket (the production standard)

Imagine a bucket that holds N tokens (the burst capacity). Every second, R tokens are added back (the sustained rate). Each API request removes 1 token. If the bucket is empty, reject with 429.

Real numbers for a typical Stripe API key: N = 200 tokens (burst), R = 100 tokens/sec (sustained rate). A customer who does nothing for 2 seconds accumulates 200 tokens and can spend them all at once on a burst. After the burst they are at 0 and fill back at 100/sec.

**Why this beats a simple counter:** it models real usage. Clients naturally batch, retry after failures, and burst on startup. A simple counter-per-window punishes legitimate clients. The token bucket lets good behavior through while still enforcing the average rate.

**Redis implementation (atomic via Lua script):**

```lua
-- KEYS[1] = tokens key, KEYS[2] = timestamp key
-- ARGV[1] = capacity, ARGV[2] = rate (tokens/sec), ARGV[3] = now (unix ms)
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local tokens = tonumber(redis.call('GET', KEYS[1]) or capacity)
local last_ts = tonumber(redis.call('GET', KEYS[2]) or now)

-- Refill based on elapsed time
local elapsed = (now - last_ts) / 1000.0  -- convert ms to seconds
tokens = math.min(capacity, tokens + elapsed * rate)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('SET', KEYS[1], tokens, 'PX', 60000)   -- 60s TTL
    redis.call('SET', KEYS[2], now, 'PX', 60000)
    return 1  -- allowed
else
    redis.call('SET', KEYS[2], now, 'PX', 60000)
    return 0  -- denied, 429
end
```

The Lua script runs atomically inside Redis. No other Redis command executes between lines. No race conditions. No external locks. One network round-trip.

---

### B. Sliding Window Counter (O(1) memory, ~1% approximation)

The pure sliding window would require storing a timestamp for every request in the window, O(requests) memory. Not practical at scale. The approximation uses two integers:

```
prev_count = requests in the previous full window
curr_count = requests in the current window so far
elapsed = how far into the current window we are (fraction 0.0 to 1.0)

estimated_count = prev_count * (1 - elapsed) + curr_count
```

If `estimated_count > limit`, reject. Two integers per API key, one subtraction. Error is at most 1% compared to a true sliding window. This is what Cloudflare uses in their rate limiting product because it is cheap enough to run on billions of requests.

Real example: limit is 100 req/min. At 0:45 (75% into the current minute): prev window had 80 requests, current window has 60 so far. Estimated = 80 * 0.25 + 60 = 80. Under the limit. No 429.

---

### C. Local + distributed hybrid (the real production optimization)

A Redis round-trip on every request is ~1ms over a local network. At 100k req/sec, that is 100k Redis writes per second per service, using 100k network calls. Expensive.

The optimization: each app pod keeps a local in-memory counter. It syncs to Redis only every 100 requests or every 100ms, whichever comes first. This means:

- 99% of requests never touch Redis (pure in-memory, ~100 nanoseconds)
- Redis is written 1,000x less often (1,000 req/sec instead of 100,000)
- Worst-case overshoot: `number_of_pods * batch_size` extra requests before the limit is enforced

For 10 pods and 100-request batches: a user could exceed their limit by at most 1,000 requests before being cut off. For most API products this is acceptable. For billing-critical hard limits (Stripe's actual billing boundary), it is not, so they use strict Redis for those endpoints only.

---

### D. Idempotency key (rate limiting's twin)

Rate limiting asks: "how many times?" Idempotency asks: "have we already done this exact thing?"

Both require a shared fast store. Stripe uses the same Redis cluster for both. An idempotency key is a client-generated UUID sent in the request header. Stripe stores `{idempotency_key -> serialized_response}` in Redis with a 24-hour TTL. If the same key arrives again (client retry on timeout), Stripe returns the cached response without re-executing the charge. No double charge. No extra DB write.

This is why Stripe's API idempotency is fast: the duplicate detection and deduplication both happen in Redis, before Postgres is ever touched.

---

### E. Graceful degradation: headers tell the client what to do

Every 429 response must include:
- `Retry-After: 14` (seconds until the client can retry)
- `X-RateLimit-Limit: 100` (what the limit is)
- `X-RateLimit-Remaining: 0` (tokens left)
- `X-RateLimit-Reset: 1687500060` (Unix timestamp when the window resets)

Without `Retry-After`, clients retry immediately. See section 6 for why this causes a cascade failure.

---

## 5. The trade-offs

### CAP theorem, made concrete per data type

| Data | Consistency choice | Reason |
|------|--------------------|--------|
| API key limits (billing boundary) | CP (strict Redis, synchronous) | Overshoot = real money lost |
| Leaderboard / analytics limits | AP (approximate, local cache) | 1% overshoot acceptable, latency matters |
| IP-based DDoS protection at edge | AP (local PoP counters, async sync) | Speed matters most, false positives okay |
| Idempotency keys | CP (single Redis write, fencing) | Duplicate execution = data corruption |

**The key insight:** not all rate limits have the same consistency requirement. A hard limit on a paid API tier is a money boundary. A soft limit on a free-tier search is a quality-of-service hint. Design each limit separately.

### Algorithm trade-offs

| Algorithm | Memory/key | Burst-friendly | Boundary burst | Latency |
|-----------|-----------|----------------|----------------|---------|
| Fixed window counter | O(1) | Yes | Vulnerable (2x burst) | Lowest |
| Sliding window log | O(requests) | No | Immune | Medium |
| Sliding window counter | O(1) | Moderate | ~Immune (1% error) | Low |
| Token bucket | O(1) | Yes, by design | Immune | Low |
| Leaky bucket | O(queue depth) | No, queues instead | Immune | Adds delay |

**Token bucket wins** for API rate limiting: O(1) memory, allows legitimate bursts, immune to boundary attacks, one Redis round-trip per request.

**Leaky bucket wins** when you want to smooth output, not reject input: a job queue, a notification sender, a billing event emitter. It adds delay instead of dropping work.

### Cost vs. latency

- Redis Cluster: ~$0.10 per million ops at cloud pricing. At 100k req/sec: ~$8,640/day in Redis ops. Expensive. The optimization (local batching) cuts this to ~$86/day.
- Edge enforcement (Cloudflare): ~$0.50 per million invocations. But blocked requests never reach your origin, saving the compute cost of every request you would have had to process and reject anyway.
- Durable Objects (for Cloudflare Workers): ~$0.00015 per request but strong consistency. Better for strict single-writer semantics. Worse for sustained high throughput because each Object is a single writer.

---

## 6. The systems-thinking lens: the retry death spiral

**The feedback loop:**

1. Traffic spike hits your service. Backend load climbs past capacity.
2. Rate limiter starts returning 429 to all requesters.
3. Every client sees 429, waits the Retry-After interval, and retries at the exact same time.
4. The retry wave is identical in size to the original spike. The service, which was already overloaded, receives a second full-sized wave before it has recovered.
5. 429 again. Another synchronized retry. The service never recovers. This is a metastable failure: the system is stuck in a stable bad state even though each individual component is "working correctly."

**The loop that breaks systems:** synchronized retries after 429.

**Naive fix (does not break the loop):** add servers. More capacity means you can absorb a bigger initial spike, but the synchronized retry storm is equally larger. You are treating the symptom.

**The senior fix (breaks the loop at its root):** all three of these together.

1. **Exponential backoff with full jitter in the client:**

```python
import random
import time

def retry_with_jitter(fn, max_attempts=5, base_delay=0.5, cap=30):
    for attempt in range(max_attempts):
        try:
            return fn()
        except RateLimitError:
            if attempt == max_attempts - 1:
                raise
            # Full jitter: uniform random within the exponential window
            delay = random.uniform(0, min(cap, base_delay * (2 ** attempt)))
            time.sleep(delay)
```

Full jitter spreads retries uniformly across the entire backoff window. Instead of 1,000 clients all retrying at t+14s, they retry randomly between t+0 and t+14s. The server sees a smooth draining trickle, not a synchronized wave. The loop is broken.

AWS published the seminal analysis of this in their "Exponential Backoff And Jitter" post and found full jitter reduces contention by 8x compared to pure exponential backoff.

2. **Circuit breaker on the server:**

If a downstream service (say, your DB) has an error rate above 50% for 10 consecutive seconds, stop forwarding requests to it entirely. Return 503 immediately from the circuit breaker (open state). After 30 seconds, let a few requests through (half-open). If they succeed, close the circuit and resume normal operation.

This matters because without a circuit breaker, a slow DB still receives every request. Timeouts pile up, threads are held open waiting, connection pools exhaust. The "timeout" spiral kills you even when the rate limiter is working fine.

3. **Backpressure at the queue boundary:**

For non-interactive work (emails, webhooks, invoice generation): do not reject. Queue it. A bounded queue upstream of your workers converts a spike into a draining stream. When the queue is full, then reject. Clients get slow responses or delayed processing, not a wall of errors.

A Celery queue or SQS FIFO queue with `maxReceiveCount=3` and dead-letter queuing is the concretize version: each job retries three times with exponential backoff before being parked in the DLQ for inspection.

---

## 7. Map to Rare.lab's stack

Rare.lab: Supabase Postgres with RLS, Cloudflare R2 (immutable scene JSON + manifests), Cloudflare Workers, WebGL runtime.

**What you already have:**

- Cloudflare Workers sits at the edge. You can enforce rate limits there before requests touch Supabase or R2. This is the correct layer for protecting public APIs.
- R2 is content-addressed and immutable. There is no rate-limiting problem for read traffic to R2 at the asset level; R2 is a CDN-adjacent store and handles parallel reads natively.

**Where your next ceiling is:**

**The shader compiler endpoint is the hot spot.** If you expose an HTTP endpoint that takes a node graph (JSON) and compiles it to GLSL/WGSL, this is compute-heavy (10-200ms per compile). One enthusiastic user (or a CI pipeline) sending 50 compile requests in parallel can saturate your Worker CPU budget and cause latency spikes for all other users.

Fix: implement a token bucket per user in a Cloudflare Durable Object. Each user gets their own Durable Object instance. The instance is the single writer for that user's token bucket, which eliminates the distributed counter problem. No Redis needed. No race conditions. Strong consistency for free.

```typescript
// Cloudflare Durable Object: one instance per user_id
export class CompilerRateLimiter {
  private tokens: number;
  private lastRefill: number;
  private readonly capacity = 10;
  private readonly ratePerSec = 2;  // 2 compile/sec sustained, 10 burst

  async fetch(request: Request): Promise<Response> {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.ratePerSec);
    this.lastRefill = now;
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return new Response("allowed", { status: 200 });
    }
    return new Response("rate limited", {
      status: 429,
      headers: { "Retry-After": "1" }
    });
  }
}
```

**The runtime embed fan-in.** If thousands of third-party sites embed your WebGL runtime via a `<script>` tag and it phones home on load (for telemetry, license validation, manifest fetching), you will see a fan-in spike every time a popular site publishes a new post. Rate limit by referring domain at the Worker layer, not by user. Block domains that exceed 100 loads/min. This pattern is different from API rate limiting: the "user" here is the embedding site, not the end visitor.

---

## 8. References with summaries

### Primary engineering sources

**1. Stripe Engineering Blog: "Scaling your API with rate limiters" (2017)**
URL: https://stripe.com/blog/rate-limiters

The original primary source for production rate limiting design. Stripe describes four rate limiter types they run simultaneously: request rate limiter (rejects after threshold), concurrent request limiter (limits parallel in-flight requests per user), fleet usage load shedder (sheds low-priority traffic under load), and worker utilization load shedder (drops traffic when the app tier's worker threads exceed 70% usage). The insight that multiple limiters serve different failure modes is the key takeaway. Read this first.

**2. GitHub Engineering Blog: "How we scaled the GitHub API with a sharded, replicated rate limiter in Redis" (2023)**
URL: https://github.blog/engineering/infrastructure/how-we-scaled-github-api-sharded-replicated-rate-limiter-redis/

GitHub migrated from a single Redis instance to a sharded, replicated Redis cluster for their API rate limiter in March 2023. The post explains the specific failure modes of a single Redis node (hot key problem: a single GitHub user's counter hashing to one node and creating a hotspot), how they used client-side consistent hashing to distribute user counters across nodes, and how they handle Redis node failure without allowing unlimited requests through. Concrete failure scenarios with real traffic numbers.

**3. Redis Official Tutorial: "Build 5 Rate Limiters with Redis"**
URL: https://redis.io/tutorials/howtos/ratelimiting/

Redis's own documentation walking through five implementations: basic fixed-window with INCR+EXPIRE, sliding window log with ZADD+ZCOUNT, sliding window counter, token bucket with Lua, and leaky bucket. Includes complete Lua scripts for each. The best hands-on reference for actually building these. The section on why you need Lua scripts (atomicity across multiple Redis commands) is essential reading before you write any rate limiter code.

**4. ByteByteGo: "Design A Rate Limiter" (System Design Interview, Chapter 4)**
URL: https://bytebytego.com/courses/system-design-interview/design-a-rate-limiter

The clearest visual walkthrough of the standard rate limiter interview design. Alex Xu covers the algorithm comparison, the distributed architecture with Redis, the client-server communication pattern, and the edge cases (race conditions, synchronization across nodes). The diagrams are the best in the field for whiteboarding. This is the chapter to read the day before a system design interview that might include rate limiting.

---

### Algorithm deep dives

**5. Arpit Bhayani: "Sliding Window Rate Limiter" (detailed with diagrams)**
URL: https://arpitbhayani.me/blogs/sliding-window-ratelimiter/

Arpit walks through the sliding window log implementation (ZADD timestamp, ZCOUNT range, ZREMRANGEBYSCORE old entries) with Python code and actual Redis commands. Shows exactly why the sorted set approach is O(requests) memory and why the sliding window counter approximation was invented as the practical alternative. Clear explanation of the boundary burst attack against fixed windows with a worked example.

**6. Arcjet Blog: "Rate limiting algorithms: Token Bucket vs Sliding Window vs Fixed Window"**
URL: https://blog.arcjet.com/rate-limiting-algorithms-token-bucket-vs-sliding-window-vs-fixed-window/

A clean modern comparison of the three most common algorithms with implementation in TypeScript. Especially useful for the section explaining that no algorithm is universally best, and the choice depends on whether you care more about strict enforcement, memory efficiency, burst tolerance, or boundary attack resistance. Arcjet is an edge security SDK built on top of these patterns.

**7. Smudge.ai: "Visualizing algorithms for rate limiting"**
URL: https://smudge.ai/blog/ratelimit-algorithms

Interactive visualization of fixed window, sliding window log, sliding window counter, token bucket, and leaky bucket using the same traffic trace. Seeing all five algorithms process identical traffic reveals their differences instantly. The bucket burst visualization (token bucket allowing a burst that the sliding window would reject) shows why algorithm choice is a product decision, not just a technical one.

---

### Distributed systems and failure modes

**8. Arpit Bhayani: "Thundering Herd Problem and addressing it with randomness"**
URL: https://arpit.substack.com/p/thundering-herd-problem-and-addressing

Explains the thundering herd from first principles with the concrete scenario of a cache expiring and 1,000 DB queries hitting simultaneously. The fix (cache-lock: only one request rebuilds the cache, others wait) versus the jitter approach (spread cache expiry times randomly). Includes analysis of the retry storm as a specific case of thundering herd. Essential for understanding WHY jitter works, not just that it does.

**9. Medium (Soumyadip Ghosh): "Deep Diving into Cloudflare's Rate Limiting Architecture"**
URL: https://medium.com/@gsoumyadip2307/deep-diving-into-cloudflares-rate-limiting-architecture-7a5fc521ffd3

A detailed reconstruction of Cloudflare's actual rate limiting architecture from their public blog posts and engineering talks. Covers: sliding window counters in Memcache per PoP, Anycast routing to spread traffic, async counter sync between PoPs, the mitigation flag (a boolean stored per IP that allows instant block without recomputing), and local caching to serve the same block decision 1,000 times without a Memcache hit. The mitigation flag pattern is clever and reusable: separate the "should this be blocked?" computation from the "is this blocked?" lookup.

**10. Cloudflare Blog: "Workers Durable Objects Beta: A New Approach to Stateful Serverless"**
URL: https://blog.cloudflare.com/introducing-workers-durable-objects/

The original announcement of Durable Objects. Explains the single-writer guarantee (one instance of each Object exists globally at a time, Cloudflare's infrastructure enforces this), the latency tradeoff (requests to an Object are routed to its home PoP, adding round-trip latency for users far from that PoP), and the use cases where this tradeoff is worth it (rate limiting, collaborative editing, live leaderboards). The rate limiter example is literally the first use case they list.

**11. OneUptime Blog: "How to Implement Rate Limiting in a Single Redis Lua Script" (2026)**
URL: https://oneuptime.com/blog/post/2026-03-31-redis-how-to-implement-rate-limiting-in-a-single-redis-lua-script/view

A production-focused walkthrough of combining fixed window, sliding window, and token bucket into a single Redis Lua script that handles all three in one EVAL call. Explains EVALSHA (load script once with SCRIPT LOAD, call by hash for efficiency) and the NOSCRIPT error handling pattern (if Redis restarts and loses the script, catch NOSCRIPT and reload before retrying). These are the last-mile production details that distinguish a working prototype from a reliable service.

---

## One-sentence summary

Rate limiting is the art of making the defense cheaper than the attack: move enforcement to the edge, use Redis token buckets (not Postgres counters) for sub-millisecond decisions, and always return `Retry-After` with jitter so the clients you push back do not all retry in a synchronized wave that breaks you again.
