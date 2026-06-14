# Uber Surge Pricing

Date: 2026-06-14
Product: Uber
Feature: Surge pricing (the live multiplier that raises a fare when an area runs hot)

---

## 1. The user

Picture Rahul, 31, standing outside a restaurant in Koramangala, Bangalore, at 9:40pm on a Friday. It started raining ten minutes ago. He just finished dinner, his shirt is getting wet, and he wants to go home to HSR Layout, about 6 km away. He opens Uber, types nothing fancy, just taps his saved Home address, and waits for a price.

On a dry Tuesday at 3pm that ride would cost him around 180 rupees. Tonight the app shows 320 rupees and a small lightning bolt with "Prices are higher due to increased demand." He is annoyed, but there are no autos in sight and the rain is getting worse. He taps Confirm. A driver named Suresh accepts in 25 seconds.

That gap between 180 and 320 is surge pricing doing its job in real time, for the exact 400 meter patch of the city Rahul is standing in.

## 2. The real problem

The pain has two sides, and they hit at the same moment.

Rahul's side: "It is raining, it is late, and everyone around me wants a cab too. I do not want to stand here for 20 minutes watching 'finding your driver' spin and then time out. I would rather pay more and just go home now."

The platform's side, which Rahul never sees: in that one wet patch of Koramangala, 200 people just opened the app in five minutes, and there are only 12 free drivers nearby. If the price stays at 180, all 200 requests chase those 12 cars. 188 people get nothing but a spinner and a "no cars available." The 12 drivers who are 3 km away in dry, quiet Indiranagar have no reason to drive into the rain and traffic for a normal fare.

So the real problem surge solves is not "make Uber more money tonight," even though it does. It is a matching failure. Demand spiked in one tiny area, supply did not, and with a fixed price the market just jams. Described like a friend would: surge is the app's way of shouting "this corner is on fire, drivers come here, riders who really need it pay a bit more, everyone else wait ten minutes for it to cool down."

## 3. The feature in one sentence

When riders in a small area suddenly outnumber the free drivers there, Uber raises the price in that exact area until the two sides rebalance, and it shows the rider the higher price before they confirm.

## 4. Jobs to be done

What Rahul is really hiring surge pricing to do, even though he would never phrase it this way:

- "Get me a car right now when everyone else also wants one, do not just fail."
- "Tell me the real price up front so I can decide, no nasty surprise at the end."
- "Pull a driver toward me who otherwise would not have come."

And what Suresh the driver is hiring it to do:

- "Tell me where the money is right now so I stop idling in a dead zone."
- "Make the wet, ugly, high-traffic trip worth my time."

And what Uber the marketplace is hiring it to do:

- "Clear the market in every 400 meter cell of every city, every few seconds, without a human touching a dial."

## 5. How it works for the user

Rahul does almost nothing. He opens the app, picks his destination, and the price he sees already has surge baked in. There is a small visible tell: a surge badge, a lightning bolt, or a line that says prices are higher than usual. In the older multiplier style he might have seen "1.8x." In the newer style many markets use, he just sees a single final number, 320 rupees, with a note that it is higher due to demand.

If he does not want to pay it, the app offers him outs. He can tap "notify me when surge drops." He can wait a few minutes and refresh. He can pick a cheaper tier, an auto instead of a sedan. He can walk two streets over, out of the hot cell, and re-check, where the price may be lower. That last one is real: surge is hyperlocal, so the price can genuinely differ between two ends of the same neighborhood.

Suresh, on the driver side, sees the same storm from the opposite direction. His driver app paints a heat map. The rainy patch of Koramangala glows orange and red. He sees that driving there earns extra right now, so he heads over.

## 6. The actual flow, step by step

1. Rahul opens Uber at 9:40pm. The app sends his GPS location to Uber's servers.
2. His location lands inside one specific small hexagon on Uber's map of Bangalore, say hexagon X covering a few hundred meters around the restaurant.
3. Behind the scenes, Uber already knows two live numbers for hexagon X and its immediate neighbors: how many people are requesting or app-open-searching here (demand), and how many free drivers are here (supply).
4. Demand badly outruns supply in hexagon X. The pricing service has already computed a surge value for that hexagon, refreshed seconds ago.
5. Rahul picks his Home destination. The app computes the base fare for 6 km, then applies the surge for hexagon X, and shows 320 rupees with the higher-demand note.
6. Rahul taps Confirm. The request, stamped with the agreed price, goes into the dispatch system.
7. Dispatch looks for the best free driver near hexagon X. Suresh, who moved toward the orange zone, is matched. He accepts in 25 seconds.
8. The fare is locked at what Rahul agreed to. Even if surge climbs to 2.5x one minute later, Rahul still pays his 320. The higher price also flows through to Suresh's earnings for that trip.
9. Minutes later, enough drivers have flooded hexagon X and enough riders have either booked or given up. Supply and demand even out. The surge for hexagon X decays back toward 1.0x. The next rider there sees a normal price.

## 7. Under the hood, like the engineer

This is the heart of it. Surge pricing is not really a pricing feature, it is a real-time geospatial state problem. You have to answer, for every tiny patch of every city, many times a minute: how many riders want a car here, how many cars are free here, and what price closes the gap. Get the geography wrong and the whole thing falls apart.

### First problem: how do you cut a city into patches?

The naive answer is a square grid, like graph paper laid over Bangalore. It is simple, but squares have an ugly flaw for anything involving movement. A square has 8 neighbors, but they are not all the same distance away. The 4 that share an edge are close. The 4 that share only a corner are farther (1.41 times farther, the diagonal). So "the cell next to me" means two different distances depending on direction. When riders and drivers are constantly moving across cell boundaries, that inconsistency creates lumpy, biased measurements. People moving diagonally jump differently than people moving straight.

Uber's answer is **H3**, their hexagonal hierarchical geospatial index, which they open-sourced in 2018 (github.com/uber/h3, docs at h3geo.org). The core idea: tile the world with hexagons instead of squares.

Why hexagons specifically? Only three regular shapes tile a flat plane with no gaps: triangles, squares, and hexagons. Among those, hexagons win for movement because a hexagon has exactly 6 neighbors and **all 6 are the same distance from the center** (center to center). There is no edge-versus-corner split. A hexagon is also the closest of the three to a circle, so it best approximates "everything within roughly this radius of a point." That matters because, in Uber's own words, people in a city are always in motion, and hexagons minimize the quantization error you get when users move across cells. Rahul walking 50 meters introduces less measurement noise in a hex grid than in a square grid.

A few concrete H3 facts that drive the engineering (documented in the H3 docs and repo):

- H3 has **16 resolutions, numbered 0 (coarsest) to 15 (finest)**. Resolution 0 carves the planet into 122 huge base cells. Each step finer splits roughly each cell into about 7, so cell counts grow about 7x per level. By resolution 15 the average cell is under a square meter.
- The grid is built on an **icosahedron**, a 20-faced solid, projected onto the globe. You cannot tile a sphere with hexagons alone, a fact from topology, so H3 is forced to include exactly **12 pentagons** (one near each vertex of the icosahedron) to close the surface. Uber deliberately places those 12 pentagons over ocean where almost no rides happen, so they rarely bother real traffic.
- Each cell has a single **64-bit H3 index**, an integer that encodes the resolution and the path down the hierarchy. Because it is just a number, you can use it as a key in a hash map, group by it in a database, or shard on it. Converting a (latitude, longitude) like Rahul's GPS into its H3 index is a fast math operation, not a search.
- Hexagons cannot perfectly nest inside bigger hexagons the way 4 squares nest inside one square. So H3's parent-child containment is approximate, a child cell sticks out slightly. This is a real wart Uber accepted in exchange for the movement benefits. For surge you usually pick one resolution and live on it, so the imperfect nesting rarely bites.

Surge lives at a fairly fine resolution, hyperlocal hexagons a few hundred meters across, which is why two ends of Koramangala can carry different prices.

### Second problem: counting supply and demand, live, per hexagon

Now the data structures. For every active hexagon you need two running numbers: free drivers and current demand. The natural structure is a **hash map keyed by H3 index**. Driver GPS pings arrive constantly (every few seconds per car). Each ping converts to an H3 index, and you increment the count for that hexagon. Rider app-opens and ride requests do the same on the demand side. So at any instant, `supply[hexagon X] = 12`, `demand[hexagon X] = 200`.

These pings are a firehose. Uber runs millions of drivers, each pinging every few seconds, which is millions of events per second globally. That is a classic **streaming** problem, and Uber's real-time stack (described in their paper "Real-time Data Infrastructure at Uber," arxiv 2104.00087) leans on **Apache Kafka** as the event backbone, with stream processing on top. Pings flow in as a stream, get bucketed by H3 index, and get aggregated in short time windows.

One important refinement: you do not look at hexagon X alone. A driver one hexagon away can serve Rahul in under a minute. So the supply count for X is usually **X plus its ring of 6 neighbors**, sometimes 2 rings out. H3 makes this trivial with a `gridDisk` (k-ring) call: give it hexagon X and k=1, it instantly returns X and its 6 neighbors. The supply number is the sum over that disk. This smoothing also stops the price from flickering wildly as one car crosses a boundary.

### Third problem: turning the gap into a price

Once you have `supply` and `demand` for the disk around X, the surge multiplier is some function of the ratio. Conceptually, if demand is roughly equal to supply, multiplier is 1.0x. As demand pulls ahead, the multiplier climbs. The exact formula is not public, and Uber has deliberately changed it over time, so the curve itself is proprietary. Label this part inference: the precise math is private, the ratio-driven shape is well grounded.

Two real, documented points about the pricing model:

- Uber historically used a **multiplicative** surge, the literal "1.8x" shown to riders. They have since shifted in many markets to **additive** surge, a flat amount added to the fare instead of a multiplier. The reason is not cosmetic. Academic work informing this change (Nikhil Garg and Hamid Nazerzadeh, "Driver Surge Pricing," arxiv 1905.07544, later in Management Science 2022) showed multiplicative surge is not incentive compatible in a dynamic setting: it overpays long trips and underpays short ones relative to the actual supply problem, and it gives drivers perverse reasons to cherry-pick. A flat additive surge is closer to incentive compatible and easier for Rahul to understand as a number.
- The price Rahul is quoted is **locked at request time**. If surge jumps to 2.5x sixty seconds after he confirms, he still pays his 320. This is a deliberate product choice: the quoted price is a contract, and it is computed and frozen server-side, not recalculated on his phone.

Where does the work happen? Server-side, always. Rahul's phone does not compute surge, does not hold the supply and demand counts, does not know the formula. It sends a location, it receives a price. All the counting, the H3 bucketing, the k-ring smoothing, the multiplier math, and the price freeze happen in Uber's backend. The phone is a thin client. This is the same lesson as any ranking system: the heavy state and the sort live on the server next to the data, not on the device.

### The scale story at three tiers

- **1,000 active cells (a single small city, quiet hour).** Trivial. One process holds a hash map of `H3 index -> (supply, demand)`. A timer recomputes surge for the active cells every few seconds. No sharding, no Kafka needed, a single machine and a plain in-memory map handles it. The k-ring neighbor sum is a handful of lookups.

- **100,000 active cells (a dense metro like greater Bangalore at peak, or many cities at once).** Now the ping firehose is the bottleneck, not the math. Millions of GPS pings per second cannot all hit one process. What breaks: a single in-memory map and a single consumer cannot keep up, and you cannot afford a lock everyone contends on. What saves it: stream the pings through **Kafka**, and **geo-shard** the work, meaning route all events for a given region or H3 prefix to the same worker, so each worker owns a slice of the map and never fights another worker for the same cell. Because H3 indexes are hierarchical, you can shard on a coarse-resolution parent cell and keep a whole neighborhood's data together on one shard, which keeps the k-ring neighbor lookups local instead of crossing the network.

- **10 million plus active cells (Uber's real global footprint, surge recomputed across thousands of cities continuously).** What breaks: even sharded, you cannot recompute every cell on demand when a rider asks, the latency would blow the request budget, and cross-shard neighbor lookups at the edges get expensive. What survives it: **precompute and cache**. Surge for each hot hexagon is computed continuously in the background, every few seconds, and written to a fast store (an in-memory cache or **Redis** keyed by H3 index). When Rahul's request arrives, the server does not compute anything heavy. It converts his GPS to an H3 index and does a single cache read to get the current surge for that cell. The expensive aggregation runs offline-ish in a tight streaming loop; the live read path is one hash lookup. That is the same shape as a good recommendation system: do the heavy thinking in a background loop, serve the live request as a cheap keyed lookup.

A blunt honesty note. The H3 grid, the hexagon reasoning, the Kafka streaming backbone, and the multiplicative-to-additive shift are documented facts with primary sources. The exact surge formula, the exact resolution surge runs at, and the precise Redis-sweep cadence are not officially published, so I have labeled those as inference or illustrative and kept them to the shape of the solution, which is how this class of real-time geospatial pricing problem is solved.

## 8. The retention and habit mechanic

Surge is unusual among the features in this ledger because the metric it moves most directly is **revenue**, not retention. It captures willingness to pay in the exact moments demand spikes, and it pays drivers more to show up, which clears trips that would otherwise have been lost. Both sides of that are revenue and marketplace liquidity.

But there is a quieter habit loop underneath, and it runs on the driver side. Suresh's heat map is a variable-reward engine aimed at drivers, not riders. The glowing orange zones are a trigger: open the driver app, there is money over there right now. The action is to drive toward the heat. The reward is a fatter fare, variable because the surge may rise, fall, or vanish before he arrives. The investment is that he learns the city's rhythms, which corners go hot on Friday nights and during rain, and he comes back to chase them. This is the same Hooked-style loop Spotify uses on listeners, pointed at drivers to keep supply liquid. A liquid supply of drivers is exactly what keeps rider wait times low, which is what keeps riders coming back. So surge buys retention indirectly, by funding the supply that makes the rider experience reliable.

On the rider side the honest read is that surge mildly hurts the moment but protects the relationship. The "notify me when surge drops" and the up-front quote exist to keep the annoyance from turning into distrust. A real observed example of the failure mode: surge during emergencies, like the 2014 Sydney hostage crisis, when automatic surge spiked prices for people fleeing the area and triggered a public backlash that pushed Uber to cap surge during disasters. The lesson Uber learned is that a pricing mechanic that ignores human context can torch trust faster than it earns revenue, so the modern system has caps and emergency overrides bolted on.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus an embeddable runtime. The surge lesson that maps hardest to your scalability goals: **pick a spatial index for your scene once, key everything on it, and make the per-frame path a cheap keyed lookup against precomputed state, never a live recompute.**

Concretely, surge works at scale because Uber turned "where is this driver" into a single integer (an H3 index) and then made every hot operation, counting, neighbor lookup, price read, a hash-map or cache hit on that integer. Borrow that exact move for the runtime:

- For any spatial effect that depends on neighbors (particle systems, fluid, collision, light gathering, screen-space blur kernels), do not scan all N elements against all N every frame, that is the O(N squared) trap that breaks at the next tier. Bucket elements into a **uniform grid or spatial hash keyed by cell index**, then for each element only test its own cell and the adjacent ring, exactly Uber's k-ring neighbor sum. This is the standard way GPU particle and physics systems hit millions of elements at 60fps, and it is the same data structure as surge's supply count.
- Prefer a **hexagonal or otherwise isotropic cell layout** when your effect involves motion or radial falloff, for the same reason Uber picked hexagons: a square grid biases neighbor distance by direction (edge versus diagonal), which shows up as visible directional artifacts in things like bloom spread, flow fields, or noise advection. An index where all neighbors are equidistant gives smoother, direction-independent results.
- Split the work the way surge splits it: a continuous **background aggregation** that maintains the spatial buckets and any expensive precomputed fields, and a **thin per-frame read** that just samples the current state by cell index. Keep anything O(scene size) out of the draw call. When a creator's effect ships into a thousand embeds, each embed should be doing keyed reads against a prepared structure, not rebuilding the structure every frame.

The through-line with last run's Spotify lesson is the same backbone: turn the heavy thing into a precomputed, keyed structure, and make the live path one cheap lookup. Uber just proved it works for 10 million live cells of a moving city, which is a stronger stress test than a weekly playlist.

---

## Sources

- Uber Engineering, "H3: Uber's Hexagonal Hierarchical Spatial Index" (why hexagons, hierarchy, surge use): https://www.uber.com/blog/h3/
- H3 official GitHub repository (16 resolutions 0 to 15, hexagonal hierarchical indexing): https://github.com/uber/h3
- H3 documentation (resolution table, base cells, pentagons, indexing): https://h3geo.org/
- Yuan He et al. / Uber Engineering, "Real-time Data Infrastructure at Uber" (Kafka backbone, streaming aggregation), arxiv 2104.00087: https://arxiv.org/pdf/2104.00087
- Nikhil Garg, Hamid Nazerzadeh, "Driver Surge Pricing" (multiplicative vs additive surge, incentive compatibility), arxiv 1905.07544; published Management Science 2022: https://arxiv.org/abs/1905.07544
- Uber Marketplace, "Surge pricing" (rider-facing description of how surge works): https://www.uber.com/us/en/marketplace/pricing/surge-pricing/
- Uber Drive, "How Surge Pricing Works" (driver-facing heat map and earnings): https://www.uber.com/us/en/drive/driver-app/how-surge-works/
