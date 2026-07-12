# Amazon: "Customers who bought this item also bought" (item-to-item collaborative filtering)

Date: 2026-07-12
Product: Amazon
Feature: The recommendation strip under a product ("Customers who bought this item also bought" / "Frequently bought together" / the personalized homepage)

---

## 1. The user

Meet Ananya. It is a Tuesday night. She is on the Amazon app looking at one
specific thing: a "Prestige clip-on gas stove lighter," the little spark gun you
click to light a burner. Hers just died. She scrolls past the price, past the
photos, past the reviews, and without deciding to, her eyes land on a row lower
down the page: **Customers who bought this item also bought.** In that row is a
pack of two lighters, a set of steel kitchen tongs, and a silicone spatula.

She was not going to buy tongs tonight. She adds them anyway.

That row is the feature. It is one of the oldest, quietest, most profitable
pieces of software on the internet, and almost nobody notices it is there. It is
also the row that trained a generation of engineers on one idea: do the
expensive thinking offline, serve the live path as a cheap lookup.

## 2. The real problem

Amazon has a shelf problem that a physical shop never has.

A corner kirana store has maybe 2,000 items and a shopkeeper who knows you. He
sees you buy a lighter and says "tongs bhi le lo, purani toh ghis gayi hogi." He
is a recommendation engine made of one human brain and a small shelf.

Amazon in 2003 had roughly **29 million customers and several million catalog
items** (the exact figure in the original paper). Today it is hundreds of
millions of buyers and hundreds of millions of items. There is no shopkeeper.
There is no shelf you can see all at once. If Amazon shows you a random grid of
products, you scroll past it forever and buy the one thing you came for. That is
a lost sale on every single page view, multiplied by billions of page views.

The real problem: **out of a catalog of millions, pick the 6 to 20 things THIS
person is most likely to want next, and pick them fast enough to render inside a
product page, for every person, on every page, forever.**

The naive answer ("find people similar to Ananya, see what they bought") is the
obvious one. It is also the one that quietly explodes. That explosion is the
whole story.

## 3. The feature in one sentence

For any item you are looking at (or anything you have ever bought), Amazon shows
a short row of the items most often bought by the same people, drawn from a
precomputed table so the live page is a cheap lookup instead of a live search
over millions of products.

## 4. Jobs to be done

What is Ananya really hiring this row to do?

- **"Finish my cart for me."** She came for a lighter. She did not think about
  the tongs. The row does the remembering.
- **"Reassure me I picked a normal thing."** Seeing what other lighter-buyers
  bought is a soft signal that she is in a sane aisle, not a weird one.
- **"Save me a search I did not know I needed."** She would never type "silicone
  spatula" tonight. The row surfaces it without a query.

And the job Amazon is hiring it to do: **turn one intended purchase into two or
three, on a page the user was already going to load anyway.** The traffic is
free. The row monetizes attention that already exists.

## 5. How it works for the user

The visible experience is almost nothing, and that is the point.

- On a product page you see one or more horizontal rows: "Customers who bought
  this item also bought," "Products related to this item," "Frequently bought
  together" (with checkboxes and a bundled "Add all three to cart" button).
- On the homepage, logged in, you see "Recommended for you, Ananya," "Inspired by
  your browsing history," "Buy again."
- In email and notifications, the same rows arrive as "We thought you might
  like."

There is no spinner. The row is just there, instantly, as part of the page. That
instant-ness is the engineering achievement hiding in plain sight.

## 6. The actual flow, step by step

1. Ananya taps the lighter listing. The app requests the product page for that
   ASIN (Amazon's item id).
2. The page assembles from many services in parallel: price, images, reviews,
   buy-box, and a call to the **recommendations service** with two inputs: the
   ASIN she is viewing, and her customer id.
3. The recommendations service does NOT go compute anything heavy. It reads a
   **precomputed "similar items" list for that ASIN** (a short, ranked row of
   item ids), and separately reads her purchase and view history to personalize
   the homepage rows.
4. It filters: drop items she already owns, drop out-of-stock, drop the current
   item itself, apply any category or region rules.
5. It returns 6 to 20 item ids. The page fetches their titles, prices, thumbnails
   and renders the row.
6. Ananya adds the tongs. That event ("customer C bought I1 and I2 in the same
   basket") flows into the logs that will, offline and later, update the similar
   items table for next time.

Steps 3 to 5 happen in a few milliseconds because the hard part already happened
last night, on a cluster, with nobody watching.

## 7. Under the hood, like the engineer

This is the heart of it. The feature is a textbook case of splitting a problem
into **matching** (which items are related at all) and then **ranking and
filtering** (which of those to show this person, in what order), and of pushing
all the expensive work **offline** so the live path is O(1)-ish.

### The obvious approach, and why it breaks (user-based collaborative filtering)

The intuitive recommender is **user-based**: to recommend for Ananya, find the
customers most similar to Ananya (people who bought a similar set of things),
then recommend what they bought that she has not.

Model it honestly. Represent each customer as a giant sparse vector over the
catalog: a 1 in each slot for an item they bought, 0 everywhere else. Ananya is
one such vector in a space with **millions of dimensions** (one per catalog
item). "Similar customer" means small angle between two of these vectors, that is
cosine similarity.

The killer: to find Ananya's neighbors you must compare her vector against **all
M customers**, and M is tens of millions. Worst case that is O(M) work per
recommendation over vectors of length N (millions), done live, while she waits
for a page. You can sample customers or cluster them first, but:

- **Sampling** throws away exactly the rare, long-tail overlaps that make a
  recommendation feel magic.
- **Cluster models** (group customers into segments offline, recommend the
  segment's popular items) scale better but the recommendations get coarse and
  generic, because Ananya is now "kitchen-buyer segment 47," not herself.

The 2003 paper's judgment, stated plainly: user-based collaborative filtering
"does not scale" to tens of millions of customers and millions of items, and the
cheaper variants "reduce recommendation quality." Search/content-based methods
(recommend items with similar text or attributes) miss the whole point, because
what makes tongs relate to a lighter is not shared words, it is shared baskets.

### The move that changed everything: flip the axis to item-to-item

Here is the pivot. Instead of asking "which customers are like Ananya," ask
**"which items are like the items Ananya already touched."**

Represent each **item** as a sparse vector, but now the vector is **over
customers**: the lighter's vector has a 1 for every customer who bought the
lighter. The tongs' vector has a 1 for every customer who bought the tongs. Two
items are "similar" when the same people bought both, measured again by cosine
similarity between their two customer-vectors.

Why this flip is the whole win:

- **There are far fewer meaningful item-item relationships than customer-customer
  ones you must check live**, and more importantly, **item-item similarity is
  stable.** The set of people who buy lighters changes slowly. Ananya's own
  tastes change fast. So you can compute the item-to-item table **offline, once
  (refreshed periodically), and reuse it for everyone.**
- The live recommendation for Ananya becomes: take the handful of items she
  bought or viewed, **look up each one's precomputed "similar items" row**,
  merge and rank those rows, filter. The cost depends only on **how many items
  Ananya has interacted with** (a few dozen), and **not at all on the catalog
  size N or the customer count M.** That is the sentence every engineer should
  tattoo somewhere.

### Building the similar-items table (the offline half)

The core offline algorithm from the paper, in plain steps:

```
For each item I1 in the catalog:
  For each customer C who bought I1:
    For each other item I2 that C also bought:
      record a co-occurrence (I1, I2)
For each co-occurring pair (I1, I2):
  compute similarity(I1, I2)   // cosine over their customer-vectors
```

The data structures in play:

- A **hash map keyed by item id**, whose value is that item's posting-style list
  of co-purchased items with similarity scores. This is essentially an
  **inverted index from item to its neighbors**, the same shape as a search
  engine's term-to-documents index, just item-to-items.
- **Sparse vectors** for the cosine math, because the customer-by-item matrix is
  almost all zeros (Ananya bought maybe 50 of 300 million items). You never
  materialize the dense N by M matrix. You only ever touch pairs that actually
  co-occur in some real basket.

The complexity honesty (this is where naive readers get it wrong):

- **Worst case is O(N^2 * M):** every item against every item across every
  customer. If it were really that, it would never have shipped.
- **In practice it is far closer to O(N * M),** and often much less, because the
  loop only ever visits **pairs of items that a real customer actually bought
  together.** Most customers buy few items, so the inner "for each other item I2"
  loop is tiny. Sparsity is not a nice-to-have here, it is the reason the
  algorithm exists. The paper leans on exactly this: real purchase data is
  extremely sparse, so the co-occurrence walk is cheap relative to the worst
  case.

Cosine, concretely, for the lighter (I1) and the tongs (I2): count the customers
who bought **both** (the dot product of the two 0/1 customer-vectors), divide by
the product of the vector magnitudes (roughly, the geometric mean of how many
people bought each). Popular items have huge magnitudes, so raw co-occurrence
would make "AA batteries" and "the best-selling novel" look similar to
everything. The magnitude division is what stops the global best-seller from
drowning the row. It is the recommendation world's version of a search engine
down-weighting the word "the."

### The online half: a cheap lookup and a small sort

At page-render time, for Ananya viewing the lighter:

1. Gather her relevant items: the lighter she is viewing, plus recent
   buys/views. Say 20 items.
2. For each, **read its precomputed similar-items row** from the hash map. That
   is ~20 memory/cache reads, each returning a short pre-ranked list.
3. **Merge** the lists, summing/boosting scores when several of her items point
   at the same candidate (tongs showing up as similar to both the lighter and
   her earlier spatula purchase is a strong signal).
4. **Filter:** remove items she owns, out-of-stock, the current item, policy
   violations.
5. **Sort** the survivors by score and take the top 6 to 20. The sort is over a
   few hundred candidates at most, done server-side, never on the phone.

Total live cost is bounded by "items Ananya touched," a constant for practical
purposes. The catalog can grow from 3 million to 300 million and this path does
not get slower. **That decoupling is the feature's superpower.**

### The scale story at three tiers

- **1,000 items (a small shop):** honestly you do not need any of this. Compute
  the full item-by-item similarity matrix (1,000 x 1,000 = a million cells),
  keep it in memory, done. Even naive user-based CF is fine at this size. Nothing
  breaks.
- **100,000 items, hundreds of thousands of customers:** now user-based CF starts
  to hurt, because comparing a user against all other users live is too slow for
  a page render, and the full item-item matrix (10 billion cells) is too big to
  hold densely. What breaks: memory and live latency. What you do: go **sparse
  and offline.** Only store co-occurring pairs, compute the table as a batch job,
  serve from a keyed lookup. This is exactly the regime where item-to-item wins.
- **10 million-plus items, tens of millions of customers (real Amazon):** the
  offline job itself is now the challenge, not the online lookup. What breaks:
  the batch computation and the storage of the table. What you do:
  - **Distribute the offline build** (a MapReduce/Spark-shaped job: map over
    baskets to emit co-occurring item pairs, reduce to accumulate similarity per
    pair). Co-occurrence counting is embarrassingly parallel.
  - **Cap the fan-out per item:** keep only the top-K (say 100 to 1,000) most
    similar items per item, not the full row. This bounds storage to O(N * K) and
    keeps the online merge small. Long-tail junk gets pruned.
  - **Handle the hot item / celebrity problem:** a mega-seller like a Kindle or a
    phone charger co-occurs with almost everything, so its posting list is huge
    and its cosine is dominated by sheer popularity. The magnitude normalization
    plus top-K capping plus down-weighting ubiquitous items keeps the hot item
    from spamming every row (same shape as a search engine's stop-word handling).
  - **Shard and replicate the serving table** by item id, cache the hottest
    items' rows, so 6-figure requests per second are cheap keyed reads.

The published record is thin on Amazon's exact modern infrastructure, so the
sharding/top-K/MapReduce specifics above are **clearly-labeled inference**: this
is how this class of problem is universally solved (and how open item-item CF at
scale, for example Spark's ALS and item-similarity jobs, is actually built). The
**offline-table + online-lookup split, the cosine over item-vectors, and the
independence from N and M are fact, straight from the 2003 paper.**

### How it evolved (fact, from Amazon's own 20-year retrospective)

The 2003 algorithm was deliberately simple and it aged remarkably well: in 2017
IEEE Internet Computing picked the original paper as the single paper from two
decades that best passed the "test of time." But Smith and Linden's 2017
retrospective ("Two decades of recommender systems at Amazon.com") is candid that
the naive "people who bought X bought Y" has real weaknesses they spent years
fixing: it recommends the **obvious** (buy a phone, get recommended phone cases
forever), it ignores **time and sequence** (what you bought two years ago should
not weigh like what you bought today), and it can trap you in a **category loop.**
The evolution added matrix-factorization-style latent factors, temporal
dynamics, and later deep and sequence models, all while keeping the same
backbone: **think hard offline, serve a cheap lookup online.**

## 8. The retention and habit mechanic

Which metric does this move? Primarily **revenue**, and secondarily
**activation** of new interests.

The loop: every page you load feeds the row, the row surfaces one more thing, you
add it, and that add-to-basket event feeds the offline table that makes tomorrow's
rows better for the next person. It is a **flywheel made of baskets:** more
purchases produce better co-occurrence data, which produces better rows, which
produce more purchases. Unlike a Spotify-style weekly refresh, there is no
scheduled hook. The habit is subtler: the store quietly feels like it "gets you,"
so search-and-leave slowly becomes browse-and-discover.

The concrete, widely reported outcome: recommendations of this kind are credited
with a very large share of Amazon's sales. McKinsey's often-cited 2013 estimate
put **about 35% of Amazon purchases** as coming from its recommendation engine.
Treat the exact percentage as an external estimate, not an Amazon-published
number, but the direction is not in doubt: this quiet row is one of the
highest-leverage surfaces in e-commerce. It also drives the **Buy Again** and
subscription behaviors that compound retention: the more it correctly predicts
your restock (Ananya's lighters, again), the less reason you have to shop
anywhere else.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. The lesson from this row is not
"build a recommender." It is **the axis flip plus the offline table**, applied to
shader authoring.

Concretely: build a **"nodes that go together" panel.** When a user drops a
Fresnel node onto the canvas, show a small row: **"Effects that use this node
also use..."** (say, a rim-light node, a normal-remap node, a specular tint).
Compute it exactly the way Amazon does:

- **Flip the axis.** Do not try to find "users like this user" live in the editor,
  that does not scale and it is slow inside a tight authoring loop. Instead treat
  **each node type as a vector over the shader graphs that contain it**, and
  precompute an **item-to-item "nodes co-occur" table** offline from your corpus
  of shared/published graphs. Cosine over graph-membership, magnitude-normalized
  so a ubiquitous node (a Time or UV node that is in everything) does not
  dominate every suggestion, the exact stop-word problem.
- **Keep the live path O(1).** The editor should never compute similarity while
  the user is dragging. It reads a top-K row for the node just placed from a
  shipped lookup table (bundle it with the app or fetch it once). Suggestion cost
  depends on the current node, not on how many graphs or nodes exist in your
  library. That is what keeps the editor at 60fps while the corpus grows from
  1,000 to 10 million published effects.
- **Let the flywheel run offline.** Every published graph feeds the co-occurrence
  build; regenerate the table nightly. The authoring loop stays instant; the
  intelligence improves in the background.

And carry over the deeper systems lesson that this whole ledger keeps repeating:
**the size of your candidate set is the dial that decouples cost from catalog
size.** Cap node suggestions at top-K, prune the long tail, shard the table by
node id if it ever gets huge. Do the expensive thinking once, offline, for
everyone; make the thing the user actually touches a cheap keyed read.

---

## Sources

- Greg Linden, Brent Smith, Jeremy York. "Amazon.com Recommendations:
  Item-to-Item Collaborative Filtering." IEEE Internet Computing, Vol. 7, No. 1,
  2003, pp. 76-80. DOI: 10.1109/MIC.2003.1167344.
  https://dl.acm.org/doi/10.1109/MIC.2003.1167344
- Brent Smith, Greg Linden. "Two Decades of Recommender Systems at Amazon.com."
  IEEE Internet Computing, 2017 (20th-anniversary "test of time" retrospective).
  https://assets.amazon.science/76/9e/7eac89c14a838746e91dde0a5e9f/two-decades-of-recommender-systems-at-amazon.pdf
- Amazon Science. "The history of Amazon's recommendation algorithm."
  https://www.amazon.science/the-history-of-amazons-recommendation-algorithm
- Paper copy (University of Maryland course mirror):
  https://www.cs.umd.edu/~samir/498/Amazon-Recommendations.pdf
- Wikipedia. "Item-item collaborative filtering."
  https://en.wikipedia.org/wiki/Item-item_collaborative_filtering
- McKinsey & Company (2013), widely cited estimate that ~35% of Amazon purchases
  come from its recommendation engine (external estimate, not Amazon-published).
