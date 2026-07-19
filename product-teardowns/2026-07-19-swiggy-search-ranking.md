# Swiggy search and ranking: what happens when you type "biryani"

Date: 2026-07-19
Product: Swiggy
Feature: The search box (restaurant and dish search, autocomplete, retrieval and ranking)

## 1. The user

It is 8:40pm on a Tuesday in Koramangala, Bengaluru. Riya just got home, tired,
a little hungry, and very sure of one thing: she wants biryani. She opens Swiggy,
taps the search bar at the top, and starts typing. By the time she has typed
"biry" the app is already showing suggestions. She taps "biryani" and a screen
fills with restaurants and specific dishes she can order right now.

She is not browsing. She has an intent in her head and she wants the shortest path
from that intent to a hot box of biryani at her door in 30 minutes.

## 2. The real problem

Here is the honest version of Riya's problem. She knows what she wants but she
does not know which restaurant near her makes a good biryani, which ones are open,
which ones will actually deliver to her flat, and which one will not take an hour.

Swiggy's catalog is enormous. There are lakhs of restaurants across India and tens
of millions of individual dishes on their menus. Almost none of that is relevant
to Riya at 8:40pm in Koramangala. A restaurant in Pune making excellent biryani is
useless to her. A restaurant 400 meters away that closed at 8pm is useless to her.
A place that spells it "Biriyani" or lists it as "Hyderabadi Dum Handi" still needs
to show up.

So the real problem has three layers stacked on top of each other:

1. Only show things she can actually order (near her, open, deliverable).
2. Among those, find the ones that match "biryani" even when the words do not match
   exactly (biriyani, dum biryani, murgh biryani, kozhi biryani).
3. Among the matches, put the best one first, because she will tap something in the
   top three or she will give up.

## 3. The feature in one sentence

Swiggy search takes a few typed letters, narrows a catalog of tens of millions of
dishes down to a few hundred that are near you and match your intent, then ranks
those few hundred so the best bet for you sits at the top, all in well under a second.

## 4. Jobs to be done

What is Riya really hiring the search box to do?

- "Read my mind before I finish typing." She types "biry" and expects "biryani"
  offered instantly. (This is autocomplete.)
- "Only show me food I can actually get tonight." Near me, open, deliverable.
- "Understand what I mean, not just what I typed." Handle spelling, Hindi and Tamil
  names, and dish synonyms.
- "Put the right answer at the top." She trusts the top of the list. Do not make
  her scroll and compare.
- "Do it now." Every extra 200ms of lag makes the app feel broken.

## 5. How it works for the user

Riya taps the search bar. Before she types anything she sees trending searches and
her recent ones. As she types "b", "bi", "bir", "biry", a dropdown of suggestions
reshuffles on every keystroke: "biryani", "biryani near me", "chicken biryani",
maybe a specific restaurant name she has ordered from before.

She taps "biryani". Now the results page loads in two flavors mixed together:
restaurant cards ("Meghana Foods", "Ammi's Biryani") and specific dish cards
(a photo of "Chicken Boneless Biryani" at 249 rupees from a place 1.2km away, with
an "ADD" button right there). The good, nearby, fast options are at the top. She
taps ADD on the dish card and she is basically done.

The whole thing felt instant. That "instant" is the entire engineering story.

## 6. The actual flow, step by step

1. Riya taps the search bar. The app sends her location (lat/long of her Koramangala
   flat) plus an empty query. Server returns trending and recent suggestions.
2. She types "b". The app fires a request: prefix "b" + her location. Server returns
   ranked autocomplete suggestions in a few tens of milliseconds.
3. She keeps typing. Each keystroke ("bi", "bir", "biry") fires a new request. Old
   in-flight requests are cancelled so a slow "bi" response cannot overwrite a fresh
   "biry" one.
4. She taps "biryani". The app sends the full query "biryani" + location + her user
   id.
5. Server does serviceability filtering first: which restaurants near her are open
   and can deliver to her exact point right now.
6. Server does retrieval: from the deliverable set, pull dishes and restaurants whose
   text matches "biryani" (including fuzzy and synonym matches).
7. Server does ranking: score those few hundred candidates with a machine-learned
   model and sort them.
8. Server returns a ready-sorted, ready-to-render list. The phone just paints it. The
   phone does no sorting.

The two heavy stages, retrieval and ranking, are two different halves of the job and
they use different tools. Mixing them up is the classic mistake.

## 7. Under the hood, like the engineer

This is the heart of it. Search at Swiggy is really four problems solved in sequence:
serviceability, autocomplete, retrieval, and ranking. Walk each one with the "biryani"
query.

### Stage 0: Serviceability, or "can I even get this?"

Before any text matching, Swiggy has to shrink the universe from "all restaurants in
India" to "restaurants that can deliver to Riya's flat, right now." This is a geography
problem, not a text problem.

Naive approach: draw a circle of radius X around Riya and keep every restaurant inside
it. To find those you would loop over every restaurant and compute distance. At lakhs
of restaurants per request, per keystroke, that is dead on arrival.

What Swiggy actually does (from their serviceability engineering posts): geohashing.
The map of India is chopped into a grid of small cells, each with a short string key
(a "geohash"). A geohash5 cell is roughly a 4.9km by 4.9km box. Every restaurant and
every serviceable zone is pre-indexed into the geohash cell(s) it covers. Swiggy builds
separate indexes per geohash5 cell on purpose: they note that if you built one giant
index, a nearest-neighbor lookup might return carts that are close in the data but far
away on the ground, and therefore not deliverable.

So when Riya's request arrives, the server converts her lat/long into a geohash key,
grabs the small set of clusters overlapping that key (and neighboring cells), and only
then runs the expensive precise test. That precise test is a point-in-polygon (PIP)
check: is Riya's exact point inside a restaurant's actual delivery polygon? Running PIP
on a few dozen candidate clusters is cheap. Running it on all of India is not.

Data structures here: a hash map from geohash key to the list of restaurants/zones in
that cell (a spatial hash), plus polygons for precise membership. The geohash is doing
the same job an inverted index does for text: it turns "search everything" into "look
up a key."

Scale note that makes this real: Swiggy has written that their distance service needs
to compute shortest distances for nearly 200 million source-destination pairs per minute
at peak. You do not get there by looping over restaurants. You get there by keying on
geography first.

Fact vs inference: the geohash5 cells, per-cell indexes, and PIP-on-reduced-set are
stated by Swiggy. The exact neighbor-cell handling and radius values I have described
generically; treat the specific radius as illustrative.

### Stage 1: Autocomplete, "biry" to "biryani"

Every keystroke is a separate search request, and autocomplete is the most
latency-sensitive surface in the whole app because a single word triggers four or five
round trips. Swiggy's autocomplete runs on OpenSearch (the open-source search engine
forked from Elasticsearch), and in 2026 they moved it to real-time machine-learned
ranking.

The important architectural decision: they split autocomplete into two stages,
candidate generation and ranking, and they run the ranking model *inside* OpenSearch
using its Learning-to-Rank (LTR) plugin. That last part matters for speed. Instead of
OpenSearch returning candidates, shipping them over the network to a separate ML service,
waiting for scores, and re-sorting, the LTR model scores the top-k candidates right
there in the engine at query time. No extra network hop. For a feature that fires on
every keystroke, deleting one network round trip is the whole game.

How the two stages work for "biry":

- Candidate generation: a fast first-stage query over an index of past queries and
  entities pulls suggestions whose text starts with or contains "biry": "biryani",
  "biryani near me", "chicken biryani", plus restaurants. This first pass is loose and
  cheap. Its only job is to get a few dozen plausible candidates fast. (This is exactly
  the matching half.)
- Ranking: the LTR model (Swiggy uses gradient boosted decision trees for this class of
  ranking, via frameworks like OpenSearch LTR / XGBoost-style models) rescores only that
  top-k. It does not score the whole index.

The ranker eats features from a feature store that serves two kinds of signal at once:
precomputed features (how popular is "biryani" as a suggestion historically, refreshed
offline) and streaming features (what is trending in the last few minutes, what Riya
herself has been doing this session). Precomputed keeps it fast; streaming keeps it
fresh. The result: "biryani" jumps to the top of the dropdown after four letters, and
it does so because millions of past sessions taught the model that people who type
"biry" almost always mean biryani.

Then there is India's language problem baked into matching: "chicken" is Murgh in Hindi
and Kozhi in Tamil, and biryani gets spelled biriyani, briyani, and biriani. Swiggy
handles this partly with fuzzy matching and synonyms in the index, and partly with
learned text embeddings (below) so that a query and a differently-spelled dish name land
close together even when the characters do not match.

Fact vs inference: OpenSearch, the LTR plugin, in-engine scoring, the candidate-then-rank
split, the feature store with precomputed plus streaming features, and GBDT models are
all from Swiggy's own 2026 write-ups. The specific candidate list for "biry" is my
illustration.

### Stage 2: Retrieval, tens of millions of dishes down to hundreds

Riya tapped "biryani". Now, over the deliverable restaurants from Stage 0, the server
has to find the matching dishes and restaurants. Swiggy describes dish search as two
steps: dish retrieval then dish ranking. Their own words: retrieval "constricts the
search space from millions of dishes to hundreds."

The workhorse is an inverted index. An inverted index is a dictionary from a word to
the list of items that contain that word. "biryani" maps to the posting list of every
dish whose name or description contains biryani: dish 88213 (Chicken Boneless Biryani
at Meghana), dish 90561 (Veg Dum Biryani at Ammi's), and so on. Looking up "biryani"
is a single dictionary lookup that returns a list, instead of scanning millions of
dish rows one by one. This is why the cost of finding matches depends on how many items
match, not on how big the catalog is.

But exact word match is not enough in India, so Swiggy layers embedding-based retrieval
on top. They train models so a query and a dish sit near each other in a vector space
when they mean the same thing, even with different spellings or languages. So "murgh
biryani" can retrieve a dish literally named "Chicken Dum Biryani". This is approximate
nearest neighbor retrieval over vectors, running alongside the classic inverted-index
match. Swiggy has also published work on fine-tuning small language models specifically
for hyperlocal food search relevance (using techniques like TSDAE and Multiple Negatives
Ranking Loss), tuned to meet a hard 100ms latency budget, precisely because a big fancy
model that takes 400ms is useless here.

The output of Stage 2 is a few hundred candidate dishes and restaurants, all near Riya,
all deliverable, all plausibly about biryani. That is a set small enough to do expensive
scoring on.

### Stage 3: Ranking, ordering the few hundred

Now the two hundred survivors get scored by a machine-learned ranker so the best one is
first. Swiggy uses deep learning for dish-search ranking. The model scores each candidate
using features across several dimensions. From Swiggy's posts, the ranking signals include:

- Relevance: how well the dish actually matches "biryani".
- Restaurant features: rating, historical quality, reliability.
- Popularity features: how often this dish and restaurant get ordered.
- Distance and time: how far Meghana is from Riya and the estimated delivery time.

Swiggy also builds ranking features along three dimensions for its rankers: customer-level
(what Riya likes, her past orders), restaurant-level (Meghana's overall behavior), and
customer-restaurant-level (has Riya ordered from Meghana before, did she like it). Those
cross features tend to give the biggest lifts in relevance and conversion.

So for Riya, "Chicken Boneless Biryani from Meghana, 1.2km, 4.4 stars, ordered a lakh
times, and she personally ordered from there last month" scores higher than an unknown
place 4km away with two reviews. The model outputs one score per candidate, the server
sorts descending, and returns the list.

Crucial point the framework insists on: the sort happens server-side, over a few hundred
scored candidates, before the response is sent. The phone never sorts millions of dishes.
It receives an already-ordered list and paints it. This is the single most important
efficiency decision in the whole feature. The heavy thinking is done where the data and
the CPUs are, and the phone gets a finished answer.

### The scale story at three tiers

Tier 1, about 1,000 dishes (a small city, early Swiggy). You could almost brute force
this. Load candidate dishes, filter by distance in a loop, sort in memory. A single
database with a decent index handles it. Text search can be a simple LIKE query. Nothing
breaks yet.

Tier 2, about 100,000 dishes (a big metro). The LIKE query dies first. You must move to
a real inverted index (OpenSearch/Elasticsearch) so "biryani" is a keyed posting-list
lookup, not a table scan. Distance filtering by looping over restaurants gets slow, so
geohash bucketing appears: key on the cell, then do precise checks only on that bucket.
Ranking with hand-tuned rules starts to feel dumb, so a learned ranker (GBDT) shows up.
Read traffic is spiky at lunch and dinner, so you add read replicas and cache popular
queries.

Tier 3, 10 million plus dishes across hundreds of cities, lakhs of concurrent users on a
peak Friday night. Now everything is about not doing work on the hot path.
- Serviceability must be a geohash lookup plus PIP on a tiny set, because 200 million
  distance pairs a minute cannot be computed live per request. Distances get precomputed
  and cached.
- The search index is sharded (naturally, by geography, since a Bengaluru user never
  needs Delhi's index) and replicated for read throughput. Sharding by city is elegant
  because it matches the query pattern: your search only ever touches your region's shard.
- Ranking cannot afford a network hop, so the model runs inside the search engine
  (OpenSearch LTR) and only rescores top-k, not the whole candidate set.
- Features that are expensive to compute get precomputed offline and served from a feature
  store; only the truly fresh signals are computed in real time.
- Autocomplete, the most brutal surface, gets the tightest latency budget (models tuned to
  100ms) because it fires five times per word.

The pattern across all three tiers is the same as every good search system: turn "search
everything" into "look up a key" (inverted index for text, geohash for space), split
matching from ranking so you only do expensive scoring on a tiny survivor set, precompute
whatever you can, and never sort on the client.

## 8. The retention and habit mechanic

Search is not usually thought of as a retention feature, but at Swiggy it quietly is,
through one loop: personalization compounding.

Every search Riya runs, every dish she taps, every order she places feeds back into the
ranking model as training data (Swiggy runs a continuous feedback loop: clicks,
conversions, and orders stream into offline training pipelines that retrain the rankers).
The customer-level and customer-restaurant-level features mean the more Riya uses Swiggy,
the more her search results bend toward what she actually likes. Her tenth "biryani"
search is more useful to her than her first, because Swiggy now knows she prefers Meghana,
likes boneless, and orders around 9pm.

That is a switching-cost moat. A competitor's fresh app does not know any of this, so its
search feels generic. Swiggy's feels like it read her mind. This is exactly the "mind
reader" framing Swiggy uses in its own data-science posts.

Which metric it moves: primarily conversion and retention. Faster, more relevant search
means a higher share of searches end in an order (activation of intent), and better
personalization over time means she keeps coming back rather than comparing apps. The
autocomplete-relevance work was justified explicitly by conversion and relevance gains.

A real observed nudge on top of the loop: the pre-typing state (trending searches and
your recent searches shown the instant you tap the box) is a re-order shortcut. Showing
"biryani" as a recent search on a tired Tuesday is a one-tap path back to last week's
order. It makes repeat ordering nearly frictionless, which is the whole point.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable
code, plus an embeddable runtime. The Swiggy lesson maps almost one to one onto asset
and node search, and more importantly onto how the runtime picks what to do.

Concrete lesson: split matching from ranking, and precompute a spatial or structural key
so live requests become lookups, not scans. Two applications.

1. Node and asset search in the editor. As Rare.lab grows to thousands of nodes, shader
   presets, and community effects, do not scan the library on every keystroke. Build an
   inverted index over node names, tags, and descriptions for the matching half, then run
   a small learned or heuristic ranker over the few survivors using signals like "did this
   user use this node before" and "how popular is this node in projects like theirs." Run
   the ranking where the data is, return a finished ordered list, never sort the full
   library in the UI thread. This is the customer-level / project-level feature idea
   applied to node discovery.

2. The deeper one, for the runtime, borrow Swiggy's serviceability trick. Serviceability
   is "which of the millions of things can I actually serve to this exact context, cheaply,
   before I do expensive work." In a shader runtime the equivalent question is "which of the
   many shader variants and effect graphs can this exact device actually run at 60fps right
   now." Do not evaluate every variant live. Precompute a capability key per device tier
   (like a geohash for GPUs: an index keyed on GPU family, memory, feature support), map the
   device to its key on load, and fetch only the small set of pre-validated variants for that
   key. Then do the expensive per-shot compile or quality decision only on that tiny survivor
   set. That is geohash-plus-PIP applied to hardware: turn "check every variant against this
   device" into "look up this device's bucket, then precisely check the few."

The one-line version: whatever the request (a search, a shader dispatch), make the hot path
a keyed lookup over a precomputed index, keep matching and ranking as two separate cheap
stages, and push all heavy scoring off the client. That is how Swiggy serves biryani in
under a second, and it is how Rare.lab stays at 60fps as the effect library explodes.

## Sources

- Swiggy Bytes: Using Deep Learning for Ranking in Dish Search - https://bytes.swiggy.com/using-deep-learning-for-ranking-in-dish-search-4df2772dddce
- Swiggy Bytes: Real-time ML Ranking for Autocomplete - Deploying Learning-to-Rank inside OpenSearch (Part 1) - https://bytes.swiggy.com/real-time-ml-ranking-in-autocomplete-part-1-3cdbbd44f85a
- InfoQ: Swiggy Improves Search Autocomplete Using Real Time Machine Learning Ranking - https://www.infoq.com/news/2026/05/swiggy-autocomplete-rt-ranking/
- Swiggy Bytes: Improving search relevance in hyperlocal food delivery using (small) language models - https://bytes.swiggy.com/improving-search-relevance-in-hyperlocal-food-delivery-using-small-language-models-ecda2acc24e6
- ZenML LLMOps Database: Two-Stage Fine-Tuning of Language Models for Hyperlocal Food Search - https://www.zenml.io/llmops-database/two-stage-fine-tuning-of-language-models-for-hyperlocal-food-search-ff103
- Swiggy Bytes: What Serviceability means at Swiggy - https://bytes.swiggy.com/what-serviceability-means-at-swiggy-c94c1aad352a
- Swiggy Bytes: Designing the Serviceability Platform at Swiggy for High Scale (Part 1) - https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-1-751a631f0379
- Swiggy Bytes: Learning To Rank Restaurants - https://bytes.swiggy.com/learning-to-rank-restaurants-c6a69ba4b330
- Swiggy Bytes: Using embeddings to help find similar restaurants in Search - https://bytes.swiggy.com/using-embeddings-to-help-find-similar-restaurants-in-search-1d1417dff304
- Swiggy Bytes: Building a mind reader at Swiggy using Data Science - https://bytes.swiggy.com/building-a-mind-reader-at-swiggy-using-data-science-5a5c38aa6c17
