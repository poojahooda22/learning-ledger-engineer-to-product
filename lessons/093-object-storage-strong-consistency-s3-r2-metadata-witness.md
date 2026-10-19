# Day 93: You PUT a file and instantly GET it from another machine. Why does it always come back fresh, when the data lives on thousands of disks and caches? How object storage (Amazon S3, Cloudflare R2) stays strongly consistent at trillions of objects

**Date:** 2026-10-19
**Difficulty:** Advanced (cache coherence plus the metadata layer, the part nobody draws in a diagram)
**Topic:** Earlier lessons covered the pieces: erasure coding for durable bytes (Day 32), content addressing (Day 23), distributed file systems (Day 41), caching and invalidation (Day 19), session guarantees (Day 31), Raft (Day 11). Today is the layer in the MIDDLE: the metadata index that answers "which version of key X is current, and where are its bytes?" and the trick that makes every read fresh without slowing it down.
**Stack relevance:** Rare.lab stores immutable scene JSON plus a manifest on Cloudflare R2. R2 is exactly this system. Section 9 shows what R2 guarantees you for free and where you still must protect yourself.
**A note on the pasted transcript:** the LeetCode mock interview is already taught in [Day 74](074-leetcode-online-judge-end-to-end.md), and its sandbox and leaderboard slices in Day 50 and Day 54. I did not repeat it. I reused its recipe (requirements, entities, API, naive design, deep dives) in Section 0.

---

## 0. The interview recipe applied to object storage

1. **Functional.** PUT an object (key plus bytes). GET it. DELETE it. LIST keys by prefix ("everything under `scenes/abc/`"). Out of scope: billing, IAM policy language, lifecycle rules, encryption key management.
2. **Non-functional.** Durability (S3 advertises 11 nines, a marketing figure for annual object loss probability). Availability (S3 Standard advertises 99.99%). Scale (S3 has passed hundreds of trillions of objects, a figure shown in AWS conference material, treat it as approximate). **Consistency: after a successful PUT, every later GET and LIST sees it.**
3. **Entities.** Bucket, Object key, Object version, Metadata record (key, size, checksum, version id, location of bytes), Data blob (the bytes), Placement (which disks hold which shards).
4. **API.** `PUT /bucket/key`, `GET /bucket/key`, `DELETE /bucket/key`, `GET /bucket?prefix=scenes/&max-keys=1000`. Note LIST is paginated by design (Day 74 made the same point about problem lists).
5. **Naive design.** One web server, one disk, one Postgres table `objects(key, bytes)`.
6. **Deep dives.** The break numbers, then the split of data from metadata, then the consistency trick.

---

## 1. Named companies and the breaking number

**Amazon S3** and **Cloudflare R2**.

- S3 serves hundreds of trillions of objects (conference slide figure, approximate) and tens of millions of requests per second at peak (a figure AWS has stated in public talks, I could not re-verify it today).
- Until December 2020, S3 was only **eventually consistent for overwrites and deletes** (and for LIST). You could PUT a new version, GET, and receive the OLD bytes for a short while. Whole ecosystems (Netflix's S3mper, Hadoop's S3Guard) existed only to paper over this.
- On 1 December 2020, AWS announced **strong read-after-write consistency for all GET, PUT, LIST, and metadata operations, at no extra cost and with no performance penalty.** Confirmed on the official AWS consistency page via search today.

**The breaking number for the naive design: one key, read by many machines that each remember an answer.**
- A hot object (say, a manifest read 50,000 times per second) is answered from CACHES. There are many caches. A write updates the source of truth, but the other caches still hold the old answer.
- How many places can hold a stale answer? In a large fleet, **hundreds or thousands of front-end and metadata nodes**. Telling all of them on every write is impossible to do synchronously at this scale. Not telling them means stale reads.

So the real question is not "how do we store trillions of objects". It is: **how does a cache know, at read time, whether it is stale, without asking the database every time?**

---

## 2. Why the naive design dies

One server, one disk, one Postgres table.

**Failure 1: Size.**
- 200 trillion objects at even 100 bytes of metadata each is 20 petabytes of METADATA alone. No single table or server holds that. Payload bytes are far larger still (exabytes).
- Analogy: one filing cabinet for every document in every library on Earth.

**Failure 2: Throughput on the index.**
- Every GET needs "find this key". Every PUT needs "update this key". Millions of requests per second hit the index. One Postgres primary manages low tens of thousands of simple writes per second (inference, class-of-problem estimate, Day 78 and Day 64 hit the same wall).
- Analogy: one librarian at the catalog desk.

**Failure 3: Hot keys and range scans fight each other.**
- Keys are strings and LIST wants them in sorted order, so you cannot just hash-spread them randomly (hash spreading destroys the ordering that `prefix=` needs). But sorted order piles popular prefixes (`2026-10-19/...` timestamps, all new keys at the end of the range) onto ONE partition. This is the hot partition of Day 16 and Day 80 again.

**Failure 4: One big cache gives stale reads, many caches give worse stale reads.**
- Add a cache in front for speed, and the cache and the database now disagree after every write. Add 1,000 cache nodes for scale and the disagreement is spread across 1,000 places.

**Failure 5: Bytes and index in one box means one failure kills both.**
- A disk dies, you lose the data AND the knowledge of what was on it.

The industry answer is three splits: **bytes from metadata, metadata by key range, and "is my cache fresh?" from "what is the value?"**.

---

## 3. The architecture, top to bottom

```
 Clients (browsers, Workers, servers, SDKs)
        |  HTTPS, signed requests
        v
 [1] DNS / EDGE / LOAD BALANCER      spread requests, terminate TLS
        |
        v
 [2] FRONT-END / GATEWAY FLEET       stateless, scale OUT. authenticate, authorize,
        |                            rate limit, parse the request, route it
        +----------------------------------------+
        v                                        v
 [3] METADATA SUBSYSTEM                     [4] DATA / STORAGE LAYER
     key -> (version, size, checksum,           bytes split into shards,
             pointer to the bytes)              erasure coded (Day 32),
     partitioned by key range                   spread over many disks and
     replicated, has a CACHE                    availability zones
        |
        +--> [3a] PERSISTENCE TIER   source of truth for metadata, replicated, durable
        +--> [3b] CACHE              hot metadata in memory, fast, might be stale
        +--> [3c] WITNESS            tiny, fast: "has this key changed since X?"
        |
        v
 [5] BACKGROUND SERVICES             repair lost shards, rebalance partitions,
                                     garbage collect orphan blobs, scrub checksums
```

Each box has one job and an analogy:

- **Front-end fleet.** Job: be the stateless door. Analogy: the reception desks of a giant hotel. Any desk can serve any guest, so you add desks when the lobby fills.
- **Metadata subsystem.** Job: answer "what is the current version of this key and where are its bytes?" Analogy: the hotel's guest register. It is tiny compared to the hotel's contents, but if it is wrong, nobody finds their room.
- **Persistence tier.** Job: be the one truth for metadata. Analogy: the register's master copy locked in the manager's office.
- **Cache.** Job: avoid trips to the manager's office. Analogy: the sticky note at each reception desk with the room number of the most common guests. Fast, usually right, sometimes out of date.
- **Witness.** Job: be a very cheap "was this page of the register changed since you wrote your sticky note?" check. Analogy: a bell that rings at every desk's wall panel each time the master register changes a line, except it is a lookup instead of a broadcast (explained below).
- **Storage layer.** Job: hold bytes cheaply and durably. Analogy: the warehouse behind the hotel, where the guest's luggage is cut into pieces and stored in several rooms so a flood in one room loses nothing.
- **Background services.** Job: janitorial work (repair, rebalance, cleanup). Analogy: the night crew.

**Key design rule: the bytes never go through the metadata subsystem.** A PUT writes the shards to the storage layer first, THEN commits one small metadata record that says "key K now points to these shards". The metadata commit is the moment the object "exists". Until then, the shards are orphans that garbage collection will clean up. This ordering gives you atomic publish: readers see the old object or the new object, never half of one.

---

## 4. One overwrite, walked end to end

Key: `scenes/9f2c/manifest.json`. Currently version 7. Two clients:

1. **Client A** (a Cloudflare Worker) sends `PUT manifest.json` with new bytes (version 8 will result).
2. The front-end writes the bytes to the storage layer as shards. Say 5 data shards plus 3 parity shards, across 3 availability zones. It waits until enough shards are durable.
3. The front-end sends the metadata update to the partition that owns this key's range: "manifest.json is now version 8, checksum X, shards at these locations."
4. The metadata persistence tier commits the update. It is replicated, and the commit is acknowledged once enough replicas have it. **This is the linearization point.** The write is now durable and official.
5. The cache entries on other nodes still say "version 7". They have not been told.
6. A second later, **Client B** (another region's front-end) sends `GET manifest.json`. That front-end asks its local metadata node.
7. The metadata node has version 7 in its cache. Before answering, it asks the **witness**: "has manifest.json changed since the version I hold?" The witness says "yes". The node **drops its cached entry and reads version 8 from the persistence tier**, then returns the new bytes.
8. For the next 50,000 reads, the witness says "no change", so the cache answers in memory.

What mattered: step 7 turns a possibly stale cache into a **validated** cache. The cost is one cheap lookup on the read path, not a full database read.

A DELETE is the same flow with a tombstone record. A LIST is a range scan over the metadata keys, and it must also be validated, which is why LIST consistency came after GET consistency and was a harder job for the S3 team.

---

## 5. Under the hood: how the freshness check works

**What is public (from Werner Vogels' post, "Diving Deep on S3 Consistency", April 2021, seen via search excerpts today because the site was not directly fetchable):**
- S3's metadata subsystem has a cache. Previously a write updated the persistence tier and cached copies elsewhere could lag, hence stale reads.
- The fix added a component that acts as a **witness**, notified every time an object changes. It is used as a **read barrier**: if the witness says the cached value is not stale, serve it. If stale, invalidate and read from the persistence tier.
- The replication logic that tracks the **per-object order of operations** is described as the core of the cache coherency protocol.
- The witness must not reduce availability. S3 states the goal of strong consistency with no tradeoff on availability or performance.

**What is from a conference talk (re:Invent 2025, STG407 by Seth Markle and James Bornholt, known to me only through a third-party transcript summary found in search):** consistency was achieved through a **replicated journal** (an ordered log of metadata changes) and a **witness system**, using quorum ideas. Treat the exact topology (how many journal nodes, how they chain) as unverified. I found no primary source for it.

**What is a patent, and so only a hint:** Amazon's patent filings with titles like "Witness service for ensuring data consistency in a distributed storage system" describe a service that holds the most recent instance (version or sequence marker) of each data item, so readers can compare. That is consistent with the public story, and I would call it supporting evidence, not documentation.

**Labeled inference: a plain-words model you can design from.**

- Every metadata change gets an ordered position in a journal for its partition (a **sequence number**, the same idea as Day 92's sequencer and Day 17's log sequence number).
- The witness tracks, for each key (or each small group of keys), the **latest sequence number** that has been committed.
- A cache entry carries the sequence number it was read at.
- On a read: `if cache.seq == witness.latest(key) then serve cache else refetch`. That is one small comparison instead of a full read.
- Why this is correct: the witness is updated as part of the commit, or before the write is acknowledged to the client. So once a client has been told "your PUT succeeded", the witness already says "newer than version 7". Any later read must pass the barrier and cannot return the older value. This is the same logic as a **fencing token** (Day 24): a version number that lets the reader reject a stale answer.

**Why this beats the alternatives:**

| Approach | Problem |
|---|---|
| Broadcast invalidations to all caches on every write | Needs N messages per write, and a lost message means a silently stale cache forever. |
| Short TTL (expire after 1 second) | Still stale for up to 1 second. That is the old eventual consistency. |
| No cache, always read the persistence tier | Correct, but the persistence tier melts at millions of reads per second. |
| **Cache plus witness validation** | Reads stay mostly in memory. Staleness is detected, not hoped away. |

**The LIST problem.** A listing is a range of keys. A new key appearing in the middle must show up in a listing that starts afterwards. So the witness idea must cover ranges or the partition's journal position, not just single keys. A simple model: each key-range partition has one monotonic sequence number, and a listing is valid if it was assembled at a position at least as new as the latest committed write you were told about. This is inference.

**The data side, briefly.**
- Bytes are erasure coded (Day 32): split into k data shards plus m parity shards, so any k of k+m rebuild the object. Far cheaper than 3 full copies (about 1.4x to 1.6x overhead instead of 3x, depending on k and m, which are not public for S3).
- Amazon's **ShardStore** is the newer storage node software, written in Rust, using an LSM-tree-like structure (Day 21) for shards. The SOSP 2021 paper (Bornholt, Joshi et al., Best Paper) says the team used **lightweight formal methods**: executable reference models and property-based tests that caught 16 subtle bugs (crash consistency, concurrency) before production. Roughly 40 to 45 thousand lines of Rust per the slides (figures differ by deck). Confirmed via search today.

**How R2 does the same job (primary source: Cloudflare docs "How R2 works" and the R2 consistency page, via search excerpts):**
- **R2 Gateway** (runs on Cloudflare Workers): receives S3-compatible API requests, handles auth and routing.
- **Metadata layer built on Durable Objects**: stores object key, checksum and so on, "to ensure strong consistency", with its own cache for lookups.
- **Tiered read cache** in front of a **distributed storage layer** that holds encrypted object data.
- Consistency promise: read after write, metadata updates, deletes and listings are strongly consistent. **IAM permission changes are eventually consistent (up to about a minute).** **Concurrent writes to the same key: last writer to complete wins.**
- Why Durable Objects fit: a Durable Object is a single-threaded actor that owns a slice of state (Day 92's single-writer principle). One owner per key range means no cross-node conflict on that slice. Whether R2 partitions exactly this way is not stated in the sources I could read, so that is inference.

**Scale story at three tiers.**

- **1,000 objects.** Postgres table plus files on disk (or `bytea` for tiny things). Correct and simple. Do not build a metadata subsystem.
- **100,000 objects.** Still one Postgres. Move bytes OUT of the database into object storage, keep only `(key, hash, size, url)` in the table. What breaks next: backups and replication lag because of fat rows. This is the split of data from metadata, and it is the single most valuable move in this lesson.
- **10 million to 10 billion objects.** Metadata alone exceeds one machine, so partition by key range (to keep LIST ordered) with automatic splitting of hot ranges (Day 77 live resharding, Day 45 secondary indexes). Add a cache, which immediately creates the staleness problem, so add validation (witness or version check). What breaks next at trillions: partition count itself, rebalancing churn, repair traffic from disk failures (with millions of disks, several fail every day, so repair is continuous background load, not an incident), and request-rate hot spots on a single prefix, which S3 handles by splitting partitions automatically and documents a per-prefix guideline of thousands of requests per second (S3's published figure is 3,500 PUT/POST/DELETE and 5,500 GET/HEAD per second per prefix, from AWS docs as I remember them, not re-verified today).

---

## 6. The transferable mechanisms

1. **Separate the index from the payload.** A small, strongly consistent metadata store pointing at a huge, cheap, dumb blob store. Used by every file system, Git (objects vs refs), and your own R2-plus-Supabase setup.
2. **Commit by publishing a pointer last.** Write the immutable bytes first, then atomically flip one small record. Readers see old or new, never half. (Same as Git moving a branch ref, and your manifest.)
3. **Validate caches instead of invalidating them.** Do not try to push "you are stale" to N caches. Make every read carry a cheap freshness check against a version, sequence number or witness. Staleness becomes impossible to miss rather than something you chase.
4. **Per-object ordering (a version or sequence number).** One monotonic number per object or partition is the thing the check compares. Also the basis for compare-and-swap writes (Day 92 map).
5. **Single owner per slice of state.** Whether it is a metadata partition, a Durable Object, or a matching engine's order book, one writer avoids distributed locking.
6. **Erasure code the bytes, replicate the metadata.** Big data uses parity math for cheap durability. Small, hot, ordered data uses replication and quorums for low latency. Choose per data type.
7. **Continuous background repair.** At scale failure is the steady state, so repair, scrubbing and garbage collection are first-class services with their own budgets, not scripts.

---

## 7. The trade-offs (CAP made concrete, per data type)

| Data | Choice | Why |
|---|---|---|
| Object metadata (which version is current) | **Consistency**, with a heavy investment to also keep availability. | After "PUT succeeded", a read returning the old version breaks applications (a build reads the old artifact, a pipeline skips a new file). S3 spent years and a new coherence protocol to remove the old trade. |
| Object bytes | **Durability and availability**, immutable once written, so no consistency problem exists. | An immutable blob cannot be stale. Only the POINTER can. |
| Listings | **Consistent now (since 2020)**, but costlier to produce than a point read. | A missing key in a listing makes sync tools delete or re-upload data. |
| Permissions / IAM changes | **Eventual** (R2 documents up to about 1 minute). | A revoked key staying alive for a minute is acceptable. Re-checking a global policy store on every request is not. This is the Zanzibar tension of Day 35. |
| Concurrent writers to the same key | **Last writer wins**, no merge. | Simple and fast. If you need a merge or a conflict error, you need conditional writes (S3 and R2 now offer `If-Match` or `If-None-Match` style preconditions in some forms, check current docs before relying on it). |
| Cross-region copies | **Eventual** (replication is asynchronous). | Speed of light. 250 ms round trips make synchronous multi-region writes expensive (Day 14 and Day 27). |

**Cost vs latency.** Strong consistency used to be a cost question. The witness design is cheap because it moves work from the write path (nothing extra) to a very small check on the read path. Erasure coding trades CPU and repair bandwidth for storage cost: you spend compute to save disks.

---

## 8. The systems-thinking lens

**The feedback loop here is repair storms and cache stampedes (a thundering herd).**

1. A rack or a metadata node fails. Its cache is gone, and its partition fails over.
2. Every request that used to hit the cache now misses and goes to the persistence tier. Load on that tier spikes (cold cache, Day 84 cold starts).
3. The persistence tier slows. Callers time out and retry, which adds more load.
4. Meanwhile, lost shards trigger **repair traffic** to rebuild data, which competes for the same disks and network as customer reads. Slower reads mean more retries.
5. The slow system looks even less healthy, so more nodes get marked failed, which triggers more repair. A metastable state.

**The senior fix breaks the loop, it does not add disks:**
- **Throttle and prioritize repair** so it yields to customer traffic (a rate limit with a lower priority class, Day 13).
- **Request coalescing** on cache misses: when 10,000 clients ask for the same missing key, only ONE goes to the persistence tier and the rest wait for that answer (the Discord lesson, Day 80).
- **Staggered cache warm-up and jittered expiry** so not everything is cold or expiring together.
- **Retry budgets and exponential backoff with jitter** at the client SDKs, so failure does not multiply load.
- **Cell-based isolation and shuffle sharding** (Day 34) so one partition's failure cannot take the whole region down.
- **Witness degrades safely:** the witness must be fast and highly available, because it sits on every read path. If it were slower than the database it protects, the design would lose.

---

## 9. Map to Rare.lab (Supabase Postgres with RLS, Cloudflare R2 with content-addressed immutable scene JSON plus a manifest, embeddable runtime with one shared WebGL context)

**What you already use, and what you get for free:**
- **R2 gives you strong read-after-write and strongly consistent LIST and delete** (per Cloudflare's consistency page). So after your publish step PUTs a scene or a manifest, any later GET from any region sees it. You do NOT need your own cache coherence for R2 objects. (The R2 consistency guarantee applies at the origin. Cloudflare's CDN cache in front of a public bucket or a custom domain is a SEPARATE cache with its own TTLs, see below.)
- **Content-addressed immutable scene JSON** is the strongest pattern in this lesson: an immutable blob cannot be stale. A URL like `/scenes/sha256-ab12...json` can be cached forever (`Cache-Control: public, max-age=31536000, immutable`). Nobody ever needs to invalidate it. Only the POINTER changes.
- **The manifest is your "metadata pointer"** and your only consistency problem. This is the exact split from Section 6, mechanisms 1 and 2.
- **Supabase Postgres with RLS** is your source of truth for ownership and project rows. Good: small, strongly consistent metadata.

**Where the next ceiling is (ordered by how soon it bites):**

1. **The manifest behind a CDN cache.** If the manifest is served through Cloudflare's CDN with a long TTL, embedders can see an old manifest after a publish, even though R2 itself is strong. That is YOUR cache, and it is the staleness problem of this lesson. Fix, cheapest first:
   - Give the manifest a **short TTL (for example 5 to 30 seconds) with `stale-while-revalidate`**, accepting that an embed may show the previous scene for that window.
   - Or have the embeddable runtime fetch the manifest with a **version check**: a tiny endpoint (a Worker reading Supabase or a Durable Object) returns `{manifestVersion: 42}` with `no-store`, and the runtime refetches the big manifest only when the version changed. That is the witness pattern, implemented in about 30 lines.
   - Do not purge by URL on every publish at scale. Purge APIs are rate limited and slow to propagate, and a purge failure leaves silent staleness.
2. **Lost updates on the manifest.** R2 is last-writer-wins on a key. Two tabs publishing at once will silently drop one. Use a **monotonic version in Postgres** with compare-and-swap (`UPDATE projects SET manifest_hash=$new, version=version+1 WHERE id=$id AND version=$expected`) as the source of truth, and write the manifest object under a version-stamped or hash-stamped key. If you want R2-level protection, use a conditional PUT precondition if the API offers it for your use case (verify against the current R2 docs).
3. **Write order on publish.** Always: (a) PUT the immutable scene JSON, (b) verify it is readable, (c) THEN flip the pointer in Postgres/manifest. Never the other way, or a client can follow a pointer to a blob that is not there yet. Run a janitor that deletes unreferenced hashes after a grace period (the garbage collection service from Section 3, in miniature).
4. **LIST is not a free query.** If the dashboard lists a user's scenes by `LIST prefix`, it is fine at hundreds and slow at tens of thousands. Keep the listing in a Postgres table (with RLS) and treat R2 as blob storage only. That is the "separate index from payload" rule again.
5. **IAM-style delays.** If you rely on revoking access (for example signed URLs for private scenes), remember signed URLs are valid until they expire regardless of revocation. Keep expiries short (minutes) for private assets.
6. **The one shared WebGL context** is a single-writer by nature (Day 92 map). The scale ceiling there is not storage but how many scenes one page can hold; plan eviction for scene resources by hash (an LRU keyed on the content hash makes the cache trivially correct, because a hash never goes stale).

**Bottom line:** your content-addressed design already dodges most of the hard problem. The only mutable thing is the pointer. Treat the pointer like S3 treats metadata: version it, validate it cheaply, and keep it small.

---

## 10. Reference summaries (what is inside each, in plain words)

Fetch note: this sandbox could not open aws.amazon.com, allthingsdistributed.com, developers.cloudflare.com or dev.to directly (DNS errors). Everything below comes from search-result excerpts. I label what I did and did not see.

- **[Diving Deep on S3 Consistency, Werner Vogels, All Things Distributed (April 2021)](https://www.allthingsdistributed.com/2021/04/s3-strong-consistency.html).** The primary source. Plain-words summary from excerpts: S3's metadata subsystem uses a cache. A new component acts as a witness to writes and is told whenever an object changes. A read asks the witness whether its cached copy is stale: if not, serve from cache, if so, invalidate and read from the persistence tier. The per-object operation order tracked by the replication logic is the heart of the cache coherency protocol. The goal was strong consistency without hurting availability. I did not read the full post, so details like node counts are not claimed here.
- **[Amazon S3 Strong Consistency, AWS page](https://aws.amazon.com/s3/consistency/).** The product promise: after a successful write of a new object or an overwrite or delete, any later read returns the latest version, and listings are consistent too, at no extra cost and with no performance trade. Gives the guarantee, not the design.
- **[Deep dive on Amazon S3, re:Invent 2025 STG407 (summary on DEV Community)](https://dev.to/kazuya_dev/aws-reinvent-2025-deep-dive-on-amazon-s3-stg407-37ne).** A third-party, auto-generated writeup of a talk by Seth Markle and James Bornholt. Per the search excerpt: it explains how S3 got read-after-write consistency in 2020 using a replicated journal and a witness system built on quorums, and talks about designing for availability. Unverified details, use it as a pointer to the real video.
- **[Using Lightweight Formal Methods to Validate a Key-Value Storage Node in Amazon S3 (SOSP 2021)](https://www.cs.utexas.edu/%7Ebornholt/papers/shardstore-sosp21.pdf).** The ShardStore paper, a Best Paper. In plain words: S3's new disk-level storage software (Rust) is checked by writing small executable models of what it SHOULD do and testing the real code against them, including crash and concurrency cases. It prevented 16 subtle production bugs (crash consistency, concurrency) and non-experts could extend the checks. Read it to learn how a storage team keeps correctness while shipping features.
- **[How R2 works, Cloudflare docs](https://developers.cloudflare.com/r2/how-r2-works/).** Describes the R2 Gateway (a Worker doing auth and routing), a metadata layer on Durable Objects with its own cache, a tiered read cache, and a distributed storage layer holding encrypted data. It is the best public explanation of what is under YOUR bucket.
- **[R2 data consistency, Cloudflare docs](https://developers.cloudflare.com/r2/reference/consistency/).** Says reads after writes, metadata updates, deletes and listings are strongly consistent, IAM changes take up to about a minute, and the last concurrent writer wins. This page is the contract your manifest design should be built against.
- **[Witness service for ensuring data consistency in a distributed storage system (USPTO publication 11741078)](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/11741078)** and the related filings for [partitioning](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/12386541) and [monotonic reads](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/12032562). Amazon patent applications, seen only as titles in search. They suggest a witness service that records the most recent instance of each item. Patents are not documentation of what is deployed, so treat them as hints.
- **Secondary blogs seen in search (not authoritative):** [Medium: How Amazon S3 Pulls Off Strong Consistency and 99.99% Availability](https://medium.com/@sudha.malpeddi/how-amazon-s3-pulls-off-strong-consistency-and-99-99-availability-c6e19b931baf), [System Design Newsletter on S3 strong consistency](https://newsletter.systemdesign.one/p/s3-strong-consistency), and [hidekazu-konishi.com's architecture writeup](https://hidekazu-konishi.com/entry/how_amazon_s3_achieves_consistency_and_durability.html). Good for vocabulary and diagrams, repeat the same story, and some claims (witness fallback to the metadata store when unreachable) are not confirmed by AWS.
- **Search for next:** the full Vogels post and the re:Invent STG407 video; Warfield's "Building and operating a pretty big storage system called S3" on All Things Distributed (a good tour of scale, hot spots and repair); and Cloudflare's engineering blog posts on Durable Objects storage.

**Inference, labeled:** the 20 PB metadata estimate, the Postgres throughput ceiling, the sequence-number-based model of the witness and LIST validation, the claim that R2 partitions metadata by key range onto Durable Objects, and the Rare.lab CDN/version-check design are my reasoning, not published internals. Per-prefix request rates and the tens-of-millions requests-per-second figure are from memory of AWS material and were not re-verified today.

**Related ledger lessons:** Day 11 (Raft), Day 16 (hot keys), Day 17 (WAL and sequence numbers), Day 19 (caching and invalidation), Day 21 (LSM trees), Day 23 (content addressing), Day 24 (fencing tokens), Day 31 (session guarantees), Day 32 (erasure coding), Day 34 (cells and shuffle sharding), Day 41 (distributed file systems), Day 74 (LeetCode end to end), Day 77 (live resharding), Day 80 (request coalescing), Day 92 (single writer and sequencer).
