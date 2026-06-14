# Day 4 — How does a CDN serve 15 million concurrent streams with near-zero origin egress?

**Date:** 2026-06-14
**Topic:** CDN architecture and the art of never asking the origin for the same thing twice
**Difficulty:** Intermediate-Advanced
**Prereqs:** HTTP basics, what caching is, rough idea of what "a server in another city" means

---

## 1. The company and the number that breaks everything

**Netflix.** 280 million subscribers across 190 countries. Friday night at 9 PM US Eastern time.

The specific number: **1 petabyte of data per hour** leaves Netflix's infrastructure at peak. Roughly 15 million people are streaming simultaneously, each consuming 4 to 8 megabits per second for HD video.

Now run the math on a naive "just put it in one AWS region" design:

- AWS data transfer out of us-east-1 costs $0.09 per gigabyte.
- 1 petabyte = 1,000,000 gigabytes.
- 1,000,000 GB x $0.09 = **$90,000 per hour** in bandwidth fees alone.
- Annualized to peak-like load: **north of $700 million per year**, just for moving bits.

And that is before you factor in the latency: a Netflix subscriber in Mumbai hitting us-east-1 sees a 270 to 310 ms round trip, and TCP video streaming over that RTT, with even 0.5% packet loss, means constant rebuffering. The service would feel broken.

This is the CDN problem: how do you serve global traffic at petabyte scale, at low latency, without the origin paying egress for every single byte?

Netflix's answer is Open Connect. More than **95% of Netflix video traffic worldwide never touches an AWS origin server.** It comes off a hard drive physically sitting inside the user's own ISP's building.

---

## 2. Why the naive design dies

Imagine you build your streaming service with a standard architecture: video files live in S3, encoded by a Lambda pipeline, behind a CloudFront CDN, with one origin in us-east-1.

At 1,000 concurrent viewers this works fine. At 100,000 it starts to strain. At 15 million here is exactly where it falls apart:

**Problem 1: Origin bandwidth wall.**
Even with CDN caching, popular new releases generate a first-miss "cache fill" wave. Stranger Things season 5 drops. Every edge PoP on earth gets a request within 60 seconds of release. Each PoP misses the cache (it's brand new), fires a request to origin. You have 600 edge PoPs. That's 600 simultaneous connections to your origin pulling a 4 GB episode. Origin falls over. This is the thundering herd.

**Problem 2: Cache eviction blows up hit ratio.**
A CDN edge PoP in Singapore serves a population of, say, 2 million users. But it has 2 terabytes of SSD for cache. Netflix has 15,000 titles in 20 different encoded formats (4K, 1080p, 720p, etc.). That's 300,000 objects. Most files are multi-gigabyte. The Singapore edge cannot hold the full catalog. Low-popularity content gets evicted by LRU. A subscriber in Singapore requesting a 2008 Korean film gets a cache miss, falls back to origin. Repeat for 10,000 niche titles. Cache hit ratio crashes.

**Problem 3: Inter-continent egress costs compound.**
Even when CDN edges DO cache, the fill path is expensive. Edge in Singapore caches miss, fetches from CDN's regional hub in Tokyo, which in turn fetches from the US origin. AWS charges egress between regions. Multiply that by 600 PoPs pulling the same 4K film from one US bucket simultaneously and you are paying origin egress three times (S3 to regional hub, regional hub to PoP, PoP to user) instead of once.

**The core structural problem:** a naive CDN still treats the origin as the source of truth that every cache eventually reaches. At petabyte-per-hour scale, that origin-facing path becomes the bottleneck and the cost center.

---

## 3. The real architecture, top to bottom

Here is how Netflix (and CDNs like Cloudflare at 330 cities, and Akamai at 325,000 servers) actually solve this. Read it as a flow from user to data and back.

```
USER (phone, TV, laptop)
    |
    | DNS query to netflix.com
    v
[ANYCAST / DNS STEERING]
    "Which server answers this request?"
    Netflix's DNS or Anycast BGP directs the user
    to the nearest OCA (Open Connect Appliance).
    For Cloudflare: same IP announced from 330 cities,
    BGP picks the closest. For Netflix: BGP session between
    OCA and ISP's router, user traffic routes to OCA.
    |
    v
[TIER 1: ISP-EMBEDDED EDGE APPLIANCES]  <-- the magic layer
    Physical servers inside the user's ISP building.
    Typically 100-500 TB of SSD/HDD per appliance cluster.
    Pre-filled PROACTIVELY with the top N titles for that ISP's
    subscriber base (popularity-weighted, filled overnight).
    Cache hit for popular content: >99%.
    If hit: bytes travel across ONE network hop inside the ISP.
    Latency: <5ms. Egress cost: zero (ISP provides connectivity
    in exchange for peering value, Netflix saves ISP transit cost).
    |
    | MISS (unpopular content, new content not yet filled)
    v
[TIER 2: IXP STORAGE APPLIANCES]
    Located at internet exchange points (IXPs) like Equinix NY,
    LINX London, DE-CIX Frankfurt.
    Each appliance holds NEAR-COMPLETE Netflix catalog (petabytes).
    Purpose: be the "origin shield" that only a small fraction of
    requests reach. Still inside the ISP ecosystem or one peering
    hop away.
    If hit: content served from IXP. Egress cost: minimal.
    |
    | MISS (new release not yet propagated, cold start)
    v
[TIER 3: AWS ORIGIN (S3 + EC2 transcoding)]
    The real origin. <5% of traffic reaches here.
    Contains master encodes, raw uploads.
    When a miss reaches here, the response is cached at the IXP
    appliance (and the edge appliance) for future requests.
    This is also where the nightly PROACTIVE FILL process fetches
    from: each OCA downloads next-day's expected popular titles
    during 2 AM to 6 AM local time.
```

**For a general-purpose CDN like Cloudflare** (no embedded ISP appliances), the same three-tier logic applies but the layers are:

```
USER
    |
[ANYCAST BGP]: same IP announced globally, nearest PoP wins
    |
[EDGE PoP, e.g. Singapore data center]
    Checks local SSD cache.
    L1 cache: in-memory (hot objects, seconds of TTL).
    L2 cache: SSD on local servers (most cached content).
    Hit ratio with tiered cache: 85-95% for typical content.
    |
    | MISS
    v
[REGIONAL UPPER TIER / ORIGIN SHIELD]
    One per major region (e.g. Asia-Pacific upper tier in Tokyo).
    All Singapore, Sydney, and Jakarta PoPs check HERE before
    hitting the customer's origin.
    A popular object missed in 5 Asian edge PoPs simultaneously
    generates 1 origin request (not 5), because the upper tier
    coalesces them via REQUEST COALESCING (see section 4).
    |
    | MISS
    v
[CUSTOMER ORIGIN]
    Your S3 bucket, your Supabase database, your server.
    Ideally sees <5% of global traffic.
```

**Annotated roles of each layer:**

| Layer | Single job | Analogy |
|-------|-----------|---------|
| Anycast BGP | Get the user to the nearest box as fast as possible | A 911 dispatcher routing to the closest fire station |
| Edge PoP | Serve from local SSD if content is there | A corner store with a fridge of popular drinks |
| Regional shield / IXP | Catch the misses before they hit origin; one request on behalf of many | A city warehouse that restocks the corner stores |
| Origin | Source of truth, master data | The factory where goods are made |
| Proactive fill pipeline | Push tomorrow's popular content to edges tonight | The overnight truck stocking every corner store before morning rush |

---

## 4. The transferable mechanisms (the primitives worth owning)

These are the reusable ideas. Learn these and you recognize them everywhere.

**Primitive 1: Content-addressed immutable objects + long TTL.**
The single most powerful CDN trick. Netflix chunks videos into 4-10 second segments with file names like `title1234_segment_00042_1080p.ts`. These names are deterministic and never change. If the content changes, it's a NEW file with a NEW name. This means TTL can be set to 1 year. No invalidation needed. No stale content ever served. You can also pre-sign URLs for auth without worrying about cache poisoning. Rare.lab already does exactly this with its R2 scene JSON: content-addressed immutable files are one of the highest-leverage decisions you can make for cache performance.

**Primitive 2: Request coalescing (also called request collapsing or thundering herd prevention).**
When 200 edge servers simultaneously miss the cache for `strangers5_ep1_segment_0001.ts`, what does the regional shield do? It collapses all 200 misses into ONE outbound origin request. The shield holds the other 199 in a wait queue. When the single response arrives, it fans out to all 199 waiting connections. Origin receives 1 request instead of 200. At peak new-release events, this reduces origin traffic by 90%+ versus a naive cache layer. This is mechanically similar to a mutex: only one goroutine fetches, the rest wait.

**Primitive 3: Proactive push / cache warming (opposite of pull).**
Pull CDN: content sits on origin, edge fetches on demand. Push CDN: you (or a scheduler) push content to edges BEFORE demand arrives. Netflix does this every night. A Steering Service looks at next-day watch probability per title per ISP region (a trained ML model, trained on historical viewing), generates a ranked list per OCA, and the OCA downloads those files during off-peak hours. When Friday morning's subscribers start streaming, the content is already on the physical drive 200 meters from their home router. No origin round trip at all. Cache hit ratio approaches 99%+ for popular content.

**Primitive 4: Consistent hashing for cache locality inside a cluster.**
Inside a single PoP, you have dozens or hundreds of cache servers. When a request arrives, which server do you ask? Random assignment means you need N servers to all independently cache the same popular object, wasting N times the space. Consistent hashing assigns each URL (the cache key) to a specific server in the ring. All requests for `/tile1234.png` always go to the SAME server in the cluster. That server's local SSD becomes the authoritative local cache for that content. Cache utilization goes from 1/N efficiency to near 100% per server. Akamai was founded in 1998 on academic research into consistent hashing; it remains a foundation of every CDN.

**Primitive 5: Stale-while-revalidate.**
Instead of serving a cache miss (slow) or a stale response (wrong), serve the stale response instantly while triggering a background revalidation. The user never waits. The cache refreshes asynchronously. Used everywhere caches age: CDNs, browser caches, service workers. The `Cache-Control: stale-while-revalidate=60` header is the HTTP-level primitive. The real lesson: in most cases, 5 seconds of staleness is undetectable by users, and trading that for zero additional latency is worth it.

**Primitive 6: TTL jitter to prevent synchronized expiry stampedes.**
If you set every cached object to expire at exactly 3600 seconds, and you pre-fill 50,000 objects at 2 AM, they ALL expire at 3 AM. Every edge simultaneously requests origin for 50,000 objects. Origin collapses. Fix: add random jitter. TTL = 3600 + random(-600, +600). Expiry spreads over 20 minutes. Origin sees a smooth trickle instead of a cliff. This is the TTL equivalent of "staggered departure" at a stadium.

---

## 5. The trade-offs

**Consistency vs. availability, broken down by data type:**

| Data type | Choice | Why |
|-----------|--------|-----|
| Video segments (immutable chunks) | Strong consistency by design (content-addressed names, never mutate) | No invalidation needed. Correct by construction. |
| Thumbnails and artwork | Accept eventual consistency (up to 24h TTL) | Slight staleness on a thumbnail is invisible to users. Long TTL = better hit ratio. |
| DRM license tokens / auth | Never cached on shared CDN edge. Short-lived per-request. | Sharing a DRM token between users is a security violation. No caching acceptable. |
| User data (what you watched, progress) | Cached for short periods (seconds), authoritative reads hit DB | The 2-second "resume" latency is fine; accuracy matters more. |

**The CAP choice a CDN makes:**
A CDN chooses **Availability over Consistency** for content. If the cache has stale data, it serves stale data rather than waiting for origin (which might be unreachable). This is the right call for video bytes but the wrong call for auth or pricing data. Know which data type you have.

**Cost vs. latency trade-offs:**

- More PoPs = lower latency + higher infrastructure cost.
- Larger cache footprint per PoP = higher hit ratio + higher storage cost.
- Proactive fill = near-zero miss latency at peak + off-peak bandwidth cost for fill.
- Tiered caching = lower origin load at the cost of one extra network hop on a miss (typically adds 5-20ms vs direct PoP-to-origin, but saves 50-300ms vs user-to-origin round trip).

**The cost math that makes CDNs an obvious yes:**
AWS S3 egress costs $0.09/GB. Cloudflare's zero-egress R2 costs $0.00/GB for CDN delivery. Netflix saves ~$700M+/year by having Open Connect absorb 95% of its traffic. CDNs are not a performance optimization; they are a cost survival mechanism at petabyte scale.

---

## 6. The systems-thinking lens: the feedback loop that actually kills you

**The failure pattern: Thundering Herd into Origin Death Spiral.**

Here is the loop that kills systems that skip proper CDN design or configure it badly:

1. Popular object's TTL expires simultaneously across 300 edge PoPs.
2. 300 PoPs simultaneously fetch from origin.
3. Origin gets 300 parallel connections for a single object. Database and storage IOPS spike.
4. Origin slows down under load, starts returning 503 or timing out.
5. CDN edge nodes, seeing timeouts, can't cache the object. They keep retrying.
6. Load on origin INCREASES (retries + original requests).
7. Origin fully falls over.
8. 100% of all requests now miss the cache (nothing to serve) and try origin, which is dead.
9. Full outage.

**Why adding capacity does not fix this:**
The problem is a feedback loop, not a resource shortage. Adding more origin servers makes the initial crash slightly less likely, but the retry storm after a partial crash will still find the new capacity threshold and exceed it. The loop is still intact.

**The senior engineer's fix breaks the loop, not the limit:**

| Loop stage | The break |
|-----------|-----------|
| Simultaneous TTL expiry (step 1) | TTL jitter eliminates synchronized expiry |
| 300 parallel origin fetches (step 2) | Request coalescing reduces to 1 fetch |
| Retries amplify load (step 6) | Exponential backoff + jitter on retry at every CDN layer |
| Cache can't re-prime (step 8) | Stale-while-revalidate: serve stale while origin recovers, no stampede on recovery |
| Single origin failure takes out CDN (global) | Origin shield: only one regional node contacts origin; edge nodes never directly reach it |

The mental model: a CDN's job is not to make origin faster. It is to **make origin irrelevant** for 95%+ of requests, and to **protect origin from itself** when it does get hit.

---

## 7. Map to Rare.lab's current stack

Rare.lab already uses the most powerful CDN primitive (section 4, Primitive 1): content-addressed immutable R2 objects for scene JSON and shader assets. That's the same architecture Netflix uses for video segments. Long TTLs, no invalidation needed. This is correct and scales to millions of reads for free.

**Where your next ceiling is:**

- **Manifest files are mutable.** The scene manifest (the file that says "fetch these 12 chunks for this shader") changes every time a user saves. Right now, if the manifest is behind R2 with a 1-hour TTL, a user sharing a link immediately after saving may get a stale manifest. The fix: serve manifests through a short TTL (5-30 seconds) or stale-while-revalidate with a Cloudflare Worker that checks a version token. Alternatively, use URL versioning for manifests too (content-addressed manifest names) and let the client always have the latest pointer.

- **WebGL runtime asset delivery.** If Rare.lab's embeddable runtime (the JS/WASM payload) is served from R2 via Cloudflare CDN with a long TTL, deploys of a new runtime version need cache invalidation. Use Cloudflare's Cache Purge API or, better, version the runtime URL itself (`/runtime/v2.1.4/rare-runtime.wasm`). Long TTL + URL versioning = zero invalidation problems.

- **The hot-key problem at SDK embed scale.** If Rare.lab gets 10,000 sites embedding the same visual effect, and a viral social media post sends 1 million hits to `rare.lab/embed/effect/123` within 60 seconds, all those requests race to the nearest Cloudflare PoP simultaneously. The first hit per PoP will miss. With 100 PoPs x 100 concurrent misses each = 10,000 origin requests in 1 second. Enable Cloudflare Tiered Cache and Workers request coalescing now, before the spike, not after.

- **The Supabase RLS origin.** Dynamic metadata (user profiles, project lists, collaboration data) cannot be cached on a shared CDN edge because it's user-specific. This is fine. The rule is: static assets (R2) go through CDN with long TTL; dynamic user data goes through Supabase with short-lived edge caching or no cache. Keep these two paths architecturally separate.

---

## References and What Each One Actually Teaches You

### Primary Sources

**1. Netflix Open Connect Overview (Official)**
Link: https://openconnect.netflix.com/en/
**What's in it:** Netflix's official explanation of how Open Connect works: ISP partner program, two-tier appliance architecture, how ISPs benefit (peering, reduced transit costs), and the basics of proactive content filling. Good starting point for understanding embedded CDN vs cloud CDN tradeoffs.
**Key insight:** Netflix gives ISPs the hardware for free. ISPs host it for free power and space. The win is mutual: ISPs move expensive transit off-net, Netflix reduces egress costs. This is a fundamentally different model from renting Cloudflare PoPs.

**2. "Delivering Netflix at Scale: An Inside Look at Open Connect" (Medium, Ashik M Hussain, 2025)**
Link: https://medium.com/@ashikmhussain.a/delivering-netflix-at-scale-an-inside-look-at-open-connect-78736be47fd5
**What's in it:** End-to-end walkthrough of Open Connect: the two-tier OCA architecture (storage at IXPs + edge at ISPs), how BGP session between OCA and ISP router works to direct user traffic, and how content popularity scoring drives the nightly proactive fill process. Concrete and engineer-focused.
**Key insight:** The OCA does not wait for a user to request content and then cache it (pull model). It predicts what will be popular TOMORROW and loads it TONIGHT (push model). Pull caches need a first-miss. Push caches don't.

**3. Cloudflare: Smart Tiered Cache for R2 (Official changelog, November 2024)**
Link: https://developers.cloudflare.com/changelog/post/2024-11-20-smart-tiered-cache-for-r2/
**What's in it:** How Cloudflare's tiered cache works with R2: an ML model selects an "upper tier" data center close to your R2 bucket, all edge PoP misses route through that upper tier before touching R2. This reduces R2 egress to near zero for popular assets. Directly relevant to Rare.lab's stack.
**Key insight:** With Smart Tiered Cache + R2, Cloudflare automatically minimizes cross-tier traffic. Enabling this for Rare.lab's R2 bucket is a one-click win.

**4. "CDN Cache Stampede and Thundering Herd Mitigation" (PrepLoop / SystemOverflow)**
Link: https://preploop.io/learn/caching/cdn-caching/cdn-cache-stampede-and-thundering-herd-mitigation
**What's in it:** Detailed explanation of why the thundering herd happens (synchronized TTL expiry across edge nodes), the four main fixes (request coalescing, stale-while-revalidate, TTL jitter, origin shield), and rough numbers for each (coalescing reduces stampede traffic by 90%+, jitter spreads load over minutes).
**Key insight:** Each mitigation targets a different stage of the loop. You need all four in production. Any one of them alone is insufficient.

**5. "Algorithmic Nuggets in Content Delivery" (Akamai research paper, Bruce Maggs and Ramesh Sitaraman, UMass)**
Link: https://people.cs.umass.edu/~ramesh/Site/PUBLICATIONS_files/CCRpaper_1.pdf
**What's in it:** The academic paper behind Akamai's founding algorithms. Covers consistent hashing for request routing inside a server cluster, the stable marriage algorithm for load distribution across clusters, and the math behind why consistent hashing achieves near-perfect cache locality with minimal rehashing when servers join or leave.
**Key insight:** Consistent hashing is why CDN clusters can grow or shrink without invalidating the entire cache. When Akamai adds a new server to a PoP, only 1/N of keys need to move (N = number of servers). Without it, adding a server would require re-caching everything.

**6. "CDN Architecture Explained: PoP Design, DNS vs Anycast and Monitoring" (Paessler blog)**
Link: https://blog.paessler.com/cdn-architectures
**What's in it:** Side-by-side comparison of DNS steering vs Anycast routing. DNS steering: authoritative DNS returns different IP based on resolver location (slower to redirect on failure, but more control). Anycast: same IP, BGP routing picks nearest PoP (faster failover, less operator control). Also covers how PoP design affects cache hit ratio.
**Key insight:** Cloudflare uses Anycast. Amazon CloudFront uses DNS steering (Route 53). Neither is universally better. Anycast is faster to failover; DNS gives more per-user routing control and enables features like A/B testing at the routing layer.

**7. Cloudflare "Regional Tiered Cache" blog post**
Link: https://blog.cloudflare.com/introducing-regional-tiered-cache
**What's in it:** How Cloudflare implemented regional upper tiers: customers who enabled it saw 50 to 100 ms improvement in tail cache-hit response times and a 16% increase in overall hit rate after the ML-based cache topology optimizer went live in 2023.
**Key insight:** The 16% improvement from ML-driven tier selection vs static upper-tier configuration shows that cache topology is not a set-and-forget decision. Traffic patterns change; the optimal upper tier for an R2 bucket in Singapore is different from one in Frankfurt. ML can tune this continuously.

**8. "CDN Request Collapsing and the Thundering Herds Problem" (OTTVerse)**
Link: https://ottverse.com/request-collapsing-thundering-herds-in-cdn/
**What's in it:** Focused explanation of request collapsing specifically in the context of video streaming CDNs. Covers how a parent cache node receives multiple parallel misses, issues one fetch to origin, queues the others, and fans out the single response. Includes the numbers: for a global CDN with 200 edge PoPs, a single hot object expiring can trigger 200 simultaneous origin connections within milliseconds; request collapsing reduces this to 1.
**Key insight:** Request coalescing is most critical in the 60 seconds after a cache purge or new content release. That is when all 200+ edge PoPs simultaneously miss. Design your CDN config to have request coalescing (or request collapsing) enabled at the upper tier specifically; edge PoPs can forward misses to the upper tier which coalesces them.

**9. "CDN Traffic Steering Guide" (CDNsun blog)**
Link: https://blog.cdnsun.com/cdn-traffic-steering-guide-dns-anycast-layer-7-multi-cdn-p2p/
**What's in it:** Comprehensive taxonomy of traffic steering methods: DNS-based (GeoDNS), Anycast-based, Layer 7 HTTP redirect, multi-CDN orchestration, and P2P. Real-world use cases for each. Also covers why multi-CDN (using two or more CDNs simultaneously, e.g. Cloudflare plus Fastly) is used by streaming platforms for resilience.
**Key insight:** At truly massive scale (Netflix, YouTube), you run multiple CDNs in parallel and route users to the one with the best performance for their ISP at that moment. Single-CDN is a single point of failure and vendor risk. The routing layer (real-user monitoring + dynamic DNS) becomes its own engineering problem.

**10. "How Google Media CDN achieves high origin offload for Warner Bros. Discovery" (Google Cloud Blog)**
Link: https://cloud.google.com/blog/products/networking/media-cdn-origin-offload-does-trick-for-warner-bros-discovery
**What's in it:** How Google's Media CDN uses three tiers for video streaming (ISP short-tail, PoP medium-tail, select PoP long-tail). Built on the same infrastructure serving 2 billion YouTube users. Describes request coalescing ("multiple simultaneous requests for same content collapse into one origin request") and the integrated architecture that removes the need for a separate origin shield configuration.
**Key insight:** Google bakes tiered caching and request coalescing into a single unified product (Media CDN) instead of requiring manual configuration of origin shields, coalescing settings, and TTLs separately. This is the "batteries included" enterprise CDN model. For a startup, Cloudflare with Tiered Cache + Workers is the closest equivalent.

### Video Resources

**11. "CDN Architecture Explained: How Content Delivery Networks Work" (YouTube, Distributed Systems Tutorial)**
Link: https://www.youtube.com/watch?v=OtMVyKirvDg
**What's in it:** Visual walkthrough of CDN architecture: PoP design, how a DNS query resolves to the nearest edge, cache hit vs miss flow, and origin shield concept. Good for building mental models before reading the deeper sources.
**Key insight:** Best watched to understand the physical geography of CDN edges before reading the algorithmic internals.

**12. "Netflix Networking: Beating the Speed of Light with Intelligent Request Routing" (InfoQ talk)**
Link: https://www.infoq.com/presentations/intelligent-request-routing/
**What's in it:** Netflix engineering talk on how they go beyond geography to route requests: they use network topology data (BGP paths, measured RTTs, ISP relationships) to pick the OCA that gives the best actual throughput, not just the geographically nearest one. Introduces real-user measurement feedback loops.
**Key insight:** Nearest in geography and nearest in network topology are often different. A Netflix subscriber in a small city might get better throughput from an OCA in a different city if the ISP has direct peering there. The routing layer is a real-time optimization problem, not just a map lookup.

---

## Quick-reference cheat sheet

| Problem | Mechanism | One-liner |
|---------|-----------|-----------|
| First request to a new PoP is slow | Cache warming / proactive push | Fill the cache BEFORE demand, not after |
| 300 PoPs simultaneously miss | Request coalescing | Collapse N misses to 1 origin fetch |
| Cache expires, stampede on origin | TTL jitter | Randomize expiry so not all miss at once |
| User gets stale content while waiting for revalidation | Stale-while-revalidate | Serve stale, refresh in background |
| New server in cluster invalidates everything | Consistent hashing | Only 1/N keys move when topology changes |
| Origin still paying egress for cache misses across regions | Tiered caching + origin shield | Misses go to a regional shield, not origin |
| Content that changes needs versioning | Content-addressed naming | Immutable URLs = infinite TTL = perfect hit ratio |
