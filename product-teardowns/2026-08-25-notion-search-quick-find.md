# Notion Search: Quick Find, and the permission-aware retrieval underneath it

Date: 2026-08-25
Product: Notion
Feature: Search (Quick Find keyword search and AI Q&A semantic search), and the permission-aware index that makes both safe

A note on scope. Notion has two different search boxes wearing one coat. Quick Find (Cmd+P) is old-school keyword search: you type "Q3 roadmap" and it finds pages with those words. AI Q&A is the newer one: you ask "what did we decide about the pricing change?" and it answers in a sentence with citations. They run on two completely different engines (Elasticsearch for one, a vector database for the other), but they share the single hardest problem in the whole feature: never show a person a result they are not allowed to open. This teardown walks both, and spends most of its time on that shared problem, because that is where the real engineering lives.

---

## 1. The user

Meet Arjun. He is a product manager at a 400-person startup that runs its entire company brain in Notion: engineering docs, HR policies, board decks, the sales team's account notes, the design team's specs. His own workspace has grown to something like 50,000 pages over three years. He remembers writing a doc about the Q3 pricing change back in June. He does not remember what he called it, which teamspace it lives in, or who he shared it with.

It is 9:40am. He is on a call. Someone asks, "wait, did we agree to grandfather existing customers on the old price?" He has fifteen seconds before it gets awkward. He hits Cmd+P and types "grandfather pricing."

That is the moment. Fifty thousand pages, one fuzzy memory, fifteen seconds, and one very important constraint he never thinks about: the search must not surface the CEO's private comp-planning page that happens to also say "grandfather" and "pricing," because Arjun was never given access to it.

---

## 2. The real problem

Told like a friend would tell it: your Notion is a filing cabinet that grew into a warehouse, and you lost the map. Everything is in there somewhere. Getting to it by clicking through the sidebar is like walking the warehouse aisle by aisle. Search is supposed to be the forklift that drives straight to the shelf.

But a company Notion is not one person's warehouse. It is a shared building where every room has a different lock. Arjun can walk into the product wing but not the HR wing. The search forklift has to know, for every single shelf, whether Arjun is allowed to stand in front of it, and it has to know that in the fraction of a second between his keystroke and the results appearing. Get that wrong in the safe direction and Arjun cannot find his own doc. Get it wrong in the unsafe direction and a search box just leaked salary data across the company.

That second failure is not hypothetical fear. It is the reason enterprises trust or distrust a wiki. One leaked snippet in a search preview and the security review is over.

---

## 3. The feature in one sentence

Type a few words or ask a full question, and Notion returns the pages you are allowed to see, ranked by how likely they are the one you meant, fast enough to feel instant.

---

## 4. Jobs to be done

What is Arjun really hiring search to do?

- "Take me back to the doc I know exists but cannot name." (Refinding. The most common job by far.)
- "Tell me what we decided, without making me read five docs." (Answering. This is the AI Q&A job.)
- "Show me only what I am allowed to see, and never make me think about that." (Trust. Invisible when it works, fatal when it fails.)
- "Do it before I lose my train of thought." (Speed. Under a second or it breaks the flow.)

Notice that trust is a job. Users do not phrase it, but they feel it the instant it breaks. A search that shows a forbidden result is not a search bug, it is a data breach with a text box.

---

## 5. How it works for the user

Two doors.

Door one, Quick Find. Arjun hits Cmd+P. A box drops down. He types "grandfather pricing." As he types, a list forms under the box: page titles first, then pages where those words appear in the body, each with the teamspace it lives in and a tiny snippet showing the matching text. He arrows down to "Q3 Pricing Change: Migration Plan," hits Enter, and he is there. Total time: about four seconds.

Door two, AI Q&A. Instead of a page, Arjun wants an answer. He asks, "did we agree to grandfather existing customers on the old price?" A few seconds later he gets a written answer: "Yes. Per the Q3 Pricing Change migration plan (dated June 12), existing customers stay on the current price for 12 months," with a clickable citation to the source page. The answer was assembled by reading the actual pages, not guessed.

In both doors, the pages he cannot access simply are not there. No "you don't have permission" row. No greyed-out teaser. They do not exist as far as his search is concerned. That silence is the whole trust mechanic.

---

## 6. The actual flow, step by step

Quick Find, tap by tap:

1. Arjun presses Cmd+P. The client opens the search overlay instantly (it is just UI, no server call yet).
2. He types "gra." The client waits a beat (debounce, roughly 150 to 200ms) so it is not firing a query on every single letter.
3. On "grandfather," the client sends one request to Notion's search service: the query text, the workspace id, and Arjun's user id.
4. The server does the real work (Section 7) and returns a small ranked list, maybe 20 results, already filtered to what Arjun can see and already sorted. The phone or browser does no ranking.
5. The client paints the list. Titles that match are shown above body matches. Arjun refines to "grandfather pricing," which narrows it.
6. He clicks a result. The client opens that page by its block id, a direct keyed lookup, no search involved.

AI Q&A, tap by tap:

1. Arjun types a full question and hits Enter.
2. The server turns his question into a vector (an embedding, a list of about 1,000 numbers that captures meaning, not spelling).
3. It searches a vector database for the chunks of text whose vectors sit closest to the question's vector, filtered to only chunks Arjun can access.
4. It takes the top chunks, pastes them into a prompt along with the question, and asks a language model to answer using only those chunks.
5. It streams back the answer with citations pointing at the source pages.

Two different machines behind two different doors. Now open the hood.

---

## 7. Under the hood, like the engineer

### 7a. First, the thing being searched: the block tree

Everything in Notion is a block. A page is a block. A paragraph inside it is a block. A to-do, a heading, a table row, all blocks. Each block is a small record: a UUID, a type, a properties hash map (its text and settings), an ordered array of child block ids, and a parent pointer. This is the data model the earlier teardowns in this ledger covered (the slash-command block model on 2026-06-25, databases on 2026-08-02). Blocks live in Postgres, partitioned by workspace id across 480 logical shards, so any single-workspace operation touches one shard.

The critical fact for search is how permissions work on this tree. You do not grant access block by block. You share a top page with someone, and every block underneath inherits that access. Arjun's "Q3 Pricing Change" page has 300 blocks under it; sharing the page with the pricing teamspace grants all 300 in one act. Permission is inherited down the parent pointers, exactly like file permissions inherit down folders.

Hold that thought. It is the source of both the elegance and the pain in this whole feature.

### 7b. Search is two halves: matching, then ranking

Every search teardown in this ledger has hammered the same spine, and Notion is no exception. Finding results is two different jobs stapled together.

- Matching (also called retrieval or candidate generation): out of billions of blocks, cheaply grab the few hundred that could possibly be relevant. This half must be fast and wide.
- Ranking: take those few hundred and carefully sort them so the one Arjun wants is at the top. This half can be slower because it only touches a small set.

Matching a query against billions of blocks by reading each one is impossible. So, like every real search engine, Notion does not store the data in a way you read row by row. It stores an inverted index.

### 7c. The inverted index, with a real word

A normal index answers "what words are in document 5?" An inverted index answers the question search actually asks: "which documents contain the word 'grandfather'?" It is a hash map from each word to a posting list, the sorted list of every document id that contains that word.

```
"grandfather" -> [doc_18, doc_204, doc_50011, ...]
"pricing"     -> [doc_12, doc_204, doc_888, doc_50011, ...]
```

To answer "grandfather pricing," you fetch the two posting lists and intersect them. Documents 204 and 50011 are in both. The cost of this is not the size of the catalog, it is the length of the posting lists for the words you typed. Rare words like "grandfather" have short lists, so the query is cheap no matter how big Notion gets. That is the single most important scaling property in all of search: cost tracks the query, not the corpus.

Notion runs this on Elasticsearch, which is a search server built around exactly this inverted-index structure (Lucene underneath). Every block's text is turned into an Elasticsearch document and indexed. When Arjun searches, Elasticsearch intersects posting lists, scores the survivors with a relevance formula (BM25, which rewards rare matching words and penalizes ones that appear everywhere), and returns the top few. Title matches get boosted above body matches. The sort happens server-side, inside Elasticsearch, never on Arjun's laptop.

### 7d. The permission problem, and why it is the hard half

Now the twist that makes Notion search different from a plain document search. Elasticsearch will happily find every block containing "grandfather" and "pricing." Some of those blocks are in the CEO's private comp doc. Arjun must never see them. How do you filter them out?

The naive answer: after Elasticsearch returns 200 candidates, loop over each one and check "can Arjun see this?" But recall Section 7a: checking access means walking up the parent pointers to find which top page granted access, then checking if Arjun is on that grant. That is a tree walk per candidate, a burst of database reads, on the hot path, while Arjun waits. For 200 candidates across shards, that is far too slow. And it gets worse under a common query where thousands of blocks match.

Notion's actual approach, confirmed in their engineering write-up on rebuilding the search reindexer, is to move that work out of the query and into indexing time. When a block is turned into an Elasticsearch document, the pipeline does the tree traversal then, once, and bakes the resolved permission set directly into the document as a field. In Notion's own words, the document-generation logic "encoded years of edge cases around per-document permission resolution, deletion semantics, and field formatting," and the transformation step performs "tree traversal and permission data construction for each block."

So every indexed document already carries "who can see this." At query time the filter becomes a cheap metadata condition inside the same Elasticsearch query: intersect the "grandfather" and "pricing" posting lists AND require that the document's allowed-scope field includes one of Arjun's access groups. No tree walk at read time. The expensive thinking was done offline; the live path is one keyed, filtered lookup. This is the offline-think, online-lookup pattern this ledger keeps finding, applied to authorization instead of ranking.

The cost did not vanish, it moved. It moved to the write side, and it moved to invalidation. When someone shares that top pricing page with a new teamspace, the access of all 300 descendant blocks just changed, which means up to 300 Elasticsearch documents need their permission field rewritten. Share a page with 10,000 blocks under it and you have 10,000 documents to re-stamp. This is the classic tradeoff: denormalizing permission onto every block makes reads fast and makes writes fan out. Notion accepts that trade because reads (searches) vastly outnumber permission changes.

### 7e. Keeping the index fresh: two pipelines

An index is only useful if it reflects what the docs say right now. Arjun edits a heading, and he expects to find it by the new heading seconds later. Notion runs two indexing paths, and understanding why there are two is half the lesson.

Path one, online, for the steady drip of edits. Every edit in Notion flows through the data lake Notion built (Debezium change-data-capture reading the Postgres write-ahead log, into Kafka, into Hudi tables on S3, with Spark on top; this is the CDC pipeline from the databases teardown on 2026-08-02). Kafka consumers pick up each block change in near real time, regenerate that block's Elasticsearch document (including re-resolving its permissions), and write it to the index. Sub-minute freshness for a single edit.

Path two, offline, for full rebuilds. Sometimes you cannot patch, you have to rebuild the entire index from scratch: you added a new searchable object type (Notion mentions making Custom Agents searchable), you changed how text is tokenized (say, better handling of Japanese or Arabic that does not split on spaces), you upgraded Elasticsearch, or you launched in a new region and need a fresh index there. A full rebuild reprocesses every block Notion has ever stored.

That rebuild is where Notion's most concrete engineering story sits, and the numbers are worth stating exactly.

### 7f. The reindexer rebuild, with real before-and-after numbers

Notion published "Rebuilding Notion's lexical search reindexer." The old full-rebuild system worked like this: Spark read the data-lake mirror of every block and wrote Apache Avro files. Then a fleet of ECS Fargate workers running Node.js consumed those Avro files, queried Snowflake for extra metadata the Avro files lacked, and wrote to Elasticsearch through its indexing API. On top of that, a separate ECS pipeline consumed Kafka changes to catch up on edits that happened during the multi-day build.

What that cost, in their reported figures:

- A full reindex took over 2 weeks.
- It reached only about 90% data consistency (some documents silently wrong or missing).
- The catchup step took around 2 days.
- It needed constant manual babysitting and on-call attention.

They rebuilt it as an Apache Spark-native pipeline that writes Elasticsearch's own snapshot format directly and uses Elasticsearch's native primitives to catch up, removing the separate ECS catchup fleet, the Node.js workers, and the live Snowflake dependency on the hot path. The reported results:

- Full reindex time: from over 2 weeks to under 2 days.
- Catchup time: from about 2 days to under 1 hour.
- Consistency: from about 90% to 100% document consistency.
- Manual effort: from weeks of engineering time to under 2 hours, with zero on-call pages.

The lesson buried in those numbers: the old pipeline's pain came from external dependencies on the critical path (Snowflake lookups, a chatty indexing API, a second catchup system). Every external hop was a place to be slow, to fail, to drift out of consistency. The fix was to remove the hops, produce the index format directly, and let one engine (Spark producing snapshots, Elasticsearch restoring them) own the whole flow.

### 7g. The other engine: AI Q&A and vector search

Quick Find matches spelling. It cannot answer "did we decide to grandfather customers?" if the doc says "existing accounts keep legacy rates," because none of the words overlap. That is the meaning gap, and it is why AI Q&A uses a different retrieval engine entirely: vector search.

Every chunk of every page is run through an embedding model that turns text into a vector, roughly 1,000 numbers positioning that text in a meaning-space. "Grandfather existing customers on the old price" and "existing accounts keep legacy rates" land near each other in that space even though they share no words. To answer Arjun's question, you embed the question and fetch the nearest chunk vectors. That is approximate nearest neighbor (ANN) search, the same primitive behind Spotify Autoplay (2026-08-16) and YouTube recommendations (2026-06-22), pointed at documents instead of songs.

Then the same permission problem returns, now on vectors. A nearest-neighbor result the user cannot see is the same breach as before. So the permission set is stored as metadata on every vector, and the ANN query filters by it: find the nearest chunks whose access metadata includes Arjun's groups. Same denormalize-at-write, filter-at-read shape as the lexical side.

The retrieved chunks are pasted into a prompt with the question and handed to a language model, which writes the answer and cites the chunks. This is retrieval-augmented generation (RAG): the model does not "know" Notion's content, it is fed the exact relevant, permission-filtered snippets at answer time and told to stick to them. That is also why it can cite sources and why it does not leak: it can only speak about what retrieval was allowed to hand it.

### 7h. The vector-search scaling story, with real numbers

Notion published "Two years of vector search at Notion: 10x scale, 1/10th cost," and it is a clean scale-breaks-then-gets-fixed narrative.

Notion launched AI Q&A in November 2023. Almost immediately a waitlist of millions of workspaces piled up. The first architecture used pod clusters, storage and compute bundled together, sharded by workspace id. Within about a month they were hitting capacity limits. Their first fix was pragmatic and worth stealing: rather than repartition everything, they spun up new index clusters tagged with a generation id and routed new workspaces to fresh generations, leaving existing workspaces where they were. New load goes to new capacity; old data does not move. It bought time.

The deeper fix, committed to in late 2024, was to migrate the entire multi-billion-object workload to Turbopuffer, a serverless vector database built on object storage. This matters for cost. Turbopuffer keeps the durable copy of vectors in cheap object storage (S3, GCS, or Azure Blob) and pulls data into NVMe SSD and RAM cache only when a workspace is actually queried. Its vector index is SPFresh, a centroid-based ANN index (group vectors into clusters, keep a small list of cluster centers, at query time download the centroids, find the closest few clusters, then fetch just those clusters). That takes only two to four round trips to object storage, which is why it can serve competitive latency (sub-10ms p50 in Turbopuffer's own benchmarks, 200ms p99 over 100 billion vectors) without keeping every vector in expensive RAM. A cold, idle workspace costs almost nothing to store; a hot one pays for cache only while it is hot.

They also fixed the embedding side. Notion built page-state caching so they do not re-embed text that did not change. When a page updates, they chunk it and compare hashes: a text hash and a metadata hash per chunk. If the text hashes are identical but the metadata hash differs, that means only the permissions changed, not the words. Re-embedding is expensive (it runs the chunk through a neural model); rewriting metadata is cheap. So in that case they skip the embedding entirely and just issue a PATCH to update the permission metadata on the existing vector. Given how often permission changes fan out across thousands of blocks (Section 7d), skipping re-embed on those is a huge saving. Embeddings generation and serving were moved to Ray on Anyscale.

The reported outcome:

- Cleared a multi-million-workspace waitlist.
- Vector database cost down about 60%.
- Embeddings infrastructure cost down over 90% (the "1/10th cost" in the title).
- Query latency improved from 70 to 100ms down to 50 to 70ms.
- Supported roughly 15x growth in active workspaces.

### 7i. The scale story at three tiers

Tier one, 1,000 blocks (a solo user's Notion, or a tiny team). Honestly, search barely needs to be clever here. Elasticsearch on one node handles it, the vector set fits in memory, and you could even brute-force scan every vector for the nearest neighbor. Permission filtering is trivial because there are few grants. Nothing breaks. Most Notion workspaces live here and never stress a thing.

Tier two, 100,000 blocks (Arjun's 400-person company). Now the naive approaches break. Re-checking permissions by tree walk at query time becomes visibly slow, which is exactly why permission must be denormalized onto each document at index time. Full reindexes stop being instant, so you need the online Kafka path to keep freshness between rebuilds. A single Elasticsearch index and a single-generation vector pod still cope, but you feel the edges: a big permission change now rewrites thousands of documents, and you notice the write fan-out for the first time.

Tier three, billions of blocks across millions of workspaces (all of Notion). Everything that was a convenience becomes a requirement. Shard by workspace id so any one workspace's search touches one shard and workspaces scale horizontally and independently (the same partition-by-tenant key as Stripe-by-account and Amazon-by-ASIN in this ledger). The old two-week ECS reindexer collapses under its own external dependencies and gets replaced by the Spark-native snapshot pipeline. The bundled-pod vector database hits capacity in a month and gets escaped first with generation-based routing, then with serverless object-storage vectors so cold workspaces cost nearly nothing. Re-embedding every changed chunk becomes unaffordable, so page-state hashing skips the ones that only changed permissions. The through-line at this tier: separate cold storage cost from hot query cost, do expensive work once at write time, and make the read path a cheap filtered lookup.

Where I am citing Notion's own posts (the reindexer numbers, the vector-search numbers, Turbopuffer, Ray, the hashing trick, generation routing) that is confirmed fact. Where I describe the inverted index, BM25 scoring, and the exact shape of the permission-filter query, that is the standard way this class of system is built, matched to what Notion has said publicly; treat those specifics as well-grounded inference, not internal confirmation.

---

## 8. The retention and habit mechanic

Search is not a feature you open the app to use. Nobody says "let me go enjoy Notion search." So its retention role is different from a feed or a recommendation. Search is a switching-cost engine.

Here is the loop. The more Arjun puts into Notion, the more he can find in Notion, the more he trusts Notion as the place his answers live, so the more he puts into Notion. Every successful search is a tiny deposit of trust that says "yes, it is all in here, and I can get it back." That compounding is what turns a note-taking app into a company's memory, and a company's memory does not churn. You do not casually migrate three years and 50,000 pages of findable, permission-correct content to a competitor.

Which metric does it move? Retention, and specifically enterprise retention, the stickiest kind. It barely touches activation (new users have little to search) and it is not direct revenue. But it is the quiet reason big accounts renew. This is why Notion invested in AI Q&A and cleared a multi-million-workspace waitlist rather than letting it sit: answering "what did we decide?" in one sentence deepens the dependence more than any keyword match can. A real observed example of the mechanic is Notion's own framing of enterprise search: the selling point is not just that you find things, it is that you find only what you are allowed to find, because a single leaked search result would break the trust the whole loop is built on. Trust is the retention product; search is how it is delivered.

The failure direction proves the mechanic too. The most common complaint about Notion historically was "search sucks, I can never find my stuff." Every failed search is a withdrawal from the trust account, and enough of them and a team starts keeping the "real" doc somewhere else, which is the first step out the door. That is why a two-week, 90%-consistent reindexer was worth rebuilding into a two-day, 100%-consistent one: the missing 10% were failed searches, and failed searches quietly erode the exact loop that keeps the account.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus an embeddable runtime. The one concrete lesson from Notion search, biased toward scalability and performance:

**Bake the expensive access-and-context resolution into the index at write time, and make every read a cheap filtered lookup. Never resolve a tree or a graph on the hot path.**

Concretely, Rare.lab will grow a searchable library of effects, nodes, presets, and user projects. When a creator searches "water caustics" or asks "which of my graphs use the bloom node," you will be tempted to answer by walking the node graph or the team's sharing tree at query time to figure out relevance and who is allowed to see what. Do not. That is Notion's naive per-candidate tree walk, and it dies at the second tier.

Instead, copy the exact pattern:

1. At save time, for every effect and node, resolve once and denormalize into a search document: its tags, the device classes it compiles for, its dependency signature, and, critically, its visibility set (which team or which sharing scope can see it). The tree walk happens once, on write, off the hot path.
2. At search time, run one filtered index query: match the terms AND require the visibility field to include the viewer's scope. No graph traversal, no permission walk, constant-ish cost per query regardless of how big the library grows.
3. When something high in the tree changes (a creator makes a whole folder of effects private, or bumps a shared sub-graph everyone depends on), fan out the invalidation on the write side, and steal Notion's hashing trick to make it cheap: hash the content separately from the metadata. If only the visibility changed and the effect's compiled output did not, do not recompile or re-embed anything, just PATCH the visibility field on the existing index entry. Recompiling a shader variant is your expensive embedding; rewriting one metadata field is your cheap PATCH. Given that a single sharing change can touch hundreds of dependent nodes, this is the difference between a snappy toggle and a multi-second stall.
4. For semantic search over effects ("something that looks like heat shimmer"), store the vectors on cheap object storage and pull hot ones into cache, the Turbopuffer shape, so an idle creator's library costs almost nothing to keep searchable while an active one pays only for what it queries. Cold storage cost and hot query cost are two different budgets; do not pay RAM prices for a library nobody is searching this minute.

The single sentence to carry: resolve context and permission once at write time, invalidate by event with a content-vs-metadata hash so you only redo the part that actually changed, and keep the runtime query a cheap filtered lookup that never walks the graph. That is how a search stays instant from 1,000 effects to 10 million.

---

## Sources

- Notion Engineering, "Rebuilding Notion's lexical search reindexer." https://www.notion.com/blog/rebuilding-notions-lexical-search-reindexer
- Notion Engineering, "Two years of vector search at Notion: 10x scale, 1/10th cost." https://www.notion.com/blog/two-years-of-vector-search-at-notion
- Notion Engineering, "Building and scaling Notion's data lake." https://www.notion.com/blog/building-and-scaling-notions-data-lake
- Notion, "Search for pages & content in Notion" (Help Center). https://www.notion.com/help/search
- Notion, "Enterprise Search security & privacy" (Help Center). https://www.notion.com/help/enterprise-search-security-and-privacy-practices
- ZenML LLMOps Database, "Notion: Scaling Vector Search Infrastructure for AI-Powered Workspace Search." https://www.zenml.io/llmops-database/scaling-vector-search-infrastructure-for-ai-powered-workspace-search
- ZenML LLMOps Database, "Notion: Rebuilding a Production Search Reindexing Pipeline at Scale." https://www.zenml.io/llmops-database/rebuilding-a-production-search-reindexing-pipeline-at-scale
- Turbopuffer, "Architecture." https://turbopuffer.com/docs/architecture
- Turbopuffer, "ANN v3: 200ms p99 query latency over 100 billion vectors." https://turbopuffer.com/blog/ann-v3
- Turbopuffer, "fast search on object storage." https://turbopuffer.com/blog/turbopuffer
- Zilliz blog, "Notion Vector Search Architecture: What Comes Next." https://zilliz.com/blog/notion-vector-search-next-problem
- Elastic, "Elastic Notion Connector reference." https://www.elastic.co/docs/reference/search-connectors/es-connectors-notion
