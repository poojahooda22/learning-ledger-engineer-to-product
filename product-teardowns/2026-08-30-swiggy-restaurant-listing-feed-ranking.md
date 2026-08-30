# Swiggy: Restaurant listing and home-feed ranking (the order of restaurants you see before you search anything)

Date: 2026-08-30
Product: Swiggy
Feature: The personalized ranking of the restaurant list on the Swiggy home screen. Not the search box (covered 2026-07-19), not the live order map (2026-06-24). This is the vertical list of restaurant cards you scroll the moment you open the app, and the machine that decides which restaurant sits at rank 1 and which sits at rank 40, for you, right now.

---

## 1. The user

It is 1:05pm on a Tuesday. Ananya works at an office in Koramangala, Bangalore. She has a 40-minute lunch break and she is hungry now, not in an hour. She opens Swiggy, and before she types a single letter, the app shows her a list of restaurants: cards with a photo, a name, a rating, a delivery time, and maybe a "20% OFF" tag.

She does not have a specific dish in mind. She is not going to search "biryani" or "salad." She is going to do what most people do at lunch: scroll the first screen, look at the first five or six cards, and tap one. If nothing on the first screen grabs her in about ten seconds, she gets annoyed, maybe closes the app, maybe eats at the office canteen instead.

So the entire lunch decision, and Swiggy's entire revenue from Ananya today, rides on which restaurants land in those first six cards. That list is not the same for the person sitting next to her. Her colleague Rohan, who orders Meghana Foods biryani three times a week, sees Meghana near the top. Ananya, who orders salads and South Indian, sees CureFit's eatfit and a Udupi place near the top. Same building, same minute, two completely different lists.

## 2. The real problem

Here is the honest version, the way you would explain it to a friend.

At lunchtime in Koramangala, there are hundreds of restaurants that can physically deliver to Ananya's office. If Swiggy just dumped all of them in a random order, or in alphabetical order, or even purely by distance, the app would be useless. The closest restaurant might be a place she hates. The highest-rated place might take 55 minutes, and she has 40. The cheapest place might be out of her taste entirely.

So the real problem is a sorting problem with a cruel twist: there is no single correct order. The "best" order depends on who is looking, what time it is, how hungry the neighborhood is right now, which restaurants are actually going to deliver on time today, and what Ananya has liked in the past. And Swiggy has to compute this personalized order in a couple hundred milliseconds, for 17.1 million people a month, on a home screen that has to feel instant.

There is a second, harder problem hiding underneath. A restaurant can look perfect and still be the wrong answer, because it cannot actually deliver to you right now. Maybe it is 8km away by road. Maybe it just got slammed with 30 orders and the kitchen is backed up. Maybe it is pouring rain in that zone and there are no delivery partners free. Showing Ananya a restaurant she loves, letting her build a cart, and then saying "sorry, not serviceable" at checkout is worse than never showing it at all. So before ranking can even begin, something has to decide which restaurants are honestly deliverable this minute.

That is two different jobs stacked on top of each other. First, who can actually feed her right now (serviceability). Then, of those, in what order should she see them (ranking). Confusing the two is the classic mistake.

## 3. The feature in one sentence

Swiggy takes the few hundred restaurants that can honestly deliver to your exact location this minute, and sorts them into a personalized order using a machine-learned ranking model that balances what you will probably like against how good the delivery experience will be, all computed server-side in a couple hundred milliseconds.

## 4. Jobs to be done

- "Show me food I will actually want, without making me search or think." (Personalized relevance.)
- "Do not show me a place that cannot deliver to me, or that will take an hour when I have 40 minutes." (Serviceability plus ETA honesty.)
- "Put the thing I am most likely to order today near the top so I can tap and get back to work." (Ranking for conversion.)
- "Feel fresh. Do not show me the exact same six cards I saw yesterday." (Freshness and exploration.)
- "Be fast. I opened the app hungry, not to wait for a spinner." (Latency.)

## 5. How it works for the user

Ananya opens the app. In well under a second, the home screen paints: a row of food-mood shortcuts at the top ("Biryani," "Healthy," "Pizza"), maybe a banner, then the main event, a long vertical list of restaurant cards.

Each card is dense with decision-making signal: a hero photo, the restaurant name (Meghana Foods, Empire Restaurant, Truffles), a star rating like 4.3, a delivery time like "35 min," a distance like "2.4 km," a price hint, and an offer tag. The order of these cards is the feature. Ananya reads it top to bottom and her eye stops on the first thing that fits her mood.

She does not know that this list was assembled just for her, seconds ago, out of a candidate pool that was itself filtered from the full catalog. To her it just looks like "the restaurants." The whole craft of the feature is that the enormous machinery behind it is invisible. It just looks like a good list.

## 6. The actual flow, step by step

1. Ananya taps the Swiggy icon. The app already knows her delivery location (saved "Work" address in Koramangala, or a live GPS fix).
2. The app sends a request to Swiggy's backend: "Home feed for user 8827x, at lat/lng 12.9352, 77.6245, at 1:05pm Tuesday."
3. The backend resolves her location to a serviceability context: which restaurants can deliver here, and at what predicted delivery time, given right-now conditions.
4. A candidate set of deliverable restaurants is fetched. Not the whole city. The few hundred that pass the serviceability gate for her exact point.
5. For each candidate, the backend gathers features: her past behavior with that restaurant and that cuisine, the restaurant's popularity and rating, the distance, the predicted delivery time, current offers, and the time-of-day and city context.
6. A ranking model scores every candidate. One number per restaurant: how good this restaurant is for this user, right now, balancing taste against delivery experience.
7. The list is sorted by that score, server-side, and the top slice is returned to the phone. Business rules and diversity tweaks may reorder a few cards (do not show five biryani places in a row, respect ad slots, inject a bit of freshness).
8. The phone renders the cards in the order it received them. It does not sort. The sort already happened on the server.
9. Ananya scrolls, taps Meghana Foods at rank 3, and the ranking model quietly logs that a restaurant shown at position 3 got a click. That log becomes training data for tomorrow's model.

The key thing to notice: by the time the list reaches the phone, all the thinking is done. The phone is a dumb display. Every hard decision happened server-side in steps 3 through 7.

## 7. Under the hood, like the engineer

This is the heart of the report. The feature is really two engines bolted together: a serviceability engine that decides who is deliverable, and a ranking engine that decides in what order. They are the "matching" and "ranking" halves, exactly like search, and Swiggy has written publicly about both.

### 7a. The matching half: serviceability, the geo gate

Before you can rank restaurants, you need the set of restaurants that can actually deliver to Ananya's precise point. Swiggy calls this serviceability, and it runs on every pre-order screen.

The naive approach is a distance circle: "show every restaurant within 7km." This is wrong for two reasons. Roads are not straight lines (a restaurant 3km away as the crow flies can be 8km by road, across a river or a rail line with one bridge). And a flat radius ignores whether there is anyone free to make the trip right now.

Swiggy's serviceability platform, described by Somsubhra Bairi on the Swiggy tech blog, works in stages (confirmed):

**Geo-filtering with polygons, not circles.** Each area Swiggy serves is a cluster drawn as a polygon on the map. The core question is "is Ananya's drop point inside a serviceable polygon for this restaurant's pickup point?" That is a Point-in-Polygon (PIP) test. The hard part is scale: Swiggy runs PIP checks across thousands of clusters spread across hundreds of cities. Doing a raw PIP test against every polygon for every request would be far too slow.

The fix (confirmed) is a **GeoHash structure for efficient spatial queries.** GeoHash chops the world into a grid of lettered/numbered cells, where a longer string means a smaller cell, and, crucially, nearby places share a string prefix. So instead of testing Ananya's point against thousands of polygons, you convert her lat/lng into a GeoHash cell, look up only the handful of polygons that touch that cell, and run PIP against just those. This is the same "turn a location into a short string and do a keyed lookup instead of a scan" trick this ledger saw with Uber's H3 hexes (2026-06-14). The data structure is a hash map from GeoHash cell to the small list of polygons overlapping it. Lookup cost becomes roughly constant instead of growing with the number of polygons in the country.

Concretely: Ananya's office in Koramangala hashes to a cell like `tdr1y`. Swiggy looks up which restaurant service polygons overlap `tdr1y` and its neighbors, and only those restaurants are candidates. Empire Restaurant on 100 Feet Road, whose polygon covers `tdr1y`, is in. A great biryani place in Whitefield, 14km away, whose polygon does not reach this cell, is out before ranking even starts.

**Route distance, not straight-line.** For the survivors, Swiggy computes actual road distance from the restaurant to Ananya, not the crow-flies distance. This matters for both the ETA and the delivery fee.

**Predicted delivery time.** The single most important serviceability output is the predicted delivery time, and it is a prediction, not a guess from distance. Swiggy's model folds in delivery fleet parameters (how many partners are free near that restaurant), store factors (how backed up this kitchen is), route distance, and traffic. This is why the "35 min" on Meghana's card at 1pm can become "50 min" at 8:30pm on a Friday even though the road distance never changed.

**Stress and graceful degradation.** Demand is not flat. At the 1pm and 8pm peaks, a zone can get more orders than its delivery fleet can absorb. Swiggy models this as a stress or "graceful degradation" system built as a Finite State Machine (confirmed): nodes are stress levels for a zone, and edges are the transitions between them. As a zone heats up, it moves along the state machine, and the system responds by widening ETAs, adding a surge fee, shrinking the serviceable radius, or in the extreme marking far restaurants unserviceable. The FSM is the honest-number discipline: rather than promise 30 minutes it cannot keep, Swiggy degrades the promise on purpose. This is the same trust-over-optimism stance the Zepto 10-minute teardown (2026-08-23) called out, raise the quote or go unavailable rather than break the promise.

So the output of the matching half, for Ananya at 1:05pm, is not "all restaurants." It is a few hundred restaurants, each already stamped with a predicted delivery time and a delivery fee, each one honestly deliverable to her point this minute. That set is the candidate list the ranking engine gets to sort.

### 7b. The ranking half: learning to rank restaurants

Now the interesting question. Given ~300 deliverable restaurants for Ananya, in what order should she see them?

The wrong instinct is a fixed formula, like "sort by rating," or "sort by distance," or "sort by some hand-tuned score = 0.4*rating + 0.3*(1/distance) + 0.3*popularity." Swiggy tried the hand-tuned and simple-model era and moved past it. The reason is that the true best order is non-linear and personal. A 4.1-rated place 1km away that Ananya orders from every week should beat a 4.5-rated place 3km away she has never touched. No fixed weight captures "every week for Ananya but never for Rohan." You have to learn it.

Swiggy frames this as **Learning To Rank (LTR)**, and has published the arc on the Swiggy tech blog (Ashay Tamhane, Jagrati Agrawal, Rutvik Vijjali, Akash Deep, and separately Jairaj Sathyanarayana on feed ranking). Here is the real progression.

**Pointwise, then pairwise, then listwise.** The first LTR models are pointwise: predict a score per restaurant independently (like "probability Ananya orders from this one"), then sort by the score. It works, but it does not directly train on the thing you care about, which is the relative order. So Swiggy experimented with pairwise and listwise losses, which train on comparisons ("restaurant A should rank above restaurant B for this user") rather than absolute scores. Their public finding (confirmed): **pairwise loss works best for them.** Pairwise learning-to-rank, in the LambdaMART/RankNet family, optimizes the order of pairs, which lines up with what the user actually experiences, a ranked list where being one slot higher matters.

**The model itself.** Swiggy's production choice has been **Gradient Boosted Decision Trees (GBDT), on Spark MLlib** (confirmed), chosen for two very practical reasons: trees capture non-linearity for free (they can learn "if user has ordered biryani 5+ times AND distance < 3km AND it is lunch, boost hard"), and they slot cleanly into Swiggy's existing Spark production setup. GBDT is the workhorse of tabular ranking across the industry for exactly these reasons.

**Then wide-and-deep with a Lattice head.** The frontier Swiggy has published moves to a neural architecture (confirmed):

- **A "wide" memorization side.** Sparse interaction features (this user with this restaurant, this user with this cuisine) are fed into a wide input so the model can memorize specific preferences. This is the "Ananya orders eatfit salads on weekdays" that you want the model to just remember.
- **Embeddings for the big entities.** Swiggy generates embeddings for restaurants, and is extending the idea to customers, food-items, and even delivery partners. It also generates **embeddings for city and for the time-slot of the order**, and feeds those into the memorization layer. So "1pm in Koramangala" becomes a dense vector the model can reason with, and the model learns that the 1pm-Koramangala context leans toward quick office lunches, not leisurely weekend brunches.
- **A Lattice head for domain knowledge.** Features like recency and similarity are fed into a **Lattice** layer (TensorFlow Lattice), which lets engineers inject monotonic domain rules directly ("more recent orders from a place should never decrease its score, all else equal"). This bakes in common sense the raw data might otherwise fight.

Swiggy computed NDCG (the standard ranking-quality metric, which rewards putting relevant items near the top) for each generation, GBT vs a pairwise-loss DNN vs the Wide-and-Deep-plus-Lattice model, and reported that **the Wide-and-Deep with the Lattice layer performed substantially better** than the others (confirmed).

**The features that go in (confirmed):** dish and restaurant relevance to this user, restaurant features (rating, cuisine, price band), popularity features (how much the neighborhood orders this place), distance of the user from the restaurant, and predicted delivery time. Note that the serviceability engine's output, the predicted delivery time, is itself an input feature to ranking. The two engines are chained: matching produces the ETA, ranking consumes it.

**Multi-objective: taste is not the only goal (confirmed).** This is the subtle, important part. If you rank purely on "what will Ananya click," you will over-serve popular far-away places and hammer the delivery network, producing late orders and unhappy customers. So Swiggy optimizes **multiple objectives at once: relevance AND delivery experience**, combined with **linear scalarization** (a weighted sum of the objective scores). In plain terms: a restaurant Ananya would love but that is 6km away in heavy traffic gets pushed down a bit because the delivery experience would be poor, even though the taste match is high. The ranking is deliberately not a pure popularity contest. It is taste balanced against "will this order actually go well."

**A concrete walk of Ananya's 1:05pm feed.** The candidate set has ~300 deliverable restaurants. For each, the model reads: Ananya's history (she has ordered eatfit 6 times, a Udupi place 4 times, never Meghana), the restaurant's rating and popularity, the distance and the freshly predicted ETA, current offers, and the 1pm-Koramangala context embedding. The Wide-and-Deep-plus-Lattice model scores all 300. eatfit (loved, 2.1km, 30 min ETA) scores high on both taste and delivery, lands at rank 1. The Udupi place lands rank 2. Meghana (4.4 rating, very popular, but 3.8km and a 45-min ETA at peak, and no personal history) scores well on popularity but is dragged down by delivery experience and weak personal fit, lands at rank 12, below the fold. For Rohan, whose history is stuffed with Meghana orders, the wide memorization side fires and Meghana jumps to rank 2. Same candidate pool, same minute, different sort, because the personalization features differ.

### 7c. Where the sorting happens

The sort happens **server-side**, not on the phone. The phone never receives 300 restaurants and sorts them. It receives an already-ordered slice (the first screen plus a buffer) and paints it in order. This matters because the scoring needs the model, the embeddings, the feature store, and the freshest serviceability numbers, none of which live on the phone. The phone's only job is to render and to log what got tapped. This is the offline-think / online-lookup spine that runs through half this ledger: the expensive learning (training the ranking model on billions of past orders) happens offline, and the live request is a comparatively cheap "fetch candidates, look up features, score, sort" pass.

### 7d. The scale story at three tiers

The catalog here is not huge the way Spotify's 100M tracks are. Swiggy's whole platform is about 233,600 monthly transacting restaurants across 680+ cities (confirmed, Q2 FY25). For one user at one point, the candidate set is only a few hundred. So the thing that explodes at scale is not the number of restaurants. It is **the number of personalized rankings you have to compute per second**, and the **freshness of the features** behind each one. Scaling this is about spreading compute and precomputing features, not storing more rows.

**Tier 1, about 1,000 restaurants in a small city (early Swiggy, one town).** Anything works. You can fetch every deliverable restaurant, score them with a simple model or even a hand-tuned formula, and sort, all in one request, and nobody notices. Do not over-engineer. A single service reading a single database is fine. The candidate set is small and the QPS is low.

**Tier 2, about 100,000 restaurants, tens of cities, millions of users.** Two things break. First, computing features live per request gets expensive: you cannot run a database query for "Ananya's last 90 days of orders with this cuisine" inside the 200ms budget, for every restaurant, for every open of the app. The fix is a **feature store**: precompute user features (her cuisine affinities, her recency vectors) and restaurant features (rolling popularity, rating, average prep time) offline or in near-real-time, and keep them in a fast key-value store so the live request is a set of O(1) lookups, not a set of joins. Second, the serviceability PIP-against-thousands-of-polygons cost bites, which is exactly why the GeoHash prefix index exists, to turn a scan into a keyed lookup. You also start **caching the candidate set per location cell** with a short TTL, because everyone in the same GeoHash cell at the same minute shares the same deliverable-restaurant set even though their personalized sort differs. Compute the expensive per-user sort per user, but share the cheap per-location candidate list across users.

**Tier 3, 200,000-plus restaurants, 680+ cities, 17M monthly users, 230M orders a quarter, with the 1pm and 8pm peaks (today's Swiggy).** Now the load is the ranking QPS at peak and the freshness of the serviceability signals under a demand wall. The survivors:

- **Two-stage ranking to bound cost.** You do not run the heavy neural Wide-and-Deep model on all several hundred candidates if you can avoid it. The standard pattern (inference, matching Swiggy's stated direction of embeddings plus a two-tower style final layer) is: a cheap first stage (embeddings-based retrieval, an Approximate Nearest Neighbor lookup over restaurant and user embeddings, or a light model) narrows a few hundred candidates to a top few dozen, then the expensive model re-scores only those. Cost stays bounded no matter how complex the final model gets, because the expensive model only ever sees a small shortlist. This is the same "fast retrieve, then expensive re-rank a shortlist" spine as Google's featured snippets (2026-08-20) and Instagram Reels (2026-08-07).
- **Geo-sharding.** Koramangala's ranking traffic is handled by infrastructure near Koramangala's data, not by one global service. Each city or zone is a mostly independent problem (a Bangalore user is never ranked against a Delhi restaurant), so you shard by geography and run many small parallel ranking services instead of one giant one.
- **The demand-wall defense.** At 1pm and 8pm the serviceability FSM does the load-shedding: it widens ETAs, adds surge, and shrinks radii per zone. This is not just a customer-facing honesty move, it is backpressure. By marking the far edges of a stressed zone unserviceable, the FSM shrinks the candidate set, which reduces both the delivery-network load and the ranking compute at exactly the moment both are most strained. During a correlated shock (heavy Bangalore rain at 8pm, delivery partners scarce everywhere at once), the radius shrinks hard rather than the app lying about delivery times.
- **Feature freshness under load.** Popularity and prep-time features need to reflect the last few minutes, not last week (a kitchen that just got slammed should sink in the ranking now). This pushes toward a near-real-time feature pipeline feeding the store, so the "predicted delivery time" and "current popularity" features are minutes-fresh, while the heavy model weights are trained offline on the long history.

The pattern across tiers: the restaurant count barely matters, the personalized-ranking QPS and the feature freshness are what explode, so you precompute features into a fast store, share the per-location candidate set while personalizing the sort, cut the expensive model down to a shortlist with cheap retrieval first, and shard by city.

### 7e. Confirmed vs inference, stated plainly

Confirmed from Swiggy's own engineering blog and financial disclosures: the serviceability stages (polygon PIP, GeoHash index, route distance, predicted delivery time, the stress FSM), the LTR progression (pointwise to pairwise, pairwise-loss-works-best), GBDT on Spark MLlib, the Wide-and-Deep-plus-Lattice model beating the others on NDCG, city and time-slot embeddings, entity embeddings for restaurants/customers/food-items/delivery-partners, the multi-objective relevance-plus-delivery-experience goal via linear scalarization, the feature families, and the scale numbers (17.1M MTU, 230M quarterly orders, 233,600 restaurants, 680+ cities). Clearly labeled inference: the exact two-stage candidate-generation-then-rerank serving pipeline, the ANN/two-tower retrieval as the cheap first stage, the per-location candidate caching, the exact latency budget and shard boundaries. Those follow the standard solution for this class of problem and Swiggy's stated direction, but Swiggy has not published the precise serving diagram.

## 8. The retention and habit mechanic

The home feed is Swiggy's core habit engine, and it works on a simple loop: **open the app hungry, see something good in the first six cards, tap, eat, feel good, come back tomorrow.** The ranking model exists to make that first-screen hit rate as high as possible, because the feed is the funnel. If Ananya finds her lunch in ten seconds, she does not open Zomato to compare. The feed's job is to end the decision before a competitor gets a turn.

The metric this moves is primarily **conversion** (opens that turn into orders) feeding **retention** (the habit of reaching for Swiggy first). It shows up in Swiggy's numbers: 17.1 million monthly transacting users placing 230 million orders a quarter is roughly 13-14 orders per user per quarter, more than one a week. That cadence is a reflex, and the reflex is protected by the feed showing you your usuals near the top (Rohan's Meghana at rank 2) while still surfacing enough freshness that the app never feels stale.

The real observed mechanic beyond ranking: Swiggy leans on the same "make the app feel alive" nudges as its rivals, rotating home-screen category shortcuts and offer tiles by time and mood (a "Healthy" or "Salads" nudge at 1pm, "Ice cream" and "Desserts" at 10pm), and injecting a bit of exploration into the ranked feed so a loyal user occasionally discovers a new place. That deliberate exploration is not noise, it is retention insurance: a feed that only ever shows your three regulars eventually gets boring, and boredom is churn. Swiggy has publicly said the next iterations aim at customer-segment-specific models and more real-time in-session intent signals, precisely to keep the first screen feeling personal and fresh rather than repetitive.

The honesty discipline is part of retention too. Because the feed only shows serviceable restaurants with realistic ETAs (the FSM degrading the promise under stress rather than breaking it), Ananya learns to trust the "35 min" on the card. A feed that showed her a great place that then failed at checkout, or promised 30 and delivered 60, would train the opposite reflex: distrust, then churn. The ranking is only as valuable as the serviceability truth underneath it.

## 9. The lesson for Rare.lab

Rare.lab is an AI shader and visual-effects product: a node-based editor that compiles to shippable code, plus an embeddable runtime. The lesson from Swiggy's feed is about **how to order a large set of candidates for a specific user or context, fast, without ever sorting on the client, and without letting relevance be the only goal.**

Three concrete moves:

1. **Split the gate from the sort, and run the gate first.** When Rare.lab shows a user a ranked set (effect templates in the editor's browser, shader presets, community nodes, "effects like this one"), do serviceability-style filtering before ranking: cut to the candidates that will actually work for this user's target this minute (their GPU tier, their target platform, WebGL vs WebGPU, mobile vs desktop, their license tier). Showing a gorgeous compute-shader preset to someone on a WebGL-only mobile target is Swiggy's "great restaurant that cannot deliver here", a guaranteed disappointment at the equivalent of checkout (compile or runtime failure). Gate on real deliverability first, using a cheap keyed lookup (a capability hash, the GeoHash-index trick) rather than testing every candidate at rank time. Then rank only the survivors.

2. **Rank on a multi-objective score, not just relevance, via linear scalarization.** Do not order the template browser purely by "most likely to be picked." Order it by a weighted blend of relevance AND runtime cost, exactly as Swiggy blends taste with delivery experience. An effect the user would love but that would blow their frame budget on their stated target (the 6km-in-traffic restaurant) should be pushed down in favor of a nearly-as-loved effect that will actually hit 60fps on their device. Bake the performance objective into the ranking so the default surfaced choices are the ones that will ship well, not just the ones that look best in a thumbnail. This turns your ranking into a scalability feature, not just a discovery feature.

3. **Precompute the expensive thinking offline, keep the live path a shortlist re-score, and never sort on the client.** Train ranking models and compute per-user and per-effect embeddings and features offline, keep them in a fast feature store, and make the live request a cheap retrieval (ANN over embeddings to a shortlist) followed by an expensive model on just that shortlist, then sort server-side and hand the client an already-ordered list. The embedded runtime and the editor should never receive the full catalog and sort it themselves, they get the answer. As the number of users and contexts explodes (the QPS, not the catalog, is what grows, just like Swiggy), this two-stage, precompute-and-lookup, shard-by-context shape is what keeps the per-request cost flat while the audience grows.

One line: gate candidates by real deliverability before you rank them, rank on relevance blended with runtime cost (not relevance alone), and keep the live path a cheap shortlist re-score over precomputed features with the sort done server-side, so surfacing scales with your audience instead of melting under it.

---

## Sources

- Jairaj Sathyanarayana, "Evolution of and experiments with feed ranking at Swiggy," Swiggy Bytes tech blog: https://bytes.swiggy.com/evolution-of-and-experiments-with-feed-ranking-at-swiggy-17204769e79f
- Ashay Tamhane, Jagrati Agrawal, Rutvik Vijjali, Akash Deep, "Learning To Rank Restaurants," Swiggy Bytes tech blog: https://bytes.swiggy.com/learning-to-rank-restaurants-c6a69ba4b330
- Somsubhra Bairi, "Designing the Serviceability Platform at Swiggy for High Scale, Part 1," Swiggy Bytes tech blog: https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-1-751a631f0379
- Somsubhra Bairi, "What Serviceability means at Swiggy?," Swiggy Bytes tech blog: https://bytes.swiggy.com/what-serviceability-means-at-swiggy-c94c1aad352a
- "Personalized Restaurant Ranking with a Two-Tower Embedding Variant," Towards Data Science: https://towardsdatascience.com/personalized-restaurant-ranking-with-a-two-tower-embedding-variant/
- Ramkishore Saravanan, "Real-time ML Ranking for Autocomplete: Deploying Learning-to-Rank inside OpenSearch," Swiggy Bytes tech blog: https://bytes.swiggy.com/real-time-ml-ranking-in-autocomplete-part-1-3cdbbd44f85a
- Swiggy Q2 FY25 results press release (17.1M MTU, 230M orders, 233,600 restaurants): https://www.swiggy.com/corporate/wp-content/uploads/2024/12/Swiggy_Press-release_Q2FY25-results.pdf
- Medianama, "Active Swiggy users surge to 17.1 million in first post-IPO earnings": https://www.medianama.com/2024/12/223-swiggy-earnings-q2fy25-active-users-increase-to-17-million-yoy/
- Ben Feifke, "Geospatial Indexing Explained: A Comparison of Geohash, S2, and H3": https://benfeifke.com/posts/geospatial-indexing-explained/
</content>
</invoke>
