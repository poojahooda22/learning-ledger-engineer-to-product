# Day 38 — How does Uber find the nearest available driver, out of millions, city-wide, in milliseconds?

*2026-07-21*

---

## 1. The company and the number that breaks a naive design

**Uber, and H3, the hexagonal hierarchical spatial index it built internally and open sourced in 2018.** Every ride request has the same underlying question buried inside it: out of every driver currently online, which handful are actually close enough to matter, right now, this second, before any of them move again. Uber's own engineering blog describes H3 as the grid system used "for analysis and optimization throughout their marketplaces," meaning it is not a side tool, it sits underneath both dispatch (who gets offered this ride) and dynamic pricing (is this neighborhood short on drivers right now).

The number that breaks a naive design here is a write-side number, not a read-side one, and it is easy to miss because ride matching feels like a search problem. Commonly cited estimates in system-design writeups (not an Uber-published figure, so treat this as a clearly labeled, order-of-magnitude estimate) put Uber's simultaneously active driver fleet at roughly **5 million**, each one streaming a GPS ping every **~4 seconds**. That alone works out to on the order of **1.25 million location updates per second**, worldwide, at roughly 100 bytes each, before a single rider has tapped "request ride." A second, independently useful way to see the same wall: multiple system-design writeups covering this exact problem title it "How Uber Finds Nearby Drivers at 1 Million Requests per Second," an estimate of the read side, the volume of "who is near me" lookups the matching system has to answer, layered on top of that already-enormous write volume. Whichever number you anchor on, the shape is the same: **a table that both a huge number of writers are constantly overwriting and a huge number of readers are constantly querying by physical location, at the same time, in the same system.**

## 2. Why the naive design dies

The naive version is one relational table, `drivers(id, lat, lng, updated_at)`, with a spatial index (a GiST or R-tree index in PostgreSQL/PostGIS terms) on `(lat, lng)`, fed by an `UPDATE drivers SET lat = ?, lng = ? WHERE id = ?` on every GPS ping, and read by a query like `SELECT id FROM drivers WHERE ST_DWithin(loc, :rider_loc, 5000) ORDER BY ST_Distance(loc, :rider_loc) LIMIT 5` every time a rider opens the app. This collapses in three concrete ways.

**a. The index and the writes fight each other, continuously.** A spatial index like an R-tree or GiST index isn't a flat lookup table, it's a tree of bounding boxes that has to be rebalanced as the points inside it move, and in this system every single point is moving every four seconds. At 1.25 million updates a second, the database spends a huge share of its time reorganizing the very structure the read queries depend on to be fast, so reads and writes end up competing for the same lock-protected pages instead of running independently. The index doesn't just fail to help at that write rate, it becomes part of the bottleneck.

**b. "Who's near me" has no cheap answer without already knowing roughly where to look.** A naive design's only tool for "nearby" is compute-distance-to-everyone-in-a-box, and a box wide enough to guarantee you don't miss a close driver in a dense downtown can still contain thousands of rows to score and sort, on every single ride request, concurrently, from every rider in that same downtown. There's no way to shrink that box without a structure that already groups drivers by roughly where they are, which a flat table with a generic spatial index doesn't provide cheaply at this write rate.

**c. One database is asked to serve both traffic that almost nobody reads and the one write that has to be exactly right.** A GPS ping that lands and is superseded four seconds later has essentially disposable value, nobody queries "where was driver 44821 at 14:32:07 and not one second later." But the moment a match is made, that trip-creation row has to be durable, correct, and never double-assigned, the same driver can't be matched to two riders at once. Mixing a firehose of throwaway writes with a handful of writes that must never be wrong, in the same table, on the same lock structure, means the throwaway traffic can slow down or contend with the traffic that actually matters.

The analogy: imagine a single dispatcher standing over one giant paper map of the whole city, with a ruler, personally measuring the distance from every single taxi to every single raised hand before deciding who gets which fare, and then, every four seconds, someone hands them a freshly repainted map because every taxi on it has moved. The dispatcher never finishes measuring before the map is obsolete again.

## 3. The architecture, top to bottom

```
Driver app (GPS ping ~every 4s: lat, lng, heading)
Rider app ("request ride" with pickup lat/lng)
   |
   v
Edge / connection gateway (stateless)
   |  terminates millions of long-lived driver connections; a phone
   |  switchboard holding open lines, not a database holding rows
   v
Load balancer -> stateless app tier
   |  Location Ingestion Service: on every driver ping, hash (lat, lng)
   |  to an H3 cell ID at the working resolution (res 8, ~0.74 km^2
   |  per cell); this is pure computation, no database write required
   |  just to know which cell a point falls in
   v
In-memory geospatial index (sharded by H3 cell)
   |  a Redis geo/sorted-set cluster, or the descendant of Uber's
   |  original in-memory quadtree "Dispatch" service; membership in
   |  a cell is one O(1)-ish SREM (leave old cell) + SADD (enter new
   |  cell), never a row update plus a tree rebalance
   |  analogy: a pegboard of city blocks, one peg-hole per block,
   |  each hole holds the driver IDs currently standing in it; moving
   |  a peg from one hole to the next is instant, no repainting needed
   v
Candidate fetch, on a ride request (the "recall" half, same split as
Day 18's search architecture: matching and ranking are different jobs)
   |  look up the rider's own H3 cell plus its ring of immediate
   |  neighbors at that resolution; widen to the next ring outward
   |  only if too few candidates come back (thin local supply,
   |  see section 6), never scan the whole city by default
   v
Matching / ranking service (the "ranking" half)
   |  batches requests and candidates over a short rolling window
   |  (Lyft's own engineering write-up describes generating and
   |  solving these batches "at frequent intervals, such as every
   |  few seconds, on a rolling basis"); scores each pair by ETA,
   |  acceptance likelihood, driver rating, and trip efficiency, then
   |  solves the batch as a bipartite assignment, not a greedy
   |  nearest-driver grab, one request at a time
   v
Trip creation (the ONE synchronous, durable write)
   |  sharded relational store; a transactional reservation of the
   |  winning driver so two riders can never both be matched to the
   |  same driver, the same idempotency-key discipline as Day 12
   v
Async pipeline (everything that does not need to block the match)
   |  every raw GPS ping and every match outcome also lands on a
   |  Kafka-style stream, feeding surge-pricing supply/demand
   |  aggregation, ETA-model training, and analytics, fully decoupled
   |  from the synchronous matching hot path above it
```

Worked example: a rider opens the Uber app standing at 5th Avenue and 42nd Street, by the New York Public Library. That point hashes to one H3 resolution-8 cell, roughly three-quarters of a square kilometer, covering a chunk of Midtown. The candidate fetch pulls driver IDs sitting in that cell plus its ring of six neighboring hexagons, maybe 40 to 60 drivers on a normal weekday afternoon, not the tens of thousands of drivers active across all five boroughs. Ranking scores those 40 to 60, not the whole city, and the batch window picks up whichever other riders requested a ride in that same rolling few-second slice of Midtown so the assignment considers all of them together, not one rider grabbing the closest car and leaving a worse leftover match for the rider one block over.

## 4. The transferable mechanisms

**a. Spatial indexing turns continuous space into a fixed, hashable vocabulary.** H3 (hexagons), geohash (rectangles), Google S2 (also hexagon/square hybrid cells on a sphere), and quadtrees all do the same underlying job: chop infinite 2D space into a finite set of named cells so "find nearby" becomes an index lookup, add-to-a-set and read-a-set, instead of a distance computation against every row. H3's specific bet is hexagons over squares: every hexagon has exactly six equidistant neighbors, so "look at the ring around me" is a clean, uniform operation, where a square grid's diagonal neighbors sit at a different distance than its edge neighbors, a subtle correctness trap for anything doing distance-based ranking.

**b. Hierarchy makes the search radius a cheap dial, not a re-architecture.** H3 cells nest, a resolution-8 cell has a specific resolution-7 parent and seven resolution-9 children, so "not enough candidates, look wider" is just walking up or out one level, not redesigning the index. This is the same recall-then-widen shape as Day 18's search ranking lesson: fetch a cheap, generous candidate set first, only pay the expensive precise-ranking cost on the much smaller set that survives.

**c. Put high-churn state in a structure built for churn, not one built for durability and joins.** A GPS ping every four seconds from millions of drivers is exactly the workload a relational table with a B-tree or spatial index handles badly, constant small mutations competing with the index's own bookkeeping. An in-memory set keyed by cell ID, or the H3-sharded successor to Uber's original quadtree "Dispatch" service, absorbs that churn as O(1)-ish set membership changes. Same lesson as Day 21's LSM trees: match the data structure to whether the workload is write-heavy-and-disposable or read-heavy-and-durable, don't force one structure to do both well.

**d. Batching turns a sequence of locally-greedy decisions into one globally-better one.** Lyft's own dispatch engineering post frames this directly: solve matching as a rolling bipartite assignment over a short window of riders and drivers together, not first-come-first-served nearest-neighbor. A longer window gives the optimizer more requests and more drivers to balance against each other, better aggregate matches, at the cost of every individual rider waiting a little longer to see a driver assigned. This is the identical shape as Day 9's queue-as-shock-absorber trade-off, deliberately holding work briefly to do it better, not the fastest thing to do request-by-request.

**e. Precompute the expensive-and-reusable, compute live only the genuinely dynamic.** DoorDash's own published approach to travel-time estimation precomputes the whole cell-to-cell travel time matrix offline: a Spark job on Databricks running the OSRM routing engine directly on the cluster (removing network-call overhead the search results describe as roughly a 10x speedup) across roughly **6 billion cell-pair estimates**, cached in Redis, at **three resolution tiers** (fine-grained, up to about a mile, mid-tier, and large-scale, up to about 100 miles), with the online path always taking the finest tier available for a given pair. Static travel time between two fixed hexagons barely changes minute to minute, so compute it once in batch; only live traffic conditions on top of that baseline need to be computed fresh. The same precompute-plus-cache split as Day 19's caching lesson, applied to a geometry problem instead of a database query.

**f. The hot-cell problem is the hot-key problem, just drawn on a map instead of a keyspace.** A single H3 cell over Times Square on New Year's Eve, or over a stadium the moment a game lets out, takes read and write traffic that dwarfs every other cell in the city, the exact same shape as Day 16's celebrity-account hot key, just spatial instead of logical. The fix is the same family: locally finer-grained sharding (sub-dividing that one hot cell further, or fanning its lookups across replicas) rather than assuming one fixed resolution serves every cell equally well everywhere.

## 5. The trade-offs

**Consistency vs. availability, split sharply by data type.** A driver's position in the geospatial index is allowed to be a few seconds stale, eventually-consistent, last-write-wins, because by the time any reader could act on last-writer-wins, four more seconds have passed and the position has already changed again, staleness here is not really a correctness bug. A completed match is the opposite: it must be strongly consistent and idempotent, the same driver can never be committed to two trips, so trip creation is the one place this whole pipeline pays for a real transactional write, the identical idempotency-key discipline from Day 12 applied to "reserve this driver," not "charge this card."

**Cost vs. latency, in the resolution choice itself.** A coarser H3 resolution means fewer cells, a smaller index, cheaper set operations, but worse match quality, a rider can land in the same coarse cell as a driver who is technically "nearby" but actually across a river or a highway with no direct route between them. A finer resolution means better precision but more cells, more index churn per ping, and more neighbor-ring lookups to cover the same physical area. Uber's own commonly-cited choice of resolution 8 (about 0.74 km² per cell) for city dispatch is exactly this trade-off, picked to land near "a city block or two," fine enough to be geographically meaningful, coarse enough to keep the cell count and churn manageable.

**Cost vs. latency again, in the precompute-vs-live-compute split.** DoorDash's roughly 6 billion precomputed cell-pair estimates are a large, continuously-recomputed batch job and a sizeable standing Redis footprint, a real ongoing cost, paid specifically so the request-time path never has to run live routing math. That is a deliberate bet that most travel-time queries are between pairs of locations whose baseline travel time is genuinely stable enough to precompute, and that the rare pairs that aren't well-served by a cached tier are an acceptable miss to backfill or approximate.

## 6. The systems-thinking lens

**The feedback loop here is localized retry-under-scarcity, a geographic sibling of the retry death spiral from Day 13.** Trace it: a stadium empties out or a storm hits, demand spikes hard inside one small ring of H3 cells → available supply in exactly those cells drains fast as the first riders get matched → the riders left behind see "no cars available" or a spiking price → many of them refresh or retry the request immediately, because the app gives them nothing better to do while waiting → every retry consumes matching-service capacity re-running "still no driver here" against the exact same starved cells → that wasted work is capacity not spent widening the search ring, rebalancing supply from a neighboring cell, or serving requests anywhere else in the city that isn't starved. Left alone, the retries pile up precisely where supply is thinnest, making the local shortage look and feel worse than it structurally is, without adding a single new driver anywhere.

**The senior fix doesn't add drivers, it breaks the loop on both sides at once.** On the supply side, dynamically widen the H3 neighbor ring the moment a cell's local candidate count drops below a threshold, trading a slightly longer pickup distance for definitely finding a match instead of returning empty and inviting a retry. On the demand side, surge pricing is functioning as backpressure with a price signal instead of an HTTP 429: it doesn't just charge more, it sheds or delays exactly the portion of demand the current local supply can't serve, which is the same load-shedding idea from Day 13, implemented as a market mechanism instead of a rejected request. And on the client side, "searching for driver" should back off between retries, not hammer the match service every second at precisely the moment that cell is already starved, capacity spent computing repeated "no" answers is capacity not spent finding the next "yes" somewhere the loop hasn't reached yet.

---

## References and summaries

**Uber Engineering Blog. "H3: Uber's Hexagonal Hierarchical Spatial Index."**
https://www.uber.com/en-US/blog/h3/
Direct fetch of this page returned HTTP 403 in this session, the same broad fetch-blocking noted in Day 36 and Day 37's references, not specific to this host. Content used here (H3 as the grid system used for analysis and optimization throughout Uber's marketplaces, including dynamic pricing and dispatch decisions, and the hexagon-over-square rationale of uniform neighbor distance) was cross-corroborated across multiple independent secondary sources quoting or summarizing the post directly, listed below.

**GitHub. uber/h3, the open-source H3 library.**
https://github.com/uber/h3
Partially fetchable; confirms H3 as "a geospatial indexing system using a hexagonal grid that can be (approximately) subdivided into finer and finer hexagonal grids, combining the benefits of a hexagonal grid with S2's hierarchical subdivisions." Points to h3geo.org for the full resolution table.

**H3 official documentation. "Tables of Cell Statistics Across Resolutions."**
https://h3geo.org/docs/core-library/restable/
Direct fetch returned HTTP 403 in this session. The resolution-8 average cell area used here, approximately 0.74 km² (about 0.7373 km² in the official table), was cross-checked against a public transcription of the same official table and against independent secondary sources describing resolution 8 as roughly "half a square kilometer" used for surge-pricing granularity, consistent within normal rounding.

**DoorDash Careers Blog. "How DoorDash achieves fast travel estimates."**
https://careersatdoordash.com/blog/doordash-fast-travel-estimates/
Direct fetch returned HTTP 403 in this session. Content used here (H3-cell travel-time precompute at three resolution tiers, fine-grained up to about a mile, mid-tier, and large-scale up to about 100 miles, using Spark on Databricks with OSRM installed directly on the cluster for roughly a 10x speedup over network calls, cached in Redis, and totaling on the order of 6 billion precomputed cell-to-cell estimates) was cross-corroborated across multiple independent secondary sources summarizing this same post.

**DoorDash Engineering Blog. "Scaling DoorDash's Geospatial Innovation with a Location-Based Delivery Simulator."**
https://doordash.engineering/2020/08/12/scaling-geospatial-innovation-with-a-location-simulator/
Direct fetch returned HTTP 403 in this session. Referenced as DoorDash's own account of building simulation tooling around its geospatial dispatch systems; used here only as corroborating context that geospatial matching at DoorDash is treated as a first-class, heavily-engineered system, not a secondary detail.

**Lyft Engineering Blog. Oussama Hanguir. "Solving Dispatch in a Ridesharing Problem Space."**
https://eng.lyft.com/solving-dispatch-in-a-ridesharing-problem-space-821d9606c3ff
Direct fetch returned HTTP 403 in this session. Content used here, dispatch modeled as a bipartite matching problem solved on a rolling batch window generated "at frequent intervals, such as every few seconds," and the explicit trade-off that a longer batching window improves aggregate match quality at the cost of longer individual rider wait time before a driver is even assigned, was cross-corroborated via independent summaries of the same post and is consistent with the broader academic literature on batched ride-hailing dispatch.

**Secondary system-design sources, used for order-of-magnitude, clearly-labeled estimates, not official company figures.**
"How Uber Finds Nearby Drivers at 1 Million Requests per Second" (GeeksforGeeks) and "Scaling a Real-Time Marketplace: Engineering Lessons from Uber's Architecture" (earezki.com Dev|Journal). These are third-party system-design write-ups, not primary Uber sources, and are the origin of the approximate 5-million-active-driver, ~4-second-ping-interval, ~1.25-million-updates-per-second estimate used in section 1, and of the Dispatch-Service-queries-geospatial-index-then-ranks-by-ETA-and-acceptance-rate flow used in section 3. Treated here explicitly as inference-level, commonly-repeated industry estimates rather than confirmed internal Uber numbers.

---

## Map to Rare.lab's stack

Rare.lab doesn't dispatch physical drivers, but the underlying pattern, index disposable high-churn state cheaply, only pay a durable-and-correct write for the thing that actually matters, shows up the moment the runtime stops being a single shared WebGL context rendering one user's scene and starts being multiplayer, several people editing or watching the same shader graph live. The moment cursor positions, live parameter tweaks, or per-frame preview state need to be broadcast to other viewers, that state has exactly the shape of a driver's GPS ping: high frequency, cheap to lose a sample of, and actively harmful to route through the same Supabase Postgres tables that hold the durable, RLS-protected scene manifests.

The concrete, actionable move: if and when Rare.lab adds any live collaborative surface, real-time cursor sharing, live parameter broadcast during a co-editing session, or a preview-farm status board showing which render workers are free, don't model that state as rows in the same Postgres tables used for the durable content-addressed manifest and asset pointers in R2. Model it the way this lesson's ingestion path does: an ephemeral, in-memory, keyed store (even something as simple as a Redis pub/sub channel or a single in-memory map behind a stateless edge function) that many small updates can hit cheaply and that nobody needs to reconstruct from history, while the one write that must be durable and correct, a scene actually being published, a manifest actually being committed, keeps going through the transactional, RLS-guarded Postgres path exactly as it does today. Keep the disposable-and-fast state and the durable-and-correct state in structures built for each, the same split this whole lesson turns on.
