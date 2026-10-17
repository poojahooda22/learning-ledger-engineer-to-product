# Day 91: A developer types `parseConfig` into a box and wants every match across billions of files in under a second. How does code search work when grep would take minutes? Trigram indexes, sharding by content hash, and verify-then-rank

**Date:** 2026-10-17
**Difficulty:** Advanced (inverted indexes for text that is NOT words: symbols, punctuation, regex)
**Topic:** Earlier lessons covered web-search ranking (Day 18), typeahead (Day 61), vector search (Day 33), content-addressed storage (Day 23), file dedup (Day 86) and LSM-style write paths (Day 21). Code search needs a different index, because code has no clean "words". People search for `->next`, `func\s+\w+Handler`, `0x7fff`. Today's lesson builds that index and shows why it is sharded by content hash.
**Stack relevance:** Rare.lab stores scene JSON immutably and content-addressed in R2, with Postgres rows in Supabase. Section 7 shows how to search shader node graphs and generated GLSL with Postgres `pg_trgm` today, and where that stops working.

---

## 0. The framework (same recipe as Day 74, the LeetCode mock interview)

The LeetCode mock interview you pasted was already taught as [Day 74](074-leetcode-online-judge-end-to-end.md), so today is a NEW topic that reuses its six-step recipe. The recipe is the thing to memorize, not the product:

1. **Functional requirements (and out of scope).** User types a query (plain text or regex), gets matching files with line snippets, can filter by repo, language, path. Out of scope: code navigation, pull requests, login, permissions UI.
2. **Non-functional requirements.** p99 under about 1 second. New commits searchable within minutes (not days). Exact results for exact queries (a missed match is a bug, not a ranking choice). Scales to billions of files. No single point of failure.
3. **Core entities.** Repository, Blob (one file version, identified by hash of its bytes), Index shard, Posting list, Query.
4. **API.** `GET /search?q=...&repo=...&lang=...&page=1&limit=20`. One route, paginated, like the LeetCode problem list.
5. **Naive high-level design.** Client, API server, database. Store files in a table, run `SELECT ... WHERE content LIKE '%parseConfig%'`.
6. **Deep dives.** Break the naive design with a number, then fix each break and say the trade-off out loud.

The interviewee's best move in the transcript was "running user code on the API server is a bad idea, isolate it, put a queue in front". Today's equivalent: "scanning every file per query is a bad idea, do the expensive work once at write time (build an index) and put a cheap lookup in front".

---

## 1. The company and the breaking number

**The company: GitHub.** In February 2023 GitHub engineers published "The technology behind GitHub's new code search", describing **Blackbird**, a search engine they wrote from scratch in Rust, built on **sharded ngram indexes**. The older Google Code Search (shut down 2012) used a trigram index, which Russ Cox documented in "Regular Expression Matching with a Trigram Index" (2012).

**The breaking number (worked estimate, labeled, not a GitHub figure).**
- Assume a code corpus of **100 TB of text**. (From memory of the GitHub post: roughly 115 TB of code in about 15.5 billion documents, about 25 TB after removing duplicate content. I could not open the post this session because the network proxy blocked it, so treat those three numbers as recalled, not verified. Check them on the GitHub blog.)
- A fast brute-force scanner like ripgrep reads on the order of **1 GB/s per core** on cached data. One query over 100 TB = 100,000 core-seconds.
- Spread across 1,000 cores that is **100 seconds per query**. At 1 query per second you already need 100,000 cores. At 100 queries per second, 10 million cores.

So the breaking number is **100,000 core-seconds for one query**. The target is **under 1 second of wall time and a few core-milliseconds of work**. That is a gap of about 10 million times. Only an index closes a gap that size.

Analogy: finding every page in a library that mentions "Bangalore". The naive way: walk the shelves and read every page. The index way: a card catalogue at the front that says "Bangalore: book 4 page 12, book 9 page 301...". Building the catalogue is slow and done once. Reading it is instant.

---

## 2. Why the naive design dies

**Naive version:** one table `files(repo, path, content)`, one Postgres, queries use `ILIKE '%parseConfig%'`.

- **A leading wildcard cannot use a normal B-tree index.** A B-tree sorts values left to right, like a dictionary sorted by first letter. `'%parseConfig%'` could start anywhere, so Postgres falls back to scanning every row (a sequential scan). That is the 100,000 core-second problem above in miniature.
- **Word-based full-text search (Elasticsearch-style, Postgres `tsvector`) breaks on code.** Tokenizers split on spaces and punctuation and then stem words ("running" becomes "run"). Code needs `->next`, `foo.bar()`, `0x7fff`, `<<=` and `snake_case_names` to survive intact. Stemming `parseConfig` as a word also cannot find `parseConfigFile`. GitHub's own post says the general-purpose engines they tried (as recalled) were too slow and too costly at this size. Another phrase in the same post, per the search excerpt I could read: an ngram index is "a special type of inverted index for looking up substrings of content".
- **One machine cannot hold the index.** Zoekt (the open-source engine Sourcegraph maintains) reports an index of **about 2 to 3 times the size of the source** (from the Sourcegraph blog excerpt). 25 TB of unique code becomes roughly 50 to 75 TB of index. No single box holds that in RAM.
- **Duplicate files multiply everything.** Forks, vendored `node_modules`, copy-pasted licenses and generated code mean the same bytes appear millions of times. Indexing each copy wastes storage and fills results with 40 identical hits.
- **Freshness.** A nightly rebuild of a 25 TB index means a commit pushed at 9am is invisible until tomorrow. Developers expect "I pushed it, I can search it".

---

## 3. The architecture

```
Clients (web search box, IDE, API)
  - job: send a query string; show path + highlighted snippet; paginate
  - analogy: you ask the librarian a question and get a short list, not the whole shelf

        |
        v
Edge / CDN
  - job: serve the static UI. Search results are mostly per-user (private repos)
    so they are NOT edge-cached. Popular PUBLIC queries can be cached for seconds.
  - analogy: the lobby signage. The answers still come from the back office.

        |
        v
Load balancer + stateless API tier
  - job: authenticate, turn the query text into a query plan (Section 4), attach
    the user's permitted repo list, fan the plan out to shards, merge answers
  - analogy: a head librarian who splits your question across all the archive rooms

        |
        v  (fan-out: ask EVERY relevant shard in parallel, the "scatter")
Index shard servers (hundreds to thousands; each owns a slice of the content)
  - job: hold a trigram -> posting list index for its slice (memory-mapped from
    SSD, so the OS pages hot parts into RAM). Intersect posting lists, return
    candidate documents.
  - analogy: each archive room has its own card catalogue for its own books

        |
        v  (the "gather")
Verify + rank stage (on the shard, then merged at the API tier)
  - job: open each candidate file and run the REAL regex or substring check,
    because the index only says "might match". Then score and keep top K.
  - analogy: the catalogue says page 12 mentions it, but you still walk over
    and read page 12 to be sure

Write path (runs in the background, never in the query path)
  Git push -> event on a log (Kafka) -> crawler reads new blobs from Git ->
  dedup by blob hash -> extract symbols and language -> route by hash to a shard
  -> shard appends to a new small index segment -> segments merge later
  - analogy: a mailroom that sorts new books into the right archive room and
    updates that room's catalogue, at its own pace
  - GitHub's post (search excerpt) says Kafka carries the events and lets each
    shard consume documents at its own pace. This is the queue-as-shock-absorber
    idea from Day 9 applied to indexing.

Blob store (object storage / Git itself)
  - job: the source of truth for file bytes, addressed by hash of the content
  - analogy: the stacks where the real books live. The catalogue only points here.
```

---

## 4. The transferable mechanisms

### 4.1 The trigram inverted index (match first, verify second)

**The idea.** Cut every file into overlapping 3-character windows. The word `limits` yields `lim`, `imi`, `mit`, `its` (the GitHub post uses this exact example). For each trigram, store a **posting list**: a sorted list of the document IDs that contain it.

**Walk one real query end to end.** Search `parseConfig`.
1. Trigrams of the query: `par`, `ars`, `rse`, `seC`, `eCo`, `Con`, `onf`, `nfi`, `fig`.
2. Look up each posting list. Suppose `par` has 40 million docs (very common), `Con` has 30 million, but `seC` has only 90,000 and `nfi` has 2 million.
3. **Intersect the lists, starting with the shortest** (`seC`), so work is bounded by the smallest list, not the biggest. A document must contain ALL nine trigrams to survive. Say 1,200 documents remain.
4. **Verify:** open those 1,200 files and run the real substring match. Maybe 1,100 truly contain `parseConfig`; 100 were false positives (they had all nine trigrams scattered in different places).
5. Rank the 1,100 and return the top 20.

Cost: a handful of list intersections plus 1,200 file reads, instead of 15 billion. This is the Russ Cox / Google Code Search design.

**Regex works too.** The query planner turns a regex into a boolean formula over trigrams. `func\s+\w+Handler` requires the literal `func` AND the literal `Handler`, so the plan is `(fun AND unc) AND (Han AND and AND ndl AND dle AND ler)`. `foo|bar` becomes an OR. A regex with no literal part of 3+ characters, such as `.*`, has no useful trigrams and degrades to a full scan; real systems cap or reject these.

**Why this separation is the key idea.** The index is allowed to say "maybe" (false positives are fine). It must never say "no" when the answer is "yes" (no false negatives). That is the same contract as a Bloom filter (Day 28). It makes the index small and fast, and the verify step restores exactness.

**Positional trigrams (Zoekt's variant).** Per the Sourcegraph excerpt, Zoekt stores WHERE each trigram appears, then looks up two trigrams from the query and checks they sit the right distance apart. That prunes most false positives before any file is opened.

**Trigrams are not always selective enough.** The GitHub excerpt says they found trigrams alone too unselective for short keywords in a code-heavy corpus, so Blackbird builds on sharded ngram indexes and tunes the gram choice. The principle: the longer the gram, the shorter the posting lists, but the more entries the index has. Picking the gram size is a space-versus-speed dial.

### 4.2 Shard by content hash (dedup comes free)

Route each blob to a shard using `hash(blob_bytes) mod N` (or a consistent hash ring, Day 10). Two consequences:
- **Duplicates collapse.** Identical bytes have the same hash, so they land on the same shard and are indexed once. The repo/path pairs that contain them are stored as a tiny side list. This is content addressing (Day 23) and dedup (Day 86) applied to search.
- **Load spreads evenly.** A hash does not care that one repo is enormous. Sharding by repo would put the Linux kernel on one machine and leave neighbors idle (a hot shard, Day 16).

Cost: every query must **fan out to every shard**, because any shard may hold a match. Fan-out latency is set by the slowest shard, which is the tail-latency problem from Day 76. Mitigations: hedged requests, per-shard timeouts, and returning partial results flagged as partial.

### 4.3 Append-only segments plus background merge (the LSM idea)

New commits must be searchable in minutes, but rewriting a 100 GB index file per commit is impossible. So each shard writes a **new small immutable segment** for new blobs and a query reads all segments. A background job merges small segments into big ones. Deletes are tombstones. This is the LSM-tree write path from Day 21, reused for search. Sourcegraph's blog (excerpt) also describes the opposite problem, a long tail of tiny repos creating many tiny shards, and "shard merging" to fix it. Zoekt flushes a shard to disk at about 100 MiB of input (excerpt).

### 4.4 Memory-map the index, let the OS be the cache

Zoekt lays shards out so they can be `mmap`'d (excerpt). The OS page cache keeps hot posting lists in RAM and evicts cold ones, with no custom cache code. This is the "cache the reads" primitive, but delegated to the kernel. Analogy: the frequently used catalogue drawers stay on your desk; the rest stay in the cabinet and are pulled out on demand.

### 4.5 Filters before index lookups (permissions as part of the query)

Private repos mean two users get different answers to the same query. Do NOT filter after ranking (a user could see "47 results" and get 3 back, leaking that private matches exist). Instead, pass the user's allowed repo set into the query plan so shards intersect it as one more posting list. See Day 35 (Zanzibar) for how the permitted set is computed.

### 4.6 Matching and ranking are two different halves

Matching (Sections 4.1 to 4.2) decides which documents qualify. Ranking decides the order. For code, useful signals are: exact symbol definition beats a mention in a comment; non-test, non-vendored paths beat tests and vendor dirs; popular repos beat abandoned forks. Blackbird's post (as recalled, unverified) describes extracting symbols at index time for this purpose. Ranking runs only on the few hundred verified candidates per shard, never on billions.

---

## 5. The trade-offs

| Decision | Choice | Why |
|---|---|---|
| Consistency vs availability for the index | **Availability, with bounded staleness.** A new commit appearing 2 minutes late is fine. | Same call as the LeetCode transcript's "990 vs 1,000 problems is fine". But GitHub's post lists query consistency as a design goal, so a single query should see a consistent snapshot per shard, not a half-written one. |
| Consistency for permissions | **Consistency.** | Showing a private file to the wrong person is a security bug, not a staleness bug. Check permissions at query time, never cache them for long. |
| Exact vs approximate | **Exact results, approximate index.** | Index returns a superset, verify step makes it exact. |
| Cost vs latency | Index is **2 to 3 times** the source size (Zoekt) and costs real RAM and SSD. | You pay storage once to avoid paying CPU on every query. At 100 qps the index wins by orders of magnitude (Section 1). |
| Write cost vs read cost | Indexing is slow and heavy; reads are cheap. | Code is written once and searched many times. If your data were written constantly and searched rarely, the trade flips. |
| Fan-out cost | Every query touches every shard. | Simple and balanced, but total CPU per query grows with shard count, and tail latency grows with it. |

---

## 6. The systems-thinking lens

**The feedback loop: the slow-shard retry spiral (a metastable failure, Day 13).**
1. One shard is slow (compaction running, cold page cache, noisy neighbor).
2. The API tier waits for it, so queries pile up and time out.
3. Clients and the API tier **retry**, doubling the load on every shard, including the healthy ones.
4. Healthy shards slow down under the extra load, more timeouts, more retries. The system stays broken even after the original slow shard recovers, because the retries themselves sustain the overload.

**Senior fixes break the loop; they do not add capacity:**
- **Per-shard deadlines and partial results.** After 300 ms, return what the fast shards gave, flagged "results may be incomplete". The slow shard no longer holds everyone hostage.
- **Hedged requests with a cap** (Day 76): a second request only after the p95 time passes, and only when under a retry budget (for example, at most 10 percent extra load).
- **Load shedding at the front** (Day 13): reject the most expensive queries first (`.*`, 1 or 2 character queries with huge posting lists) and keep cheap ones flowing.
- **Query cost estimation before fan-out.** The planner knows posting list sizes. A query whose smallest list has 500 million entries is refused with "be more specific". This is a circuit breaker based on the query's own price tag.
- **Keep indexing off the query path.** Index merges are rate-limited so they cannot steal I/O from readers.

---

## 7. Map to Rare.lab's own stack

Rare.lab keeps scene JSON as immutable, content-addressed files in R2, plus Postgres in Supabase with RLS. Useful search questions: "which scenes use the `noise` node with `octaves` above 6?", "find every shader that samples `u_time`", "find generated GLSL containing `discard`".

| Pattern from today | Rare.lab status | What to do |
|---|---|---|
| Content-addressed blobs (hash is identity) | **Already have it** (R2 immutable scene JSON + manifest) | The hash is already your dedup key. Two users who publish the same scene share one blob, so you only ever need to index each hash once. |
| Match-then-verify | Not built | Keep it simple first: a `search_text` column per scene version holding node type names, uniform names, and the generated GLSL. Index it with Postgres `pg_trgm` (a real Postgres extension). `CREATE INDEX ... USING gin (search_text gin_trgm_ops)` makes `ILIKE '%u_time%'` use a trigram index instead of a table scan. Supabase supports enabling `pg_trgm`. |
| Permissions inside the query | Already have RLS | RLS applies a policy filter to the query. Check with `EXPLAIN` that the trigram index and the RLS policy are both used; a policy that calls a slow function can turn the index lookup into a scan. |
| Shard by content hash | Not needed yet | One Postgres handles a GIN trigram index well into the **millions** of rows. |
| Segments + merge (LSM) | Postgres GIN has a "pending list" that does the same job | Tune `gin_pending_list_limit` if inserts get slow. |
| Fan-out and retry control | Not needed yet | Becomes relevant only when you split search onto its own cluster. |

**Where the next ceiling is.**
- **~100k to ~1M scenes:** GIN trigram index inside Supabase Postgres is fine. Index build and size (the index can approach or exceed the table size) are the first costs. Watch write latency on publish.
- **~10M+ scenes or long generated GLSL per scene:** the GIN index competes with your OLTP traffic for the same CPU, RAM and disk. Move search to a separate read replica first (Day 78 on pooling, the replica idea), then to a dedicated engine (Zoekt, or Tantivy/Meilisearch-style) fed by CDC from Postgres (Day 17). Index off the replica's WAL or a queue, never in the request path.
- **Do not index the editor's hot path.** The node-editor search ("add node") is a small in-memory list (Day 61 typeahead), not code search.

**One-line lesson for Rare.lab:** you already store code-like artifacts by content hash, so index each hash ONCE, search with a trigram index that is allowed to over-match, then verify exact matches on the few candidates, and make permission filtering part of the query, not a post-filter.

---

## 8. What is inside the video transcript you pasted (plain-language summary)

This is the same "Design LeetCode" mock interview summarized in [Day 74](074-leetcode-online-judge-end-to-end.md) Section 8. The recap of what it teaches as a **method** (since you said you want the whole structure for any product):

- **Minutes 0 to 4, functional requirements and out of scope.** Say what the product does in 3 or 4 verbs, and what you are NOT building (login, payments, analytics). This stops the interview, and real projects, from sprawling.
- **Minutes 4 to 11, non-functional requirements.** The interviewee chose availability over consistency and defended it with a concrete example (a user seeing 990 problems instead of 1,000 is fine). Also: low latency, isolation of untrusted code, scale (100k contest users, millions overall), no single point of failure.
- **Minutes 11 to 20, entities and API.** Nouns first (Problem, User, Submission, Leaderboard), then one REST route per requirement, with pagination even for 4,000 items.
- **Minutes 21 to 32, simplest design, then break it.** Client, API, database. Then the sandbox debate: run code on the API server (bad), VM (strong isolation, heavy), container (light, good utilization). The interviewer corrected one point: VMs actually isolate better; containers win on cost and elasticity.
- **Minutes 32 to 46, the leaderboard and scaling.** Partition submissions by competition ID, cache the leaderboard instead of polling the database, add a queue between API and workers to smooth spikes, retries with exponential backoff, primary plus replicas.
- **The closing advice:** study Hello Interview and the Gaurav Sen / Shrayansh Jain style YouTube channels (the transcript names "Shrian Jen" and "GKCS").

How today's lesson applies that method: same six steps, but the "break it" step is a 10-million-times gap in work per query, and the fix is an index plus a verify step instead of a sandbox plus a queue.

---

## 9. References and what is actually in them

**Tool limits this session:** the network proxy blocked direct page fetches (GitHub blog, swtch.com, Cursor blog, Simon Willison). I could only read **search-result excerpts**. So the facts below are from excerpts; anything I added from memory is marked "recalled". Open the links to confirm.

- [The technology behind GitHub's new code search (GitHub Engineering, Feb 2023)](https://github.blog/engineering/architecture-optimization/the-technology-behind-githubs-new-code-search/): the primary source. Excerpt confirms: engine called Blackbird, written in Rust, sharded ngram indexes, ngram index defined as an inverted index for substrings, Kafka in the indexing pipeline, query consistency as a design goal, trigrams too unselective on their own. Recalled but unverified: 115 TB / 15.5B docs / about 25 TB after dedup, sharding by blob hash.
- [Regular Expression Matching with a Trigram Index, or How Google Code Search Worked (Russ Cox, 2012)](https://swtch.com/~rsc/regexp/regexp4.html): the classic explanation. Excerpts confirm it describes Google Code Search as a trigram index plus a DoS-resistant regex engine, written after Cox's 2006 intern work on gsearch. The query-to-boolean-formula idea in Section 4.1 is from this post as I know it.
- [Google Code Search (Wikipedia)](https://en.wikipedia.org/wiki/Google_Code_Search) and [Trigram search (Wikipedia)](https://en.wikipedia.org/wiki/Trigram_search): short background on the Google product and the technique.
- [Zoekt design doc (Gerrit mirror)](https://gerrit.googlesource.com/zoekt/+/master%5E/doc/design.md) and [Sourcegraph: Tackling the long tail of tiny repos with shard merging](https://sourcegraph.com/blog/tackling-the-long-tail-of-tiny-repos-with-shard-merging): positional trigrams, shards flushed at about 100 MiB, mmap layout, index about 2 to 3 times source size, per-blob branch bitmasks, partial query evaluation to skip shards, regex limits.
- [Cursor: fast regex search](https://cursor.com/en-US/blog/fast-regex-search): a recent write-up on indexing for agent regex search. Only the title and result listing seen; not read.
- [How GitHub Indexes Code For Blazing Fast Search (scaleyourapp.com)](https://scaleyourapp.com/system-design-github-code-search-engine/): secondary case study. Excerpt claims 5,184 vCPUs, 40 TB RAM, 1.25 PB storage, 53 billion files. Treat as unverified secondary.
- [Hound (Etsy) via Changelog](https://changelog.com/posts/hound-lightning-fast-code-search-tool): an open-source tool based on Cox's trigram algorithm, a good one to read and run locally.
- [PostgreSQL pg_trgm docs](https://www.postgresql.org/docs/current/pgtrgm.html): the real extension used in Section 7 (GIN/GiST trigram indexes for `LIKE`/`ILIKE`). Stated from general knowledge; confirm there.

**Inference, labeled:** all capacity math (100 TB, 1 GB/s per core, 10 million times gap, fan-out and retry-budget numbers) is my back-of-envelope estimate. GitHub's internals beyond the published post are not public.

**Related ledger lessons:** Day 9 (queue as shock absorber), Day 10 (consistent hashing), Day 13 (backpressure and load shedding), Day 17 (WAL and CDC), Day 18 (search ranking), Day 21 (LSM trees), Day 23 (content-addressed storage), Day 28 (Bloom filters), Day 61 (typeahead), Day 74 (LeetCode end to end), Day 76 (tail latency and hedged requests), Day 86 (file sync and dedup).
