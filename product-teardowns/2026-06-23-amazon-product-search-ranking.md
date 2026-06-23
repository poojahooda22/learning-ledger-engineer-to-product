# Amazon Product Search: how typing "watch" returns the right ten things out of 350 million

Date: 2026-06-23
Product: Amazon
Feature: Product search ranking (the results page after you type a query)

## 1. The user

Meet Anjali. It is 11pm, she is on the sofa, phone in hand, and her dad's
birthday is in four days. She opens the Amazon app and types one word into the
box: `watch`. She is not browsing. She has a job, a deadline, and a vague
picture in her head of a leather-strap analog watch under 3,000 rupees. She
expects the first screen of results to already be close. If it is not, she will
add a word, or she will leave.

That one word is the entire interface. She did not pick a category, a brand, a
price band, or a strap material. She gave the system four letters and a tap, and
she expects it to read her mind in under a second.

## 2. The real problem

Here is the honest version of the pain. The word `watch` is a mess. It could
mean a wristwatch (analog, smart, kids', luxury). It could mean a wall clock.
It could mean a watch battery. It could mean a watch strap. It could mean the
video game "Watch Dogs". It could mean "watch" as a verb that leaked in from
someone searching for a movie. Amazon has more than 350 million items listed,
and a huge slice of them contain the token "watch" somewhere in the title,
description, or reviews.

So the problem is not "find products that contain the word watch". That returns
millions of items, which is useless. The problem is two problems stacked on top
of each other: first, narrow 350 million down to a few thousand plausible
candidates fast, and second, order those few thousand so the ten that Anjali
actually wants sit at the top. Get the first half wrong and the right product is
not even in the running. Get the second half wrong and it is on page 9, which
for a shopper is the same as not existing.

And it has to happen while roughly 60,000 other people are searching in the same
second.

## 3. The feature in one sentence

Amazon product search takes a short, ambiguous query and returns a ranked list
of items by splitting the work into two halves: matching (cheaply fetch a
candidate set from a catalog of hundreds of millions) and ranking (carefully
sort that candidate set using machine-learned models that lean heavily on what
shoppers actually clicked and bought).

## 4. Jobs to be done

What is Anjali really hiring this search box to do?

- "Read my vague intent from one word and show me the obvious good options first."
- "Do not make me scroll. Put the thing I would have bought on screen one."
- "Filter out the junk: the wall clocks and watch batteries when I clearly want a
  wristwatch."
- "Respect my budget and my taste without me having to spell them out."
- "Be fast enough that searching feels free, so I search five more times tonight."

Notice the job is not "match my keywords". It is "rank by how likely I am to be
happy with this purchase". That reframing is the whole game.

## 5. How it works for the user

Anjali types `watch` and the results appear almost instantly. The top of the
page is wristwatches, not wall clocks. There is a band of sponsored results, then
organic results: popular analog watches from brands like Titan and Fossil,
mostly in her rough price range, mostly with high review counts and four-plus
stars. Down the left (or behind a "Filters" button on mobile) are refinements:
brand, price, strap material, dial color. A "watch for men" and "watch for women"
nudge sits near the top.

If she now types `watch for men under 2000`, the list reshuffles in under a
second to analog and digital men's watches below 2,000 rupees. She never sees the
machinery. To her it feels like the box simply understood.

## 6. The actual flow, step by step

1. Anjali types `watch` and hits search.
2. The query hits Amazon's front end, which does query understanding: tokenize
   the text, lowercase it, correct obvious spelling, expand with known synonyms
   (so "watch" can also reach items tagged "timepiece" or "wristwatch"), and
   guess intent (this looks like a product-class query, not a specific ASIN).
3. The matching service takes those tokens and fetches a candidate set from the
   index. This is the "narrow 350 million to a few thousand" step. It returns
   candidates, not an order.
4. The ranking service scores each candidate with a machine-learned model and
   sorts them. This is the "put the best ten on top" step.
5. Business layers blend in sponsored placements, apply filters Anjali set,
   remove out-of-stock or ineligible items, and apply diversity rules so the page
   is not ten near-identical listings.
6. The page is returned and rendered. The sort already happened on Amazon's
   servers. Anjali's phone just draws the list it was handed; it does not sort
   anything.
7. Anjali taps the third result. That click, and whether she then buys, is logged
   and becomes training fuel for tomorrow's ranking. The loop closes.

## 7. Under the hood, like the engineer

This is the heart of it. Matching and ranking are two different machines with two
different cost profiles. Keep them separate in your head and everything else
falls into place.

### The matching half: an inverted index

You cannot scan 350 million product rows on every keystroke. So Amazon, like
every large search engine, precomputes an inverted index.

A normal (forward) index maps a product to its words: ASIN B07X -> {titan,
analog, leather, watch, men}. An inverted index flips that around. It maps each
word to the list of products that contain it:

```
"watch"   -> [B07X, B091, B0A2, B0KK, ... several million ASINs ]
"analog"  -> [B07X, B0A2, B0ZZ, ... ]
"leather" -> [B07X, B0LM, ... ]
"men"     -> [B07X, B091, ... ]
```

Each of those per-word lists is called a posting list. The classic data
structure is a hash map (term to posting list) where each posting list is a
sorted array of document ids, often delta-compressed (store the gaps between ids,
not the ids, because small numbers pack tighter).

When Anjali searches `watch for men`, the matcher does not look at 350 million
products. It grabs three posting lists ("watch", "men", and it drops the stop
word "for"), then intersects or unions them. Intersection of sorted arrays is a
linear merge: walk both lists with two pointers, keep the ids that appear in
both. The cost is proportional to the length of the posting lists for the words
you typed, not to the size of the whole catalog. That is the magic trick. Typing
a rare word like `tourbillon` is cheap because its posting list is short. Typing
`watch` is more expensive because its list is long, which is exactly why the next
stage uses pruning so it does not score all of them.

Each candidate gets a fast first-pass relevance score so the matcher can keep
only the top slice. The textbook score here is BM25, which rewards a product for
containing the query words and dampens the reward for very common words. A
production matcher keeps a small heap of the best K candidates (K is often around
1,000 to a few thousand) and uses dynamic pruning to skip posting-list entries
that provably cannot beat the current K-th best. The output of the matching half
is a candidate set of roughly a few thousand ASINs. The expensive ranker never
has to look at more than that.

### The gap that pure keywords leave, and the semantic patch

Keyword matching has a famous hole. If a shopper types `sneakers` but a great
product is titled "running shoes", a pure inverted-index match on the word
"sneakers" misses it. The words do not overlap even though the meaning does.

Amazon's fix, described in their KDD 2019 paper "Semantic Product Search" (Nigam
et al.), is a neural two-tower model. One tower reads the query text and produces
a vector. A second tower reads the product text and produces a vector in the same
space. The model is trained so that a query vector lands close to the vectors of
products people actually bought for that query, and far from random products and
from products that were shown but not bought. The paper uses a deliberately cheap
encoder (average the embeddings of the tokens rather than run a heavy recurrent
network) precisely because it has to run at Amazon's latency and scale, and it
tokenizes into unigrams, bigrams, and character trigrams so that misspellings and
unseen words still land somewhere sensible. Out-of-vocabulary tokens are hashed
into a fixed set of buckets so the vocabulary cannot blow up.

At serving time the product vectors are precomputed offline and loaded into an
approximate nearest neighbor index. The live query is encoded once into a vector,
and a nearest-neighbor lookup pulls in semantically close products that the
keyword matcher would have missed. So `sneakers` now also fetches "running
shoes". This semantic candidate set is merged with the lexical (inverted-index)
candidate set. Two matchers, lexical and semantic, feeding one candidate pool.

### The ranking half: gradient boosted trees on behavioral features

Now we have a few thousand candidates for `watch`. Ordering them is where Amazon
spends its intelligence. From their SIGIR 2016 paper "Amazon Search: The Joy of
Ranking Products" (Sorokina and Cantu-Paz), the main ranking model is gradient
boosted trees. They picked boosted trees because the trees discover complex
interactions between features on their own and work well without heavy tuning.
The model is trained with a pairwise objective, and the default thing it
optimizes is NDCG, a metric that rewards putting the items a shopper engaged with
near the top of the list rather than buried.

The single most important point in that paper: Amazon leans heavily on behavioral
features, because product text is often nearly identical across listings. Two
Titan watches can have almost the same title and description, so the words cannot
tell them apart. What tells them apart is behavior: for the query `watch`, how
often did shoppers click this exact ASIN, add it to cart, and buy it? Click-through
rate, purchase rate, conversion, review count and rating, price, and
availability carry the ranking. Text relevance gets you into the candidate set;
behavior decides the order.

Behavioral data has a nasty bias baked in. People click the top result more often
just because it is on top, not because it is better. If you train naively on
clicks, the model concludes "things at the top get clicked, so keep them at the
top", and it freezes. The paper describes correcting for this position bias, and
crucially adapting that correction day to day rather than using one fixed curve.
That daily refresh is also how a brand-new product, with no click history yet,
gets a fair chance to climb.

One more structural idea from the 2016 paper: Amazon does not run one global
ranker. Different categories have their own ranking functions (books behave
nothing like fashion, which behaves nothing like electronics), and a layer called
All Product Search blends those separate category rankings into the single list
you see when your query is not pinned to one category. The query `watch` spans
wristwatches, smartwatches (electronics), and watch straps (accessories), so the
blend matters.

### Walk the real query end to end

Anjali types `watch`.

- Query understanding: tokenize to ["watch"], it is a broad product-class query,
  expand with synonyms like "wristwatch" and "timepiece".
- Lexical match: grab the posting list for "watch" and friends, intersect with
  filters, BM25 first-pass score, keep the top few thousand via a pruned heap.
- Semantic match: encode "watch" to a vector, nearest-neighbor lookup pulls in
  items titled "analog timepiece" or "wrist watch" that the keyword pass might
  rank weakly. Merge into the candidate pool.
- Ranking: gradient boosted trees score each candidate using behavioral signals
  (this Titan analog watch converts well on the query "watch", has 12,000 ratings
  at 4.2 stars, is in stock, ships fast) plus relevance and price. The wall clock
  and the watch battery score low on purchase behavior for this query and sink.
- Business rules: insert sponsored band, drop out-of-stock, diversify so it is not
  ten identical Titans.
- Server sends the sorted page. Phone renders. Sort already done server-side.

The wall clock contained the word "watch" too. It lost not on words but on
behavior: almost nobody who types "watch" buys a wall clock.

### The scale story at three tiers

Tier 1, about 1,000 products. You do not need any of this. Load all 1,000 rows,
filter the ones containing "watch" with a simple scan, sort them in memory by a
hand-tuned score. A single machine answers in milliseconds. An inverted index
here is over-engineering.

Tier 2, about 100,000 products. The full scan per query now hurts, especially
under concurrent load. You build an inverted index so a query touches only the
posting lists for the typed words, not all 100,000 rows. You add a ranking model
because hand-tuned weights stop scaling to the variety of queries. This still
fits on one beefy box with the index in memory. What broke at the jump from tier
1: the linear scan. What saved you: the inverted index turning "search the
catalog" into "merge a few short lists".

Tier 3, 10 million and beyond (Amazon is past 350 million). Now several things
break at once and each needs a fix:

- The index no longer fits on one machine. Fix: shard the index across many
  machines, then scatter the query to all shards and gather the partial results
  (scatter-gather). Each shard finds its local best candidates; a coordinator
  merges them.
- Scoring every matched product with the heavy model is too slow. Fix: the
  two-stage split itself. Cheap matching narrows to a few thousand; only those
  few thousand reach the expensive gradient boosted trees. The ranker's cost is
  bounded by the candidate-set size, not the catalog size.
- 60,000 queries per second is brutal traffic. Fix: replicate the read path
  heavily (search is overwhelmingly reads), and cache. Popular queries like
  `watch`, `iphone`, `shoes` repeat constantly, so cache their candidate sets and
  even their ranked pages with a short time-to-live and serve most traffic from
  cache.
- All the heavy thinking is moved offline. Building the inverted index, training
  the boosted trees, computing product embeddings, and building the nearest
  neighbor index all happen in batch, off the hot path. The live query is a cheap
  lookup-and-sort over a small candidate set. This is the same pattern that shows
  up again and again in this ledger: precompute the expensive part, keep the
  request path cheap.

The deepest scale lesson is the two-stage architecture. Without matching, ranking
would have to score hundreds of millions of items per query, which is impossible
in a second. Without ranking, matching would return millions of items in no
useful order. Splitting the work is what makes the whole thing tractable, and the
candidate-set size (a few thousand) is the dial that decouples ranking cost from
catalog size.

What is public and what is inference: the matching/ranking split, gradient boosted
trees with a pairwise NDCG objective, the heavy use of behavioral features, the
daily position-bias correction, and per-category rankers blended in All Product
Search are stated in Amazon's own 2016 SIGIR paper. The two-tower semantic model
with cheap averaged token embeddings, character-trigram tokenization, OOV
hashing, and nearest-neighbor serving is from Amazon's 2019 KDD paper. The
specific sharding, caching, BM25 first-pass, and heap-pruning details are the
standard way this class of problem is solved at this scale and are my clearly
labeled inference for Amazon's exact current stack, not quoted internals. Amazon
has also layered newer systems on top (a commonsense knowledge graph called COSMO
for intent, and the Rufus conversational assistant), which are publicly announced
but whose serving internals are not detailed.

## 8. The retention and habit mechanic

The loop here is quieter than a Monday playlist drop, but it is stronger, because
it is the front door to the entire store. The mechanic is a self-reinforcing
flywheel:

1. Anjali searches `watch`, the ranking puts a good watch on top, she buys it.
2. That purchase is logged against the query `watch`.
3. Tomorrow's daily retrain sees that signal (position-bias corrected) and nudges
   that watch, and watches like it, slightly higher for that query.
4. The next shopper who types `watch` gets a marginally better first screen, is
   marginally more likely to buy, which feeds step 2 again.

Every search makes the next search better. The metric this primarily moves is
revenue: Amazon's own 2016 paper opens by saying search powers the majority of
Amazon's sales and that even small relevance improvements significantly impact
revenue. It moves retention too, through trust. When search reliably nails the
first screen, the box becomes a reflex. People do not browse to Amazon, they
search on Amazon, often skipping a general web search entirely. The real observed
example is the behavior of the market itself: a majority of US product searches
now start on Amazon rather than on a search engine, and Amazon fields on the order
of 60,000 searches every second. That habit is the moat. The ranking quality is
what built it.

## 9. The lesson for Rare.lab

Rare.lab has a search surface hiding in plain sight: the node and effect library,
and the template gallery. A creator types "glow" or "water ripple" or "heat
haze" into the node search and expects the right node or starter graph
immediately. Treat that exactly like product search, with the same two-stage
split, and bias every decision toward keeping the hot path cheap.

Concretely:

- Build the cheap matcher offline. Index every node, effect, and template by
  name, tags, and description in an inverted index, and also precompute a semantic
  embedding for each one (so "water ripple" finds an effect tagged "caustics" or
  "wave distortion" even with no word overlap, the sneakers-equals-running-shoes
  problem). At query time do a cheap candidate fetch from both, never a scan of
  the whole library.

- Rank the shortlist by behavior, not by name match. The killer insight from
  Amazon is that text barely separates near-identical items; behavior does. So
  rank the candidate nodes and templates by what creators actually insert and,
  more importantly, keep in their graph and ship, not by how well the name
  matches. A node that everyone drags in and then deletes should sink even if its
  name is a perfect match. Log insert, keep, and compile-to-ship events and feed
  them back, with the same position-bias correction so a newly published
  community node still gets a fair shot at the top.

- Keep the expensive part off the keystroke path. For Rare.lab the expensive
  thing is not a boosted-tree score, it is compiling and previewing a shader. Do
  not compile candidates to rank them. Precompute thumbnails, embeddings, and
  cost estimates offline in batch, and let the live search be a lookup and a small
  sort over a few hundred candidates. The runtime stays responsive because the
  catalog size never touches the request path. That is the same lever (candidate
  set size decouples ranking cost from library size) that lets Amazon rank inside
  350 million items in under a second.

The one-line version: split matching from ranking, precompute both halves
offline, rank the shortlist by what creators keep rather than what the name says,
and the search box stays fast no matter how big the library grows.

## Sources

- Daria Sorokina and Erick Cantu-Paz, "Amazon Search: The Joy of Ranking Products", SIGIR 2016 (Amazon Science): https://www.amazon.science/publications/amazon-search-the-joy-of-ranking-products
- ACM Digital Library entry for the same paper: https://dl.acm.org/doi/10.1145/2911451.2926725
- Priyanka Nigam et al., "Semantic Product Search", ACM SIGKDD (KDD) 2019: https://dl.acm.org/doi/10.1145/3292500.3330759 and PDF: https://arxiv.org/pdf/1907.00937
- "Exploring Query Understanding for Amazon Product Search" (2024): https://www.arxiv.org/pdf/2408.02215
- Amazon A9 algorithm overview (history of A9.com, matching vs ranking, COSMO and Rufus layers): https://www.amalytix.com/en/knowledge/seo/amazon-alogrithm-a9/ and https://feedvisor.com/university/a9-search-engine/
- Background on inverted indexes, posting lists, BM25, and dynamic pruning at scale: https://www.systemoverflow.com/learn/search-ranking/ranking-algorithms/bm25-implementation-inverted-index-and-dynamic-pruning-at-scale and https://milvus.io/ai-quick-reference/how-does-an-inverted-index-work
- Amazon search scale figures (350M+ products, ~60,000 searches/second, share of product searches starting on Amazon): https://wifitalents.com/amazon-search-statistics/ and https://www.emarketer.com/content/do-most-searchers-really-start-on-amazon
