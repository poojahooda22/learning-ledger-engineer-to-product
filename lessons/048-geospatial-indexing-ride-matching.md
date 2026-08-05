# Day 48 — How do you find the nearest available driver among 100,000 moving cars, in under a second?

**Date:** 2026-08-05
**Difficulty:** Expert
**Topic:** Geospatial indexing and real-time matching at scale: Uber's H3 hexagonal hierarchical grid (successor to Google's S2 library), the DISCO dispatch system, Ringpop's consistent-hash-ring-plus-gossip sharding for a stateful service, and why a plain latitude/longitude database column cannot answer "who is near me" fast enough once the map gets crowded.
**Stack relevance:** Rare.lab does not dispatch cars, but it will eventually need the same primitive: "which of these thousands of live scene viewers, or nodes, or shader instances, are near this one in some space" (viewport region, dependency graph distance, GPU cluster locality). The mechanism here, discretize a continuous space into a hierarchy of cells and index on the cell ID, is the transferable idea, not the ride-hailing wrapper around it.

---

## 1. The company and the breaking number

**Uber**, since it grew past its first few cities. The concrete failure is well documented across systems that all hit the same wall independently (Uber's own DISCO dispatch system, and later PostGIS-based ride-hailing clones that rediscovered the same limit): a **naive spatial query against a live table of moving points falls over between roughly 5,000 and 10,000 concurrently tracked drivers in a single dense city**, with query latency reported to jump from around 50 milliseconds to over 2 seconds once concurrent match requests climb into the low thousands against a table of that size. That is not a rare edge case, it is Tuesday-evening rush hour in one metro area.

The volume behind that number is real and well established from Uber's own engineering writing and talks. Driver apps push a GPS ping roughly **every 4 seconds**, and aggregated across hundreds of cities that adds up to figures Uber engineers have cited in public talks in the range of **hundreds of thousands of location updates per second** and **tens of billions per day** globally (these aggregate figures come from Uber engineering talks and secondary write-ups of them, not a single canonical blog post, so treat the exact figure as a well-sourced estimate rather than an audited statistic). Whatever the precise number, the shape is the same: a continuous, high-frequency stream of writes ("where is every driver right now") has to be cross-referenced, on every single ride request, against a query of the form "who is close to this one point, ranked by real travel time, right now." That is a spatial nearest-neighbor query running continuously against a dataset that never stops moving, at write volumes that swamp anything a naive design was built for.

---

## 2. Why the naive (demo) design dies

The demo version of "find nearby drivers" is a single table: `drivers(id, lat, lng, status, updated_at)`, with a B-tree index on `lat` and another on `lng`, and a query like:

```sql
SELECT id, lat, lng FROM drivers
WHERE lat BETWEEN :lat_min AND :lat_max
  AND lng BETWEEN :lng_min AND :lng_max
  AND status = 'available';
```

This looks reasonable and works fine in a demo with 50 drivers. It dies in production for three specific, compounding reasons.

**A B-tree index on one column cannot answer a two-dimensional range query.** A standard index sorts rows along a single axis. Given an index on `lat` and a separate index on `lng`, the database engine can use *one* of them to narrow the search (say, all rows with `lat` in range), but it then has to scan every one of those rows and check `lng` in the application layer or via a bitmap-and, because rows that are close in latitude can be scattered anywhere in the physical table with respect to longitude. A bounding box that looks small on a map can still match thousands of rows that have to be individually fetched and filtered. This is precisely the failure PostGIS's own spatial indexes (GiST, R-tree family) exist to fix, and precisely the failure that persists when a system reaches for two independent B-tree indexes instead of a real spatial index, which is a mistake made constantly by teams that did not expect to need geospatial queries when they first modeled the table.

**Every candidate row still needs an expensive distance calculation.** Even with a correct spatial index narrowing the box, "who is nearest" is not the same question as "who is inside this box." Ranking by real distance means computing haversine (great-circle) distance, or worse, an actual road-network ETA, for every candidate row, at query time, for every single ride request arriving concurrently. At a few requests a second this is invisible. At a few thousand concurrent requests against a table of tens of thousands of live rows, the CPU cost of that per-row math, repeated per query, becomes the dominant cost, exactly the pattern reported for naive PostGIS ride-hailing prototypes: latency does not degrade gracefully, it cliffs, because the work is O(candidates × concurrent queries) and both numbers grow together during rush hour, the worst possible correlation.

**The write side collides with the read side on the same hot table.** Every driver pings its location roughly every 4 seconds. With even 10,000 concurrently online drivers in one city, that is roughly 2,500 UPDATE statements per second hitting the same `drivers` table that every ride request is also querying. Each UPDATE has to maintain the spatial index, each read query has to see a consistent-enough view to not double-book a driver, and the two workloads (very high write frequency for movement, latency-sensitive read for matching) compete for the same locks, the same cache pages, the same I/O bandwidth on the same rows, over and over, every 4 seconds, for every driver, forever. A single relational table was never going to survive being both a write-hot stream and a read-hot index at the same time.

---

## 3. The architecture

The shape Uber's dispatch stack (and every serious ride-hailing or delivery-matching system that converged on the same answer) settles into, top to bottom:

```
[Driver app: GPS ping ~every 4 seconds]
   |
   v
[Edge ingestion / API gateway]
   analogy: a mail sorting office, not a mailbox — it does not
   store anything, it just routes the letter to the right bin
   |
   v
[Stream ingestion (Kafka-style partitioned log)]
   analogy: a conveyor belt wide enough that thousands of
   letters can be dropped on it per second without collision,
   each one tagged for later pickup
   |
   v
[Stream processor: resolve (lat, lng) -> H3 cell ID at a
 chosen resolution, write into an in-memory geo-index keyed
 by cell ID -> set of driver IDs]
   analogy: instead of one giant citywide phonebook, a set of
   small index cards, one per city block, each listing who is
   currently standing on that block
   |
   v
[Rider requests a ride: dispatch service resolves rider's own
 H3 cell, then expands outward ring by ring (k-ring: 1 -> 7
 cells, 2 -> 19 cells, 3 -> 37 cells) until enough candidates
 are found]
   analogy: asking "who's on this block, and the six blocks
   touching it" before ever widening the search further out
   |
   v
[Candidate scorer: for the small candidate set only (tens of
 drivers, not thousands), compute real road-network ETA, not
 straight-line distance]
   analogy: ranking a short list of nearby taxis by how long
   the actual one-way streets take, not by a ruler on a map
   |
   v
[Reservation: lock the chosen driver for this rider with a
 short TTL, so no second rider can be matched to the same
 driver while confirmation is pending]
   analogy: holding a restaurant table for a few minutes while
   the party finishes deciding, not selling the same seat twice
   |
   v
[Trip ledger: durable write to a sharded database, replicated,
 once the match is confirmed]
   |
   v
[Async pipeline for everything that does not block the match
 itself: receipts, ETA notifications to the rider, surge
 recalculation for the affected cells]
```

The key structural decision is where in this chain the driver's *location* lives versus where the *match state* lives. Location is an in-memory, sharded-by-geography, frequently-overwritten index. Match state (this driver is now reserved, this trip is now confirmed) is a durable, strongly consistent write. Those are two different consistency requirements sharing one workflow, and conflating them into one table is exactly the naive design that failed in section 2.

Uber's dispatch service, called **DISCO**, was originally sharded using Google's **S2 geometry library**, which divides the Earth's surface into a hierarchy of cells and gives each cell a numeric ID usable as a partition key. Uber later built and open-sourced its own successor, **H3**, a hexagonal hierarchical spatial index, specifically because hexagons have a property square or S2-style quadrilateral cells do not: **every hexagon has exactly six neighbors, all at the same distance from its center** (except twelve unavoidable pentagons forced by the geometry of wrapping a hexagonal grid around a sphere). That uniform adjacency is what makes "expand outward ring by ring" (the k-ring operation) a clean, deterministic operation instead of one with distance distortion baked in, which square grids have at their corners versus their edges. Uber's own published guidance is that resolutions 7 through 9 (cells of roughly 0.1 to 5 square kilometers) cover the large majority of dispatch, ETA, and surge-pricing use cases, coarse enough to keep candidate sets small, fine enough to keep them locally relevant.

To make the dispatch service itself horizontally scalable and fault tolerant, given that it is a **stateful** service (it holds the live driver location index in memory, so a plain stateless round-robin load balancer does not work; a request for cell X must land on the node that owns cell X), Uber built **Ringpop**: a consistent hash ring, so each node in the cluster owns a deterministic slice of the cell-ID space, combined with the **SWIM gossip protocol**, so nodes discover and monitor each other's health without a central coordinator. When a node dies, the ring rebalances its cells onto the remaining nodes automatically, the same consistent-hashing mechanism this ledger covered on Day 10, applied here to a hexagonal cell ID instead of a cache key.

---

## 4. The transferable mechanisms

**a. Hierarchical spatial indexing: turn continuous 2D space into a hash-map lookup.** The core trick underneath H3, S2, and simpler cousins like geohash is the same: pick a resolution, map every (lat, lng) to a single discrete cell ID, and now "who is near this point" becomes "look up this cell ID and its immediate neighbors in a hash map," an O(1) lookup plus a small, bounded k-ring expansion, instead of an O(n) scan with a per-row distance calculation. This is the single mechanism that converts an unbounded geometric search into a bounded, cache-friendly one.

**b. Reservation with a TTL lock, not an eventually-consistent flag.** The moment a driver is offered to a rider, that driver must be provably unavailable to every other concurrent match attempt until the offer resolves, one way or another. A short-lived lock (seconds, not minutes) with an expiry handles both the happy path (rider confirms, trip starts, lock converts to a real assignment) and the failure path (rider does not respond, driver does not accept, lock simply expires and the driver returns to the pool automatically, no cleanup job required). This is the same pattern this ledger named on Day 24 (distributed locks and fencing tokens) applied to a physical, moving resource instead of a database row.

**c. Push, not poll, for the write-hot side.** Drivers push their location on a fixed cadence; nothing on the read side has to poll a driver to find out where it is. This keeps write volume predictable and bounded (one ping per driver per interval, not one query per rider per second fanning out to every driver) and is the same push-vs-poll trade this ledger has returned to before: polling multiplies with the number of readers, pushing does not.

**d. Sharding by geography is naturally embarrassingly parallel.** A ride request in Mumbai never needs to consider a driver in São Paulo. Partitioning the live index by region (which is exactly what a consistent hash ring over H3 cell IDs achieves) means adding a new city adds a new shard of load, not new load on every existing shard. This is the geographic instance of the same sharding principle this ledger covered generally on Day 10, but it is worth naming separately here because the partition key falls directly out of the problem domain, not out of an arbitrary hash of a user ID.

**e. Separate candidate generation from ranking, and only rank the small set.** The geo-index's job is to cheaply produce a short list, tens of candidates, not to rank them well. The expensive operation, real road-network ETA instead of straight-line distance, runs only on that short list. This is the identical two-phase shape (cheap broad recall, then expensive precise ranking on a narrowed set) that this ledger named for search and feed ranking on Days 7 and 18; geospatial matching is the same shape with a hexagon instead of an inverted index doing the narrowing.

**f. Idempotency on the match itself.** A dispatch attempt that times out and retries must not produce two trips for one ride request. A stable idempotency key per match attempt, the same mechanism from Day 12, guards the reservation-to-confirmation transition so a retried request either finds the prior attempt's result or safely starts a fresh one, never both.

---

## 5. The trade-offs

**Consistency versus availability, split cleanly by data type, inside one workflow.** Driver *location* is a natural candidate for eventual consistency: a car moves perhaps 20 to 30 meters in the 4 seconds between pings, so a read that is a few seconds stale is operationally fine, and demanding strong consistency here (every reader sees the exact latest position, synchronously) would mean paying a coordination cost on every single write, 2,500-plus times a second in one city, for precision the matching algorithm does not actually need. Match *state* (this specific driver is now reserved for this specific rider) cannot tolerate that same looseness: two riders both believing they got the same driver is a broken product experience, not a rounding error, so that narrow slice of state needs a strongly consistent, linearizable write, exactly the TTL-locked reservation from mechanism b. The lesson generalizes past ride-hailing: the right consistency model is a property of what a specific field means, not a single blanket decision for an entire service.

**Cost versus latency in where the live index lives.** Keeping the entire city's live driver locations in memory, sharded across a cluster, costs real money in RAM that a disk-backed table would not. It is worth it here because the alternative, hitting disk-backed storage for a query that has to answer in well under a second while competing with thousands of pings per second on the write side, cannot hit the latency bar at all, at any cost. This is the same trade this ledger named for caching in general on Day 19, made concrete: sometimes the "expensive" option is the only one that is actually available at the required latency, which makes it not a trade-off in practice, just the price of entry.

**Cell resolution: precision versus fan-out cost.** A finer H3 resolution (smaller hexagons) gives a more precise notion of "nearby," fewer irrelevant candidates per lookup, but it also means more cells to track, and a k-ring expansion at a fine resolution may need to reach further out (a larger k) to gather enough candidates in a sparse area, which is more lookups, not fewer. Uber's choice to lean on resolutions 7 through 9 for most dispatch logic is a concrete answer to that trade-off: coarse enough that candidate sets stay small and stable, fine enough that "nearby" still means something useful to a rider standing on a specific street corner.

---

## 6. The systems-thinking lens

**The feedback loop: match churn under surge, a metastable failure that outlives the surge itself.** Trace it end to end. A big event lets out (a stadium, a concert) and demand spikes sharply in one small geographic area, a handful of adjacent H3 cells. Every rider in that area requests a match at once, and the small local pool of available drivers gets contended for by far more requests than there are drivers → many of those match attempts land a TTL-locked reservation on a driver, but because so many riders are competing for so few drivers, a meaningful share of those reservations fail (driver already taken by a faster request, driver rejects, rider gives up and re-requests before the lock even expires) → each failure releases the driver back into the pool and immediately gets re-contended by the same crowd of still-unmatched riders, all of whom are also retrying → the dispatch service is now spending its capacity relocking and re-releasing the same small set of drivers over and over, doing enormous work while the number of *successful* matches barely moves, and this keeps happening even minutes after the actual surge in requests has leveled off, because the retries themselves are now the dominant source of load, not the original demand spike. This is the same shape this ledger has named repeatedly under different names, a thundering herd that becomes self-sustaining because the system's own failure response (retry immediately) is what keeps regenerating the load that caused the failure in the first place.

**The senior fix breaks the loop structurally, it does not add driver capacity to the hot cells.** Uber's actual lever here is **surge pricing**, which is usually framed as a revenue mechanism but is structurally a demand-throttle: raising price in the specific overloaded H3 cells reduces the rate of new match requests entering that hot region right when the matching engine is already saturated, exactly the economic equivalent of backpressure. Underneath that, the dispatch engine itself applies queuing discipline instead of parallel blind retries (batch a short window of unmatched requests in a hot cell and solve them together instead of racing independent point-to-point matches against each other, so retries stop competing against each other for the identical scarce driver), and widens k-ring search radius gracefully under load instead of leaving riders to manually retry a failed match, which is the client-side source of the retry storm in the first place. None of these fixes add a single new driver to the road. They all change the shape of how demand hits the already-scarce supply, which is the same lesson this ledger keeps landing on: the fix for a retry-driven feedback loop lives in the request path (throttle, batch, queue, price) not in the resource pool underneath it.

---

## Map to Rare.lab's stack

**Where the same shape shows up, stripped of the ride-hailing wrapper.** Rare.lab does not have drivers or riders, but it has the identical underlying problem waiting once the product scales: a large, live, continuously-updating population of things (open scene sessions, active nodes in a big node graph, GPU shader instances across many customers' embedded runtimes) where a common operation is going to be "find the ones near this one," in whatever space matters, viewport locality for collaborative editing presence, dependency-graph locality for incremental recompilation, or GPU-cluster locality for load-balancing render work across a shared WebGL context pool. The naive version of that query, a table scan filtered by two independent range conditions, is exactly section 2's failure, and it will not announce itself until session counts or node-graph size cross whatever threshold turns "a filtered scan" into "a table scan under contention with a write-hot stream," the same cliff this lesson describes for a drivers table.

**The concrete move to make before that day arrives:** if and when Rare.lab needs "what else is near this node in the dependency graph" or "which collaborators are viewing near this region of the canvas" at real scale, do not reach for a live SQL range query against a hot table. Discretize the relevant space (graph distance buckets for dependency locality, a coarse spatial grid over canvas coordinates for viewport locality) into a small, fixed set of bucket IDs, keep a sharded in-memory index from bucket ID to the current members, and update it via a push, not a poll, the same mechanism as a driver's 4-second ping. This is a smaller, cheaper version of H3 for a domain-specific space rather than the Earth's surface, but the mechanism, and the reason it is needed once the naive version falls over, is identical.

---

## References and summaries

**Uber Engineering Blog: "H3: Uber's Hexagonal Hierarchical Spatial Index"**
https://www.uber.com/en-LT/blog/h3/ (also mirrored at https://www.uber.com/us/en/blog/h3/)
Uber's own explanation of why it built H3: hexagons give every cell exactly six equidistant neighbors (aside from twelve unavoidable pentagons), which makes ring-based expansion (k-ring search) geometrically uniform in a way square or S2-quadrilateral grids are not. Each cell at a chosen resolution encodes to a compact 64-bit ID usable directly as an index or shard key. Uber's stated primary uses are dispatch (finding nearby drivers), ETA estimation, and surge-pricing computation, mostly at resolutions 7 through 9.

**GitHub: uber/h3, "Hexagonal hierarchical geospatial indexing system"**
https://github.com/uber/h3
The open-source implementation and reference documentation for the k-ring formula used in this lesson (k=1 gives 7 cells including center, k=2 gives 19, k=3 gives 37, following 1 + 6·k(k+1)/2) and the multi-resolution hierarchy (roughly 0.1 km² cells at resolution 9, coarsening at each lower resolution).

**High Scalability: "How Uber Scales Their Real-Time Market Platform" (summarizing a 2015 talk by Matt Ranney, then Uber's Chief Systems Architect)**
http://highscalability.com/blog/2015/9/14/how-uber-scales-their-real-time-market-platform.html
Primary account of Uber's DISCO dispatch system in its earlier architecture: originally sharded using Google's S2 geometry library for cell IDs as a partition key, built as a stateful Node.js service (explicitly not stateless, because the live location index has to live in memory on the node that owns a given cell), and paired with Ringpop for cluster membership and request routing.

**Uber Engineering Blog: "Ringpop: Application-Layer Sharding for Node.js Applications"**
https://eng.uber.com/ringpop-open-source-nodejs-library/
Source for Ringpop's mechanism: a consistent hash ring assigns each node a deterministic slice of the key space (here, cell IDs), combined with the SWIM gossip protocol so nodes detect failures and disseminate membership changes to each other without a central coordinator, letting the ring rebalance automatically when a node joins or leaves.

**Secondary write-ups on Uber's location platform scale (Medium/dev.to summaries of public Uber engineering talks, not a single canonical Uber blog post)**
https://medium.com/@simranjeetsingh1497/uber-architecture-part-1-why-tracking-5-million-drivers-every-second-is-one-of-techs-hardest-6ca606892497 and https://singhajit.com/how-uber-finds-nearby-drivers-1-million-requests-per-second/
These are secondary sources, not Uber's own primary publications, and are cited here as clearly-labeled inference rather than confirmed fact: they report driver GPS pings roughly every 4 seconds, aggregate figures in the range of hundreds of thousands of location updates per second globally, and driver-matching request volume reported in the range of roughly a million per second across Uber's combined Rides, Eats, and Freight services. Treat the specific figures as order-of-magnitude estimates from people summarizing public talks, not as officially audited Uber statistics.

**Vitess/Citus/CockroachDB context on why a single B-tree cannot serve a 2D range query, and PostGIS's GiST-based spatial index as the standard fix**
General PostGIS and PostgreSQL documentation on the GiST index family (used by PostGIS for spatial types) versus plain B-tree indexes, referenced here to explain why `WHERE lat BETWEEN ... AND lng BETWEEN ...` against two independent B-tree indexes does not compose into an efficient 2D query, which is the standard, well-documented reason spatial databases build R-tree-family indexes (GiST in PostgreSQL/PostGIS) instead of relying on ordinary single-column indexes for bounding-box queries.
