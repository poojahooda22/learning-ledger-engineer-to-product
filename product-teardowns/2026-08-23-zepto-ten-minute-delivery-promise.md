# Zepto: the "delivered in 10 minutes" promise

Date: 2026-08-23
Product: Zepto (Indian quick-commerce grocery app)
Feature: The delivery-time promise. The "Delivery in 8 mins" line you see before you order, and the machinery that actually makes the packet of Maggi show up that fast.

A note on scope. The 2026-06-17 teardown in this ledger covered Zepto's dark-store inventory and order routing: where the stock physically lives and which store owns your pincode. This report is about a different thing that sits on top of that: the number in minutes. How Zepto computes the promise before you have paid, why that number is a prediction and not a countdown, and how a real order gets picked, packed, assigned to a rider, and clubbed with a neighbour's order so the promise holds. Different question, different machinery.

---

## 1. The user

It is 9:40 on a Tuesday night. Anjali is in her flat in Powai, Mumbai. She is cooking, the onions are already in the pan, and she just discovered the coriander is finished and there is no salt in the spare packet either. The nearest kirana is a 12-minute walk and it is raining. She opens Zepto. Before she has typed anything, the top of the screen already says "Delivery in 9 mins" for her address. She searches coriander, adds one bunch and a 1 kg salt pack, taps Pay, and goes back to the onions. The doorbell rings while the onions are still soft. That number at the top, "9 mins", is the entire product. Everything else is groceries.

The concrete stakes: the coriander has to arrive before the dish is ruined. Anjali is not buying groceries in the weekly-list sense. She is buying against a timer that is already running on her stove.

---

## 2. The real problem

Here is the honest version, the way you would tell a friend.

Normal grocery delivery makes you plan. You order in the afternoon for an evening slot, or you order tonight for tomorrow morning. That is fine for a monthly stock-up. It is useless for the thing you just ran out of. When you are mid-recipe and the coriander is gone, a two-hour slot is the same as no delivery at all. You will walk to the shop or you will skip the coriander.

So the pain is not "I want groceries." The pain is "I want this one small thing, now, and the cost of walking out in the rain is higher than the thing is worth." The old model priced that convenience out of reach because it could not promise a time you could trust. If the app said "10 minutes" and it took 40, you would never believe it again, and you would go back to the walk.

The promise is the product, and a promise you cannot keep is worse than no promise. That is the real problem: not delivering fast, but being able to say a number out loud, before payment, that turns out to be true almost every time.

---

## 3. The feature in one sentence

Before you order, Zepto shows a delivery time for your exact location that it predicts it can hit, and behind that number it runs a pick-pack-assign-and-club pipeline tuned to make the prediction come true.

---

## 4. Jobs to be done

What is Anjali really hiring that "9 mins" line to do?

- Tell me the truth I can act on right now. Not a marketing "fast", an actual number I can weigh against the walk in the rain.
- Let me not plan. Remove the whole mental step of choosing a slot. If it is always about ten minutes, I never think about timing again.
- Protect the dish on the stove. Get the coriander here before the onions burn.
- Make the small order feel allowed. One bunch of coriander should be as normal to order as a full trolley.
- Stay believable. If it says nine and delivers nine, I trust the next nine. The number has to be a habit, not a gamble.

The deepest job: kill the deliberation. A trustworthy number means there is nothing to decide.

---

## 5. How it works for the user

The visible experience is almost nothing, which is the point.

1. Open the app. The header already reads "Delivery in 9 mins" for the saved address. No search, no tap needed for the number to appear.
2. Add items. The number usually stays the same as you fill the cart. It is a property of your location and the store's current state, not of your specific basket, most of the time.
3. Pay. The moment you pay, the number becomes a live tracker: "Packing your order", then "Rider on the way", then a small map with a dot moving toward you.
4. Doorbell. Total elapsed time lands near the promise. Occasionally it is faster. When the area is slammed (heavy rain, dinner rush), the up-front number quietly rises to "16 mins" or the store shows "Delivery unavailable", so the promise stays honest instead of lying and failing.

The user never sees the store selection, the picker route, the rider assignment, or the fact that their coriander shared a scooter with a neighbour's ice cream. They see a number and a doorbell.

---

## 6. The actual flow, step by step

Walking one real order end to end. Anjali, Powai, one bunch of coriander plus 1 kg salt.

1. App open. The client sends Anjali's location (lat, long from GPS, or the saved-address geocode) to the backend.
2. Serviceability check. The backend maps that location to a serviceability polygon and to the dark store that owns it. Powai maps to, say, store BOM-PWI-03, roughly 1.5 km away. (This mapping is the 2026-06-17 report's territory; here it is just the first hop.)
3. Promise computation. Before any item is chosen, the backend asks the ETA service: given this store's current load, current rider availability, current traffic, and this delivery distance, what time can we promise? It returns 9 minutes. The header renders it.
4. Browse and add. Coriander and salt go into the cart. Each add checks live stock at BOM-PWI-03. Both are in stock, so the number holds.
5. Pay. Payment clears (UPI deemed-success or card auth). The order becomes a real work item with an id.
6. Stock decrement. The two units are atomically decremented from BOM-PWI-03's inventory so two people cannot be promised the last salt pack. (Contested-inventory decrement, again from the earlier report.)
7. Picking. The order lands in the store's picking queue. A picker's handheld shows a route through the aisles: salt in aisle 4, coriander in the chiller by the door. The layout is fixed and memorised, so picking is seconds, not a hunt.
8. Assignment and clubbing. In parallel with picking, the dispatch engine looks for a rider. It may club Anjali's order with a second Powai order dropping 300 m away, if that does not blow either promise. It assigns rider R-217, who is already near the store finishing a return.
9. Handoff. Picker bags both orders, hands them to R-217. The app flips to "Rider on the way" and starts the moving dot.
10. Ride. R-217 rides the 1.5 km on the pre-suggested route, drops the neighbour first, then Anjali.
11. Doorbell at about minute 8. The promise made at minute 0, before payment, held.

Two things to notice. The promise is computed before step 5, on almost no information about the basket. And the "countdown" the user watches after payment is not what generated the promise. The promise was a prediction; the tracker is a live readout of a prediction already made.

---

## 7. Under the hood, like the engineer

This is the heart. The 10-minute promise is really three engineering problems wearing one number: predict a time you can commit to, decide who delivers, and keep both cheap enough to answer for lakhs of people at once. Zepto's exact internals are not fully public, so below I separate the confirmed public facts from the clearly-labeled inference about how this class of system is built. The inference is grounded in published engineering from DoorDash, Instacart, and Uber, who solve the identical class of problem and who do publish.

### 7a. The promise is a prediction, so it is a regression, not a clock

Confirmed (public reporting on Zepto's data science): the up-front ETA is treated as a regression problem. The model takes features like distance to the customer, current traffic, historical delivery patterns for that store and time, and the rider's typical speed, and predicts a delivery time using models in the gradient-boosted-trees family (random forests, gradient boosting) rather than a fixed rule.

Why regression and not "distance divided by speed". A flat formula cannot know that BOM-PWI-03 has only two riders free right now, that it is raining, that 9:40pm is a rush spike, or that this store's pickers are slower on weekends. All of those move the real delivery time by minutes. A learned model eats those features and outputs a number that already accounts for them. That is why the header can say 9 tonight and 14 during Sunday lunch, from the same store, same distance.

The right mental model, and the one DoorDash publishes for the same problem, is to break the delivery into stages and predict each, because each stage depends on different features:

- Time to assign a rider (depends on how many riders are free near the store).
- Picking and packing time inside the store (depends on store load, number of items, staffing).
- Rider travel to the customer (depends on distance, traffic, rain).
- A safety buffer.

The promise is the sum of the stage predictions plus buffer. Splitting it this way is the key move: each stage is more predictable on its own than the whole, and "pickup readiness" (store load) is a very different signal from "road speed" (traffic). DoorDash's ETA work uses exactly this staged decomposition, and more recently a multi-task mixture-of-experts model with probabilistic forecasts, so the system predicts a distribution and not just a point. That matters, because you should promise a time you beat 90% of the time, not the average time. If you promise the mean, you are late half the time by definition.

Concrete example. For Anjali's order the model might predict: assign 40s, pick+pack 3 min (two items, store moderately busy), travel 3.5 min (1.5 km, light night traffic, wet roads), buffer 1.5 min. Sum: about 8.6 min, shown as 9.

The data structure here is boring on purpose: a fixed-length feature vector per request, fed to a tree ensemble, returning one float. The intelligence is in the features and the training data (millions of past deliveries per city), not in a fancy structure. Boring and fast is the correct choice when you must answer in tens of milliseconds on the home-screen load.

Inference, clearly labeled: I am inferring the exact staged breakdown and the buffer-as-percentile choice for Zepto specifically. What is public is the regression framing and the feature families. The staging and the "promise a high percentile" discipline are how DoorDash and Instacart describe solving the same problem, and Zepto's numbers only make sense if they do something equivalent.

### 7b. Who delivers: dispatch is a matching problem, then a routing problem

Once the order is paid and being picked, a second engine decides which rider carries it. This is the classic food and grocery dispatch problem, and its shape is well documented.

Confirmed for the class (DoorDash engineering, published): the first-generation dispatch is a bipartite matching problem. On one side, ready-or-nearly-ready orders. On the other, available riders. Each order-rider pair has a cost (mostly extra time). You want the assignment that minimises total cost. The textbook exact solver for that is the Hungarian algorithm, which finds the minimum-cost perfect matching in a bipartite graph in O(n^3). DoorDash states plainly that its earlier dispatch framed one-delivery-per-route as bipartite matching solved with the Hungarian algorithm.

Why a graph and the Hungarian algorithm and not just "give it to the nearest rider". Greedy-nearest is locally smart and globally dumb. If rider R-217 is nearest to both order A and order B, greedy hands R-217 to A and leaves B stranded to a far rider, when the better global answer was R-217 to B and a slightly-farther rider to A. Minimum-cost matching over the whole bipartite graph optimises the sum across all pairs at once, so nobody gets stranded to make one order look good. The graph is the data structure; the cost matrix is its weighted edges; the Hungarian algorithm is the solver.

The clubbing (batching) upgrade. Quick commerce lives on clubbing two or three nearby orders onto one rider trip, because that is what makes the unit economics survive at a sub-15-minute promise. The moment you allow more than one order per trip, the problem stops being clean bipartite matching and becomes a small vehicle-routing problem (VRP): for each candidate bundle you must also decide the drop order and route, and VRP is NP-hard, so you cannot solve it exactly at speed. DoorDash's published answer, which is representative of the class, is a ruin-and-recreate heuristic run under a tight time budget with multithreading: start from a decent assignment, tear out part of it, rebuild that part better, repeat until the clock says stop, keep the best found. You are not proving the optimum; you are getting a very good answer inside a few hundred milliseconds, over and over.

The clubbing constraint that protects the promise: a bundle is only allowed if it keeps every order in it inside its promised time. Anjali's coriander can share R-217's scooter with the neighbour's ice cream only because dropping the neighbour first still lands Anjali near minute 8. If clubbing would push her to minute 13, the bundle is rejected and she rides alone. The promise from section 7a is a hard constraint on the optimiser in 7b. That coupling is the whole trick: the prediction is not just displayed, it is fed back in as a rule the dispatcher must obey.

Confirmed for Zepto specifically (public reporting): the dispatch solves rider allocation (using signals like proximity and even scooter battery level) together with traffic modeling, and it will shrink the serviceable radius in real time if a road clogs, rather than keep promising a time it can no longer hit. That last behaviour is the honest-number discipline made physical: when the machine cannot keep the promise, it stops making it (the store shows "unavailable" or the number climbs) instead of lying.

### 7c. Geospatial: turning "where" into a fast lookup

To find riders near a store, and to map a customer to a store, you need proximity queries over a moving set of points, thousands of times a second. You do not scan every rider and compute a distance; that is O(riders) per query and dies fast.

Inference, grounded in Uber's published H3 work (already covered in this ledger's 2026-06-14 surge report): the standard solution is a geospatial index. Cover the map in a grid of cells (Uber uses H3 hexagons; a geohash grid is the simpler cousin). Bucket every rider by the cell they are in. To find riders near store BOM-PWI-03, you look up the store's cell and its ring of neighbour cells and read those buckets only. Cost becomes proportional to the number of nearby riders, not the total fleet. Rider location updates are just "move this rider id from bucket X to bucket Y", an O(1) hash-map operation. The store-to-customer serviceability check is the same idea: precomputed polygons and cell lookups, not live geometry per request.

Hexagons over squares for the same reason the surge report gave: all six neighbours of a hex are equidistant and share a full edge, so "expand the search ring" and "measure travel between adjacent cells" behave uniformly, which square grids (with their awkward diagonal neighbours) do not.

### 7d. The scale story, at three tiers

This is where the design choices earn their keep. Same feature, three sizes.

Tier 1: one dark store, about 1,000 orders a day. Everything is easy. One store's inventory fits in memory. The ETA model is a single service call. Dispatch has a handful of riders, so even a naive nearest-rider assignment mostly works, and the Hungarian algorithm over a 10x10 cost matrix is instant. You could almost run this on one box. Nothing is interesting yet. Example: a single pilot store in one Mumbai neighbourhood at launch.

Tier 2: one city, dozens of stores, about 100,000 orders a day, sharp evening peaks. Now things break.
- The hot spike. Demand is not flat; 7pm to 10pm is a wall. A model trained on daily averages will promise 9 minutes at 8:30pm and miss, because it did not see the rush coming. Survival: demand forecasting. Confirmed for Zepto, the system uses time-series models (ARIMA, Prophet, LSTM) on historical order data to predict spikes by store and hour, so riders and pickers are pre-positioned before the wall hits, and the promise for that window is set from predicted load, not current load.
- The dispatch load. Assignment now runs continuously across many stores and hundreds of riders. A single Hungarian solve over the whole city is too big and too slow, and most pairings are pointless (a rider in one suburb will never serve a store 8 km away). Survival: geo-sharding. Partition the problem by area (the H3 / geohash cells from 7c) and run many small assignment solves in parallel, one per neighbourhood cluster, instead of one giant one. Each solve is small, so it is fast, and they run concurrently.
- The read load. Lakhs of home-screen opens want the ETA number. Recomputing a fresh regression for every open is wasteful, because the answer barely changes second to second for a given store. Survival: cache the promise per store (and per coarse distance band) with a short TTL of a few seconds, and let almost every home-screen open be a cheap cached read. The expensive model runs a few times a second per store, not once per user.

Tier 3: national, 1,000-plus dark stores, 10 million-plus orders, monsoon and festival surges. New things break again.
- Correlated shocks. Heavy rain in Mumbai does not slow one order; it slows every order in the city at once, collapses rider availability, and clogs every road together. A per-order model that treats delays as independent will keep promising times it cannot keep, city-wide, in the exact moment it matters most. Survival: the real-time radius shrink and store "unavailable" toggle (confirmed behaviour). The system deliberately narrows what it promises, and to whom, so the promises it does make stay true. It sheds load rather than degrading the number everyone trusts.
- The single-store hot partition on a sale. A festival sale on one popular store can pin one shard while the rest of the country idles. Survival: this is why the promise is decoupled per store and cached per store. One store going red does not stall Anjali's unrelated Powai store, because their promises and their inventory decrements live on different shards.
- Fairness and freshness under queueing. When orders pile up faster than pickers clear them, the picking queue itself becomes the bottleneck, and the ETA model must read live queue depth as a feature or it will lie. Survival: feed store-load and queue-depth signals straight into the promise (section 7a), so a backed-up store automatically quotes 15 minutes instead of 9 and stays honest.

The through-line across all three tiers is the same as several features in this ledger: do the expensive thinking off the hot path (train the ETA model offline, forecast demand ahead of time, precompute serviceability polygons and store-cell maps), and keep the live per-user request as a cheap cached lookup plus a small constrained optimisation. The home-screen "9 mins" is mostly a cache read. The genius is in what was computed before you opened the app.

---

## 8. The retention and habit mechanic

The loop is trust compounding into reflex.

The mechanic is not a streak or a badge. It is reliability itself. Every time the number says 9 and the doorbell rings at 8, Anjali's brain files one more data point: this number is true. After a dozen true numbers, she stops evaluating it. "I ran out of X" now maps directly to "open Zepto", with no deliberation in between, because the deliberation (is it worth it, will it actually be fast) has been answered so many times that it collapses. That is the definition of a habit: a cue (ran out) firing a behaviour (open app) without conscious cost-benefit each time.

Which metric it moves: primarily retention, then frequency and revenue. Activation is the first believed promise; the habit is the retention engine. Quick commerce's whole business case rests on order frequency, and frequency only climbs if the promise is trusted enough to be used for tiny, spontaneous, low-stakes orders (one coriander, one ice cream at 11pm) that no planned-slot service would ever capture. Each of those tiny orders is unprofitable alone; the clubbing engine in 7b is what makes them collectively pay, which is why the batching constraint (never break a promise to save a rupee) and the retention loop (never break a promise or lose the habit) are the same rule seen from two sides.

The honest-number discipline is the retention mechanic doing defense. Showing "16 mins" during rain, or "unavailable", feels like a worse experience in the moment. It is the single most retention-protective thing the system does, because one loudly-broken promise costs more trust than ten quietly-honest higher numbers. The feature guards its own habit by refusing to lie.

Real observed example of the same principle across the category: quick-commerce apps compete on the displayed minutes more than on price, and will raise the quoted time or gray out a store under load rather than let the doorbell arrive long after the promise. The number on the home screen is the product they are actually defending.

---

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into a shader and runs it in an embeddable runtime. The Zepto lesson is about the promise, not the pizza: commit to a number the user can trust, before the work is done, and make the number a hard constraint on the engine, not just a label on the output.

Concretely, three moves:

1. Show a predicted cost before the user commits, and predict it as a staged sum. When a designer wires up a node or drops in an effect, Rare.lab should display a predicted runtime cost for the target device before they hit compile: "about 4.1 ms/frame on a mid-range phone". Predict it the way Zepto predicts delivery: not one flat formula, but a sum of per-stage estimates (this many texture samples, this many ALU ops, this much overdraw, this dependency chain depth), each learned or measured from real runs on real hardware. A staged prediction is both more accurate and more explainable ("your blur pass is 2.3 of the 4.1 ms"), exactly as DoorDash's staged ETA is more accurate than one number.

2. Promise the high percentile, not the average. Zepto promises a time it beats most of the time, then buffers. Rare.lab should quote a frame-time budget it hits on the P90 device in the target range, not the average GPU, because a shader that averages 60fps but stutters to 30fps on a quarter of phones has broken its promise where it hurts. Predict a distribution across the device population, show the pessimistic-but-true number, and let the honest number steer the design.

3. Make the predicted budget a hard constraint the compiler must obey, and shed gracefully when it cannot. This is the deepest Zepto move: the ETA is fed back into the dispatcher as a rule (no bundle may break a promise), and when the system cannot keep the promise it shrinks the radius instead of lying. Rare.lab's runtime should do the same. Let the user set a frame budget (say 6 ms), treat it as a constraint on compilation and on runtime, and when a scene cannot hold it on a given device, degrade deliberately and visibly (drop an optional pass, lower an effect's sample count, reduce resolution on the heavy layer) rather than silently dropping frames. Ship a real-time quality governor that narrows what it promises under load, like Zepto shrinking its serviceable radius in the rain, so the runtime keeps the frame-rate promise it displayed instead of quietly missing it. On the shader side as much as the grocery side, the trustworthy number, defended by graceful degradation, is the product.

---

## Sources

- Zepto data science and 10-minute delivery, overview of ETA-as-regression, demand forecasting (ARIMA/Prophet/LSTM), and dispatch: Analytics Vidhya, "The Data Science Behind Zepto's 10-Minute Delivery Success" (2025). https://www.analyticsvidhya.com/blog/2025/10/zepto-data-science/
- Zepto dark-store model, store size (2,000-4,400 sq ft), 1.5-2 km radius: TechResearchOnline, "How Zepto Delivers Groceries in 10 Minutes." https://techresearchonline.com/blog/zepto-10-minute-delivery-business-model/
- Zepto dark store as a "time engine," founder detail: BigStory, "Zepto's Dark Store: How Kaivalya Vohra Built India's Time Engine." https://www.bigstorynetwork.com/content/zepto-business-model-kaivalya-vohra-playbook
- DoorDash dispatch as bipartite matching solved with the Hungarian algorithm, and the DeepRed engine: DoorDash Engineering, "Using ML and Optimization to Solve DoorDash's Dispatch Problem." https://careersatdoordash.com/blog/using-ml-and-optimization-to-solve-doordashs-dispatch-problem/
- DoorDash next-generation dispatch, batching, and ruin-and-recreate routing under multithreading: DoorDash Engineering, "Next-Generation Optimization for Dasher Dispatch at DoorDash." https://careersatdoordash.com/blog/next-generation-optimization-for-dasher-dispatch-at-doordash/
- DoorDash routing at scale, multithreading and ruin-and-recreate: DoorDash Engineering, "Scaling a routing algorithm using multithreading and ruin-and-recreate." https://careersatdoordash.com/blog/scaling-a-routing-algorithm-using-multithreading-and-ruin-and-recreate/
- DoorDash staged ETA and mixture-of-experts probabilistic forecasting: DoorDash Engineering, "Precision in Motion: Deep learning for smarter ETA predictions." https://careersatdoordash.com/blog/deep-learning-for-smarter-eta-predictions/
- Evolution of food delivery dispatching (greedy to matching to batching/VRP): Ilya Zinkovich, "Evolution of Food Delivery Dispatching." https://ilyazinkovich.github.io/2020/06/16/delivery-dispatching-evolution.html
- H3 hexagonal geospatial index (grounding for the proximity-index inference; also covered in the 2026-06-14 Uber surge teardown in this ledger): Uber Engineering, "H3: Uber's Hexagonal Hierarchical Spatial Index." https://www.uber.com/blog/h3/
