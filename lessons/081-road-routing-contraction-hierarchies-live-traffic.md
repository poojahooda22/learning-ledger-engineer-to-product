# Day 81: How does a maps app find the fastest route across a continent in milliseconds, while traffic changes every minute? Contraction hierarchies, customizable metrics, and learned travel times

**Date:** 2026-10-05
**Difficulty:** Advanced (graph algorithms as a serving problem, offline/online split, separating graph shape from edge weights)
**Topic:** Shortest path on a graph with tens of millions of nodes, as a live service. Precompute structure offline, answer queries online, re-weight cheaply when traffic moves.
**Stack relevance:** A Rare.lab node graph is also a graph that you compile and query again and again. The same "precompute the shape, keep the weights cheap to change" move applies. Section 7 maps it.

---

## 0. The framework (same six steps as Day 74)

1. Functional: given origin and destination, return the fastest route and an ETA; re-route when the driver goes off route; show alternatives.
2. Non-functional: answer in tens of milliseconds, handle a huge number of concurrent queries, reflect traffic that is minutes old, never return a route through a closed road.
3. Entities: Road segment (edge), Intersection (node), Route, Traffic observation.
4. API: `GET /route?from=lat,lng&to=lat,lng&depart=now` returns polyline, distance, ETA.
5. Naive design: load the road graph in memory on one server and run Dijkstra's algorithm per request.
6. Deep dives: why Dijkstra is too slow, contraction hierarchies, why traffic breaks precomputation, customizable metrics, ETA prediction as a separate ML problem.

---

## 1. The company and the breaking number

**Google Maps (and any maps or ride app: Uber, Rapido, Swiggy), and one number: a continental road graph has about 18 million nodes and 42.5 million directed edges, and plain Dijkstra takes seconds per query on it.**

- The standard research benchmark, the road network of Western Europe from PTV, has **18.0 million vertices and 42.5 million directed arcs** (per the survey "Route Planning in Transportation Networks", via search excerpt, Section 9).
- The same literature states that on graphs this size Dijkstra "incurs running times in the order of seconds even for a single path query".
- Inference arithmetic: a long query makes Dijkstra settle roughly half the graph, so about 9 million nodes. At about 100 ns each (priority queue pop plus edge relaxations, a generous guess) that is about **1 second of one CPU core per query**.
- Illustration, not a measured Google number: if a service sees 100,000 route requests per second at peak, one-second queries need **100,000 busy cores** just for routing. At 1 millisecond per query you need about **100 cores**. A 1000x algorithmic gap is the whole lesson.

Analogy: finding a route by checking every street in Europe, outward in a growing circle from your house, until the circle touches your destination. Correct, but you read half a continent to plan a drive to the next town.

---

## 2. Why the naive design dies

**Naive: whole graph in RAM, Dijkstra per request.**
- **CPU.** About a second per long query, as above. Throughput per core is about 1 query per second. Scaling out means paying for thousands of servers to redo the same work.
- **Wasted work.** Dijkstra explores in a circle. A trip from Delhi to Jaipur explores Pakistan, Nepal and Gujarat before it finds Jaipur. Most of the explored roads cannot possibly be on the answer.
- **Stale weights.** Edge cost is travel time, and travel time changes with traffic. If you precompute anything using the weights, a jam on one highway can invalidate it.
- **Memory.** 42.5M edges with weights and geometry is gigabytes. Fine for one box, but you need many copies to serve load, and each copy must be updated when traffic changes.

Real query walked end to end (naive): user in Gurugram asks for a route to Jaipur. Server pops the origin, relaxes its few neighbours, pops the nearest, and so on. After about 5 million pops it reaches Jaipur. The user waited, the CPU core was pinned, and the next request waits behind it.

---

## 3. The architecture, top to bottom

```
Clients (phone nav app, web)
   |   "from A to B, leave now"
   v
Edge / load balancer (route by region, so a query hits the server that holds that region)
   |
   v
Routing API (stateless: snap GPS to nearest road, validate, format response)
   |   snap origin and destination to graph nodes (spatial index, see Day 48)
   v
Routing engine (in-memory, read-only graph per region)
   |   * topology index: contraction hierarchy (built offline, rarely changes)
   |   * metric: edge weights (travel times, rebuilt every few minutes from traffic)
   |   * bidirectional upward search, then unpack shortcuts into real roads
   v
ETA service (ML model over the chosen route's segments, live + historical traffic)
   ^
   |  Traffic pipeline (offline/stream): phone GPS pings -> map-matched speeds -> new weights
   |
Map data pipeline (offline, batch): OpenStreetMap or vendor data -> graph -> hierarchy build
```

- **Snap step:** turns a lat/lng into the nearest road node. Analogy: telling the taxi "outside the blue gate" and the driver mapping that to an exact street corner.
- **Routing engine:** a read-only index in RAM. Job: answer one shortest-path question in about a millisecond. Analogy: a metro map that has shortcuts drawn in, so you never trace every side street.
- **Traffic pipeline:** keeps the weights fresh. Analogy: a radio traffic report updating the cost of each road without redrawing the map.
- **ETA service:** a separate learned model. Analogy: the route engine picks the road, the ETA service says how long it will really take.
- **Regional partitioning (inference):** big providers split the world by region so each server holds a manageable graph. Exact partitioning of Google's system is not public.

---

## 4. The transferable mechanisms

### 4.1 Offline think, online lookup (precompute so queries are cheap)
- Contraction Hierarchies (CH), by Geisberger, Sanders, Schultes and Delling (2008), spend minutes to hours of preprocessing once so that each query takes about a millisecond. The original work reports queries under 1 ms, and an "economical" variant at about **200 microseconds**, on the Europe benchmark (per the thesis excerpt in Section 9). That is roughly **1000x to 5000x** faster than Dijkstra.
- Real analogy: highways. You do not plan Delhi to Mumbai by choosing every street. You get on the local road, join the national highway, drive far, then leave it near the end. CH discovers that hierarchy automatically.

### 4.2 Contraction with shortcuts (how CH is built)
- Rank every node by "importance" (a side street ranks low, a highway junction ranks high). Then remove nodes from least to most important. When you remove node `v`, check each pair of its neighbours `u` and `w`. If the shortest `u` to `w` path goes through `v`, add a **shortcut edge** `u -> w` with weight `weight(u,v) + weight(v,w)`.
- Tiny example: a village road `A - V - B` with weights 3 and 4. Remove `V`. Add shortcut `A - B` with weight 7, remembering it "skips V". The shortest distance A to B is preserved exactly. The graph got smaller and the answer did not change.
- Cost: shortcuts add edges (a few times the original edge count is the usual range, from memory of the paper, not re-checked today). Bought back by tiny queries.

### 4.3 Bidirectional upward search (how the query works)
- Run two searches at the same time: one forward from the origin, one backward from the destination. Each is only allowed to go **up** to more important nodes. They meet at a high-ranked node. The shortest route is the best meeting point.
- Because everything above a village is highway-like and sparse, each search touches only a few hundred to a few thousand nodes, not millions.
- Then **unpack** the shortcuts back into real roads. The thesis excerpt gives about 300 microseconds for unpacking.
- Walked query: Gurugram to Jaipur. Forward search climbs from local lanes to NH48. Backward search climbs from Jaipur's streets to the same NH48 junctions. They meet near Kotputli. The shortcut `Manesar -> Kotputli` is unpacked into the actual highway segments. Total work: thousands of node visits, not millions.

### 4.4 Separate the shape of the graph from the weights (customizable routing)
- Plain CH bakes the weights into the shortcuts. A traffic jam changes weights, so you would rebuild the hierarchy, which is far too slow to do every minute.
- Fix: **Customizable Route Planning** (Delling et al., Microsoft Research) and **Customizable Contraction Hierarchies** (Dibbelt, Strasser, Wagner). Phase 1 (slow, once): compute the graph's structure using only topology. Phase 2 (fast, repeated): "customize", meaning fill in weights for the already-built structure. Phase 3: query.
- The research claim is that customization is fast enough to apply traffic updates or user preferences (avoid tolls, bike vs car) without redoing phase 1. Exact times per update are in the papers, which I could not open today, so I give no number.
- OSRM, the open source engine behind many apps, offers two modes: CH (fastest queries, slow re-weighting) and **MLD** (multi-level Dijkstra, a customizable approach, faster to update with traffic). Per the OSRM docs and secondary summaries. It is a direct engineering example of this trade.

### 4.5 Keep the graph immutable and swap it atomically
- Build the new metric into a new read-only structure. Serve from the old one until the new one is ready. Then swap a pointer. No locks on the query path, and a query never sees half-updated weights.
- Analogy: printing a new edition of the road atlas, then putting the new one on the shelf in one move, instead of editing every copy in people's hands.

### 4.6 Separate the "which path" problem from the "how long" problem
- The path search minimizes a cost that is a fairly simple number per edge. The ETA that you show is a different, harder prediction problem, because traffic changes while you drive.
- DeepMind and Google Maps (2020) split roads into **Supersegments**, consecutive road pieces that share traffic, and used a **graph neural network** to predict travel time for each. They report improving ETA accuracy by **up to 50%** in cities such as Berlin, Jakarta, Sao Paulo, Sydney, Tokyo and Washington DC (per DeepMind blog and VentureBeat, Section 9).
- Lesson: a fast exact algorithm on approximate weights gives a fast, exact answer to the wrong question. Spend effort on the weights too.

---

## 5. The trade-offs accepted

| Data type | Choice | Why |
|---|---|---|
| Road topology (the graph) | **Strong, offline, versioned** | Built in batch from map data, validated, then published as an immutable snapshot. A wrong edge is a wrong route. |
| Live traffic weights | **Eventual, minutes old, availability over consistency (AP)** | A route using traffic from 2 minutes ago is fine. A routing service that blocks until traffic is perfectly current is not. |
| Incidents and closures | Faster path, still eventual | Safety relevant, so pushed with higher priority, but still not transactional. |
| Route result | Not cached globally (many unique pairs), cached for repeat popular pairs (inference) | Origin and destination pairs are enormous in number, so a naive cache has a low hit rate. |

- **Cost vs latency:** CH preprocessing costs hours of CPU and extra RAM for shortcuts, and in exchange each query costs microseconds. You pay once per map update to save on every query.
- **Speed vs freshness:** the fastest query structure (plain CH, hub labels) is the least flexible when weights change. Customizable variants give up some query speed to update quickly. You choose per product. A turn-by-turn app with live traffic needs updates. A static "driving distance" feature does not.
- **Exactness vs speed:** CH stays exact. Some systems accept approximate answers (landmarks, heuristics) for even lower cost. CH shows you can often keep exactness and still be fast.

---

## 6. The systems-thinking lens

**The loop: reroute herding (a feedback loop between the router and the traffic it measures).**

```
jam on road A -> router sends everyone to road B -> road B jams
     ^                                                    |
     +----- router sees B jam, flips everyone back to A <-+
```

- This is not a server failure loop. It is a **control loop with delay**: traffic data is minutes old, so the router acts on the past, and every driver reacts to the same signal at once. Same shape as a thundering herd (Day 16) with roads instead of cache keys.
- Adding capacity (more routing servers) does nothing. The senior fix breaks the loop:
  1. **Spread the load across near-equal routes:** return alternatives, and give different users different near-optimal routes. Same idea as jittered retries (Day 13), applied to traffic.
  2. **Predict, do not just react:** the ETA model forecasts how traffic will look when you arrive at each segment (15 to 45 minutes ahead is the range cited for GNN forecasting in the excerpts), so the router does not chase the current snapshot.
  3. **Damp updates:** blend new traffic into old rather than replacing it, so a single noisy minute does not flip routes.
- Server-side version of the same lesson: a traffic update that triggers an expensive full rebuild creates a **rebuild storm**. Customizable structures break that loop by making the update cheap, and the atomic swap means queries never wait on it.

---

## 7. Map to Rare.lab's stack (Supabase Postgres with RLS, R2 immutable scene JSON plus manifest, one shared WebGL context)

| Routing pattern | Rare.lab touchpoint | Status | Action |
|---|---|---|---|
| Offline think, online lookup | Compiling a node graph to shippable code | **Already doing the core of this** | The compiler is your "contraction step": expensive once, cheap per frame. Keep every expensive analysis (dependency order, dead node removal, constant folding) at compile time. |
| Separate shape from weights | Node graph structure vs parameter values (colour, speed, intensity) | Strong fit, check the code | The compiled program should treat parameters as uniforms, not baked constants, so changing a slider never recompiles. This is exactly "customize weights without rebuilding the hierarchy". If a slider change triggers a recompile, that is your rebuild storm. |
| Immutable snapshot plus atomic swap | R2 content-addressed scene JSON plus manifest | **Already doing this** | The manifest pointer flip is the atomic swap. Keep it as the only mutable thing. Builds never edit objects in place. |
| Shortcuts (precomputed skips) | Cached intermediate results (static subgraphs, baked textures) | Not yet | A subgraph whose inputs never change can be collapsed to one baked node, like a shortcut. Store it content-addressed on R2 keyed by subgraph hash plus compiler version. |
| Graph queries in Postgres | "Which scenes use this node preset?", dependency lookups | Watch out | A recursive CTE over a deep graph is a poor man's Dijkstra. Fine at 1,000 rows. At millions, precompute a closure table or materialize dependencies on write, not on read. |
| Reroute herding | Many clients reloading the manifest at once after a publish | Possible | Add jitter to manifest polling and cache with short TTL plus stale-while-revalidate (Day 80 pattern). |

**Where the next ceiling is (inference, not measured):** recompile latency for large graphs. Past a few thousand nodes, compiling the whole graph on each edit gets slow. First fix: incremental compile, where you recompute only the changed node and its downstream dependents (same idea as customizing only affected cells in CRP). Second fix: cache compiled subgraph results by hash so two users editing near-identical graphs share work.

**One-line lesson for Rare.lab:** do all the structural analysis of the graph once at compile time, keep parameters as cheap-to-change inputs rather than baked constants, and publish results as immutable snapshots, because the routing world shows that this one split buys a 1000x speedup.

---

## 8. What is inside the video you shared (recap, already covered)

The "Design LeetCode" mock interview (Vanika Agarwal, Google) is fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md), so I did not repeat it. Plain-language spine for your notes:
1. Requirements first: functional (list problems, view one and code in any language, submit and get instant results, live contest leaderboard), then out of scope (auth, payments, analytics), then non-functional (availability over consistency, isolated code execution, low latency, scale to about 100k contest users).
2. Entities, then REST API, then a simple diagram, then fix it step by step.
3. Her fixes: never run user code on the API server, run it in containers per language, put a queue between API and runners with exponential backoff retries, cache the leaderboard, replicate the submissions database with failover.
4. **Link to today:** her leaderboard cache is "do the work once, serve it to everyone", the same as the offline step here. Her queue-between-API-and-runners is the shock absorber from Day 9. The interviewer's push on VMs vs containers vs serverless is the same cost-vs-isolation trade-off.

Use the same recipe on any product: requirements, entities, API, naive design, find the breaking number, fix one layer at a time, name the trade-off.

---

## 9. References and what is actually in them

**Honest note on access:** arxiv.org was blocked by the network proxy in this session, so I could not open the papers. Facts below come from web search excerpts and are marked. Verify numbers before quoting them.

- [Route Planning in Transportation Networks (Bast et al., survey)](https://arxiv.org/pdf/1504.05140). The best single overview. Compares Dijkstra, A*, CH, transit node routing, hub labels and customizable approaches on the Western Europe graph. Source of the 18.0 million vertex, 42.5 million arc figure and the "seconds per query" statement (via excerpt).
- [Contraction Hierarchies: Faster and Simpler (Geisberger diploma thesis)](https://ae.iti.kit.edu/download/diploma_thesis_geisberger.pdf) and [Exact Routing in Large Road Networks Using Contraction Hierarchies](https://publikationen.bibliothek.kit.edu/1000028701/142973925). The primary CH sources. Explains node ordering by edge difference, shortcuts, bidirectional upward query. Source of "under 1 ms" and "about 200 microseconds" (via excerpt).
- [Customizable Contraction Hierarchies (Dibbelt, Strasser, Wagner)](https://arxiv.org/pdf/1402.0402) and [Customizable Route Planning in Road Networks (Delling et al., Microsoft Research)](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/01/crp_web_130724.pdf). Describe the split into topology preprocessing, fast weight customization, and query, for traffic and user preferences. Not opened today.
- [Time Dependent Contraction Hierarchies](https://arxiv.org/pdf/0804.3947). Extends CH to edge weights that depend on time of day, the stricter version of the traffic problem.
- [OSRM (Open Source Routing Machine)](https://github.com/Project-OSRM/osrm-backend). Production-grade open source engine with CH and MLD modes. The wiki page did not load, so the CH vs MLD description is from search summaries.
- [Traffic prediction with advanced Graph Neural Networks (DeepMind blog)](https://deepmind.com/blog/article/traffic-prediction-with-advanced-graph-neural-networks) and [VentureBeat summary](https://venturebeat.com/ai/deepmind-claims-its-ai-improved-google-maps-travel-time-estimates-by-up-to-50). Supersegments, the GNN, and the up to 50% ETA improvement claim.
- [Improving Google Maps Using GNNs (Substack)](https://vkeerthivikram.substack.com/p/improving-google-maps-using-gnns). Lower authority, plain-language retelling of the DeepMind post.
- **From the video you shared:** Hello Interview (hellointerview.com) and Shreyansh Jain's YouTube channel, as recommended by the guest. Not opened today.

**Inference, labeled:** the 1 second per Dijkstra query estimate and its per-node cost, the 100,000 requests per second illustration, regional partitioning, route caching behaviour, reroute herding discussion beyond what the sources state, and all of Section 7.

**Related ledger lessons:** Day 9 (queue as shock absorber), Day 13 (backpressure), Day 16 (hot key and herds), Day 19 (caching), Day 48 (geospatial indexing), Day 60 (shader compilation caching), Day 74 (LeetCode), Day 80 (Discord).
