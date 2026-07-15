# Day 33 — How do you find the nearest neighbor among a billion vectors in under 2 milliseconds?

*2026-07-15*

---

## 1. The company and the number that breaks a naive design

Meta AI Research built **FAISS** (Facebook AI Similarity Search) to answer one question fast: given a query vector, which of N stored vectors is closest to it. The library's flagship benchmark, described in Johnson, Douze and Jégou's "Billion-Scale Similarity Search with GPUs" (2017), is **Deep1B: 1 billion 96-to-128-dimensional vectors**. That is not a toy number. It is the exact scale at which "just compute the distance to every vector" stops being something you can wave your hands at.

Here is the arithmetic that makes this concrete, worked from first principles (this specific calculation is mine, not lifted from a paper, but the input numbers, 1 billion vectors, 128 dimensions, are the real Deep1B/SIFT1B benchmark FAISS was built against). Store each vector as 128 float32 numbers: 128 × 4 bytes = 512 bytes per vector. For 1 billion vectors that is **512 gigabytes** of raw vector data. A brute-force nearest-neighbor query has to read every one of those bytes at least once, there is no shortcut, you do not know which vector is closest until you have compared against all of them. Typical single-core DRAM bandwidth for this kind of strided read pattern is on the order of 10 to 20 GB/s in practice. 512 GB at that bandwidth is **25 to 50 seconds per query**, and that is before you even count the multiply-add arithmetic for the distance computations themselves. Pinterest, running visual search over a corpus its engineering blog describes as multiple billions of Pins, hits the same wall from the other direction: their real, published constraint is keeping each search shard under **16 GB of RAM per 10 million images**, specifically because a shard that does not fit in RAM, or that has to scan too much of what does fit, cannot answer in real time.

That is the breaking number: not "the query is slow," but **the size of the data itself, multiplied by the fact that exact search has no algorithmic shortcut in high dimensions**, guarantees the naive approach costs tens of seconds per query at a scale a demo will never accidentally reach. Every mechanism in this lesson exists because "compare against everything" cannot be optimized away, it has to be replaced with something that promises a good enough answer instead of the guaranteed-correct one.

---

## 2. Why the naive design dies

The naive design: store every vector in a table, and for each query, compute the distance (cosine or L2) against every row in application code, sort, take the top k. It fails in three concrete, compounding ways.

**a. It is memory-bandwidth bound, not just slow.** Even with all 512 GB resident in RAM (itself an unrealistic ask on a single machine), the CPU has to stream every byte of it through cache on every single query. This is like re-reading an entire library's card catalog, cover to cover, every time someone asks "do you have a book about whales." A demo never notices this because a demo's catalog fits in L2 cache.

**b. Cost grows linearly with corpus size, forever, with no ceiling.** Double the vectors, double every future query's cost. Unlike a B-tree index on a SQL column, which turns an O(N) scan into an O(log N) lookup, there is no exact data structure in high-dimensional space that gives you the same free lunch. This is the "curse of dimensionality": space-partitioning tricks that work beautifully in 2 or 3 dimensions (k-d trees, quadtrees) degrade back toward brute-force scanning once dimensionality climbs into the hundreds, because in high dimensions almost every point ends up roughly equidistant from almost every other point, so partitioning stops usefully narrowing the search.

**c. You cannot "add an index" the way you would for a slow SQL query, because exact nearest neighbor has no fast exact algorithm at this dimensionality.** The fix for a slow `WHERE` clause is usually a B-tree or hash index and the query is still exact. Here, the only way to get sub-millisecond answers is to **stop guaranteeing the exact answer** and instead guarantee a high-probability approximate one. That is a different kind of engineering decision than most systems have to make, and it is the pivot this whole lesson turns on.

---

## 3. The architecture, top to bottom

```
Client (search box, "similar images", recommendation request)
   |
   v
Embedding service — turns the raw query (text, image, click history) into a vector
   |  a trained model does this, not the database; the vector IS the search key
   v
Load balancer, stateless query-serving tier
   |  routes the query vector to the ANN service, no session state to pin to a node
   v
ANN index shards — each shard sized to fit in RAM
   |  Pinterest's real number: keep each shard under 16 GB for 10M images,
   |  by holding compact codes, not full-precision floats, per vector
   |  like splitting one giant library across many small branch libraries,
   |  each one small enough that its whole card catalog fits on one desk
   v
Per-shard coarse search: IVF cell probe OR HNSW graph walk
   |  IVF: vector space is pre-clustered into cells (Voronoi regions); only
   |  probe the nprobe nearest cells to the query instead of every vector,
   |  like checking only the few nearest aisles in a warehouse instead of
   |  walking every aisle
   |  HNSW: a multi-layer graph where top layers have long-range "highway"
   |  edges and bottom layers have short local edges; search greedily hops
   |  down from highway to side street to driveway
   v
Candidate merge across shards
   |  each shard returns its local top-k; a coordinator merges the lists
   |  into one global top-k, same fan-out/fan-in shape as Day 18's search
   v
Exact rerank on the shortlist
   |  the shortlist (hundreds, not billions) gets scored with full-precision
   |  vectors or a more expensive metric, buying back accuracy lost to
   |  compression, cheaply, because the list is now small
   v
Cache layer for popular queries / query embeddings
   |  Day 19's stampede-safe caching applies directly: a popular query's
   |  result list is a cache-aside read, computed once, served many times
   v
Response: top-k results
```

Offline, alongside the above, sits a **batch index-building pipeline**: periodically (Pinterest's blog describes this as Hadoop-driven) retrain the coarse clustering and the compression codebooks against the current data distribution, build a fresh read-only index, and swap it into serving. The live serving path never writes into the ANN structure directly; new items wait for the next build cycle to become searchable. That staleness window is a deliberate trade, covered in section 5.

---

## 4. The transferable mechanisms

**a. Approximate nearest neighbor (ANN) search.** Replace "guaranteed exact top-k" with "top-k correct with high probability, measured as recall@k against ground truth." This is the foundational move: once you accept approximate answers, every other mechanism below becomes available. FAISS's own documentation frames its indexes explicitly as trading precision for speed.

**b. Inverted File index (IVF) / coarse quantization.** Cluster the vector space offline into cells (k-means centroids), assign every stored vector to its nearest cell, and at query time only scan the `nprobe` cells closest to the query vector instead of the whole corpus. This turns an O(N) scan into "look up the right aisle, then scan only that aisle," the same inverted-index instinct from Day 18's text search applied to continuous vector space instead of discrete tokens.

**c. Product Quantization (PQ), from Jégou, Douze and Schmid's 2011 paper.** Split each vector into subvectors, learn a small codebook per subvector, and represent the original vector as a short sequence of codebook indices instead of raw floats. A 512-byte float vector can become an 8-to-64-byte code, an order of magnitude smaller, which is exactly how billions of vectors fit in RAM at all. Pinterest's own binarization module pushes the same idea further, compressing to bits instead of bytes, because at multi-billion scale even a few extra bytes per vector multiplies into terabytes.

**d. Graph-based search (HNSW), from Malkov and Yashunin's 2016 paper.** Build a navigable small-world graph with multiple layers: sparse long-range edges on top, dense short-range edges at the bottom. A query starts at the top layer and greedily walks toward the target, descending layers as it gets close, similar to how you would navigate a highway to the nearest exit before switching to local roads for the last mile. This gives search cost that scales roughly logarithmically with corpus size instead of linearly.

**e. The retrieve-cheap, rerank-expensive funnel.** Do the approximate, compressed-vector pass across the full corpus (or all shards in parallel) to get a short candidate list, then spend the expensive, full-precision computation only on that short list. This is the identical two-stage shape Day 18 used for text search (cheap candidate generation, expensive ranking), applied here to compressed vs. full-precision distance instead of BM25 vs. a learned ranker.

**f. Offline build, online serve, with a staleness window instead of a live write path.** Train the coarse quantizer and codebooks in a periodic batch job against the current data distribution, build a complete new index, and swap it in wholesale rather than mutating a live index vector-by-vector. This is the same "build once, serve many, accept a delay" pattern Rare.lab already leans on for its content-addressed manifest.

---

## 5. The trade-offs

**CAP made concrete, per data type.** The thing being sacrificed here is not "correctness of an existing answer," it is **freshness of the searchable set**. A newly inserted vector is not necessarily findable the instant it is written; it waits for the next index build cycle. That is a deliberate choice for availability and query latency over immediate consistency of what is searchable, made explicit rather than accidental: the system decides in advance how stale "just added" is allowed to be, instead of discovering it under load.

**Recall vs. latency vs. memory, three dials on one knob set.** Raising `nprobe` (IVF) or `ef` (HNSW's search-time parameter) improves recall by examining more candidates, at the direct cost of latency. Compressing vectors with PQ or binarization shrinks memory enough to fit a shard in RAM, at the direct cost of distance-computation accuracy, which the exact-rerank stage exists specifically to buy back on the shortlist. None of these three dials can be maximized simultaneously; every real ANN deployment is a chosen point on this triangle, not a solved optimum.

**Cost vs. latency: GPU acceleration is a real option, not a default.** FAISS's GPU implementations exist because GPU parallelism helps distance computation at scale, but Pinterest's production shards are sized around a plain RAM budget per CPU node, not GPU throughput. The lesson is not "always use GPUs for vector search," it is "measure your actual QPS and corpus size before paying for that infrastructure"; a few million vectors on CPU with IVF or HNSW is often the entire answer, and GPU or exotic quantization only earns its cost once the corpus and query volume genuinely demand it.

---

## 6. The systems-thinking lens

**The feedback loop here is cell imbalance, the vector-space cousin of Day 16's hot-key/celebrity problem.** Trace it: the coarse quantizer's clusters are trained once, offline, against the data distribution at training time → real-world data is never uniform, some regions of vector space (one popular visual style, one common query intent) are naturally denser than others → the cells covering those dense regions end up holding a disproportionate share of both stored vectors and, because popular items get queried more, incoming query traffic → those specific shards run hotter than the rest of the fleet, their latency degrades under load even though every other shard looks perfectly healthy → because candidate merge waits on every shard's response before returning a result, the single slow shard sets the latency for every query system-wide, not just queries that "belong" to that cell.

**The senior fix is not "give the hot shard more replicas and call it done."** That treats a distribution problem as a capacity problem, and the imbalance keeps recurring as the underlying data keeps drifting, requiring a manual rebalance every time. The fix that actually breaks the loop is to (1) retrain the coarse quantizer periodically against the *current* data distribution, not a frozen snapshot from launch day, so cell boundaries track where the data actually lives, and (2) route and replicate based on measured per-cell load rather than a fixed one-shard-per-cell assumption, so a cell's popularity, not a static config, determines how many copies of it exist. The failure mode is not "one shard is slow," it is "the index's structure and the data's real shape have quietly diverged," and the fix targets that divergence directly instead of papering over its symptom with more hardware.

---

## Map to Rare.lab's stack

Rare.lab's shader and node-preset library today is small enough, thousands to perhaps low hundreds of thousands of presets, that this entire lesson is not yet a problem. A "find shaders similar to this one" feature can run on exact search, or on Postgres's `pgvector` extension directly inside the existing Supabase database, computing distance against every stored embedding with a basic index. That is the honest, correct-sized answer today: do not build an IVF/HNSW/PQ pipeline for a corpus that fits comfortably in one machine's RAM.

The ceiling is real and predictable, though. Once the preset and shader corpus crosses roughly a million-plus embeddings, a single Postgres node doing exact or lightly-indexed vector search runs into the same wall this lesson describes: too many bytes to scan per query for the latency budget a live editor needs. At that point, the transferable pattern is not "use FAISS specifically," it is the shape underneath it: shard embeddings across nodes sized to a measured RAM budget the way Pinterest sizes shards to 16 GB per 10 million images, and separate the expensive index-build step (retraining clusters or codebooks) from the cheap serving path, mirroring the build-once, serve-many split Rare.lab already uses for its content-addressed R2 manifest. The mechanism to reach for first, before any of the heavier machinery here, is simply `pgvector`'s own IVFFlat or HNSW index types, since Rare.lab is already on Postgres; only graduate past that into dedicated ANN infrastructure once measured shard size or query latency actually crosses the line this lesson draws.

---

## References and summaries

**Johnson, Douze, Jégou: "Billion-Scale Similarity Search with GPUs"** (arXiv:1702.08734, IEEE Transactions on Big Data, 2019)
https://arxiv.org/abs/1702.08734
The primary source for FAISS's design and its headline benchmark: Deep1B, 1 billion 96-to-128-dimensional vectors, used to validate GPU-accelerated IVF and product-quantization based indexes at a scale where brute-force search is not viable. This lesson's opening arithmetic (512 GB of raw vector data, tens of seconds per brute-force query) is derived from this paper's stated corpus size, not quoted from it directly.

**Malkov, Yashunin: "Efficient and Robust Approximate Nearest Neighbor Search Using Hierarchical Navigable Small World Graphs"** (arXiv:1603.09320, IEEE TPAMI 2018)
https://arxiv.org/abs/1603.09320
The source for the HNSW mechanism described in sections 3 and 4: a multi-layer proximity graph with long-range edges at coarse layers and short-range edges at fine layers, searched with a greedy descent that gives roughly logarithmic-scaling query cost instead of linear.

**Jégou, Douze, Schmid: "Product Quantization for Nearest Neighbor Search"** (IEEE TPAMI, 2011)
https://inria.hal.science/inria-00514462v1/document
The source for the Product Quantization mechanism in section 4: splitting a vector into subvectors, learning a small codebook per subvector, and representing the original as a short sequence of codebook indices, the technique that lets billions of vectors' compressed codes fit inside a practical RAM budget.

**Pinterest Engineering: "Unifying Visual Embeddings for Visual Search at Pinterest"** (Medium, Pinterest Engineering Blog)
https://medium.com/pinterest-engineering/unifying-visual-embeddings-for-visual-search-at-pinterest-74ea7ea103f0
The source for this lesson's real production numbers: a multi-billion-Pin visual search corpus, ANN shards kept under roughly 16 GB of RAM per 10 million images, periodic Hadoop-driven construction of coarse token indices and in-memory feature trees, and a binarization module used to shrink embedding storage and serving cost, the direct real-world counterpart to this lesson's PQ and sharding mechanisms.
