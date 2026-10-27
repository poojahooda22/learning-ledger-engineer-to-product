# Day 100: One photo, sixty shapes. How on-the-fly image variants (Cloudflare Images, Facebook photo serving) survive 1 million originals without storing or computing every size

**Date:** 2026-10-27
**Difficulty:** Intermediate-Advanced (builds on CDN Day 4, content-addressed storage Day 23, caching Day 19, hot keys Day 16, request coalescing Day 80)
**Topic:** The same picture must reach a 360 pixel phone, a 4K monitor, a retina laptop, a browser that speaks AVIF and one that only speaks JPEG. Storing every combination wastes money. Computing each one per request melts your CPUs. The whole design is about choosing WHICH variants exist, WHEN they are made, and WHAT the cache key is.
**Stack relevance:** Rare.lab has scene thumbnails, preview renders and embeds on R2. Section 9.
**A note on the pasted transcript:** the LeetCode mock interview was already taught in [Day 74](074-leetcode-online-judge-end-to-end.md) (sandbox Day 50, leaderboard Day 54). I did not repeat it. Section 0 reuses its interview recipe on today's topic and Section 10 summarizes the interview in plain language again.
**Honesty labels:** CONFIRMED = read in a search result excerpt today. Limit: I could not open full pages (blog pages came back only as search excerpts or mirrors). INFERENCE = my reasoning, labeled. MEMORY = standard knowledge I did not re-read today. All arithmetic is shown so you can check it.

---

## 0. The interview recipe applied to an image service

The candidate in the transcript went: functional requirements, non-functional requirements, entities, API, naive design, deep dives. Same recipe.

1. **Functional.** Upload an original. Request it at a width, a format, a quality. Serve it fast anywhere.
2. **Non-functional.** Low latency (a hero image should paint in well under a second on mobile). High availability (a broken image is a visible hole in a page). Cost control (storage and CPU are the bill). Eventual consistency is fine for "a new variant appears a second later", but a replaced original must not show stale pixels forever.
3. **Entities.** Original, variant (original + parameters), cache entry, transform job.
4. **API.** `GET /img/{originalHash}?w=800&fmt=auto&q=80`. Upload is separate (Day 79).
5. **Naive design.** Resize on every request, or precompute everything.
6. **Deep dives.** Cache key design, thundering herd on a cold variant, abuse, format negotiation, invalidation.

Same instinct as the candidate: start simple, say what is wrong with it, then add a cache and a queue-like smoothing layer.

---

## 1. Named companies and the breaking numbers

- **Facebook photo serving (CONFIRMED via course slides citing the 2010 Haystack paper, secondary source).** About 260 billion images (about 20 PB) at the time. A separate tier does resizing and cropping so the storage layer only stores bytes. Reported share of daily views by size class: thumbnails 10.2 percent, small 84.4 percent, medium 0.2 percent, large 5.2 percent. Lesson: a few size classes carry almost all traffic, so the variant ladder can be short.
- **Facebook f4 (CONFIRMED via paper abstract excerpt).** Over 65 PB of logical blobs for warm data, cutting effective replication from 3.6 to 2.8 or 2.1. Lesson: old photos are rarely read, so you pay less to keep them (Day 32).
- **Cloudflare Image Resizing / Cloudflare Images (CONFIRMED via excerpts of the Sept 2021 engineering post and docs).** The resizer is a Rust web server that runs on every server of the network, handled by the nearest data center, and it only does work on a cache miss because it is integrated with the normal cache. The same Rust libraries are shared with Polish, a separate recompression service. Predefined variants are limited to width, height, fit and blur, up to 100 per account. "Flexible variants" (any size via the URL) is opt-in.
- **AWS image-optimization sample (CONFIRMED via excerpt).** Normalizes the transformation URL: validates requested transforms, orders them, lowercases them, so that equivalent requests share one cache key. Also turns on CloudFront Origin Shield, an extra cache layer that collapses many simultaneous misses into one origin fetch.

**The breaking number (INFERENCE arithmetic):**
- A shop has 1,000,000 product photos.
- Responsive ladder: 20 widths x 3 formats (AVIF, WebP, JPEG) = 60 variants per photo.
- 1,000,000 x 60 = 60,000,000 variant objects. At 1 CPU second to encode an AVIF (MEMORY: AVIF is slow to encode, often far over a second for large images), that is 60,000,000 / 3,600 = about 16,700 CPU hours, mostly for variants nobody will ever request.
- The opposite failure: a launch email sends 50,000 requests in one second for ONE cold variant of a new 12 MB, 24 megapixel hero image. If each request decodes and resizes it (about 150 ms CPU, INFERENCE), that is 50,000 x 0.15 = 7,500 CPU seconds of identical work in one second.

---

## 2. Why the naive designs die

**Naive A: precompute every variant at upload.**
- 60 million objects, hours of CPU, and a new device size (say a 1,440 pixel width becomes popular) means a backfill of the whole catalog.
- Analogy: printing every photo in every frame size before anyone orders a frame.

**Naive B: resize on every request, no cache.**
- Each request pays decode + resize + encode. Latency is hundreds of milliseconds even when it works. CPU scales with traffic, not with catalog size.
- Analogy: a photo lab that develops the negative again for every customer who asks for the same print.

**Naive C: cache, but the key is the raw URL.**
- `?w=800&q=80` and `?q=80&w=800` are two cache entries for the same bytes. `w=801`, `w=802` ... `w=4000` lets anyone create 3,200 variants of one image on demand. That is both a cache-poisoning-by-volume problem and a CPU attack: every new width is a guaranteed miss that costs CPU.
- Analogy: a coat check that accepts any ticket text, so the same coat gets many tickets and a stranger can invent tickets.

**Naive D: cold-variant stampede.**
- Many viewers hit a variant that is not yet cached. Every one of them misses, every one triggers a transform. This is the thundering herd (Day 19, Day 80).

---

## 3. The architecture, top to bottom

```
Browser / app / embed
  |  <img srcset> or CSS picks a width, browser sends Accept: image/avif,...
  v
Edge CDN (hundreds of cities)
  |  Job: serve the cached variant bytes. Key = originalId + normalized params.
  |  Analogy: a local newsstand holding today's most-asked-for prints.
  v  (miss only)
Normalizer / allow-list (still at the edge)
  |  Job: snap w=801 to w=800, sort params, resolve fmt=auto to a concrete format,
  |  reject unknown params. Makes equivalent requests ONE key.
  |  Analogy: the clerk who rewrites every order slip into the same standard form.
  v
Regional cache / origin shield
  |  Job: collapse N simultaneous misses for one key into 1 upstream request.
  |  Analogy: one person goes to the warehouse, everyone else waits at the counter.
  v
Transform workers (stateless, CPU-heavy)
  |  Job: decode original once, resize, encode, return bytes. Scale on CPU.
  |  Analogy: the darkroom. Expensive, so used only on a true miss.
  v
Original store (object storage, immutable, content-addressed)
  |  Job: the one source of truth for pixels. Variants are NEVER the source of truth.
  |  Analogy: the negative archive. Prints can be remade, negatives cannot.
  v
(optional) Async warm-up queue
     Job: after upload, pre-generate the 3 or 4 variants that carry most traffic.
     Analogy: printing the popular sizes before the shop opens.
```

Where each layer saves you:
- Edge cache turns 99 percent of requests into zero CPU.
- Normalizer keeps the key space small.
- Shield/coalescing turns a herd into one job.
- Immutable originals make invalidation almost disappear (Section 4, primitive 5).

---

## 4. The transferable mechanisms

1. **Derive, do not store: variant = pure function of (original, params).**
   - Because it is a pure function, you can throw any variant away and recompute it. The cache is a performance feature, not a database. This is the same idea as derived data in Day 90 (deletion) and build caches in Day 95.
2. **Bound the key space (cache key cardinality control).**
   - Cardinality = how many distinct keys exist. Snap widths to a ladder (for example 320, 480, 640, 800, 1200, 1600, 2400). Allow-list parameters. Sort them. Lowercase. Reject the rest.
   - Why: every distinct key is a miss that costs CPU, and a miss you did not intend is an attack surface. Cloudflare's own fixed variants (max 100 per account) are this idea in product form.
3. **Request coalescing at the miss (single flight).**
   - If 50,000 requests want key K and K is not cached, the first one starts the transform, the other 49,999 wait on the same in-flight promise. One decode, not 50,000.
   - Origin Shield (AWS) does this across edge locations; inside a worker you do it with a per-key lock or in-flight map (Day 80).
4. **Content negotiation done safely.**
   - Browsers send `Accept: image/avif,image/webp,...`. If the response differs by that header, the cache must know (`Vary: Accept`), but a raw `Vary` fragments the cache because header strings differ slightly. Better: reduce the header to a small enum (avif, webp, jpeg) before it enters the key. AWS sample does the equivalent normalization. (INFERENCE on the enum detail; the normalization idea is CONFIRMED.)
5. **Immutable content-addressed originals.**
   - If the URL contains the hash of the original, then "replace the image" means a NEW hash and a NEW URL. Old cache entries are never wrong, only unused. You can set `Cache-Control: public, max-age=31536000, immutable`. No purge storms (Day 19, Day 23).
6. **Warm the head, compute the tail lazily.**
   - Use the size-class skew (84 percent of Facebook views were the small class, secondary source). Pre-generate the top few variants on upload via a queue; make the long tail on demand. Rule of thumb: precompute what is requested often, lazily compute what is rare.

---

## 5. The trade-offs

**Consistency vs availability, per data type:**
- Original pixels: strong consistency matters (never serve a half-written original). Write once, address by hash, publish only after the upload is verified.
- Variants: eventual is fine. A variant missing for a few hundred milliseconds just means one slow first view.
- "Which hash is current for this product": this pointer needs correctness, and lives in the product database. The image bytes do not.

**Cost vs latency:**
- More precompute = lower first-view latency, higher storage and CPU bills, and most of it wasted on the long tail.
- Lazy compute = cheap, but the first viewer pays 100 to 1,000 ms (INFERENCE range).
- Formats: AVIF is smallest but slow to encode; WebP is a middle path; JPEG is fastest and largest. Choice: precompute cheap formats, lazily compute AVIF only for popular images (MEMORY on relative encode speeds).

**This design chooses:** availability and low cost over eager freshness. Safe because immutable hashes make staleness impossible rather than merely unlikely.

---

## 6. The systems-thinking lens

**The loop: cold-variant thundering herd, plus a cardinality feedback loop.**
- Herd loop: a cold key causes many misses, many misses cause many identical transforms, transforms saturate CPU, saturated CPU raises latency, slow responses make clients and the CDN retry, retries add more misses. This is a retry death spiral with a metastable state (Day 13).
- Cardinality loop (INFERENCE): more distinct keys lower the hit ratio, a lower hit ratio sends more traffic to workers, loaded workers get slower, and slow images make pages feel broken so users refresh, which creates more requests.

**Fixes that BREAK the loop (not just add servers):**
- Single flight: the herd cannot multiply work.
- Normalize and allow-list: the key space cannot grow without bound.
- Concurrency limit on transform workers plus load shedding: serve a slightly larger already-cached variant, or the original, instead of queuing forever (Day 13).
- Negative caching: if a transform fails for a corrupt original, cache the failure briefly so retries do not hammer it.

---

## 7. A worked example, end to end

Shop "Kapda" uploads `saree-red.jpg`, 6000 x 4000, 9 MB. Hash is `a91c...`.

1. Upload (Day 79) stores the original at `orig/a91c...`. Product row now points to `a91c...`.
2. Upload fires a queue message. A worker pre-generates three variants: 640 wide WebP, 1200 wide WebP, 640 wide JPEG. Those carry the home grid and the product page for most phones.
3. Diwali email goes out. 30,000 people open the product page within 10 seconds. Their browsers request `/img/a91c...?w=1200&fmt=auto`.
4. Edge normalizes: `fmt=auto` plus `Accept: image/avif,...` becomes `avif`. A request with `w=1180` is snapped to 1200. Key = `a91c.../w1200/avif`.
5. AVIF 1200 was not pre-generated. First 30,000 requests miss at the edge. The shield collapses them to ONE request. One worker decodes the 9 MB original, resizes, encodes AVIF (maybe 1 to 2 seconds, INFERENCE), stores and returns.
6. The other 29,999 wait up to 2 seconds, get the same bytes. Next 100,000 requests are pure edge hits.
7. The shop replaces the photo. New hash `b27d...`, new URLs. No purge. Old `a91c...` entries expire by LRU.
8. A scraper requests `w=1`, `w=2`, ... `w=4000`. The allow-list snaps or rejects all of them. The key space stays at the ladder, CPU stays flat.

---

## 8. What breaks at the next tier

- **1,000 images:** precompute everything, serve from object storage. Nothing breaks.
- **100,000 images:** precompute-all becomes slow and costly (6,000,000 objects at 60 per image). Move to lazy variants plus CDN cache.
- **10 million+ images:** the miss tail itself is expensive (long-tail images are rarely cached, each miss is a decode of a big original). Add: a shield layer, tiered storage for old originals (f4-style erasure coding, Day 32), CPU autoscaling with a hard concurrency cap, pre-decoded smaller "mezzanine" intermediate (INFERENCE: store one 2400 pixel master and derive all smaller sizes from it, which cuts decode cost for the long tail).

---

## 9. Map to Rare.lab's stack

What Rare.lab already has:
- **Cloudflare R2 with content-addressed immutable scene JSON plus a manifest.** This is exactly primitive 5. Previews and thumbnails can use the same rule: key = `sceneHash + renderParams`, immutable, one-year cache.
- **Cloudflare edge.** Cloudflare Images / image transformations run on the same network, with the cache integration described in the engineering post.
- **Embeddable runtime.** An embed that loads a poster image before the WebGL context starts benefits directly from small, fast variants.

Patterns to use now:
1. **Never put free-form numbers in a URL that does work.** Use a fixed ladder (for example 320, 640, 1280, 1920) and a small set of formats. This protects R2 operations and CPU.
2. **Render the poster once, at publish time.** A scene thumbnail is produced by running the shader, which costs GPU time. That is far more expensive than an image resize. Render ONCE at the top size on publish (a queued job), store the master, derive smaller sizes with ordinary resizing. Do not render shaders on cache miss.
3. **Key by hash, not by scene id.** If the scene changes the hash changes and nothing needs purging.
4. **Cap concurrency of any worker that renders.** A viral embed must degrade to a placeholder, not queue GPU jobs.

Where the next ceiling is:
- **GPU rendering for previews** is the first real cost wall. The cure is rendering once and deriving, plus a dedupe by scene hash so two users of the same scene share one render.
- **Many-variant requests from the embed itself** (device pixel ratio, container width) can multiply keys. Snap them in the runtime before building the URL.
- **R2 Class B read operations** are cheap but nonzero; a high edge hit ratio is the actual cost lever (CONFIRMED in spirit by Day 4 and Day 93; check current R2 pricing before budgeting).

---

## 10. References, summarized in plain language

**The pasted LeetCode mock interview (YouTube, a Google engineer designing LeetCode).** What is inside:
- She starts with what the product must do: list problems, open one and code in any language, submit and get instant pass or fail, and a live leaderboard for weekly contests. Out of scope: login, payments, analytics.
- Quality targets: stay available (a little inconsistency in counts is fine), answer within a couple of seconds, run strangers' code safely, handle about 100,000 people in a contest and millions overall.
- Main objects: problem, user, submission, leaderboard. Main calls: list problems with page and limit, get one problem for a language, submit code as text, get the leaderboard for a contest.
- Storage: DynamoDB, because there are few relations and no joins. The interviewer points out that submissions need a user id and agrees the contest id works as the key for leaderboard queries.
- Running code: not on the API server (malware, CPU hogs, one crash takes everything down). VMs are isolated but heavy, containers are lighter (one pool per language), plus a time limit to stop infinite loops. The interviewer notes VMs isolate better but containers use resources better, and suggests serverless as an option.
- Scale: cache the leaderboard instead of polling the database, add servers sideways, put a queue between the API and the code runners to smooth spikes, retry with exponential backoff, define test cases once in JSON and convert them per language, and use primary plus replica databases for the busy submissions table.
- Resources she names: Hello Interview, and a YouTube channel that the transcript spells "Shrians Jen" (the interviewer adds GKCS). I could not match those spellings to exact channels, so verify before relying on them.
- **How it connects to today:** the cache plus the queue as a shock absorber is the same shape as the shield plus transform workers here. A cold variant is like a contest starting: many people want the same thing at the same second.

**Cloudflare blog, Building Cloudflare Images in Rust and Cloudflare Workers (Sept 2021).** Explains that the resizer is a Rust server that receives an image URL with options, fetches the original, processes it, and returns the result, running on the same machines as the cache so work happens only on a miss. Shared Rust libraries are reused by other products (Polish). Takeaway: put the transformer next to the cache and share code, so every product resizes the same way. Read via search excerpt and mirrors, not the full original.

**Cloudflare Images docs (variants, flexible variants).** Plain version: you can define named presets with width, height, fit and blur (up to 100 per account), or opt in to free-form sizes. Free-form is more flexible and also opens the cache-key and abuse problems in Section 2. Read via search excerpt. A community summary adds warnings about cache keys and signed URLs; I flagged those as unofficial and did not rely on them.

**AWS sample, image-optimization (GitHub) and CloudFront Origin Shield docs.** The sample builds on-the-fly resizing from CloudFront plus a function that normalizes URLs (validate, order, lowercase) so equivalent requests hit one cache entry, and enables Origin Shield. Origin Shield is an extra cache layer in front of the origin that lets one request fetch an object while other layers wait. Docs also say it suits on-the-fly image processing but fits less well for rarely requested or low-cacheability content. Read via search excerpt.

**Facebook, Finding a needle in Haystack (OSDI 2010) and f4 (OSDI 2014), via course slides.** Haystack stores photos as big append-only files so one disk read fetches one photo. A separate tier does resizing. Slides report 260 billion images and 84.4 percent of views on the small size. f4 stores older "warm" blobs with erasure coding, dropping effective replication from 3.6 to 2.8 or 2.1. I read only slide and abstract excerpts, not the papers. Statistics are secondary.

**Where to look next on the wider web (leads only, not read today):** Stack Overflow questions on `Vary: Accept` and CDN cache fragmentation; web.dev articles on responsive images and AVIF; Cloudflare blog posts on Polish and on AVIF support; r/webdev and r/devops threads on "image CDN cost surprises"; Netflix TechBlog posts on artwork personalization for a variation on "many variants per title". I have not verified any of these.

---

## 11. One-line takeaway

An image variant is a pure function of an immutable original, so the whole game is shrinking the key space, collapsing the herd on a miss, and never storing or computing a pixel nobody asked for.

---

## Sources

- Cloudflare, Building Cloudflare Images in Rust and Cloudflare Workers: https://blog.cloudflare.com/building-cloudflare-images-in-rust-and-cloudflare-workers (read via search excerpt)
- Plushcap summary of the same post: https://plushcap.com/content/cloudflare/blog/building-cloudflare-images-in-rust-and-cloudflare-workers (search listing)
- Cloudflare Images, flexible variants: https://developers.cloudflare.com/images/optimization/hosted-images/enable-flexible-variants/ (read via search excerpt)
- Cloudflare Images, key concepts: https://developers.cloudflare.com/images/get-started/key-concepts/ (search listing)
- Cloudflare Image Resizing: https://developers.cloudflare.com/image-resizing (search listing)
- AWS sample, image-optimization: https://github.com/aws-samples/image-optimization (read via search excerpt)
- AWS CloudFront, Origin Shield: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html (read via search excerpt)
- Facebook Research, f4: Facebook's Warm BLOB Storage System: https://research.facebook.com/publications/f4-facebooks-warm-blob-storage-system (abstract excerpt)
- Course slides on Haystack: https://cse.buffalo.edu/~stevko/courses/cse486/spring15/lectures/29-haystack.pdf (search listing, secondary)
- Course slides on f4: https://www.cse.buffalo.edu/~stevko/courses/cse486/spring15/lectures/30-f4.pdf (search listing, secondary)
