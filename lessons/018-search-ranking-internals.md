# Day 18 — How does search rank 500 million products in under 200 milliseconds?

**Date:** 2026-06-29
**Difficulty:** Advanced
**Topic:** Search Ranking Internals — inverted indexes, BM25, two-stage retrieval, learning-to-rank
**Stack relevance:** Any product catalog search, Rare.lab node/shader search, Cloudflare R2 content addressing

---

## 1. The company and the breaking number

**Amazon Search, 2024. 500 million product listings. 30 billion queries per month. A naive SQL scan takes 45 seconds per query.**

Type "iphone 13 case" into Amazon. In under 100 milliseconds Amazon scores every remotely relevant listing from 500 million products, re-ranks them for your personal purchase history, boosts sponsored items, applies inventory filters, and returns 48 results. At peak (Prime Day 2023), Amazon Search handles roughly 1.2 million queries per minute.

The naive design: `SELECT * FROM products WHERE title LIKE '%iphone 13 case%' ORDER BY sales DESC`. Run that on 500 million rows even with all 256 GB of RAM loaded: it is a linear scan, 5 to 45 seconds per query. That is the breaking number. One SQL LIKE query at that scale costs more time than a user will wait.

The entire architecture of search exists to pre-answer this question at write time so that query time is just a lookup.

---

## 2. Why the naive (demo) design dies

The one-table, one-query demo looks like this:

```
User types query
App runs: SELECT * FROM products WHERE title LIKE '%query%'
Database scans every row
App sorts results by sales_count
App returns page 1
```

Three concrete ways it collapses:

**a. LIKE is a sequential scan.**
A B-tree index on `title` is useless for leading-wildcard LIKE (`%iphone%`). Postgres cannot use the index because it does not know where in the string the match occurs. It reads every row. At 500 million rows averaging 500 bytes each, that is 250 GB of data to scan per query. Even at 10 GB/s sequential disk read speed, that is 25 seconds. No cache helps because 250 GB does not fit in RAM for most servers.

**b. Boolean match returns noise, not ranked results.**
A match means the word "iphone" appears in the title. With 500 million products, millions of items match the string "iphone". Sorting by `sales_count` alone ignores the query: a popular phone charger outranks an exact iPhone 13 case because it has more total sales. The user wanted relevance, not just popularity.

**c. No scale path.**
Every new query hits the primary database. At 30 billion queries per month (roughly 1 million per minute at peak), a single Postgres instance is destroyed. Adding read replicas only replicates the scan problem across more machines.

**The demo fails because it treats the database as a query engine. The correct mental model is: search pre-processes documents at write time, so query time is just a fast lookup into structures built ahead of time.**

---

## 3. The architecture

Top to bottom, each layer doing one job:

```
[User types query]
     |
     v
[Query Understanding]
     spell correction: "iphoen" -> "iphone"
     synonym expansion: "cell phone cover" -> "phone case"
     intent classification: product search vs. brand search vs. question
     analogy: a librarian who rewrites your vague request into a precise catalog term
     |
     v
[Query Router / Cache]
     exact-match cache for hot queries: "iphone 13 case" is searched 10,000x/min
     cache hit rate: ~55-60% of queries are repeated across users in the same hour
     analogy: the front desk that answers common questions from a FAQ board before routing to anyone
     |
     v
[Stage 1 — Candidate Retrieval] (milliseconds, 500M -> 5,000 candidates)
     Inverted Index (Lucene shards, one per product segment)
     For each query term, pull the posting list (list of product IDs containing that term)
     Intersect/union the posting lists across query terms
     Score each candidate with BM25 (a term-frequency-inverted-document-frequency formula)
     Approximate Nearest Neighbor search (FAISS/HNSW) for semantic/vector matches
     Output: top 5,000 candidate product IDs with a cheap score
     analogy: a warehouse worker who pulls all boxes from the "iphone" aisle without inspecting each box
     |
     v
[Stage 2 — Reranking] (tens of milliseconds, 5,000 -> 200 candidates)
     Feature fetch: for each candidate, retrieve 100+ signals
       - Query-document match score (BM25, TF-IDF)
       - Click-through rate for this query-product pair (historical)
       - Purchase conversion rate
       - Review count and average rating
       - Price, availability, Prime eligibility
       - Recency of listing
     Gradient Boosted Tree model (XGBoost / LightGBM): 500-1000 trees, each a fast lookup
     Neural re-ranker for top 500 (BERT-based cross-encoder: scores the pair (query, product) together)
     analogy: a specialist who reads each box label carefully and ranks them by relevance to your exact need
     |
     v
[Stage 3 — Business Logic Layer] (microseconds, 200 -> 48 shown to user)
     Sponsored item injection (auction-based, interleaved at fixed positions)
     Inventory filter (in-stock, ships to user's ZIP code)
     Personalization boost: items matching user's past purchases are promoted
     Diversity rule: no more than 3 items from same brand in top 10
     analogy: a store manager who sets up the front shelf to balance what sells vs. what makes money
     |
     v
[Cache Fill + Response]
     Top 48 results cached at the query level for 5 to 60 minutes
     Product cards (image, price, title) served from separate CDN-backed catalog service
     First byte to user: <100ms on fast connections
```

**The key insight:** Stage 1 (retrieval) is about recall. Get every possibly-relevant document. Speed is paramount, relevance is secondary. Stage 2 (ranking) is about precision. Score accurately. Speed is secondary. These two goals are fundamentally in tension, which is why they are split into separate stages.

---

## 4. The transferable mechanisms

### a. The Inverted Index

This is the core data structure of all search. Build it at write time, query it in constant time.

At write time, for every product:
```
Product ID 101: "Apple iPhone 13 case clear slim"
Product ID 204: "iPhone 13 protective case"
Product ID 890: "Samsung Galaxy case clear"
```

The inverted index stores, for each term, the sorted list of documents containing it:

```
"apple"   -> [101, 3040, 7201, ...]
"iphone"  -> [101, 204, 502, ...]
"13"      -> [101, 204, ...]
"case"    -> [101, 204, 890, ...]
"clear"   -> [101, 890, ...]
```

Each entry in the posting list also stores term frequency in that document (how many times "iphone" appears in product 101). This enables BM25 scoring.

At query time for "iphone 13 case":
1. Fetch three posting lists: [iphone], [13], [case].
2. Intersect them (AND semantics) or union (OR). For AND: merge the sorted lists in O(min(N)) where N is the shortest list. Skip pointers let the merger jump forward when one list is far ahead of the other.
3. Score each matching document with BM25.

This lookup takes under 5 ms for 500 million products because you never touch documents that do not contain the query term. The scan is proportional to the size of the posting list, not the size of the corpus.

**Lucene/Elasticsearch implementation detail:** Posting lists are stored in a compressed columnar format called `.doc` files. Compression is done with Frame of Reference (FOR) encoding: since document IDs are sorted, you store the deltas between consecutive IDs, then pack those deltas into fixed-width groups. A posting list for "case" in 500 million products might occupy 50 MB on disk and decompress to posting entries in microseconds using SIMD instructions.

### b. BM25 — the relevance formula

BM25 (Best Match 25) is the standard term-based relevance formula. It improves on raw TF-IDF by:
1. Saturating term frequency: a term appearing 10 times is not 10x more relevant than appearing once. BM25 uses a logarithmic saturation curve.
2. Normalizing document length: a short title with "iphone" is more relevant than a 500-word description with one mention of "iphone".

```
BM25(query_term t, document d) =
  IDF(t) * (TF(t,d) * (k1 + 1)) / (TF(t,d) + k1 * (1 - b + b * |d|/avgdl))
```

Where:
- IDF(t) = log((N - df + 0.5) / (df + 0.5) + 1): inverse document frequency. Rare terms score higher.
- TF(t,d): term frequency in this document.
- |d|: document length (in tokens). avgdl: average document length across corpus.
- k1 = 1.2: controls TF saturation. b = 0.75: controls length normalization.

BM25 is fast enough to score 5,000 candidates in under 1 ms. It is the standard first-stage ranker in Elasticsearch, Lucene, Solr, and OpenSearch.

**What BM25 cannot do:** It is purely lexical. It does not understand that "cell phone cover" and "smartphone case" are semantically the same. That is where vector/dense retrieval comes in.

### c. Two-stage retrieval: recall first, precision second

Stage 1 maximizes recall. "Get everything that could possibly be relevant." BM25 on the inverted index does this. The output is a candidate set of 1,000 to 10,000 items.

Stage 2 maximizes precision. "From the candidate set, find the best 48." Here you can afford expensive signals: neural cross-encoders that read the full query and document together, click-through rate from historical data, conversion rate, user-specific purchase history.

The trade-off is explicit: Stage 2's ML model is too slow to score 500 million documents (it would take hours). But it is fast enough to score 5,000 candidates in 20-50 ms. Stage 1 is too inaccurate to serve the final ranking directly. Together they achieve both speed and quality.

**Learning-to-rank (LTR):** Stage 2 uses a supervised ML model trained on past search logs. The training signal is implicit: if a user clicked product A and ignored product B (shown in a higher position), that is a weak signal that A was more relevant. Pointwise LTR trains a regression model per document. Pairwise LTR trains on pairs (A should rank above B). Listwise LTR optimizes the ranked list as a whole (NDCG, MAP). Amazon's published research uses pairwise LTR with gradient boosted trees as the base model.

### d. Dense Retrieval (ANN search for semantic matches)

A user who types "something to protect my new Samsung" will miss lexical matches for "Galaxy S24 case" entirely. Dense retrieval solves this:

1. Encode every product title + description with a BERT-based encoder into a 768-dimensional embedding vector. Store all 500M embeddings in a vector index (FAISS).
2. At query time, encode the query with the same encoder. Find the nearest neighbors by cosine similarity in the embedding space.
3. The K nearest neighbors (K=1000) are semantically similar documents, even without shared words.

The challenge: finding exact nearest neighbors in 768 dimensions across 500M vectors takes too long. The solution is Approximate Nearest Neighbor (ANN) search. HNSW (Hierarchical Navigable Small World) is the dominant algorithm. It builds a multi-layer graph where each layer is a subset of nodes connected by proximity edges. Query traversal starts at the top layer (fewest nodes, long-range edges) and drills down to the bottom layer (all nodes, short-range edges). HNSW achieves >95% recall at 10x the speed of brute force.

Dense retrieval is merged with BM25 retrieval at Stage 1: take the union of BM25 candidates and ANN candidates, then pass to Stage 2 for reranking.

### e. Segment-based incremental indexing

You cannot rebuild the entire inverted index every time a new product is added. Lucene solves this with segments.

- Incoming product writes go to an in-memory buffer (the "indexing buffer").
- When the buffer is full, it is flushed to disk as an immutable segment (a self-contained mini-index).
- A new segment appears as searchable within 1 second of the write (near-real-time search).
- Background merge: small segments are periodically merged into larger segments, reducing the total number of segments a query must search.
- Merge policy controls segment count: Elasticsearch defaults to TieredMergePolicy, which targets a maximum of 10 segments in a shard.

**The write-vs-search trade-off in segments:** During a merge, the search thread must also search the pre-merge segments until the merge completes. Query latency spikes during heavy merges. Production Elasticsearch clusters throttle merge speed to protect query latency.

### f. The feature store for ML ranking

Stage 2 needs 100+ features per document-query pair. Computing them at query time is too slow. The solution is a feature store: a low-latency key-value store (Redis, DynamoDB) that holds precomputed features per product.

- Offline features: computed daily. Average rating, total review count, brand reputation score.
- Near-real-time features: computed hourly. Conversion rate for this product this week.
- Online features: computed at query time. BM25 score (query-dependent, cannot precompute), user's click history for this session.

The feature store decouples feature computation from feature serving. The ML model at Stage 2 just does a batch key-value lookup for all 5,000 candidates in parallel, merges online features, and runs inference.

---

## 5. The trade-offs

### Consistency vs. availability, per data type

| Signal | Staleness tolerance | Why |
|--------|-------------------|-----|
| Inverted index | 1 to 10 seconds (Lucene NRT) | A new product listing is fine to appear with 5-second lag |
| BM25 score | Same as index freshness | Recomputed per query, no cache |
| Click-through rate | 1 to 24 hours | CTR is stable over hours; hourly refresh is fine |
| Price / inventory | 5 to 60 seconds | Inventory must not be grossly wrong; user expects price to be current |
| Personalization model | Daily or weekly | Re-training a neural ranker takes hours; daily is typical |
| Sponsored ads | Real-time (per query auction) | Revenue-critical; cannot be stale |

CAP trade-off: Amazon's product catalog is AP (available + partition tolerant). You can add a product and it appears in search after a few seconds of eventual consistency. You cannot block the add waiting for the inverted index to confirm. The cluster accepts the write, returns success, and the index update propagates asynchronously.

### Cost vs. latency

- Serving a BM25 inverted index query on one shard: ~1 ms CPU. Cheap.
- Serving 5,000 candidates through a gradient boosted tree ranker: ~10-20 ms CPU. Moderate.
- Serving 500 candidates through a BERT cross-encoder: ~50-200 ms GPU. Expensive.
- Caching 60% of queries at the query level eliminates the expensive path for repeated queries.

The design choice: build cheap L1 to avoid running expensive L2 on every query. Cache aggressively at L1 output. Only run L2 when the L1 cache misses.

### Precision vs. recall

BM25 recall: high for lexical matches, zero for semantic misses. Dense retrieval recall: high for semantic matches, slower and approximate. Hybrid retrieval (BM25 union ANN) has the highest recall but the largest candidate set to re-rank. The engineering decision is where to cap the candidate set: 1,000 is cheaper to re-rank but risks missing good semantic matches; 10,000 is safer but pushes Stage 2 latency past 100 ms.

---

## 6. The systems-thinking lens

**The feedback loop that causes failure: the thundering herd on hot query cache miss.**

1. Query "iPhone 15 Pro Max" is searched 50,000 times per minute. It is cached with a 60-second TTL.
2. At second 60, the cache entry expires. One request misses, goes to Stage 1 + Stage 2.
3. Before that request returns (say, 80 ms), 6,000 more requests for the same query miss the empty cache and also go to Stage 1 + Stage 2.
4. 6,001 concurrent ML ranking jobs compete for GPU capacity. Latency spikes to 2 seconds. Some time out.
5. The system is more degraded than before the cache existed.

**Named pattern: thundering herd / cache stampede.** Popular caches do not experience gradual miss; they experience synchronized avalanche because all TTLs expire at the same moment.

**The senior fix breaks the loop at three places:**

1. **Probabilistic early expiration (PER, also called "cache jitter" or "eager expiration").** Before the cache entry officially expires, randomly begin re-fetching it early. The probability of early re-fetch increases as the TTL deadline approaches. Only one request triggers the re-fetch; others continue to read the stale (still-valid) cached result. The entry is warm before expiration. No thundering herd.

   Implementation: on each cache hit, generate a random number. If `(current_time + random * scale) > expiry_time`, trigger a background re-fetch while returning the cached result. Scale defaults to `TTL * 0.1`.

2. **Single-flight / request coalescing.** When multiple goroutines (or worker threads) request the same missing key simultaneously, they are coalesced into ONE upstream call. All waiters block on the same future. When that one call returns, all get the result. The thundering herd reduces from N concurrent calls to 1.

3. **Lazy hot-key detection.** Count query frequency at the cache layer itself. If a query crosses a threshold (say, 500 hits per second), pin it in a local in-process cache (L1, on the app server) with a 5-second TTL. Even a cache miss at the distributed cache (Redis) does not go upstream: the local cache absorbs it. Hot keys live in RAM on every app server and never reach the search tier.

**The meta-lesson:** the failure mode for search is not gradual degradation. It is a sudden phase transition: the moment a popular query's cache expires, the system snaps from healthy to catastrophic load in 80 ms. The senior fix is proactive (prevent the expiration from being synchronized) rather than reactive (detect the degradation and add capacity).

---

## Map to Rare.lab's stack

**What you already have:**
- Cloudflare R2 stores immutable scene JSON objects with content-addressed keys. Each unique scene file has a unique SHA-based URL. This is already segment-friendly: you never update a key in place, you always write a new one. A search index over these objects can be rebuilt incrementally by following the manifest.
- Supabase Postgres holds the manifest: which scene IDs exist, their metadata (name, tags, user ID, created_at). This is your searchable catalog.

**What you do not have yet but will need at 50,000 scenes:**
A simple `WHERE name ILIKE '%query%'` on Supabase will work up to about 10,000 scenes with a GIN index on the tsvector column. At 50,000 scenes with complex tag-based queries, you hit the single-shard wall: no personalization, no semantic search, no ranking by relevance.

**The ceiling and the next step:**
1. Enable Postgres full-text search today: add `tsvector_update_trigger` on the scene metadata table, index it with GIN. This gets you BM25-equivalent ranking for free in Postgres up to 50k scenes.
2. At 50k scenes or when semantic search is needed: pipe scene metadata from Supabase CDC (Supabase Realtime) into Typesense (self-hosted) or Algolia (managed). Both support BM25 + vector hybrid search with sub-10ms queries on 1M documents.
3. The ML ranking layer (Stage 2) is not needed until Rare.lab has enough usage data (click logs, node graph opens) to train a model. Stage 1 with BM25 + tag filters will be the right ceiling for 12 to 18 months.

**The node-based editor angle:** when a user searches for a shader node by type ("blur", "chromatic aberration", "voronoi"), they are doing a mix of lexical ("voronoi") and semantic ("noise with hexagonal pattern" should match "voronoi noise") search. Dense retrieval on node descriptions and tags is exactly the right tool when the node library grows past 1,000 entries.

---

## How to approach this in a system design interview

The LeetCode mock interview format (from the video transcript linked in references) applies directly to any search system:

**Functional requirements first:**
- View list of items (paginated, filtered by category/difficulty/tag)
- Search by query string
- Rank results by relevance

**Non-functional requirements:**
- Latency: under 200ms for query results
- Scale: millions of items, millions of daily queries
- Consistency: eventual is fine for search index freshness; inventory must be near-real-time

**Core entities:** Item (the document), SearchQuery, Ranking Score, User (for personalization)

**API shape:**
```
GET /search?q=iphone+13+case&page=1&limit=48&category=accessories&sort=relevance
```

**Walk the architecture:** start at the user's keypress, go through query understanding, inverted index lookup, BM25 scoring, ML ranking, business rules, return. Name each component, its single job, and its latency budget. For a 200ms budget: 5ms retrieval + 50ms ranking + 10ms business logic + 5ms network = 70ms, well under budget. This leaves room for cache misses.

**Scaling story (three tiers):**
- 10,000 items: single Postgres GIN index, no sharding, sub-5ms queries.
- 1,000,000 items: Elasticsearch with 3 shards, no ML ranking, BM25 only.
- 100,000,000 items: Elasticsearch with 100 shards across 20 nodes, FAISS for dense retrieval, LightGBM Stage 2 ranker, query cache at Redis.

---

## References and summaries

### Core documentation and official resources

**Lucene BM25Similarity API Documentation**
https://lucene.apache.org/core/4_6_0/core/org/apache/lucene/search/similarities/BM25Similarity.html
The official Lucene API docs for BM25. Shows the exact formula Elasticsearch and OpenSearch use under the hood. Key defaults: k1 = 1.2 for term-frequency saturation; b = 0.75 for document-length normalization; IDF = log(1 + (numDocs - docFreq + 0.5) / (docFreq + 0.5)). Read this before debugging Elasticsearch relevance issues. Lucene's BM25 is different from textbook BM25 in one subtle way: it adds +1 inside the IDF log to prevent negative IDF for terms that appear in every document. The diff from raw TF-IDF: term frequency saturates, so seeing a word 20 times is not 20x more useful than seeing it twice.

**Elasticsearch from the Bottom Up, Part 1**
https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up
Elastic's deep-dive on Lucene segment architecture: how shards are split into immutable segments, why writes go to an in-memory buffer first, how segments are flushed to disk and later merged. The key insight in this article: search queries must hit every open segment and merge the results, so reducing segment count via merges directly reduces query cost. Covers why segment immutability is a feature, not a limitation: once a segment is written, it never needs locking, so parallel reads are free. Essential reading for anyone managing an Elasticsearch cluster under write load.

**Frame of Reference and Roaring Bitmaps (Elastic Engineering)**
https://www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps
Explains how Elasticsearch compresses posting lists using Frame of Reference (FOR) encoding and Roaring Bitmaps. FOR: since posting list document IDs are sorted, store deltas between consecutive IDs instead of absolute values; pack deltas into fixed-width groups (32-byte blocks); decode with SIMD in microseconds. Roaring Bitmaps: for dense posting lists, switch to bitmap representation where each bit position represents a document ID. The result: a posting list for a common word like "the" in 1 billion documents can fit in 1-2 GB and decompress in under 1 ms. This is why lexical search at scale is faster than it looks.

**FAISS GitHub: Billion-Scale Similarity Search**
https://github.com/facebookresearch/faiss/wiki
Facebook AI Research's library for ANN search. Covers all index types: Flat (exact, slowest, O(N)), IVF (inverted file index, partitions space into Voronoi cells for fast approximate search), HNSW (graph-based, fastest at high recall, O(log N)), and PQ (product quantization, smallest memory at the cost of recall). Benchmark numbers: HNSW on 1 billion 128-d vectors, recall@10 of 97% at 5ms per query on a single CPU thread. The algorithm comparison table in the wiki is the best single-page guide for choosing between index types based on your dataset size, memory budget, and recall-vs-latency target.

### Engineering blogs from real companies

**Spotify: Rethinking Spotify Search**
https://engineering.atspotify.com/2021/04/rethinking-spotify-search
How Spotify rebuilt their search pipeline into two explicit stages. Stage 1 generates candidates by querying the inverted index with BM25 for lexical matches plus ANN for semantic matches. Stage 2 re-ranks those candidates using a model that combines query-document match signals with personalization signals (what you have listened to, saved, and searched before). Key insight Spotify shares: the biggest recall gains came not from better ranking models but from better query understanding (synonym expansion, transliteration, entity recognition). Their index covers 100 million tracks, 5 million podcasts, and 500k audiobooks, all queried under 200ms.

**Spotify: Introducing Voyager — Spotify's ANN Library**
https://engineering.atspotify.com/2023/10/introducing-voyager-spotifys-new-nearest-neighbor-search-library
Spotify used approximate nearest-neighbor search internally for over a decade (first with Annoy, then migrated to Voyager, which is built on hnswlib). This post explains why they replaced Annoy: Annoy builds random-projection trees which are fast to build but less accurate at high recall than HNSW graphs. Voyager (HNSW-based) hits the same recall@10 as Annoy but at 3x lower query latency. Key production detail: Voyager loads the full index into memory on startup (no disk I/O during queries), so startup time is proportional to index size; they use incremental index snapshots to avoid full rebuilds on every model update.

**Netflix: Innovating Faster on Personalization Algorithms Using Interleaving**
https://netflixtechblog.com/interleaving-in-online-experiments-at-netflix-a04ee392ec55
Netflix cannot run classic A/B tests on ranking algorithms because users see one list, not two. Their solution: interleaving. Take algorithm A's ranked list and algorithm B's ranked list, interleave them into one list while tracking which algorithm contributed each item, then measure which algorithm's items got clicked more often. Interleaving detects ranking quality differences with 100x smaller user samples than a classic A/B test. This means Netflix can evaluate a new ranking model on 1% of users in 1 day instead of 1% of users for 100 days. The article explains the Team Draft interleaving algorithm step by step.

**Amazon OpenSearch Service: Learning to Rank**
https://docs.aws.amazon.com/opensearch-service/latest/developerguide/learning-to-rank.html
Amazon's production LTR integration for their managed search service. Covers the full pipeline: upload a training dataset (judgment lists with query-document relevance grades), train a model with XGBoost or RankLib, deploy the model as a rescore phase on top of BM25 retrieval. Key numbers from the docs: reranking 100 candidates with a gradient-boosted tree model adds roughly 3-8ms to query latency. The judgment list format (query, document ID, relevance grade 0-3) is exactly what you would build by logging user clicks and applying a demotion heuristic (click = grade 2, skip above a click = grade 0). Read this to understand the concrete data pipeline from search logs to trained ranking model.

**Uber Eats: Query Understanding**
https://www.uber.com/en-IN/blog/uber-eats-query-understanding/
Uber Eats processes millions of food search queries where the same dish is spelled dozens of ways ("cheeseburger", "cheese burger", "chz brgr"). Their three-layer pipeline: normalization (lowercase, strip punctuation, expand abbreviations), entity recognition (BERT-based model trained to classify tokens as dish-type, cuisine, restaurant-name, or modifier), and intent classification (is this a craving search or a restaurant lookup?). They run a multilingual BERT fine-tuned on their query logs, inference in under 15ms. The most practical lesson: query reformulation (expanding "pizza" to include "Italian restaurant" and "pizzeria") gave bigger recall improvements than any ranking model change.

### Research papers

**HNSW: Efficient and Robust ANN Search Using Hierarchical Navigable Small World Graphs**
https://arxiv.org/abs/1603.09320
The paper behind the algorithm used by FAISS, pgvector, Qdrant, and Weaviate. HNSW builds a multi-layer proximity graph. Higher layers have fewer nodes connected by long-range edges (for fast approximate traversal), lower layers have all nodes connected by short-range edges (for precise local refinement). Query traversal starts at the top layer and drills down. Query time scales as O(log N). The paper benchmarks HNSW at 99% recall@10 on SIFT1M (1 million 128-dimensional vectors) in 0.4ms per query on a single CPU core. This is a 360x speedup over brute-force exact search with only 1% recall loss. The key parameters to tune: M (edges per node; M=16 is a good default), efConstruction (graph quality at build time; higher is better but slower), efSearch (query quality at search time; higher recall costs more latency).

**TF-Ranking: A Scalable TensorFlow Library for Learning-to-Rank**
https://arxiv.org/abs/1812.00073
Google's open-source LTR framework used internally in Gmail search, Google Drive search, and Google Play. Covers all three learning-to-rank paradigms in one framework: pointwise (predict a score per document), pairwise (predict which of two documents is more relevant), and listwise (optimize a list-level metric like NDCG directly). Key contribution: listwise loss functions were previously hard to implement in distributed training because they require all documents in a ranked list to be in the same gradient step. TF-Ranking provides a custom `StratifiedSampler` that colocates list members on the same GPU. Read this if you want to understand why NDCG-based training is harder than accuracy-based training.

**Two-Stage Retrieval with FlashRank Reranking and Query Expansion**
https://arxiv.org/abs/2601.03258
A recent paper formalizing the two-stage paradigm for production retrieval-augmented systems. Stage 1 (retrieval) optimizes for recall: get every possibly-relevant document using BM25 and/or dense retrieval. Stage 2 (reranking) optimizes for precision: re-score the top-K candidates using a cross-encoder that reads query and document together. FlashRank introduces dynamic subset selection: instead of always reranking the top 100, it adaptively selects a subset based on Stage 1 confidence scores, reranking only candidates above a threshold. This cuts GPU inference cost by 40-60% with negligible NDCG loss. Key numbers: FlashRank on BEIR benchmark achieves the same NDCG@10 as full reranking at 55% of the compute.

**Learning to Rank for E-Commerce Search**
https://arxiv.org/pdf/1903.04263
A practical survey of how pointwise, pairwise, and listwise LTR approaches perform on real e-commerce product search. Key finding: pairwise LTR (specifically LambdaMART) consistently outperforms pointwise approaches on NDCG, but requires 10x more training data. For small catalogs (<100k products), a pointwise gradient-boosted tree (XGBoost) trained on click logs outperforms a neural model because there is insufficient data to train the neural model's extra parameters. The paper's training data pipeline (implicit feedback from click logs, with position bias correction) is the most practical section: it shows how to convert raw server logs into a training dataset with correct relevance labels despite the position bias in what users actually see.

### YouTube videos and talks

**System Design: Typeahead Search (Gaurav Sen)**
https://www.youtube.com/watch?v=xrYTjaK5QVM
The best structured system design walkthrough for search typeahead (autocomplete). Covers trie data structures for prefix matching, the RAM problem at scale (you cannot fit 100 billion phrases in one trie), and how to shard the trie by prefix (all queries starting with "A" go to shard 1, "B" to shard 2, etc.). Includes a capacity estimation that you can reproduce in an interview: 1 billion daily users, 5 searches per day, 5 characters per search = 25 billion autocomplete requests per day, roughly 290,000 QPS. Serving 290k QPS with sub-50ms latency requires roughly 100-150 servers. This video is a template for how to do capacity math in an interview: start with users, multiply through to requests, convert to QPS, divide by server capacity.

**Designing LeetCode: Full System Design Mock Interview**
https://www.youtube.com/watch?v=p6Q7yrGj7VM
A Google software engineer walks through designing LeetCode end to end with an interviewer. This video is a masterclass in interview structure: requirements first (view problems, code solution, submit, get feedback, live leaderboard), non-functional requirements second (scale to 1M DAU, 100k concurrent during contests, low latency, code isolation security), core entities third, API design fourth, HLD fifth, optimizations last. Key design decisions covered: containers vs. VMs for isolated code execution (containers win on resource efficiency and startup speed), queue between API tier and execution containers (smooths contest submit spikes), Redis sorted set for the live leaderboard (O(log N) update per submission, O(1) range query for top-K). The submit queue section directly applies to any system where spiky writes hit a slow backend, like Rare.lab processing shader compile requests.

**How Search Engines Work: Crawling, Indexing, and Ranking (Google Search Central)**
https://www.youtube.com/watch?v=BNHR6IQJGZs
Google's official explainer for the three phases of web search. Crawling: discovering and downloading pages. Indexing: parsing content, extracting links, building the inverted index. Ranking: matching the query to the index using 200+ signals, personalizing, and returning results in under 100ms. Best stat from this video: Google's index contains hundreds of billions of web pages, but a typical query only needs to look at a tiny fraction because the inverted index narrows the candidate set immediately. Watch this to get the canonical mental model of how search pipelines work before drilling into implementation details.

**Elasticsearch Segment Merging and Write Path (Elastic YouTube Channel)**
https://www.youtube.com/watch?v=PpX7J-G2PEo
Covers the Lucene write path in detail: why new documents go to an in-memory buffer first, how the buffer flushes to a new immutable segment, what happens during background merges (both the pre-merge and post-merge segments are available to queries simultaneously, which temporarily doubles disk usage). The key monitoring advice: track not just total segment count (should stay under 10 per shard) but also merge throughput (MB/s). When merge throughput drops below write throughput, segments accumulate and query latency climbs. The throttle.max_bytes_per_sec setting controls how aggressively merges run; the default is conservative to protect query latency, but during initial bulk indexing you should raise it.

### Articles

**Algolia Engineering: Inside the Engine, Part 3 (Query Parsing and Tokenization)**
https://www.algolia.com/blog/engineering/inside-the-engine-part-3-query-parsing-and-tokenization/
Algolia processes 1.75 trillion search queries per year. This post explains their tokenization and typo-tolerance pipeline. Spell correction works via a prefix trie where each node stores an edit-distance-1 neighborhood pre-computed at index build time, so query-time correction is a single O(length) trie traversal with no runtime edit-distance computation. Key calibration numbers: 1-typo correction kicks in after 4+ characters, 2-typo after 8+ characters. Below these thresholds, correction hurts precision more than it helps recall (too many false-positive "corrections"). These thresholds were calibrated from click-through-rate data, not from theory.

**Elasticsearch Learning to Rank: Introduction**
https://www.elastic.co/search-labs/blog/elasticsearch-learning-to-rank-introduction
Elastic's introduction to native LTR (available from Elasticsearch 8.13+). Explains the two-stage flow: standard Elasticsearch BM25 retrieval returns top-K candidates, then the LTR rescorer runs a trained XGBoost or PyTorch model on those candidates to produce the final ranked list. The training data format is a judgment list: triples of (query, document ID, relevance grade 0-3). Shows how to generate judgment lists from user click logs using the assumption that a click is grade 2 and a skip-below-click is grade 0. Powers production search at Wikimedia (Wikipedia search) and Snagajob.

**Why Search Engines Don't Use Linear Search (Medium)**
https://medium.com/@raghavattri13/why-search-engines-dont-use-linear-search-understanding-the-inverted-index-ca796466ee6f
A clean explanation for engineers new to search of why SQL LIKE queries and linear scans break at scale, and how the inverted index fixes it. Includes a worked example: for the query "cat sat mat", the inverted index returns the posting list intersection in O(k) where k is the size of the rarest term's posting list, not O(N) where N is the corpus size. For a corpus of 1 billion documents where "mat" appears in 10,000 documents and "cat" appears in 5 million, the intersection starts from the 10,000 "mat" entries and runs in milliseconds, not seconds. This mental model (work is proportional to the rarest term, not the corpus) is what you should communicate in an interview when asked why search is fast.
