# Day 19 — How do you cache a trillion items without a stampede taking down the database?

**Date:** 2026-06-30
**Difficulty:** Advanced
**Topic:** Caching strategies and invalidation (cache-aside, write-through, leases, request coalescing, TTL stampede prevention)
**Stack relevance:** Cloudflare R2 content-addressed scene JSON, Cloudflare edge cache, Supabase Postgres + Realtime, the embeddable runtime's shared WebGL context

---

## 1. The company and the breaking number

**Facebook, September 23, 2010. One mis-flagged cache key. Hundreds of thousands of queries per second hit one database tier. 2.5 hours of downtime.**

Facebook's own postmortem (widely quoted, original phrasing preserved across multiple secondary write-ups of the incident): "Today we made a change to the persistent copy of a configuration value that was interpreted as invalid." An automated system that validates cached configuration values flagged one value as invalid. Every web server that read that value tried to "fix" it by re-querying a backing database cluster directly. The fix-queries themselves overwhelmed the database. Worse: the database errors caused by the overload were misread by the same automated system as "still invalid," so it deleted the cache key again, which made every subsequent reader miss again, which generated another wave of fix-queries. The system was correcting itself into a deeper hole. It took 2.5 hours to break the loop, at the time described as Facebook's worst outage in roughly four years.

That single incident is a big part of why Facebook's later memcache engineering, published as ["Scaling Memcache at Facebook"](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala) (NSDI 2013), exists. By 2013 Facebook's memcache tier was, by the paper's own description, the largest memcached deployment in the world: thousands of memcached servers per region, caching on the order of trillions of items, fielding traffic widely reported (in secondary summaries of the paper) as billions of requests per second across the fleet. The exact sentence-level numbers are worth re-reading in the primary PDF before you quote them in a slide deck, but the order of magnitude is not in dispute: this is a cache tier sized to stand between "the entire internet's worth of page views" and "one shared MySQL fleet."

**The breaking number to hold onto:** one popular cache key, read by thousands of stateless app servers, that goes missing for even a few milliseconds. If even 1% of those servers happen to ask for it in the same instant, every one of those requests independently decides "cache miss, go to the database," and the database receives a burst that looks identical to a denial-of-service attack, generated entirely by your own well-behaved code.

---

## 2. Why the naive (demo) design dies

The one-server demo looks like this:

```
Request -> check cache (key) -> hit? return value
                                -> miss? query DB, write to cache, return value
```

This is correct. It is also one bad afternoon away from an outage, for three concrete reasons:

**a. The thundering herd at TTL expiry.**
A cache entry expires after its TTL. The next N requests for that key all see a miss in the same instant, because nothing coordinates them. All N independently query the database for the same row and all N independently write the same value back to the cache. If the underlying query is expensive (a join, an aggregate, a cross-shard fan-out) and N is in the thousands, the database receives thousands of duplicate copies of one query at once. This is the exact mechanism behind the Facebook 2010 outage, just triggered by an explicit delete instead of a TTL.

**b. The stale set race.**
Client A reads a miss, starts fetching from the DB. Before A's fetch finishes, the underlying row changes and a delete is sent to the cache (correctly invalidating the stale entry). A's fetch, which read the *old* value, now finishes and writes that old value back into the cache. The cache now serves stale data for the next full TTL, and nothing about the system "knows" it is wrong. A demo with one writer never sees this. A system with concurrent writers and concurrent cache fills hits it constantly.

**c. Invalidation that never reaches every copy.**
The moment you have more than one cache node (or more than one region), "delete this key" is no longer a single operation. If invalidation is implemented as "the app code calls `cache.delete(key)` after every write," every code path that writes to that row must remember to call delete, in every service, forever. Miss one code path (a backfill script, an admin tool, a new microservice) and that node serves stale data indefinitely, with no error and no alert. This is why Phil Karlton's line, attributed via [Martin Fowler's bliki](https://martinfowler.com/bliki/TwoHardThings.html) (who traces it to Tim Bray's blog, where Bray says he first heard it around 1996-97), has lasted: "There are only two hard things in Computer Science: cache invalidation and naming things."

The demo fails the same way every time: it treats the cache as a fast copy of the database, but never asks **who is responsible for telling the cache when the database changes**, and **what happens when many readers discover the same gap at the same instant**.

---

## 3. The architecture

Top to bottom, annotated with each layer's single job:

```
[Client]
   |  HTTPS request for /scene/42
   v
[Edge / CDN cache]            Cloudflare-style edge PoPs
   serves immutable, content-addressed assets directly, zero origin trip
   analogy: a vending machine restocked overnight, never asks the warehouse mid-sale
   |  (cache miss or non-cacheable request)
   v
[Load balancer]               routes to any stateless app server
   no session affinity required, because state lives in cache/DB, not the box
   |
   v
[Stateless app tier]          the only thing that talks to both caches and the DB
   analogy: a teller who never remembers your last visit, looks it up every time
   |
   v
[L1 cache: in-process]        local memory inside the app server, microsecond latency
   tiny, per-instance, holds the hottest few thousand keys
   analogy: the note taped to your monitor vs. walking to the filing cabinet
   |  (miss)
   v
[L2 cache: distributed]       memcached / Redis cluster, shared across all app servers
   sharded by consistent hashing across nodes (Day 10's hash ring), so losing one node
   only loses 1/N of the keyspace, not all of it
   analogy: a shared filing cabinet down the hall, still faster than the archive building
   |  (miss)                                    |  (node down)
   v                                            v
[Database primary + read replicas]      [Gutter pool: spare cache nodes]
   the source of truth, the expensive path     absorbs traffic from a dead L2 node so
   reads prefer replicas, writes go to primary  the DB never sees the failover spike
   analogy: the archive building, slow but      analogy: a backup teller window opened
   authoritative                                 only when the regular one is broken
   |
   v
[Invalidation pipeline]       tails the DB's write-ahead log / replication stream (Day 17)
   converts committed DELETEs/UPDATEs into cache-delete messages, batched and routed
   to every cache node that might hold the key, AFTER commit, never before
   analogy: a wire-service ticker that only reports facts that already happened
```

**The key structural decision:** invalidation does not live in application code that calls `cache.delete()` after a write. It lives in a pipeline that watches the database's own commit log and generates deletes from what actually committed. This is the same WAL-tailing pattern as Day 17's CDC lesson, applied specifically to cache correctness: the database becomes the one place that has to remember to invalidate the cache, instead of every caller.

---

## 4. The transferable mechanisms

### a. Cache-aside (lazy loading) vs. write-through vs. write-behind

Three different answers to "when does the cache get filled":

- **Cache-aside (lazy loading):** app checks cache, on miss queries the DB and populates the cache, returns the value. Per [AWS's ElastiCache caching strategies guide](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Strategies.html), this is cost-efficient because the cache only ever holds data that was actually requested, and a freshly-replaced empty cache node degrades latency rather than correctness. Downside: nothing proactively keeps it fresh, so it needs an invalidation story.
- **Write-through:** every write goes to the cache and the DB in the same operation, so the cache is never stale for that key. Downside: doubles write latency, and you can end up caching values nobody ever reads.
- **Write-behind (write-back):** writes land in the cache immediately and are flushed to the DB asynchronously. Lowest write latency, highest risk: a cache node failure before the flush loses data that was never persisted.

AWS's own stated best practice is to combine cache-aside reads with a write-through update on the hot path, plus a TTL as a safety net for anything the invalidation pipeline missed.

### b. TTL with jitter

A flat TTL ("every key expires exactly 300 seconds after it was written") creates synchronized expiry: if you cold-started a cache or deployed a config change that touched thousands of keys at once, they all expire together, and you get a self-inflicted thundering herd on a timer. The fix is trivial and load-bearing: add random jitter to every TTL (`300 + random(-30, 30)` seconds) so expirations smear across a window instead of landing on one instant.

### c. Leases and request coalescing (stopping the herd at the source)

Facebook's memcached fork adds a **lease**: on a miss, instead of just returning "not found," memcached hands the requesting client a 64-bit token bound to that key, and will not hand out a second lease for the same key for a fixed window (reported as roughly 10 seconds in the paper). Every other concurrent requester for that same key is told to wait briefly instead of being sent to the database. Only the lease-holder fetches from the DB and writes back, presenting the lease as proof it was the one who was told to. If a delete arrives for that key while the lease is outstanding, the lease is invalidated and the eventual `set` is rejected, closing the stale-set race from section 2b in the same mechanism. Reported in secondary summaries of the paper: this dropped a stampede's peak database query rate from roughly 17,000/s to roughly 1,300/s for the affected key (treat the precise figures as paper-summary numbers worth re-verifying against the primary PDF before quoting in print).

You do not need to fork memcached to get the same effect. Go's [`golang.org/x/sync/singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight) package implements the application-level version: `Group.Do(key, fn)` "executes and returns the results of the given function, making sure that only one execution is in-flight for a given key at a time. If a duplicate comes in, the duplicate caller waits for the original to complete and receives the same results." Same idea as a lease, no protocol changes needed, works with any cache and any backing store.

### d. Probabilistic early expiration (XFetch)

Leases stop the herd once a miss has already happened. XFetch stops the miss from happening in the first place, by recomputing a hot key's value *before* it expires, probabilistically, so that recomputation work spreads out across time instead of synchronizing on the expiry instant. From the algorithm described in ["Optimal Probabilistic Cache Stampede Prevention"](https://archive.org/details/xfetch) (Vattani, Chierichetti, Lowenstein; VLDB 2015) and implemented at the Internet Archive:

```
recompute_early_if:  now - delta * beta * log(random()) >= expiry
```

where `delta` is the measured cost (in seconds) of recomputing the value, `beta` is a tunable multiplier (1.0 by default; higher recomputes earlier), and `random()` draws uniformly from (0, 1). Because `log(random())` is always negative, the formula always adds a positive amount of "early slack" before the real expiry, and because the draw is random, different processes holding the same key independently pick different moments to refresh it. No locking, no coordination between processes, and the paper's claimed property is that the exponential distribution this produces is provably optimal versus refreshing on a uniform schedule.

### e. Negative caching

A miss that goes to the database, finds nothing, and is *not* cached means every subsequent request for a nonexistent key (a typo'd ID, a deleted record someone still has bookmarked, an enumeration attack probing for valid IDs) goes straight through to the database every single time. Caching the absence itself, with a short TTL, turns repeated "not found" lookups into cache hits and protects the DB from exactly the kind of automated retry behavior that made the 2010 Facebook incident self-reinforcing.

### f. The gutter pool: degrade gracefully, don't fail open to the database

When a regular cache node dies, the obvious fallback is "route those keys to another live cache node" or "skip the cache and go straight to the DB." Facebook's paper describes neither: they reserve a separate, isolated pool of spare memcached servers, sized at roughly 1% of a cluster's regular memcached count, called **Gutter**, used only during failures. A client that gets no response from its assigned node retries against Gutter; if Gutter also misses, the client queries the DB and populates *Gutter*, not the original dead node. Reported effect: a roughly 99% reduction in client-visible failures during a node outage, with Gutter's hit rate climbing past 35% within about four minutes of a failure. The reasoning given for not just spilling onto a neighboring regular node: that risks turning a moderate hot key into an overloaded hot spot on whichever node absorbs it, converting one failure into a cascading one. An isolated, intentionally-spare pool absorbs the blast radius instead of spreading it.

---

## 5. The trade-offs

### Consistency vs. availability, per data type

| Data type | Caching choice | Why |
|-----------|----------------|-----|
| Static assets (images, compiled shaders, immutable JSON) | Cache forever, no invalidation needed | Content-addressed by hash; a changed value is a new key, not a stale one |
| User profile / config | Cache-aside + CDC-driven invalidation | Must reflect writes within seconds; staleness is visible and embarrassing |
| Counts (likes, views) | Cache with short TTL, approximate is fine | Off-by-a-few is invisible to users; exactness is not worth the write load |
| Session/auth state | Skip cache or very short TTL | Stale auth state is a security bug, not a UX nit |
| Search/ranking results | Cache-aside, TTL in seconds to minutes | Freshness matters less than the cost of recomputation (Day 18) |

Facebook's own system makes this trade explicit at the protocol level: memcached is explicitly a **demand-filled, look-aside cache**, not the system of record. MySQL is always right; memcached is always allowed to be momentarily wrong, and the entire leases/mcsqueal/gutter machinery exists to bound *how* wrong and for *how long*, not to make the cache perfectly consistent.

### Cost vs. latency

RAM is roughly 10 to 100 times more expensive per gigabyte than disk-backed database storage, but a cache hit is roughly 100 to 1,000 times faster than a database round trip. The economic argument for caching only holds for data that is read far more often than it is written, and skewed enough that a small hot set (often single-digit percent of total rows) accounts for the large majority of reads. Caching cold, rarely-read data is pure cost with no latency win, which is exactly why Facebook's "regional pools" exist: large, rarely-hit items get one shared copy per region instead of one copy per frontend cluster, trading a small amount of extra latency for not paying N times over for memory nobody is reading from N times.

---

## 6. The systems-thinking lens

**The feedback loop that actually causes failure: cache stampede as a self-reinforcing retry storm, not just "the DB got busy."**

Walk the 2010 Facebook outage as the canonical example:

1. A cached config value is flagged invalid by an automated checker.
2. Every reader of that value gets a miss and re-queries the backing database to "fix" it.
3. The database, hit by every reader at once, slows down and starts erroring under the load.
4. The automated checker interprets those errors as "still invalid" and deletes the cache entry again.
5. Step 2 repeats, except now the database is already degraded, so it fails faster and harder.
6. The loop tightens on its own. More capacity does not break it, because every unit of new capacity gets immediately consumed by the next wave of retries.

This is a **metastable failure**: the system cannot recover by adding resources while the loop is running, because the loop itself is what's consuming the resources. It looks identical, from the outside, to a successful self-healing system right up until it doesn't recover.

**The senior fix breaks the loop at the points where one miss turns into many duplicate fetches, not by making the database bigger:**

1. **Coalesce concurrent misses into one fetch** (leases or singleflight). The fix is structural: there is physically only one in-flight database query per key, no matter how many callers are waiting on it.
2. **Recompute before expiry, probabilistically** (XFetch). Removes the synchronized "everyone misses at the same instant" trigger entirely, by spreading recomputation across time before the cliff is ever reached.
3. **Isolate failure-mode traffic from steady-state traffic** (gutter pool). When something does go wrong, the overflow has its own dedicated, bounded capacity instead of competing with (and overwhelming) the primary path.
4. **Treat repeated errors as a signal to back off, not retry harder.** The actual 2010 bug was that errors were being read as "the data is still wrong, delete it again" instead of "the system is overloaded, stop hammering it." A circuit breaker that opens after N consecutive failures and serves stale-but-available data (or a clear error) instead of retrying is what turns a self-reinforcing loop into a self-limiting one.

Notice the common shape with Day 9 (queue as shock absorber) and Day 13 (backpressure): the fix is never "add more database." It's "stop the system from generating more duplicate work than the failure actually requires."

---

## Map to Rare.lab's stack

**What you already have, and it's a bigger win than it looks:**

R2's content-addressed, immutable scene JSON sidesteps almost everything in section 2 by construction. If the cache key is the content hash, there is no such thing as a stale value: a changed scene produces a *new* hash, which is a cache miss for a brand-new key, not staleness on an old one. Section 4's invalidation pipeline, McSqueal, the CDC tailing from Day 17, the entire "who tells the cache the data changed" problem, simply does not exist for this part of your stack. This is the single most transferable lesson from today: wherever you can make data immutable and content-addressed, cache invalidation stops being a hard problem because there is nothing to invalidate, only new keys to add. Lean into this pattern for anything else cacheable in Rare.lab's pipeline: compiled shader bytecode, exported runtime bundles, rendered thumbnails. Hash the output; key the cache by the hash; never invalidate, only evict by LRU/age.

**Where you still need a real invalidation story:**
The manifest (mutable, lives in Supabase Postgres) is exactly the case section 4d/4e describes. Don't invalidate it with app-code `cache.delete()` calls scattered across every write path. You already have the primitive from Day 17: subscribe Supabase Realtime (WAL-based CDC) to the manifest table, and let that stream generate cache-delete events the same way mcsqueal generates them from MySQL's commit log. One pipeline, every write path covered automatically, including the ones you haven't written yet.

**The next ceiling:**
The embeddable runtime's shared WebGL context is doing exactly what section 3's L1/L2 split describes, whether or not it's named that way yet: in-memory compiled shader programs and parsed scene graphs are your L1 (microsecond, per-instance), R2 plus Cloudflare's edge cache are your L2 (network-fast, shared). The risk you haven't hit yet is the thundering herd version of a viral moment: if a scene trends and thousands of embeds load it within the same second, and the edge cache for that specific asset has just been evicted or the deploy just rotated the CDN cache, every one of those thousands of runtime instances can independently fire a fetch to R2 for the same bytes at once. Apply section 4c here before it bites: put a singleflight-style coalescing layer in the runtime's asset loader, so concurrent loads of the same scene hash within one instance (and ideally across instances behind one edge PoP, via Cloudflare's own request coalescing) collapse into one R2 fetch instead of N.

---

## References and summaries

### Primary engineering source

**Facebook / Meta: "Scaling Memcache at Facebook" (Nishtala et al., NSDI 2013)**
https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala
The foundational paper for this entire lesson. Describes memcached used as a demand-filled, look-aside cache in front of MySQL at the scale of a social graph serving over a billion users. Introduces leases (a 64-bit token issued on miss that throttles repeat misses for the same key and rejects stale writes), mcsqueal (a daemon that tails the MySQL commit log, parses committed DELETEs, and batches them into cache-invalidation messages routed through mcrouter), regional pools (one shared cache copy per region for large, infrequently-read items instead of one copy per cluster), cold cluster warmup (a new cluster fetches misses from a warm neighbor instead of the database), and the gutter pool (a small reserved pool of spare cache servers that absorbs traffic during a node failure so the database never sees the failover spike). Note: several numeric figures in this lesson (the 17K/s to 1.3K/s lease impact, exact server counts) are drawn from secondary summaries of the paper rather than a verbatim read of the PDF; re-read the primary source directly before quoting those numbers precisely.

### The real outage behind the engineering

**TechCrunch: "Facebook Goes Down, Twitterverse Reacts" (September 23, 2010)**
https://techcrunch.com/2010/09/23/facebook-downtime/
Contemporaneous coverage of the outage referenced in section 1 and section 6. Facebook's own explanation, as widely quoted: an automated system that validates cached configuration values flagged a value as invalid; every reader's attempt to "fix" it queried the database directly; the resulting load caused database errors that the same checker misread as "still invalid," triggering repeated cache deletes and repeated retries. About 2.5 hours of downtime, described at the time as the company's worst outage in roughly four years. Read this alongside the NSDI 2013 paper, since the gutter pool and lease mechanisms described there are direct engineering responses to exactly this failure mode.

### Distributed cache architecture at a second company

**Netflix / EVCache wiki**
https://github.com/netflix/evcache/wiki
EVCache ("Ephemeral Volatile Cache") is Netflix's memcached-based distributed cache, built on the spymemcached client and deeply integrated with AWS EC2 and Netflix's Eureka service discovery. Confirms two architectural choices directly relevant to this lesson: shards are distributed across the cluster using the Ketama consistent hashing algorithm (the same family of mechanism as Day 10's hash ring), and writes replicate to every configured availability zone while reads are served from the local zone first, falling back to a random other zone only if the local copy is unavailable. The wiki states this design gives "linear scalability of overall data size or network capacity" versus vertically scaling a single larger cache box.

### The request-coalescing primitive, as a library

**Go: `golang.org/x/sync/singleflight` package documentation**
https://pkg.go.dev/golang.org/x/sync/singleflight
The clean, application-level version of Facebook's lease mechanism, with no protocol fork required. `Group.Do(key, fn)` executes `fn` for a given key, and any concurrent caller using the same key waits for the in-flight call to finish and receives the same result rather than triggering a second execution. This is the exact primitive to put in front of any cache-miss-triggers-expensive-fetch code path to prevent duplicate concurrent work, which is the mechanical core of stampede prevention.

### Stampede prevention, the probabilistic version

**Vattani, Chierichetti, Lowenstein: "Optimal Probabilistic Cache Stampede Prevention" (VLDB 2015)**
https://archive.org/details/xfetch
The paper behind the XFetch algorithm: instead of waiting for a key to expire and then coordinating who refetches it, each reader probabilistically decides to recompute the value slightly before its real expiry, using the formula `now - delta * beta * log(random()) >= expiry`, where delta is the measured recomputation cost and beta is a tunable multiplier. Because the trigger is randomized per-reader, recomputation work spreads out across time instead of synchronizing on the expiry instant, and the paper proves the exponential spread this produces is optimal. The Internet Archive's engineering team implemented and presented this at RedisConf 2017; see also a clear walkthrough of the formula at https://www.michal-drozd.com/en/blog/cache-stampede-xfetch/.

### The quote that names why this is hard

**Martin Fowler's Bliki: "TwoHardThings"**
https://martinfowler.com/bliki/TwoHardThings.html
The most commonly cited discussion of Phil Karlton's line, "There are only two hard things in Computer Science: cache invalidation and naming things." Fowler traces the earliest documented sighting to Tim Bray's blog, where Bray says he first heard it around 1996-97; an unverified secondhand account places Karlton saying it at Carnegie Mellon as early as 1970. There is no single canonical published origin; it's an oral-tradition engineering quote, which is itself worth knowing before repeating it as a sourced fact.

### Vendor-neutral reference for the basic patterns

**AWS: ElastiCache caching strategies documentation**
https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/Strategies.html
A clean, vendor-neutral statement of cache-aside (lazy loading) versus write-through, with the stated trade-offs: cache-aside only ever holds data that was actually requested and degrades gracefully on a cold cache, but can serve stale data until the next read after a write; write-through keeps the cache current at write time but pays the cost in write latency and can waste memory caching data that's never read. AWS's stated best practice is combining the two, with a TTL as a backstop for anything an explicit invalidation path missed.

### Video, technical deep-dive

**Hussein Nasser / freeCodeCamp.org: "Memcached Architecture - Crash Course with Docker, Telnet, NodeJS"**
https://www.youtube.com/watch?v=NCePGsRZFus
An hour-long hands-on walkthrough of memcached internals: slab allocation and memory management, LRU eviction, connection handling and threading, and the read/write path, demonstrated live with Docker and Telnet. Useful as concrete grounding for what's actually happening inside the L2 cache node referenced in section 3, beneath the Facebook-specific mechanisms.

### Video, interview-framed

**ByteByteGo: "Cache Stampede Problem Explained: What is Thundering Herd & How to Fix It | System Design Interview"**
https://www.youtube.com/watch?v=TAZGA-aScPA
A shorter, system-design-interview-style explainer focused specifically on the thundering herd / cache stampede problem and standard fixes. Good for rehearsing the section 6 explanation out loud in interview format: name the failure mode, name the loop, name the fix that breaks the loop rather than just adding capacity.
