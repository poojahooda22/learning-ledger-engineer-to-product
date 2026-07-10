# Day 28 — How do you check a URL against a blocklist ten billion times a day, without a network round trip and without a database that grows forever?

**Date:** 2026-07-10
**Difficulty:** Expert
**Topic:** Probabilistic data structures at scale: the Bloom filter behind Google Chrome's Safe Browsing, HyperLogLog for cardinality estimation (Redis's implementation, used industry-wide for unique-visitor counting), and the Count-Min Sketch for streaming frequency estimation (the "heavy hitters" / trending-topics problem, as studied on Twitter's firehose), and why an exact hash set or an exact counter is the naive design every one of these replaces
**Stack relevance:** Rare.lab's Cloudflare R2 content-addressed scene JSON and compiled-shader artifacts (Day 23), where "have we already stored this exact blob" is asked on every save, and the embeddable runtime's shared WebGL context, where "how many distinct sessions are actually running right now" and "which compiled shader is hottest across the fleet" are questions worth answering cheaply

---

## 1. The company and the breaking number

**Google Chrome, checking every URL and file a user's browser touches against Safe Browsing's list of known-dangerous sites.** Google states plainly that Safe Browsing now protects more than **5 billion devices** worldwide, and assesses more than **10 billion URLs and files every single day**. That is the number that breaks a naive design, and it breaks it twice, once on the network side and once on the storage side.

The threat list itself is not small or static either. It is a continuously updated, multi-million-entry corpus, fed by crawlers, machine-learning classifiers, and user reports, with entries added and removed constantly as phishing campaigns spin up and get taken down. Google's own Safe Browsing engineers, working within their documented client-side budget, targeted keeping the local data structure **under 2MB** on-device, specifically because shipping and re-syncing anything larger, to billions of phones and laptops, on every list update, was not viable.

So the breaking number is really two numbers pulling in opposite directions: **10 billion checks a day** says "do this locally, with no network call," while **millions of entries, growing without bound** says "you cannot just ship the exact list to the client and keep it small." A naive design that satisfies one of those constraints fails the other.

---

## 2. Why the naive (demo) design dies

The demo version of "check if this URL is on a blocklist" is one of two obvious approaches, and each one is fine for a college project and collapses at Chrome's scale for a different, specific reason.

**Naive design A: every navigation makes a network call to a central lookup service.** The browser hashes the URL, sends it to a Safe Browsing server, the server does an exact lookup (a hash map or a database index) against the canonical threat list, and returns yes or no. This is functionally simple and always exactly correct, because the server holds the real, current list. It fails three concrete ways:

**a. It puts a network round trip on the critical path of every single page load, for every user, everywhere.** Even a fast round trip, 30 to 80 milliseconds, is pure added latency stacked onto the thing users notice most: how fast a page appears to start loading. Multiply that by 10 billion checks a day and you have added a meaningful, permanent latency tax to the entire web, for a check that is "no" the overwhelming majority of the time.

**b. It turns one lookup service into the single point of failure for browsing itself.** If that service is slow or unreachable, every browser depending on it must decide between blocking navigation (breaking the web for everyone during an outage) or failing open (silently disabling protection for everyone during an outage). Neither answer is acceptable at this scale, and both are direct consequences of routing every check through one authority.

**c. It is a live feed of every URL every user visits, straight to one company's servers.** This is not just a scale problem, it is the privacy problem this exact architecture is designed to avoid: even hashed, a server that resolves a hash for every navigation is, in aggregate, watching where billions of people go on the internet in real time.

**Naive design B: ship the exact blocklist to every client and check locally.** This fixes all three problems above, no network call, no single point of failure, no server-side visibility into browsing, by keeping an exact copy of the threat list on-device and matching against it directly, the way a spam filter might keep an exact table of blocked sender addresses. It dies on the storage and bandwidth side instead:

**a. An exact, collision-proof set entry costs real bytes, and the list has millions of entries.** To be exact and collision-free you need something like a full or near-full hash digest per URL, 32 bytes for a SHA-256. A list of even 5 million entries at 32 bytes each is roughly 160MB, an order of magnitude past the roughly 2MB client-side budget Safe Browsing's own designers were working within, and the list only grows.

**b. The list is not static, it is a stream.** New phishing and malware URLs are discovered continuously. An exact structure has to be re-synced, in full or via deltas, constantly, and every re-sync ships more bytes to billions of devices, many of them on metered mobile data. Exact storage that must track an ever-growing set has no ceiling; it either eats more device storage and bandwidth every year, or the product quietly caps how much protection it ships.

Chrome's actual, documented answer to naive design B's problem, going back to its earliest Safe Browsing implementation, was a **Bloom filter**: keep a compact, fixed-size, probabilistic local structure that can answer "definitely not on the list" for the overwhelming majority of URLs with zero network call and zero false negatives, and only fall back to a real server check, naive design A, on the rare cases the local filter cannot rule out. Chromium's own source history shows the literal file `bloom_filter.cc` in `chrome/browser/safe_browsing/`, and a later, published engineering change ("Transition safe browsing from bloom filter to prefix set") replaced that specific structure with a related but more update-friendly one, a sorted set of variable-length hash prefixes, which is what today's Safe Browsing Update API (v4) uses: hash prefixes of 4 to 32 bytes, mostly 4, checked locally, with a full hash sent to the server only when a local prefix happens to collide with a bad one, at which point the server confirms or denies using its complete, unhashed-to-the-client list. Same core idea (approximate locally, verify rarely, on the true answer's authority), better-tuned implementation.

---

## 3. The architecture

The shape Safe Browsing converges on, top to bottom:

```
[Browser, every navigation]
   analogy: a bouncer at the door checking a name against a list
   before letting anyone in
   |
   v
[Local probabilistic structure: hash-prefix set / Bloom-filter-style
 local database, a few MB, synced periodically]
   compute the URL's hash, check the local structure
   -> "definitely not on any list": let the page load, ZERO network
      calls, this is the outcome for the overwhelming majority of
      the ~10 billion checks a day
   -> "possibly on a list" (a local prefix collision): proceed to
      the next layer
   analogy: the bouncer's own memorized rough description of
   trouble-makers; if a face doesn't match anything he remembers,
   he waves you through without radioing anyone
   |
   v  (only for the rare local "maybe")
[Edge / load balancer -> stateless app tier: Safe Browsing lookup /
 real-time API]
   analogy: the rare case the bouncer radios the front office to
   confirm a name against the master list, instead of trusting his
   own memory
   |
   v
[Cache: recently resolved verdicts for hot / repeatedly-queried
 hash prefixes]
   the same handful of currently-active phishing domains get
   re-queried by many different users' browsers in the same
   window; caching the verdict avoids re-hitting the DB for
   something already just answered
   |
   v
[DB primary + read replicas: the canonical, exact threat corpus]
   this is the one place the FULL, exact list lives; it is never
   shipped whole to any client, only queried, and only for the
   small fraction of checks that got this far
   |
   v
[Sharding: the corpus is partitioned, e.g. by hash prefix range,
 across many machines, because the exact corpus itself is too big
 and too hot to serve from one box at this query volume]
   |
   v
[Async ingestion pipeline: crawlers, ML classifiers, user "report
 this page" signals feed new entries into the corpus continuously]
   analogy: the master list at the front office is never
   "finished"; new photos are added to it all day, every day
   |
   v
[Periodic delta update, pushed back down to every client's local
 structure, keeping it small and current without ever containing
 the exact, complete list]
```

The load-shedding move here is architectural, not reactive: the entire point of the local structure is that it sheds essentially all load, roughly the whole 10 billion checks a day, before it ever reaches the network, the cache, or the database. What would otherwise be the busiest read path in the system is arranged so that, on the fast path, it does not exist.

---

## 4. The transferable mechanisms

These five mechanisms generalize far past URL blocklists. Bloom filters answer "is this thing in the set." HyperLogLog answers "how many distinct things have I seen." Count-Min Sketch answers "how many times has this specific thing happened, and what are the biggest few." All three are instances of the same underlying trade: **give up exactness, in a mathematically bounded and controllable way, in exchange for a fixed, small, predictable memory footprint that does not grow with the true size of the data.**

**a. Approximate membership testing, with a one-sided error guarantee (Bloom filter).** A Bloom filter is a fixed-size bit array plus k hash functions; adding an item sets k bits, checking an item tests those same k bits. It can never produce a false negative, if it says "not present," that is always true, but it can produce a false positive, saying "present" for something never added, at a rate you choose at construction time by trading bits for accuracy (the classic result, going back to Burton Bloom's original 1970 paper, is roughly m ≈ -(n ln p) / (ln 2)^2 bits for n items at false-positive rate p). Apache Cassandra uses exactly this for SSTable lookups: every SSTable on disk carries its own Bloom filter, and a read consults each one before touching disk at all, skipping any SSTable the filter says definitely does not contain the key; Cassandra's own documented default false-positive rate is 1% for size-tiered compaction, and operators can trade roughly 2x the memory for a 10x reduction in false positives (0.1 down to 0.01) when disk seeks are the more expensive resource. The one-sided guarantee is the whole point: a false positive just costs one wasted expensive check, naive design A's server round trip in Safe Browsing's case, a disk read in Cassandra's, but a false negative would mean silently missing a real threat or a real row, which is why the structure is built so that can never happen.

**b. Cardinality estimation in constant space (HyperLogLog).** Counting distinct items exactly requires remembering every item you've already seen, an exact set, which grows linearly with cardinality: counting a billion unique visitors exactly means storing something close to a billion distinct identifiers somewhere. HyperLogLog, from Flajolet, Fusy, Gandouet, and Meunier's 2007 paper, instead hashes each item and tracks, per register (a small fixed number of them, commonly a few thousand), the position of the leftmost 1-bit seen so far, then combines all registers with a harmonic mean; the position of the leftmost set bit is a cheap proxy for "how many distinct hashes have plausibly landed near this register," and averaging across many independent registers cancels out the noise from any single one. Redis's production implementation of this is a genuinely striking real number: a Redis HyperLogLog structure occupies **at most 12 kilobytes**, regardless of whether it is counting a thousand unique items or a billion, with a **standard error of about 0.81%**, accessible via `PFADD` to add an item and `PFCOUNT` to read the estimate, in O(1) time. Twelve kilobytes to approximately count a billion of anything is the mechanism in one sentence: fixed memory, unbounded input.

**c. Frequency estimation and heavy-hitter detection in sublinear space (Count-Min Sketch).** Where a Bloom filter answers "is it in the set" and HyperLogLog answers "how many distinct things," Count-Min Sketch, from Cormode and Muthukrishnan's 2005 paper "An Improved Data Stream Summary," answers "how many times has this specific item occurred," approximately, using a small 2D array of counters (d rows by w columns) and d independent hash functions, one per row. Adding an occurrence of an item increments one counter in every row; estimating an item's count takes the **minimum** across its d row-counters, which is the trick: any single row can overestimate because of hash collisions with other items, but the true count is never overestimated by more than a bounded amount in every row simultaneously, so taking the minimum across independent rows cancels most of that noise. This is the standard mechanism behind streaming "trending now" and heavy-hitter detection, exactly the shape of finding which words or phrases are spiking across a live firehose of posts: a common practical pattern keeps exact counts (via something like the Misra-Gries algorithm) for a bounded set of current top candidates, maybe the top 1,000, and falls back to a Count-Min Sketch for the enormous long tail everything else belongs to, giving exact accuracy exactly where it matters and cheap fixed-memory approximation everywhere it doesn't.

**d. Verify-on-collision: let the cheap structure absorb the load, and only escalate the ambiguous cases to the expensive, authoritative check.** This is the single mechanism all three structures share with plain old L1/L2 caching and CDN edge logic: a fast, approximate, local layer resolves the overwhelming majority of requests by itself, and only the rare case it cannot confidently resolve pays the cost of the slow, exact, authoritative path. Safe Browsing's local hash-prefix check absorbing ~10 billion checks a day and escalating only real collisions to a server round trip is this pattern applied to security; a Bloom filter guarding a database read is the same pattern applied to storage.

**e. Mergeability: these structures compose across shards for free.** A Bloom filter's bit array, an HLL's register set, and a Count-Min Sketch's counter grid are all just fixed-size arrays that combine with simple, associative, per-slot operations, bitwise OR for a Bloom filter, taking the max register value for HLL (`PFMERGE` in Redis does exactly this), and elementwise addition for Count-Min Sketch. That means you can build one of these structures independently, per shard, per time window, or per worker, entirely in parallel with no coordination, and then merge the results afterward to get the answer for the combined data, the same divide-and-conquer shape that makes MapReduce-style aggregation work, applied to approximate counting instead of exact.

---

## 5. The trade-offs

**Accuracy is the axis this system trades, not the classic availability-versus-consistency split, and it is traded per data type on purpose.** For "is this URL malicious," the design accepts a bounded rate of false positives (a safe URL occasionally, harmlessly, triggers one extra server check) but engineers away false negatives entirely (a Bloom filter/hash-prefix structure mathematically cannot miss an entry that was added), because the two error types have wildly different costs: a false positive costs one wasted network round trip, a false negative lets a real phishing page through undetected. For "how many unique visitors" or "what's trending," by contrast, both over- and under-counting by roughly 1% is simply fine, because the number feeds a dashboard or a ranking decision, not a security gate or a financial ledger. The same family of data structures, used for the wrong data type, would be a mistake: nobody should back a billing counter or an inventory count with a HyperLogLog, because "approximately how much money you owe someone" is not an acceptable sentence.

**Cost versus latency, made concrete in bytes.** The entire justification for all three structures is a direct trade of a small, bounded, quantifiable error rate for orders of magnitude less memory and, in Safe Browsing's case, the near-total elimination of network latency from the hot path. Twelve kilobytes standing in for what could be gigabytes of exact storage, or a few megabytes of local Bloom-filter data standing in for a 160MB+ exact list re-synced constantly, is not a rounding-error optimization, it is the difference between a workable client-side design and an impossible one at this scale.

---

## 6. The systems-thinking lens

**The feedback loop: silent accuracy decay, not a crash, is how these structures actually fail in production.** A Bloom filter or a Count-Min Sketch sized correctly for n items at construction time keeps getting more items added to it as the underlying dataset (a growing threat list, a longer streaming window, more distinct visitors) grows past n, and unlike a full disk or an out-of-memory crash, nothing about this failure is loud. The false-positive rate simply, smoothly, and continuously rises as more items get packed into the same fixed-size structure, more bits collide, more counter cells fill up, and every downstream system relying on "mostly local, rarely escalate" quietly starts escalating more often, or, worse for a Count-Min Sketch tracking heavy hitters, starts reporting inflated counts for ordinary items that happen to collide with genuinely popular ones. There is no error, no alert, no stack trace, just a slow, compounding degradation that looks like "the system is getting a little slower and a little less accurate" until someone eventually notices the local hit rate has crept down and the escalation path, naive design A's expensive round trip, is now firing far more often than it was ever provisioned for.

**The senior fix is capacity planning for the sketch itself, and bounding it with rotation, not adding more precision after the fact.** The instinctive patch, "make the hash functions better" or "just make the filter bigger," treats the symptom in isolation and still leaves the structure growing without bound forever. The actual, structural fix, visible in how real systems use these structures, is to bound growth explicitly: Cassandra ties Bloom filter sizing to the expected key count of each SSTable and rebuilds it on compaction rather than letting one filter absorb an unbounded stream of keys forever; a trending-topics pipeline keeps its Count-Min Sketch scoped to a rolling time window (the last hour, not all of history) and starts a fresh sketch each window rather than growing one sketch indefinitely; Safe Browsing's local structure is entirely replaced by each periodic update rather than patched in place. The transferable move is the same one this ledger has returned to before: don't let a component that degrades gracefully under growth quietly absorb that growth forever, give it an explicit lifetime, a resize trigger, or a rotation boundary, so degradation is bounded and visible instead of open-ended and silent. A second, sharper version of this same lens: because these structures rely on hash functions, an attacker who can predict which bucket their crafted input lands in can deliberately flood a small number of buckets, a documented, real attack class (hash-flooding denial of service, first widely disclosed in 2011 against several languages' unkeyed hash tables), degrading a Count-Min Sketch's or Bloom filter's accuracy on purpose rather than by accident; the defense is the same principle applied at construction time, seed the hash functions with a secret, per-instance key so an outside attacker cannot predict, and therefore cannot deliberately engineer, which buckets their input will collide into.

---

## Map to Rare.lab's stack

**Where this applies directly: the content-addressed R2 save path from Day 23.** Every time Rare.lab's editor saves a compiled shader artifact or a scene-JSON blob, the content hash becomes its address, and the natural question before writing anything is "do we already have a blob at this hash." Done naively, that is an R2 `HEAD` request, a network round trip, on every single save, exactly naive design A's shape: fine at low volume, a real latency and request-cost tax once users are saving frequently, especially from a node-based editor where small edits can trigger frequent autosaves. A small Bloom filter of known content hashes, kept warm at the edge (a Cloudflare Worker, or client-side, refreshed periodically from the manifest that already tracks what's in R2), answers "definitely new, just write it" locally for the common case, a genuinely novel scene edit or shader variant, with zero network call, and only falls through to the real R2 existence check on the rarer case of a probable duplicate, precisely mirroring Safe Browsing's local-filter-then-verify shape, and directly reusing the manifest infrastructure Day 23 already established rather than adding new coordination.

**Two secondary opportunities worth naming, not yet worth building.** First, the embeddable runtime shares one WebGL context across potentially many embeds; knowing how many genuinely distinct sessions are actively rendering right now, for capacity and pricing signals, is a HyperLogLog-shaped question, an approximate live count in a few kilobytes, not a query that should hit Postgres per render frame. Second, identifying which compiled shader program is the current heavy hitter across the whole runtime fleet, the one worth keeping warm in a shared GPU program cache versus the long tail worth evicting, is exactly the Count-Min Sketch heavy-hitters pattern from section 4c, and it composes directly with Day 16's hot-key lesson: the sketch is how you'd detect the hot key in the first place, cheaply, before deciding what to do about it.

---

## References and summaries

**Google, "How Google Safe Browsing's Enhanced Protection Mode keeps you safe online"** (blog.google)
https://blog.google/products/chrome/google-chrome-safe-browsing-one-billion-users/
Source for this lesson's headline scale numbers: Safe Browsing protects more than 5 billion devices worldwide and assesses more than 10 billion URLs and files every day.

**Google for Developers, "Safe Browsing Update API (v4)"**
https://developers.google.com/safe-browsing/v4/update-api
The primary source for the current mechanism: clients download a local database of hash prefixes and check locally first; only a local prefix match ("collision") triggers a request to Google's servers, which respond with full hashes for verification, and the server never learns the actual URLs a client visits unless a local match occurs.

**Google for Developers, "URLs and Hashing," Safe Browsing APIs (v4)**
https://developers.google.com/safe-browsing/v4/urls-hashing
Source for the hash-prefix design detail: prefixes range from 4 to 32 bytes, with most lengthened only when they collide with a popular URL's hash.

**Chromium source, `chrome/browser/safe_browsing/bloom_filter.cc`, and Chromium Code Review issue 10896048, "Transition safe browsing from bloom filter to prefix set"**
https://chromium.googlesource.com/chromium/chromium/+/refs/heads/main/chrome/browser/safe_browsing/bloom_filter.cc
https://chromiumcodereview.appspot.com/10896048/
Primary evidence that Chrome's original Safe Browsing implementation used a literal Bloom filter client-side, later replaced by a prefix-set structure, the direct engineering ancestor of today's v4 hash-prefix design.

**Burton H. Bloom, "Space/Time Trade-offs in Hash Coding with Allowable Errors"** (Communications of the ACM, 1970)
The original Bloom filter paper; source for the general false-positive-rate-versus-bit-count relationship referenced in section 4a.

**Apache Cassandra Documentation, "Bloom Filters"**
https://cassandra.apache.org/doc/latest/cassandra/managing/operating/bloom_filters.html
Source for the SSTable-read-path use case and the documented default false-positive rate of 0.01 (1%) for size-tiered compaction, plus the roughly 2x memory for 10x accuracy trade-off between 0.1 and 0.01.

**Redis Documentation, "HyperLogLog"**
https://redis.io/docs/latest/develop/data-types/probabilistic/hyperloglogs/
Source for the concrete Redis implementation numbers: at most 12 kilobytes of memory per HyperLogLog structure, a standard error of about 0.81%, and O(1) `PFCOUNT`.

**Flajolet, Fusy, Gandouet, Meunier, "HyperLogLog: The Analysis of a Near-Optimal Cardinality Estimation Algorithm"** (AofA 2007)
https://hal.science/hal-00406166v1
The original HyperLogLog paper, source for the leading-zero-position-plus-harmonic-mean mechanism described in section 4b.

**Cormode and Muthukrishnan, "An Improved Data Stream Summary: The Count-Min Sketch and its Applications"** (Journal of Algorithms, 2005)
http://dimacs.rutgers.edu/~graham/pubs/papers/encalgs-cm.pdf
The original Count-Min Sketch paper, source for the d-rows-by-w-columns structure and the take-the-minimum estimation rule described in section 4c.

**Redis, "Count-Min Sketch: The Art and Science of Estimating Stuff"**
https://redis.io/blog/count-min-sketch-the-art-and-science-of-estimating-stuff/
Practical, production-oriented explanation of Count-Min Sketch used for streaming frequency estimation and heavy-hitter detection.
