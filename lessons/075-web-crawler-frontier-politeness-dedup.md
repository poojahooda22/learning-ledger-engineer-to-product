# Day 75: How does a search engine crawl a billion pages a day without knocking over the sites it visits?

**Date:** 2026-09-29
**Difficulty:** Advanced (a full pipeline with three hard data-structure problems inside it)
**Topic:** The web crawler. The URL frontier (what to fetch next), politeness (never hurt a host), and dedup (never fetch or store the same thing twice). Day 18 covered ranking, Day 28 covered Bloom filters, Day 10 covered consistent hashing. This lesson shows all three doing real work in one system.
**Stack relevance:** Rare.lab has no crawler. But it has the SAME three problems in miniature: a work queue with per-key fairness, content-addressed dedup in R2, and a shared resource (one WebGL context) that must not be hammered. Section 7 maps them.

---

## 0. The framework (same six steps as Day 74)

1. Functional requirements: given seed URLs, download pages, extract links, follow them, store the content. Out of scope: ranking, indexing, rendering JavaScript.
2. Non-functional: polite (never overload a host), robust (the web is hostile and broken), scalable (add machines, get more pages), fresh (re-visit changing pages), extensible.
3. Entities: URL, Host, Page (content plus fingerprint), robots.txt rules, Crawl job.
4. API: internal only. `enqueue(url)`, `next_url()`, `store(page)`. A crawler has no public API. Say that out loud in an interview.
5. Naive design: one loop. `while queue: fetch, parse, push links`.
6. Deep dives: frontier, politeness, dedup, distribution, traps.

---

## 1. The company and the breaking number

**Googlebot-style crawling, and one number: 1 billion pages a day.**

Real anchors (from the sources, Section 9):
- Common Crawl, a nonprofit, adds roughly 3 to 5 billion pages every month, about 100 to 300 TB compressed, on a 9.5+ PB archive.
- IRLbot (Texas A&M) crawled 6.3 billion valid HTML pages on ONE server in 41 days at an average of 1,789 pages/sec.
- BUbiNG (Boldi, Vigna and colleagues) reports over 10,000 pages/sec on one 64-core machine while respecting per-host and per-IP politeness.

**Worked arithmetic (labeled estimate, my back-of-envelope, not any company's number).**
- 1,000,000,000 pages / 86,400 sec = **about 11,600 pages/sec**, sustained, all day.
- Average page about 100 KB (assumption): 11,600 x 100 KB = **about 1.2 GB/sec, roughly 9 Gbit/sec** of download.
- 11,600 fetches/sec means 11,600 DNS lookups/sec if you do nothing clever.
- Frontier: say 10 billion known-but-unfetched URLs x 100 bytes each = **1 TB of queue**. It cannot live in RAM on one box.
- "Have I seen this URL?" asked for every extracted link. A page has about 50 links, so 11,600 pages/sec x 50 = **580,000 URL-seen checks/sec**.

The breaking number is not the bandwidth. It is **580,000 membership questions per second against a set of 10+ billion strings, while never sending one host more than about one request per second.**

---

## 2. Why the naive design dies

**Naive version:** one machine, one FIFO queue in memory, a `seen` hash set, a loop that fetches, parses and appends links. Breadth-first.

It collapses four ways:

- **You DDoS somebody.** BFS from a popular page queues 5,000 links to the same host in a row. You fire 5,000 requests at a small blog in seconds. Its server falls over, its owner blocks your IP range, and you have done real harm. Analogy: a hundred delivery drivers all ringing one doorbell at once.
- **Memory dies.** A Python `set` of 10 billion URL strings at about 100 bytes each is roughly 1 TB. The queue is another 1 TB. Analogy: trying to keep the entire phone book in your head.
- **You crawl the same thing forever.** A calendar page links to "next month", which links to "next month", forever. A site adds `?session=abc123` to every link, so every URL is "new" but the page is identical. These are crawler traps. Analogy: a hallway of mirrors.
- **One slow host stalls everything.** A single-threaded loop waits 30 seconds on a dead server. At 11,600 pages/sec you would need about 350,000 in-flight connections just to hide 30-second stalls. Analogy: one slow customer at the only register.

---

## 3. The architecture

```
Seed URLs + newly discovered links
        |
        v
URL normalizer + filter
  - job: make equal URLs look equal (lowercase host, strip #fragment,
    sort query params, drop tracking params, resolve ../), reject
    non-HTTP schemes, apply URL-length and depth limits
  - analogy: writing every address in the same format before filing it

        |
        v
"URL seen?" test  (Bloom filter in RAM, exact set on disk behind it)
  - job: drop URLs already known. 580,000 asks/sec.
  - analogy: the bouncer who says "you were already here tonight"

        |
        v
URL FRONTIER  (the heart of the crawler)
  |-- FRONT queues: F FIFO queues, one per PRIORITY level
  |     job: decide WHAT is worth fetching first (page importance,
  |     change frequency, freshness deadline)
  |     analogy: triage lanes at an emergency room
  |
  |-- Router: picks a front queue (biased to high priority), moves URL
  |     to the back queue that belongs to its host
  |
  |-- BACK queues: B FIFO queues, and each queue holds URLs of ONE host only
  |     job: guarantee POLITENESS. One host is in exactly one queue,
  |     and one worker owns that queue at a time.
  |     analogy: one checkout lane per store, so a store never gets
  |     two deliveries at once
  |
  |-- Min-heap of (next_allowed_time, back_queue_id)
        job: sleep-free scheduling. Pop the queue whose host is allowed
        to be hit soonest. If its time is in the future, wait.
        analogy: a kitchen timer per table

        |
        v
Fetcher fleet (thousands of async connections per machine)
  - job: DNS (own cache), robots.txt check (cached), HTTP GET with timeout,
    size cap, redirect cap
  - analogy: many couriers, each holding a route, none allowed to stall the depot

        |
        v
Content-seen test (fingerprint: hash for exact, SimHash for near-duplicate)
  - job: "have we already stored this page under another URL?"
  - analogy: recognizing the same newspaper printed with a different cover

        |
        v
Parser / link extractor  ---->  back to the normalizer (the loop closes)
        |
        v
Page store (compressed, append-only: WARC files on object storage)
  - job: durable raw copy for the indexer. Write once, read many.
  - analogy: a warehouse of sealed boxes, never opened in place
```

**Distribution (how it goes from one machine to a fleet).** Hash the HOST NAME (not the URL) with consistent hashing (Day 10) to pick the owning crawler node. Then every URL of one host lives on one node, so politeness and the per-host queue stay local with no cross-machine coordination. A discovered link to another node's host is sent to that node over the network. This is the design idea behind UbiCrawler and BUbiNG (per the papers' abstracts and my reading of their descriptions; see Section 9 for what I did and did not open).

---

## 4. The transferable mechanisms

- **Two-level queue: priority in front, fairness in back (the Mercator frontier).**
  - One queue cannot do two jobs. Priority says "important first". Politeness says "never the same host twice in a row". Split them: front queues for the first, back queues (one host each) for the second, a heap of "next allowed time" in between.
  - Worked example: `en.wikipedia.org` has 40,000 pending URLs, `tiny-blog.example` has 3. Wikipedia's back queue holds 40,000, the blog's holds 3. The heap alternates hosts, so the blog gets its 3 pages fetched in seconds and Wikipedia is paced.
  - This is the same shape as fair dispatch on the Day 74 judge fleet and per-tenant queues in Day 67 (noisy neighbor).

- **Per-host rate limiting from real signals, not a constant.**
  - Default idea: after fetching, wait a multiple of the fetch time before the next hit on that host (Mercator used a factor of roughly 10x the download time, from my memory of the paper; verify in the PDF). A slow server answers slowly, so it is automatically hit less often. That is backpressure built from a measurement.
  - Politeness math: a 10-million-page site at 1 request/sec takes 10,000,000 / 86,400 = **116 days**. This is why big hosts get a larger rate budget and why crawls are never "complete".
  - Honor `robots.txt` (RFC 9309). Google's documented behavior: cache it up to 24 hours, read only the first 500 KiB, and treat repeated 5xx as "site unreachable, pause crawling" rather than "everything allowed".

- **Bloom filter in front of an exact store (Day 28).**
  - 10 billion URLs at a 1% false-positive rate needs about 9.6 bits per URL = **about 12 GB**. That fits in RAM on one node, versus 1 TB for raw strings.
  - The trade: a Bloom filter says "definitely new" or "probably seen". A false positive means you SKIP a real new URL (lose 1% of coverage). It never causes a duplicate fetch. For a crawler that is the right error direction.
  - IRLbot took a different route for exactness at scale: DRUM (Disk Repository with Update Management), which batches millions of key lookups and merges them against disk sequentially instead of doing random I/O. The lesson: when the set outgrows RAM, turn random reads into sorted batch merges (Day 21, LSM).

- **Content fingerprinting: exact hash and SimHash.**
  - Exact: SHA-256 of the body. Catches identical pages at different URLs (`/a` and `/a?utm=x`).
  - Near-duplicate: SimHash gives a 64-bit fingerprint where similar pages differ in few bits. Two pages within a Hamming distance of about 3 bits are treated as duplicates. This catches "same article, different ad banner".
  - Cost: 8 bytes per page. 10 billion pages = 80 GB. Cheap.

- **Crawl budget per host or domain (trap defense).**
  - Cap pages per domain and URL depth. IRLbot's STAR algorithm gave each domain a budget proportional to how many OTHER domains link to it (in-degree), so a spam farm with millions of self-generated pages gets a small budget, and a real site gets a large one.
  - Plus cheap rules: max URL length (about 2,000 chars), max path depth, repeated path segments (`/a/b/a/b/a/b`), too many query parameters.

- **Async I/O and your own DNS cache.**
  - The fetcher is I/O bound. One machine holds tens of thousands of open sockets with an event loop (epoll), not one thread per connection.
  - Resolve DNS yourself with a big cache. Public resolvers will rate-limit you at 11,600 lookups/sec, and Mercator found the OS resolver was a bottleneck (from memory; verify).

---

## 5. The trade-offs

| Data | Choice | Why |
|---|---|---|
| The frontier | Availability, loose ordering | Fetching URL A a few seconds before B does not matter. Losing the whole frontier does matter, so persist it (disk-backed queues, checkpoint). Duplicated queue entries are fine. |
| URL-seen set | Approximate (Bloom), biased to skip | A missed new URL costs 1% coverage. A duplicate fetch costs bandwidth AND host goodwill. Skip is the cheaper error. |
| Page store | Durable, append-only, immutable | Raw pages are the source of truth for indexing. Never mutate. Same reasoning as content-addressed storage (Day 23). |
| Freshness | Cost vs staleness | News front pages re-crawled every few minutes, an old forum thread once a month. You cannot re-crawl everything often: 1B pages/day is your whole budget. Spend it where pages change and matter. |
| Politeness vs throughput | Politeness wins, always | A crawler that gets blocked crawls nothing. Throughput comes from MANY hosts in parallel, never from hammering one. |

**Cost vs latency:** A crawler has no user latency. It trades freshness (how stale the index is) for cost (bandwidth, machines). Depth-first and BFS both lose to importance-first ordering, so the front queues earn their complexity.

---

## 6. The systems-thinking lens

**The feedback loop: the retry spiral against a struggling host, and the trap explosion.**

- **Retry spiral.** A host slows down under load (perhaps partly from your crawl). Your fetches time out. A naive crawler retries immediately. That adds load, the host slows more, more timeouts. You are now a slow DDoS on a site that was merely busy.
- **Trap explosion.** A trap generates infinite unique URLs. Each fetched page adds 50 new URLs to the frontier, so the frontier grows without limit while useful coverage stops. The queue that was 1 TB becomes 10 TB and pushes real URLs down.

**Senior fixes break the loops, not add machines:**
- **Adaptive per-host delay:** timeouts and 429/503 responses INCREASE that host's delay (exponential backoff with jitter, Day 8, Day 71). After N failures, park the host for hours. A `Retry-After` header is honored exactly.
- **Circuit breaker per host:** open the circuit on repeated failures, stop sending, probe occasionally.
- **Hard budgets per host and domain:** a trap can only burn its own budget, never the whole frontier (bulkheads, Day 34 cell-based thinking).
- **Bounded frontier with priority eviction:** when full, drop the lowest-priority URLs, not the newest. Load shedding (Day 13) applied to a queue.
- **Idempotent fetch jobs:** the job key is the normalized URL, so a restarted worker re-doing a URL writes the same page once (Day 12).

---

## 7. Map to Rare.lab's own stack

| Crawler concept | Rare.lab equivalent | Already have? | Next ceiling |
|---|---|---|---|
| Content fingerprint dedup | Content-addressed immutable scene JSON in Cloudflare R2 | Yes, exactly. Same hash = same object, stored once. | Hash the CANONICAL form of a node graph (sorted keys, stripped editor-only fields like node x/y position), or two visually identical graphs hash differently and dedup fails. This is URL normalization for graphs. |
| URL normalization | Graph canonicalization before compile | Not formalized | Write a `canonicalize(graph)` function and version it. A change to it changes every hash, so put the version in the hash input. |
| Per-host politeness / fairness | Fair scheduling of any future server-side compile or render jobs, keyed per user or per project | Not yet needed | The first time jobs exist, use one queue per tenant with round-robin dispatch (the back-queue idea), or one heavy project starves the rest. Cloudflare Queues plus a Durable Object per tenant is a natural fit. |
| Rate limiting a shared resource | One shared WebGL context in the embeddable runtime | Partly | The GPU is Rare.lab's "one host". Give each effect a per-frame time budget (like crawl-delay) and skip or degrade an effect that overruns, instead of letting one slow shader stall the page. |
| Frontier durability | Supabase Postgres as the system of record | Yes | A Postgres table used as a queue (`SELECT ... FOR UPDATE SKIP LOCKED`) is fine to about a few thousand jobs/sec. Past that, move to a real queue. Keep the connection through the Supabase pooler. |
| Budget per domain (trap defense) | Limits on graph size: max nodes, max loop bounds, max texture reads | Not yet | Enforce at save time and compile time. A pathological or malicious graph is a crawler trap for your compiler. |

**One-line lesson for Rare.lab:** canonicalize before you hash. Content-addressing only dedups if equal things produce equal bytes, so write and version a `canonicalize(graph)` step now, before there are a million scenes in R2 that hash differently for the same picture.

---

## 8. What is inside the video you shared in this task (recap, already covered)

The mock interview is "Design LeetCode". It was taught in full in [Day 74](074-leetcode-online-judge-end-to-end.md), with the timestamped summary, the six-step framework (requirements, non-functional, entities, API, naive HLD, deep dives) and the deep dives on sandboxing, the queue, and the Redis leaderboard. To avoid repeating a covered topic, today's lesson applies the SAME framework (Section 0) to a new system. Use that six-step order as your spine for any product you study.

---

## 9. References and what is actually in them

**Honest note on access:** the network proxy in this environment blocked opening every page (research.google, arxiv.org, Wikipedia, university PDFs, crawlex.net). Everything below was read as SEARCH-RESULT EXCERPTS only. I did not read the full papers. Facts marked "from memory" need checking against the PDF.

- [Mercator: A Scalable, Extensible Web Crawler (Heydon and Najork)](https://courses.cs.washington.edu/courses/cse454/15wi/papers/mercator.pdf) (also [mirror](https://resources.mpi-inf.mpg.de/d5/teaching/ss05/is05/papers/mercator.pdf)). The classic paper. Excerpt confirms: the frontier is a front end of FIFO queues for priority and a back end of FIFO queues for politeness, with a heap between the back queues and workers for timing; each URL goes to a back queue by host name so at most one thread downloads from a given server; real frontiers reach hundreds of millions of URLs so most sit on disk. Read this first.
- [IRLbot: Scaling to 6 Billion Pages and Beyond (Lee, Leonard, Wang, Loguinov)](https://dl.acm.org/doi/10.1145/1367497.1367556). One server, 6.3 billion pages, 41 days, 1,789 pages/sec. Argues that URL-uniqueness checking, BFS ordering and fixed per-host rate limits all break at this size. Introduces DRUM (batched disk key-value lookups) and STAR (per-domain budget from in-degree to fight spam and infinite sites). [PDF copy](https://www.khoury.northeastern.edu/home/vip/teach/IRcourse/3_crawling_snippets/other_notes/paper_IRLbot.pdf). Also see Greg Linden's short take, [Crawling is harder than it looks](https://glinden.blogspot.com/2008/05/crawling-is-harder-than-it-looks.html).
- [BUbiNG: Massive Crawling for the Masses (Boldi, Marino, Santini, Vigna)](https://vigna.di.unimi.it/ftp/papers/BUbiNG.pdf). Open-source Java, fully distributed. Excerpt: over 10,000 pages/sec on a 64-core, 64 GB machine with politeness by host AND by IP, storing more than 160 MB/s; avoids batch MapReduce style in favor of high-speed messaging between agents. Read for how a modern crawler is assembled.
- [Google: How Google interprets the robots.txt specification](https://developers.google.com/crawling/docs/robots-txt/robots-txt-spec) and the standard [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309). Excerpt: 500 KiB size limit, cache up to 24 hours, 5xx treated as unreachable and crawling paused. This is the politeness contract your crawler signs.
- [Common Crawl: size of monthly archives](https://commoncrawl.github.io/cc-crawl-statistics/plots/crawlsize) and [Common Crawl's move to Nutch](https://commoncrawl.org/blog/common-crawl-move-to-nutch). Real scale: billions of pages per month, hundreds of TB, built on Apache Nutch (batch, Hadoop style). Compare with BUbiNG's streaming style. Heritrix (Internet Archive) is the other open crawler and needs its machine count fixed before you start.
- [Stanford CS276 crawling lecture](https://web.stanford.edu/class/cs276/19handouts/lecture18-crawling-1per.pdf) and [Neel Mishra: Design a Web Crawler](https://neelmishra.github.io/blog/hld/real-world/web-crawler.html) (excerpt only): plain-language teaching versions of the front/back queue frontier. Good for a first pass.

**Inference, labeled:** all capacity math in Section 1 (11,600 pages/sec, 1.2 GB/sec, 1 TB frontier, 580,000 checks/sec, 12 GB Bloom filter), the 116-day politeness example, and every Rare.lab mapping are my estimates and reasoning, not published figures. Google's and Bing's real internals are not public.

**Related ledger lessons:** Day 8 (rate limiting), Day 10 (consistent hashing), Day 12 (idempotency), Day 13 (backpressure), Day 21 (LSM), Day 23 (content-addressed storage), Day 28 (Bloom filters), Day 34 (cells and bulkheads), Day 67 (noisy neighbor), Day 74 (online judge).
