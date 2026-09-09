# Zepto: order picking and SKU slotting inside the dark store

Date: 2026-09-09
Product: Zepto
Feature: How a picker grabs your milk, bread, eggs and Maggi in about a minute. The slotting of items on the shelves, the pick path through the store, and the batching of orders that makes the 10-minute promise physically possible.

A note on scope. Two earlier Zepto teardowns in this ledger looked at the layers around this one. The 2026-06-17 teardown ("dark-store inventory and order routing") covered which store serves you (point in polygon), the atomic stock decrement so the last carton of milk never oversells, and which rider gets the trip. The 2026-08-23 teardown ("the 10-minute promise") covered the ETA shown before you order and the dispatch to the rider. Neither one looked at what happens between "order accepted" and "rider picks it up." That gap is a human walking through a small warehouse with a phone. This teardown is about that walk, and about the invisible offline work that makes the walk short.

---

## 1. The user, and their day

It is 8:12 pm on a Tuesday. Riya just got home from work in Powai, Mumbai. She opens the fridge and it is basically empty. She needs milk for tomorrow's chai, bread for the morning, a six-pack of eggs, and a four-pack of Maggi because she is tired and dinner has to be easy. She is hungry now, not in an hour.

She opens Zepto, taps four items, and the app says "Delivery in 9 minutes." She half believes it. She has believed it before and it has been true, which is exactly why she opened Zepto and not a 40-minute grocery app.

She does not think about a warehouse. She does not think about a person. But somewhere within about 2 to 3 km of her flat, in a shuttered shop with no customers inside, a worker's phone just buzzed with her four items, and a 60-second clock started.

---

## 2. The real problem

Here is the honest version, told like a friend would.

The promise "10 minutes" sounds like it is about the rider on the bike. It is not, mostly. The bike ride from a store 2 km away is 4 to 6 minutes. That leaves only a few minutes for everything that happens before the bike leaves. If the worker inside the store takes 5 minutes to find Riya's four items, the promise is already dead before the rider even touches the throttle.

So the real problem is brutal and physical: a human being has to locate four specific items among a few thousand, walk to each one, grab the right pack (not the wrong flavour, not the expired one), scan it, and hand it off, and all of that has to finish in about 60 to 90 seconds, hundreds of times a day, every day, in a room the size of a small shop.

The pain has three sharp edges:

1. **Walking is the enemy.** In warehouses of every size, the single biggest chunk of a picker's time is not grabbing items, it is walking between them. The classic review of order picking (de Koster, Le-Duc and Roodbergen, 2007) reports that travel is about 50% or more of total picking time, and that order picking overall can be as much as 55% of a warehouse's total operating cost. Every extra step Riya's picker takes is money and a broken promise.

2. **Wrong items break trust harder than slow items.** If the milk shows up 3 minutes late, Riya shrugs. If the app sends her full-cream milk when she wanted toned, or a dented dabba, she stops trusting the whole thing. A mispick is worse than a slow pick.

3. **It has to survive the rush.** At 8 pm on a rainy Tuesday, that one store is not handling Riya's order, it is handling a flood of orders in the same ten minutes, all competing for the same picker's feet and the same last packs of bread.

The feature that solves this is not one algorithm. It is a layout, a route, and a batching trick, all leaning on the same idea the rest of this ledger keeps finding: do the expensive thinking offline, ahead of time, so the live path is a short cheap walk.

---

## 3. The feature in one sentence

When your order lands, the store's software hands a worker a pick list already sorted into the shortest walking route through shelves that were arranged in advance so your most-likely items sit closest to the packing counter, and it often bundles your order with a couple of others so the worker collects them all in one trip.

---

## 4. Jobs to be done

What is Riya really hiring this feature to do? She never sees it, so the jobs are indirect. She is hiring it to:

- **Make "10 minutes" not a lie.** She wants the number on the screen to be real, because that reliability is the entire reason she opened this app instead of cooking with what she had.
- **Send the right thing.** Toned milk, not full cream. The Maggi four-pack, not the single. Eggs that are not cracked.
- **Never make her think about the seams.** She does not want to know about pickers or shelves. She wants groceries to feel like they teleport.

And the store operator is hiring the same feature to do a different job: **get more orders out of the same worker and the same rent.** A picker who packs an order in 60 seconds instead of 180 seconds does three times the work per hour. In a business with famously thin margins, that ratio is the difference between a store that makes money and one that does not.

---

## 5. How it works for the user (the visible experience)

From Riya's side, there is almost nothing to see, and that is the point.

She taps four items. She taps "Place order." A small screen appears: a progress strip that says something like "Packing your order," then "Rider assigned," then "On the way," then a live map with a moving dot (that live map is the 2026-06-24 Swiggy tracking teardown and the 2026-08-08 Uber "where's my car" teardown, the same moving-marker machinery).

The only visible artifact of all the picking work is the speed of that first step. "Packing your order" flips to "Rider assigned" shockingly fast, often within a minute or two. Zepto's co-founder Aadit Palicha has said the operational benchmark is that within 76 seconds of an order being placed, the order is packed and ready for pickup. Not the average, the standard.

Occasionally Riya sees the one place the picking layer leaks into the app: an item marked "out of stock" at checkout, or a rare "item unavailable, refunded" note. That is the inventory truth (the 2026-06-17 atomic-stock teardown) meeting the physical shelf.

---

## 6. The actual flow, step by step

Follow Riya's four items (Amul Taaza toned milk 500 ml, Britannia bread, a six-pack of eggs, Maggi 2-minute noodles four-pack) from tap to bike.

1. **8:12:00 pm. Order placed.** The app sends the four line items to the backend. The store that will serve her is chosen (point in polygon against dark-store service zones, from the 06-17 teardown). Say it is the Powai dark store, 2.1 km away.

2. **8:12:01. Stock reserved.** For each of the four items the system does an atomic check-and-decrement so nobody else can grab her last six-pack of eggs while she is paying (06-17 again). All four are in stock. Good.

3. **8:12:02. Pick list generated and sorted.** This is the new part. The four items are not handed over in the order Riya tapped them. The software looks up where each item lives in this specific store (a shelf address, like "A3-L2" meaning aisle A3, level 2), then sorts the four into the shortest walking sequence through the store. It may also decide to bundle Riya's order with one or two other orders that came in the same few seconds (batching). The result is a digital pick list on a worker's handheld: an ordered checklist with shelf locations and a small map.

4. **8:12:05. Picker starts walking.** A worker near the front grabs the list. Because eggs, bread and milk are all high-velocity items, they have been slotted near the packing counter, so the picker grabs three of the four almost immediately. Maggi is one aisle deeper. The route the phone drew takes the picker down that aisle and back without doubling over.

5. **8:12:05 to 8:13:00. Grab and scan.** At each shelf the picker scans the item barcode. The scan does two jobs: it confirms this is the right SKU (not the full-cream milk sitting next to the toned) and it marks that line item done. A wrong scan beeps. This is the mispick defense.

6. **8:13:10. Pack.** All four items are in a bag or crate at the packing station, which is right where the walk ended because the layout was designed to end there.

7. **8:13:16. Ready for pickup.** About 76 seconds after 8:12:00. The order is now waiting for the rider who was being assigned in parallel (06-17, 08-23). Bike leaves. Riya's live map dot starts moving.

The whole in-store portion is roughly a minute. Everything that made it a minute instead of five was decided before Riya ever opened the app.

---

## 7. Under the hood, like the engineer

This is the heart of it. There are three distinct problems stitched together: **where to put things (slotting)**, **what route to walk (routing)**, and **how many orders to carry at once (batching)**. Two of the three are classic, well-studied computer science problems with real algorithms and real names. Where Zepto's exact internals are not public, I say so and give the well-grounded standard-practice version, labeled as inference.

### 7a. The route is a Traveling Salesman Problem, but a rare solvable one

Start with the picker's walk. The picker must start at the packing station, visit a set of shelf locations, and return, in the least total distance. That is exactly the Traveling Salesman Problem (TSP): given a set of points, find the shortest tour that visits them all.

In general the TSP is NP-hard. If you have 20 items to pick, there are more possible orderings than you could ever check. So you would expect this to be intractable and for stores to just use rough rules of thumb.

Here is the beautiful part. A warehouse is not a random scatter of points. It is a **grid**: parallel aisles, with cross-aisles connecting them only at specific places (often just the two ends). That structure collapses the hard general TSP into a solvable special case.

Ratliff and Rosenthal proved this in 1983, in a paper with a title that says it all: "Order-Picking in a Rectangular Warehouse: A Solvable Case of the Traveling Salesman Problem" (Operations Research 31(3):507-521). They built a sparse graph of the warehouse and gave a **dynamic programming** algorithm that finds the provably shortest pick tour in time that is **linear in the number of aisles**, not exponential in the number of items. On 1983 hardware, a 50-aisle problem solved in about a minute. Roodbergen and de Koster later extended it to warehouses with more cross-aisles (2001).

The data structure is a graph. Nodes are the pick locations and the aisle junctions. Edges are the walkable aisle segments with their lengths. The DP sweeps aisle by aisle, and at each aisle it only needs to remember a tiny number of "states" describing how the partial tour connects across that aisle. That small, bounded state is why the cost grows linearly. It is the same spirit as any good DP: the problem looks exponential, but the number of things you actually need to remember at each step is small.

**Do dark stores run the full Ratliff-Rosenthal DP?** Probably not the exact 1983 algorithm, and this is inference: a Zepto store is tiny (2,000 to 4,400 sq ft) with maybe a dozen short aisles, and a single order is often just 4 to 8 items. At that size even a simple heuristic gets you a near-optimal route. The standard warehouse heuristics (de Koster et al., 2007) are:

- **S-shape / traversal:** enter an aisle that has a pick, walk it all the way through, snake to the next. Simple, avoids backtracking.
- **Return:** enter an aisle, grab, come back out the same end.
- **Largest gap / midpoint / combined:** smarter variants that decide per aisle whether to traverse or return based on where the items sit.

For a 5-item order in a 12-aisle store, an S-shape route sorted by shelf address is within a whisker of optimal and costs almost nothing to compute. So the likely truth: the pick list is **sorted by shelf location into a traversal order**, which is a cheap heuristic sitting on top of the same graph model the exact algorithm uses. The concrete effect for Riya: her four items come back sorted "packing-counter cluster first, then aisle A3," never "milk, then far aisle, then back to the counter for bread."

### 7b. Slotting: the layout is precomputed from order history

Routing decides the path given where items are. **Slotting decides where items are in the first place**, and it is the higher-leverage lever, because a great route through a badly organized store is still slow.

This is the **storage assignment problem**. Two ideas do almost all the work, and both are computed offline from past order data.

**Velocity slotting (ABC analysis).** Rank every SKU by how often it is ordered. The top sellers (the "A" items) go in the golden zone: closest to the packing counter, at waist-to-eye height so no bending or reaching. In a Zepto store the A items are the obvious ones: milk, eggs, bread, bananas, atta, cold drinks, onions, baby products. Put them a few steps from where every tour ends, and the average tour gets dramatically shorter, because most tours include at least one A item. This is why Riya got three of her four items instantly: milk, bread and eggs are all A items sitting near the counter. Multiple operator write-ups confirm high-velocity SKUs are placed closest to the packing zone.

**Affinity slotting (correlated storage assignment).** Look at which items are frequently bought together and put them near each other. This is market-basket association mining, the same co-occurrence idea behind Amazon's "Customers who bought this also bought" (the 2026-07-12 item-to-item teardown), reused for physical geography instead of a recommendation row. If bread and butter, or Maggi and ketchup, or chips and cold drinks, show up in the same cart constantly, slot them adjacent so a two-item trip becomes a one-stop trip. Operator descriptions of dark-store layout explicitly say items ordered together are slotted adjacent.

The data structure underneath slotting is a big offline table: for every SKU, an order-frequency count (velocity) and, for affinity, a co-occurrence matrix or a set of association rules mined from order logs. This is a whole-corpus computation over months of orders. It is exactly the wrong thing to do live. So it runs offline, on a schedule, and produces a simple output: a shelf-address assignment per SKU that the live pick path just reads. Offline think, online lookup, the ledger's oldest refrain (Discover Weekly, YouTube, Amazon, Google's index build).

One more offline input keeps the shelves full: **min-max replenishment**. Every SKU has a minimum level that triggers a restock and a maximum ceiling that prevents overstock and expiry. Demand forecasts (per store, per SKU) feed these thresholds so the fast movers are physically present when the picker arrives. A perfect route to an empty shelf is a failed order.

### 7c. Batching: carry more than one order per trip

The third trick attacks travel directly. If travel is 50% of pick time, the way to beat it is to **not walk the same aisle five times for five separate orders**. Instead, combine several orders that arrived close together into one tour, pick all their items in a single walk, then split the haul into separate bags at the counter. This is the **order batching problem**.

The payoff is large and measured: batch picking typically cuts travel by 40 to 60% versus picking one order at a time, because the fixed cost of walking to a zone is shared across many orders. During the evening rush, when many orders for overlapping items land in the same minute, batching is what keeps the store from falling behind.

The catch is that batching and routing are entangled. The best batch depends on the route, and the best route depends on the batch. The research problem of solving them together is the "joint order batching and picker routing problem," and it is genuinely hard at large scale. For a tiny dark store with small baskets and a hard latency deadline (you cannot make Riya wait while you assemble a fat batch), the practical version is a **greedy, time-boxed batch**: group the handful of orders in a short window whose items cluster in the same aisles, cap the batch size so no single tour gets too long, and send it. This is inference for Zepto specifically, but it is the standard way latency-constrained micro-fulfillment does it, and it fits the confirmed "algorithmic batch picking within 120 seconds" descriptions of the sector.

### 7d. Where the sorting happens

Worth stating plainly, because this ledger keeps making the point: none of this thinking happens on the picker's phone. The phone shows a pre-sorted list. The route, the batch and the slotting were computed on the server (route and batch live at order time, slotting offline in advance). The handheld is a thin display plus a barcode scanner. The scan validates and check-marks; it does not compute the plan. Same shape as server-side search ranking never happening on your phone.

### 7e. The scale story, three tiers

What grows here is not a catalog to search. It is **order volume against a fixed, tiny store, and then the number of stores**. Name what breaks at each tier and what you do about it.

**Tier 1: about 1,000 items, a few dozen orders a day (a single corner kirana with a delivery guy).** Nothing needs optimizing. The one worker has the whole shop memorized. A paper list in the order the customer said it works fine. Building a WMS, a slotting engine and a routing DP here would be pure over-engineering. Real example: your neighborhood shop that delivers. It has no pick-path algorithm and does not need one.

**Tier 2: about 2,500 to 4,400 sq ft, roughly 3,000 SKUs, but hundreds to low thousands of orders a day (one real Zepto dark store).** Now the small numbers multiply. 3,000 SKUs is few enough that the assortment is curated (only high-velocity items, target availability 95 to 98%), but the order rate is high enough that walking waste dominates. This is exactly the tier where the three tricks earn their keep: velocity + affinity slotting to shrink the average tour, sorted pick paths to kill backtracking, batch picking to amortize travel across the rush, min-max replenishment so the golden-zone shelves are never empty. What broke at the jump from Tier 1 was **travel time per order under load**, and the fixes are layout and batching. The 76-second benchmark lives here.

**Tier 3: 10 million-plus orders across the network, hundreds of stores, peaks like a rainy evening or a big-match night (the whole of Zepto).** The scaling trick is the same one Notion used with workspaces and Stripe used with accounts and the 06-17 teardown used with stores: **each dark store is an independent shard.** One store's inventory, pick queue and workers have nothing to do with another's, so the system scales horizontally just by adding stores, with no cross-store coordination on the hot path. The heavy shared work moves offline and central: demand forecasting per store per SKU, and slotting recomputed on a schedule from the whole network's order history, then pushed down to each store as a shelf map. What breaks at this tier is not the software, it is **the human picker's throughput ceiling.** You cannot make a person walk faster. So the network response splits two ways. Most stores stay small and manual and just multiply (more stores closer to more customers, which also shortens the ride). And the largest "metro" dark stores start to **mechanize**: reported 6,000 to 8,000 sq ft nodes stocking up to 25,000 SKUs with conveyors and automated micro-fulfillment, which is the physical-world version of adding hardware when software optimization runs out. Also at this tier the contested-inventory problem from 06-17 gets sharper (many concurrent orders fighting for the last few packs of a hot SKU during a sale), handled by the atomic decrement and sub-counter tricks already covered there. And a quiet feedback loop protects availability: if a store stocks out of a SKU three times in a week, that product drops in the app's search ranking for that pin code, so the catalog you see is quietly shaped by what the store can actually deliver.

---

## 8. The retention and habit mechanic

The loop is simple and it is the whole company: **reliable speed builds a reflex.**

The first time Riya orders and it genuinely arrives in about 9 minutes, a small belief forms. The fifth time, it is a habit. By the twentieth time, "I'll just Zepto it" has replaced "I'll pick it up on the way home" and "I'll add it to the weekly big shop." The 10-minute promise, kept over and over, converts groceries from a planned chore into an impulse. That impulse is order frequency, and order frequency is the metric this whole picking machine exists to protect.

Which metric does the picking feature move? Two, tightly linked:

- **Retention, through trust.** The promise only builds a habit if it is almost always true. Fast, correct picking is the necessary (invisible) condition. This is the same invisible-papercut retention the ledger found in Spotify's loudness normalization and shuffle: the user cannot name why it feels good, they just keep coming back, and the one thing that would break the habit (here, a late or wrong order) is exactly what the barcode scan and the 76-second layout are engineered to prevent. A mispick is the head-turn moment where trust cracks, so the scan gate is a retention feature disguised as a quality check.

- **Revenue / unit economics, through throughput.** The 76-second pick is also what makes the money work. Same rent, same worker, three times the orders out the door because the pick is a minute not three. In quick commerce, where the margin on a grocery basket is razor-thin, that throughput ratio is survival.

The real observed mechanic is the 76-second benchmark itself, held up publicly as the operational standard, plus the stockout-to-ranking penalty: the app actively hides products a store keeps failing to deliver, so what Riya sees is biased toward what will actually show up fast. The system protects the promise by only making promises it can keep.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor that compiles a visual-effects graph to shippable code, plus an embeddable runtime. The dark-store pick is, structurally, the exact problem your runtime's per-frame render loop faces, and most VFX runtimes get it wrong the same way a badly run store does: they walk the store in the order the artist happened to wire the nodes.

The expensive thing on a GPU is not the work, it is the **switches**. Binding a different shader, swapping a texture, changing a render target: these state changes are the picker's walking. A naive runtime that executes 40 effect nodes in graph order re-binds the same noise texture eight times because eight scattered nodes reference it, exactly like a picker who walks back to aisle A3 four separate times because the list was not sorted. The fix is the store's three tricks, one to one:

1. **Slotting equals data locality, computed at compile time.** Lay out resources so co-used ones sit together and hot ones sit where access is cheapest. Pack co-sampled textures into an atlas (affinity slotting: items bought together sit adjacent). Keep the highest-frequency resources (a noise LUT sampled by half your nodes) resident in the fastest memory tier, the runtime's golden zone near the "packing counter." Do this analysis offline in the compiler from the graph itself, the way slotting is computed offline from order history, not per frame.

2. **Routing equals draw-call sorting to minimize state changes.** Your compiler should emit passes sorted by a state key: pipeline/shader first, then material/texture, then geometry, so the GPU never backtracks to re-bind something it already had. All consumers of the noise texture run consecutively: slot it once, visit it once, like the picker clearing an entire aisle before moving on. This is the Ratliff-Rosenthal insight applied to frames: the general ordering problem looks hard, but your graph has structure (a DAG with a bounded set of distinct pipeline states), and that structure makes a cheap near-optimal sort possible.

3. **Batching equals instancing.** One tour for many orders is one draw call for many instances. Where many nodes or many objects share a pipeline, batch them into a single instanced draw so the per-call setup cost is paid once, not N times. Batch picking cuts travel 40 to 60%; instancing cuts draw-call overhead by a similar order, and for the same reason: a fixed cost amortized across many payloads.

And the meta-lesson, the one this ledger keeps landing on: **precompute the plan offline, execute a flat cheap sequence online.** The store computes slotting from months of data and computes the route at order time, so the picker's live job is a 76-second sorted walk with zero decisions. Rare.lab's compiler should do the same: from the node graph, emit a **state-sorted, batched command list** as a build artifact, so the runtime's per-frame job is to replay a pre-sorted sequence, never to re-derive the render order every frame. The artist wiring nodes in a messy order is Riya tapping items in a random order. Neither should ever decide the walk. The compiler is the pick-list generator.

One line to keep: the picker's walk is a Traveling Salesman Problem that a grid makes solvable, but the real speed comes from moving the thinking off the live tour (slot high-velocity and co-ordered SKUs near packing, offline, from order history; batch orders into one trip), so your GPU runtime should likewise slot resources for locality, sort passes by state to kill re-binds, and instance to amortize setup, all decided at compile time so the per-frame path is a flat replay.

---

## Sources

- Ratliff, H.D. and Rosenthal, A.S. (1983). "Order-Picking in a Rectangular Warehouse: A Solvable Case of the Traveling Salesman Problem." Operations Research 31(3):507-521. https://pubsonline.informs.org/doi/abs/10.1287/opre.31.3.507
- de Koster, R., Le-Duc, T., Roodbergen, K.J. (2007). "Design and control of warehouse order picking: A literature review." European Journal of Operational Research 182(2):481-501. PDF: https://repub.eur.nl/pub/7322/ERS%202006%20005%20LIS.pdf
- Semantic Scholar entry for the de Koster et al. review (travel is ~50% of pick time; order picking ~55% of operating cost): https://www.semanticscholar.org/paper/Design-and-control-of-warehouse-order-picking:-A-Koster-Le-Duc/d482423c1f77da795531fed05046d13c9669704b
- "Exact algorithms for the order picking problem" (context on Ratliff-Rosenthal DP, linear in aisles, and later extensions), arXiv: https://arxiv.org/pdf/1703.00699
- "Improved formulations of the joint order batching and picker routing problem," arXiv: https://arxiv.org/pdf/2207.05305
- MetricsCart, "From Order to Delivery: Behind Dark Stores Operations" (optimized picking path, digital pick lists, shelf maps, barcode validation, high-velocity SKUs near packing, items ordered together slotted adjacent, min-max replenishment): https://metricscart.com/insights/dark-store-operations/
- 42Signals, "Deconstructed: How Zepto's 10-Minute Delivery Model Redefined Commerce" (Aadit Palicha's 76-second order-to-packed benchmark; data-driven layout): https://www.42signals.com/blog/zepto-business-model-explained/
- AppsRhino, "How Zepto Delivery Works: The Logic Behind the Speed" (store proximity 2 to 3 km, sub-2-minute picking): https://www.appsrhino.com/blogs/zepto-10-minute-grocery-delivery
- Prozo, "Quick Commerce Fulfilment - Blinkit, Zepto, Swiggy Instamart" (shelf position mapped to picking frequency; micro-picking; mechanized 6,000 to 8,000 sq ft metro stores, 25,000 SKUs; algorithmic batch picking within 120 seconds; stockout-to-ranking penalty; 95 to 98% target availability): https://www.prozo.com/warehousing/quick-commerce/
- OneVisionMedia, "Zepto Case Study" (store size 2,000 to 4,400 sq ft; curated ~2,500 to 3,000 SKUs per store): https://onevisionmedia.in/zepto-business-model-case-study-dark-store-unit-economics/
- NetSuite, "What Is Batch Picking?" and Descartes/Finale guides (batch picking reduces travel 40 to 60%): https://www.netsuite.com/portal/resource/articles/inventory-management/batch-picking.shtml
