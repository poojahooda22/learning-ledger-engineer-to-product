# Zepto grocery search: the search bar that only shows what is in the box near you

Date: 2026-10-18
Product: Zepto
Feature: Grocery search (the search bar), with its hyperlocal, in-stock-only candidate set

A note on sourcing before we start. Zepto has not published the internals of its
search stack. So this teardown does two honest things. Where I describe the live
behavior of Zepto search, that comes from using the app and from third-party
analyses (clearly labeled as secondhand). Where I describe how the engineering
is actually built, I lean on the best-documented grocery search in the world,
Instacart's, which has published papers and engineering posts on exactly this
problem. Instacart runs the same shape of problem as Zepto: a short messy query,
a catalog that differs store by store, and results that must respect what one
specific store physically holds right now. I mark Instacart facts as CONFIRMED
and everything applied to Zepto specifically as INFERENCE.

---

## 1. The user

Meet Priya. It is 9:20pm on a Tuesday in Koramangala, Bengaluru. She is making
Maggi for a late dinner, opens the packet, and realizes there is no butter left
in the fridge and the milk is almost gone. She does not want to walk to the
store. She opens Zepto, taps the search bar, and types "amul". She wants the
butter and the milk in her hands in ten minutes, and she wants to be back on the
couch in under a minute of tapping.

She is not browsing. She is on a mission with a shopping list in her head
(butter, milk, maybe bread). Every extra second of typing, every wrong result,
every "out of stock" she hits after tapping is friction between her and dinner.

## 2. The real problem

Here is the pain, said plainly. Priya knows exactly what she wants, but the shop
does not have everything, and which "shop" she is even buying from changes with
where she is standing.

Zepto does not deliver from one big national warehouse. It delivers from a tiny
neighborhood "dark store" (a mini warehouse with no customers inside) that sits
within about two miles of her, so a rider can reach her in ten minutes. That
dark store is small. It stocks a curated couple of thousand items, not the whole
grocery universe. So when Priya searches "amul", the honest answer is not "every
Amul product that exists". It is "every Amul product this particular Koramangala
dark store is holding, in stock, right now, at 9:20pm, after a busy evening of
orders have already emptied some shelves".

If search shows her Amul Butter 100g and she taps it and it says "out of stock",
she is annoyed and she trusts the app a little less. If search is slow, she
starts typing in a competitor's app instead. The real problem is: show me, fast,
only the things I can actually get in the next ten minutes from the box near me.

## 3. The feature in one sentence

Zepto search takes a short, often messy query and returns a ranked list of
products that the single nearest dark store is actually holding in stock at this
moment, ordered so the thing most people like Priya buy sits at the top.

## 4. Jobs to be done

What is Priya really hiring this search bar to do?

- "Find the exact thing I already decided on, in two taps." (She typed "amul"
  because she wants Amul butter, not to browse dairy.)
- "Do not lie to me about what I can get." (Only show in-stock items from the
  store that will serve me.)
- "Guess what I mean even when I type badly." ("maggi" means instant noodles,
  "doodh" means milk, "collgate" means Colgate.)
- "Remember what I always buy." (Milk, bread, eggs every few days. Make the
  repeat order one tap.)
- "Be instant." (Results should appear as I type, not after I wait.)

## 5. How it works for the user

Priya taps the search bar. As she types "a", "am", "amu", a dropdown of
suggestions fills in under her finger: "amul butter", "amul milk", "amul cheese",
"amul dahi". This is autocomplete, and it is showing her the searches other
people near her actually make, ranked by what they end up buying.

She taps "amul butter". A results page appears almost instantly: Amul Butter
100g at the top with a price and an "Add" button, then Amul Butter 500g, then
Amul Mozzarella, then a couple of close cousins (Amul Cheese Slices). Every card
she sees has an "Add" button that works, because everything shown is in stock at
her store. There is no "notify me", no greyed-out card. She taps Add on the 100g
butter, taps the search bar again, types "doodh", gets Amul Taaza Toned Milk
500ml and Nandini Toned Milk (the big Bengaluru brand) at the top, adds one, and
checks out. Total time in the app: about forty seconds.

## 6. The actual flow, step by step

1. Priya's app already knows her delivery location (a saved pin in Koramangala).
   That pin was resolved earlier to one specific dark store. Every search she
   does is silently scoped to that store. (This is the serviceability gate,
   covered for Swiggy on 2026-09-28: the app decides which store can serve this
   pin before anything else happens.)
2. She taps the search bar. The app sends each keystroke, or a debounced version
   of it, to the search service along with her store id.
3. The server returns autocomplete suggestions. The phone just draws them. The
   ordering was decided on the server.
4. She taps a suggestion, "amul butter". The phone sends that full query plus the
   store id.
5. The server does two things at once: it figures out what she means (butter, the
   brand Amul), and it fetches candidate products.
6. The candidates are filtered down to only what this Koramangala store carries
   and has in stock at this second.
7. The survivors are ranked by a model, best first.
8. The server sends back an ordered list of product cards. The phone displays
   them in that exact order. The phone does no sorting.
9. She taps Add. At that instant the system checks and decrements the live stock
   count for that unit at that store, so the shelf stays honest for the next
   person searching.

The key thing: steps 5, 6, 7 all happen on the server, in a few tens of
milliseconds, and the phone is a dumb display. The phone never holds the catalog
and never sorts.

## 7. Under the hood, like the engineer

This is the heart of it. Search is always two different machines bolted together:
MATCHING (cheaply find a few candidates out of everything) and RANKING
(expensively order those few). The twist that makes grocery different from
Amazon or Google is where the difficulty lives.

### The twist: the catalog is tiny, but it is different in every box

On Google, matching is the hard half, because you are pulling a few hundred
candidates out of hundreds of billions of pages. On Zepto, one dark store holds
only about 2,500 to 3,000 items (per-store assortment figures from quick-commerce
industry blogs, treat as secondhand). Searching 3,000 rows is trivial. A laptop
can scan that.

So the hard problem flips. It is not "find the needle in billions". It is "the
haystack is a different, small, constantly-changing haystack for every customer".
Priya's Koramangala store and another store three kilometers away stock
different items and have different live counts. The same query "amul" returns
different results from the two pins. And across all of Zepto, the DRHP (the IPO
filing) was reported to show roughly 1,139 dark stores across 66 cities, with a
weighted average near 49,602 SKUs per dark-store geography (that larger number
is the per-city catalog, not what one store holds; secondhand reporting, verify
against the filing). So the thing that explodes is not one catalog. It is
(stores x items-per-store x live-inventory-updates x concurrent searchers).

### MATCHING: understand the query, then fetch in-stock candidates

Step one is query understanding, and grocery queries are nasty. They are short
("maggi"), brand-as-category ("maggi" means instant noodles, not literally the
Maggi label), multilingual ("doodh" is Hindi for milk), and misspelled
("collgate"). CONFIRMED (Instacart's engineering blog and their "Intent Engine"
work): a query understanding layer sits upstream, classifies the query into a
product category, corrects spelling, expands synonyms, and validates the result
against a controlled product taxonomy. Instacart's newer version uses an LLM with
retrieval-augmented prompting and then runs the output through validators that
throw away hallucinations and force alignment to the taxonomy. The query category
it predicts "directly powers recall and ranking". INFERENCE for Zepto: the same
layer turns "doodh" into the category "milk" and the Hinglish into a clean intent
before any lookup happens.

Step two is candidate fetch, and this is where the inverted index earns its keep.
An inverted index is the index at the back of a book: for every word, a sorted
list of the products that contain it. Search "amul butter" and the server walks
the posting list for "amul" and the posting list for "butter" and intersects
them, which is fast because the lists are short and sorted. CONFIRMED as the
standard approach (Elasticsearch and its inverted index are the default engine
for this class of problem). Out falls a candidate set: Amul Butter 100g, Amul
Butter 500g, Amul Mozzarella, and so on.

But keyword match alone misses the ambiguous and the differently-worded. So the
modern stack adds a second retrieval path running in parallel: embedding-based
retrieval. CONFIRMED (Instacart, "An Embedding-Based Grocery Search Model",
SIGIR eCom 2022, arXiv 2209.05555): a two-tower transformer learns a vector for
the query and a vector for every product, so that semantically close things land
near each other in the vector space. You then find candidates by approximate
nearest neighbor search (ANN) over those vectors. Instacart serves this with
FAISS, and this is the single most important detail for our story: they keep
"separate per-retailer FAISS indexes that handle store-level inventory". In plain
words, the vector index itself is scoped to one store's stock. You do not search
a global index and then filter. The index you search already only contains what
that store can sell. INFERENCE for Zepto: the retrieval index is scoped (or hard
filtered) to the serving dark store's live assortment, so the availability check
is not a late filter, it is baked into what you even look at.

Then the two candidate lists, keyword and embedding, are merged. CONFIRMED
(Instacart's "hybrid retrieval" work): results from keyword retrieval and
embedding retrieval are combined, and the embedding similarity score is carried
forward as a feature into the ranker. So "matching" ends with maybe 50 to 200
in-stock candidate products for "amul butter" at the Koramangala store.

The availability filter deserves its own line because it is the whole feature.
On Amazon an out-of-stock item ranks lower but still shows. On Zepto, CONFIRMED
by third-party quick-commerce analyses (marketing agencies that study the apps,
treat as secondhand): if an item is not in stock at the nearest dark store, it
does not rank lower, it disappears. That is the honest-shelf promise. It also
means the filter must read a near-real-time stock signal, because the last 6
packs of Amul Butter 100g at Koramangala can sell out between Priya typing and
Priya tapping.

### RANKING: order the survivors by what people actually buy

Now the expensive half, but it only runs on the 50 to 200 survivors, not on the
whole catalog. CONFIRMED (Instacart): ranking blends relevance, popularity, and
personal preference, and the model is trained to predict an action, add-to-cart.
Their embedding search drove, in online A/B tests, 4.1% more cart-adds per search
and 1.5% more GMV, with a 10% relative lift in offline Recall@20. One subtle
honest detail they published: cart-adds are a noisy label, because shoppers buy
many items per trip and sometimes add an irrelevant result by accident, so the
"did they buy it" signal has to be de-noised during training.

For a quick-commerce app the dominant ranking signals (CONFIRMED by secondhand
quick-commerce ranking analyses) are availability and fill rate, then sales
velocity (how fast an item sells, so bestsellers float up), then text relevance,
then personalization. Walk "maggi" through it. Candidates: Maggi 2-Minute Masala
70g, the 280g 4-pack, the 12-pack, Maggi Oats Noodles, Maggi Hot and Sweet
sauce. The 70g single and the 4-pack sit on top because they are the velocity
leaders (everyone buys them), the sauce sinks because it sells slower and is a
looser match to the intent "instant noodles". Priya, who buys the 4-pack every
week, might see the 4-pack lifted by her personal history.

Crucially, the sort happens server-side, in the ranking service, across the
candidate set, and the ordered list is handed to the phone. The phone draws
cards top to bottom. It never sorts.

### Substitution: when the last unit sells mid-session

Grocery has a problem a bookstore does not: the thing can run out while you shop.
CONFIRMED (Instacart, "how Instacart suggests replacements for out-of-stock
products"): a funnel does it. Heuristic retrieval narrows candidates using past
replacement pairs (people who could not get fresh blueberries accepted frozen
blueberries) plus catalog-based semantic similarity, then an ML ranker orders
the replacements. INFERENCE for Zepto: if Amul Butter 100g sells out at
Koramangala while Priya is deciding, the natural substitute surfaced is Amul
Butter 500g or Nandini Butter 100g, learned from what other shoppers accepted.

### The scale story at three tiers

What grows is not catalog size. It is (dark stores) x (items per store) x
(live inventory changes per second) x (concurrent searchers across cities).

Tier 1, about 1,000 items, one store, one city. Put the catalog in one
Elasticsearch or Postgres index. Keyword match, filter by an in_stock boolean,
sort by a popularity column. No embeddings, no learned ranker, no ANN. Building
those here would be pure over-engineering. Ship the inverted index and a velocity
sort and move on. At 3,000 rows a brute-force scan is fine.

Tier 2, about 100,000 scale, many stores each with its own 3,000-item assortment
that differs by neighborhood. Now a single flat global index lies, because it
will happily show Priya an item her Koramangala store does not carry. So you
scope retrieval to the serving store or zone. You add real query understanding,
because the messy queries ("doodh", "collgate") now cost you real conversions if
you get them wrong. You keep a per-store inventory signal that updates as orders
land. Embedding retrieval starts to pay off for the ambiguous and long queries.
This is the tier where the architecture stops being optional.

Tier 3, Zepto's real scale: roughly 1,139 dark stores across 66 cities (DRHP,
secondhand), lakhs of concurrent searchers on a busy evening, inventory moving by
the second. Four things break and here is the survival move for each.

1. You cannot scan a global catalog per keystroke. Survive with the inverted
   index plus per-store-scoped ANN. CONFIRMED that Instacart precomputes product
   embeddings daily and keeps separate per-store FAISS indexes. The heavy thinking
   (training embeddings, building indexes, computing velocity aggregates) runs
   OFFLINE in a nightly or daily batch. The live query does only cheap work:
   tokenize, resolve intent by lookup, scatter to the store's scoped index, merge,
   light re-rank, return. This is the same offline-think / online-lookup spine as
   Spotify Discover Weekly (2026-06-13) and Google core ranking (2026-10-10).

2. Inventory is contested and changes by the second (40 people eyeing the last 6
   packs of butter at Koramangala). Survive by reading a fast near-real-time stock
   store for the availability filter, leaning conservative (better to hide a
   borderline item than to show one that is gone), and decrementing stock
   atomically at add-to-cart. Same atomic-stock pressure as Amazon Lightning Deals
   (2026-08-29) and Zepto inventory routing (2026-06-17).

3. One machine cannot hold or serve it all. Survive by sharding the search and
   inventory index by store or zone id, so a Bengaluru query hits Bengaluru
   shards and never does a global scan. Same shard-by-key idea as Notion by
   workspace (2026-06-25) and Gmail by mailbox (2026-10-03). Locality is free
   accuracy here: a store's shard only holds a store's reality.

4. Query understanding and ranking models are expensive to run live. Survive by
   training them offline and serving them as cheap lookups and small scoring
   passes on the 50 to 200 survivors, never a model run over the whole catalog on
   the request path.

The through-line across all three tiers: the catalog per store is small, so the
engineering is not about retrieval scale, it is about keeping the availability
truth fresh and local and cheap to check, and pushing every heavy computation off
the live path into a precomputed, shardable, store-scoped form.

## 8. The retention and habit mechanic

Grocery is the most habitual category there is. You run out of milk every few
days forever. That is the loop Zepto search is built to capture.

The engine is "Buy Again" and personalized autocomplete. Because search learns
your repeat basket, typing "d" can surface "doodh, Amul Taaza 500ml" that you buy
weekly, turning a recurring errand into a two-tap reorder. The cue is a real-life
trigger (the fridge is empty), the routine is open-Zepto-and-search, the reward
is it arrives in ten minutes, and the more Zepto nails the reward, the more
"I am out of X" compiles straight to "Zepto it" with no conscious choice. The
metric this moves is retention and frequency, specifically orders per user per
month, which is the north-star for quick commerce.

But the sharpest retention mechanic here is invisible craft: an availability-
accurate, fast search IS the retention. This is the same trust-cracker pattern as
Swiggy serviceability (2026-09-28), Netflix per-title encoding quality
(2026-09-11), and Spotify loudness (2026-08-28). If search shows Priya an item
that is actually out of stock, or takes two seconds to respond, trust cracks and
she keeps the competitor's app open as a backup. One good search where everything
she taps actually adds to cart is worth more than any festival animation. The
honest-shelf promise (we only show what you can get) is not a nicety, it is the
product, and it is precisely why the availability filter is a first-class part of
matching and not a cosmetic badge.

A real observed example of the "keep it alive" layer on top: quick-commerce apps
rotate home-screen categories and run festival and rainy-day merchandising (the
same instinct dissected in the Zomato push-notification teardown, 2026-09-30), so
the app feels fresh between the purely functional reorders. But the spine is the
reorder loop, and search is the fast lane into it.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code plus an embeddable runtime. The feature that maps onto Zepto
search is the in-editor search over your node, preset, and asset library:
a creator types "noise" or "glass" and wants the right node now, without
breaking flow.

The one lesson, biased to scalability and performance: make availability a
first-class FILTER in the matching half, not a penalty in the ranking half, and
scope the candidate set to what is actually usable here before you rank anything,
exactly the way Zepto scopes search to one dark store's live stock.

Concretely:

1. Scope by target, do not filter late. A node or shader that cannot compile on
   the user's current target (a WebGPU-only node on a Metal build, or an effect
   too heavy for the phone preview) should disappear from results the way an
   out-of-stock SKU disappears, not rank lower. A result the creator cannot use
   is worse than no result, because tapping it and hitting a wall is the thing
   that cracks trust. Build a per-target "in-stock" index: the subset of the
   library that compiles and runs on this device class and this backend.

2. Precompute the scoped indexes offline. CONFIRMED that Instacart precomputes
   product embeddings daily and keeps per-store FAISS indexes. Do the same per
   backend and per device class at build time: an inverted index over node
   names, tags, and descriptions, plus an ANN index over precomputed embeddings,
   one scoped set per target. The live editor query is then a cheap scoped lookup
   plus a light sort, never a global scan and never a model run on the 60fps
   thread.

3. Keep the "stock" signal fast and lean conservative. Whether a node runs here
   at frame budget is your availability bit. If it is borderline on this device,
   omit it rather than show it and jank the preview, the same way you lean
   conservative on contested inventory.

4. Rank for the action that sticks. Instacart ranks for add-to-cart and measured
   a 4.1% lift in cart-adds. Your equivalent is "dropped into a graph and it
   actually compiled and shipped", not "clicked and previewed and deleted two
   seconds later". Log that outcome per query and per context, aggregate it
   offline, and read it at edit time as an O(1) re-rank table.

5. Shard by project or target so search stays O(scoped set), not O(global
   library), as your asset library grows to millions. A creator building for
   mobile should hit the mobile-scoped shard, the way a Bengaluru query hits
   Bengaluru shards.

One line: build Rare.lab's asset search like a Zepto dark store, scope the
candidate set to what is genuinely in stock for this target before you rank,
precompute the scoped indexes offline, keep the availability signal fresh and
conservative, and rank for the result that actually compiles and ships, so the
live editor query is a cheap local lookup that never lies and never stalls a
frame.

---

## Sources

Instacart (CONFIRMED engineering, the class-of-problem reference):

- An Embedding-Based Grocery Search Model at Instacart, SIGIR eCom 2022:
  https://arxiv.org/abs/2209.05555 and the PDF
  https://sigir-ecom.github.io/ecom22Papers/paper_8392.pdf
- How Instacart Uses Embeddings to Improve Search Relevance:
  https://www.instacart.com/company/tech-innovation/how-instacart-uses-embeddings-to-improve-search-relevance
- Optimizing search relevance at Instacart using hybrid retrieval:
  https://tech.instacart.com/optimizing-search-relevance-at-instacart-using-hybrid-retrieval-88cb579b959c
- How Instacart Uses Machine Learning-Driven Autocomplete to Help People Fill
  Their Carts:
  https://company.instacart.com/tech-innovation/how-instacart-uses-machine-learning-driven-autocomplete-to-help-people-fill-their-carts
- Building The Intent Engine: Revamping Query Understanding with LLMs:
  https://tech.instacart.com/building-the-intent-engine-how-instacart-is-revamping-query-understanding-with-llms-3ac8051ae7ac
- How Instacart Uses ML to Suggest Replacements for Out-of-Stock Products:
  https://www.predictiveanalyticsworld.com/machinelearningtimes/how-instacart-uses-machine-learning-to-suggest-replacements-for-out-of-stock-products/14118/
- Two-tower models in industry (per-retailer FAISS, daily embeddings summary):
  https://machinelearningatscale.substack.com/p/recsys-part-2-two-tower-models-in

Zepto and quick commerce (facts and secondhand analyses, labeled in text):

- Business Today, how dark stores power quick commerce (pack in about 2 minutes,
  deliver in 8 to 10):
  https://www.businesstoday.in/magazine/deep-dive/story/how-dark-stores-are-powering-growth-of-quick-commerce-players-374569-2023-03-23
- Quick-commerce search ranking, availability-first and sales velocity
  (secondhand marketing analysis):
  https://zane.marketing/quick-commerce-search-ranking/
- Zepto IPO / DRHP reporting (dark store and SKU counts, verify against filing):
  https://hdfcsky.com/ipo/zepto-limited-ipo
