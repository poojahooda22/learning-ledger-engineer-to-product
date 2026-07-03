# Rapido: turn-by-turn navigation for the captain (the routing engine)

Date: 2026-07-03
Product: Rapido (India's largest bike-taxi and mobility app)
Feature: The in-app turn-by-turn navigation that draws the blue line for the captain and re-draws it the moment he misses a turn

A note on sourcing before we start. Rapido has published about its navigation
work on its engineering blog (Rapido Labs) and in a Google Maps Platform case
study. Those give us the confirmed shape: Rapido runs a multi-provider in-app
Navigation SDK for the captain app, it does two-wheeler routing, and it operates
at roughly 4 million captains and 60 million rides a month. The exact internal
routing engine (which graph, which algorithm, how the shortcuts are built) is
not fully public. So the "under the hood" section separates two things clearly:
what Rapido has said (FACT), and how this exact class of problem, shortest path
on a country-sized road graph, is universally solved (INFERENCE). The inference
is not hand-waving. It is the same math that OSRM, GraphHopper, Valhalla, and
Google Maps all run, with real papers and numbers attached.

---

## 1. The user

Meet Suresh. He is a Rapido captain in Bengaluru. It is 9:10 am, peak hour, and
his phone buzzes with a ride. Pick up Ananya from a lane behind Indiranagar 100
Feet Road, drop her at Manyata Tech Park, 11 km away. Suresh has never been down
that exact lane. He is sitting on a scooter in moving traffic, helmet on, one
glance at the phone allowed every few seconds, no second person to read the map
for him. He needs the app to tell him, out loud and with a fat blue line, "in
200 meters, turn left," and he needs it to be right, because a wrong turn on a
one-way in Indiranagar costs him four minutes and a U-turn he is not allowed to
make.

The rider Ananya is the other user. She is watching Suresh's little scooter icon
crawl toward her on her screen and she is deciding, second by second, whether to
trust the "arriving in 3 min" or cancel and book an auto.

---

## 2. The real problem

Told like a friend would tell it: a car has a co-pilot seat and a windshield
mount and a driver who can afford to stare at a screen. A bike captain has none
of that. He is balancing a two-wheeler, he cannot look down for more than a
blink, and the road he needs is often a 3-meter gully that Google's car
navigation would never send a car into but a scooter can take as a shortcut.

Worse, the two most common navigation apps are built for cars. Car navigation in
India will happily route you onto a flyover or an expressway where two-wheelers
are legally banned, and it will refuse the narrow cut-through that every local
scooter uses. It also will not U-turn a bike the way it U-turns a car, because
the turn rules are different. Confirmed by OpenStreetMap's own routing notes:
standard engines like OSRM do not even read road width, so they cannot tell a
scooter-width lane from a truck road.

And the whole thing has to survive being wrong for a second. GPS on a cheap
Android phone in a concrete canyon drifts 20 to 40 meters. So the app constantly
thinks the captain is on the wrong road, and it has to decide, calmly and fast,
"is he actually off route, or is this just GPS lying again?" Get that wrong and
you either nag a captain who is on the right road, or you fail to reroute a
captain who genuinely missed the turn.

Rapido's own stated business pain sits right on top of this: ride
cancellations. A captain who cannot find the pickup, or takes a bad route, is a
cancelled ride, a lost fare, and a rider who books the competitor next time.

---

## 3. The feature in one sentence

Given the captain's live GPS dot and a destination, compute the best legal
two-wheeler route on India's road network, speak it turn by turn, and recompute
it in well under a second every time he deviates.

---

## 4. Jobs to be done

What Suresh is really hiring this feature to do:
- "Get me to the pickup by the shortest legal scooter path, not the car path."
- "Tell me the next turn early enough that I can change lanes safely."
- "When I miss a turn, do not scold me, just quietly give me the new line."
- "Do not send me onto a flyover I am not allowed on."
- "Keep working when my data signal drops under a metro bridge."

What Ananya is hiring it to do:
- "Show me a scooter icon that moves smoothly and an ETA I can trust, so I do
  not cancel."

What Rapido as a business is hiring it to do:
- "Turn every accepted ride into a completed ride. Cut the cancellations that
  come from a captain getting lost or taking a slow route."

---

## 5. How it works for the user

Suresh accepts the ride. The captain app instantly draws a blue line from his
current spot to Ananya's pin, shows "1.2 km, 4 min to pickup," and starts
speaking: "Head north on 100 Feet Road." A big arrow card sits at the top:
"Turn left in 200 m." As he rides, the line shortens under his icon, the
distance-to-turn counts down, and the voice fires again at 200 m, 50 m, and at
the turn. When he overshoots the left because a bus blocked it, the app does not
freeze. Within a second the blue line snaps to a new shape that takes the next
left instead, the voice says "rerouting," and the ETA nudges up by one minute.
After pickup, the destination flips to Manyata Tech Park and the same dance
repeats for the 11 km main leg.

The rider Ananya never sees the routing engine. She sees its output: a smooth
scooter icon and a shrinking "arriving in X min."

---

## 6. The actual flow, step by step

1. Ride assigned. Server sends the captain app two points: current location and
   pickup location (later, pickup to drop).
2. The app asks the routing layer for a route between those two points, for a
   two-wheeler profile, biased to time not distance, avoiding banned roads.
3. The routing layer returns an ordered list: a polyline (the blue line) plus a
   list of maneuvers ("turn left onto 12th Main, 200 m") plus a total time and
   distance.
4. The app draws the polyline, shows the first maneuver card, and starts the
   voice guide.
5. Every second or so the phone gets a fresh GPS fix. The app map-matches that
   raw dot onto the route's road (snaps it to the line) so the icon sits on the
   road, not in a building.
6. The app checks: is the matched position still on the planned route, within
   tolerance? If yes, it just advances the "distance to next turn" counter.
7. If the position has clearly left the route (off by more than a threshold for
   more than a beat), it fires a reroute: ask the routing layer for a fresh route
   from the new position to the same destination. Redraw. Re-speak.
8. On arrival at pickup, the leg ends, the destination switches to the drop, and
   the loop restarts.

The heavy computation is in steps 2 and 7. Everything else is cheap bookkeeping
on the phone.

---

## 7. Under the hood, like the engineer

This is the heart of it. A navigation feature is really two separate engineering
problems wearing one coat. Call them the two halves, the same way search splits
into matching and ranking:

- Half one, the model: build the right graph. What roads exist, which direction
  they go, which ones a scooter may legally use, and how long each takes right
  now. This is where "two-wheeler" actually lives.
- Half two, the algorithm: search that graph fast. Given two points on it, find
  the lowest-cost path, in under a millisecond, a few thousand times a second.

Most people think navigation is half two. The hard, India-specific value is
actually half one.

### The data structure: a road network is a directed weighted graph

FACT (this is universal, not a guess): every routing engine models the road
network as a graph. Each intersection is a node. Each road segment between two
intersections is a directed edge. The edge carries a weight, which is not
distance but estimated travel time. A one-way street is a single directed edge.
A two-way street is two edges pointing opposite ways.

Make it concrete. The corner of 100 Feet Road and 12th Main in Indiranagar is a
node, call it N_4471. The stretch of 12th Main from there to CMH Road is an edge
with a weight of, say, 48 seconds at 9 am. Turn restrictions ("no left here 8 to
11 am") are modeled as turn penalties on the pair of edges, or as a small
expanded sub-graph at the junction. India's full OpenStreetMap road graph is on
the order of tens of millions of nodes and edges.

Why a graph and not something simpler? Because "shortest path" is a graph
question with a 65-year-old exact answer (Dijkstra, 1959), and because the graph
lets you attach per-edge rules. The two-wheeler profile is literally a filter and
a re-weighting on this graph: drop the edges tagged as motorway or expressway
(two-wheelers banned, confirmed as a real India problem in OSM routing
discussions and the reason Google shipped a separate Two-Wheeler mode for
India), keep the narrow residential edges a car profile would down-rank, and set
speeds to scooter speeds. Same nodes, different allowed edge set and different
weights. That single idea, "a profile is a re-weighting of one shared graph," is
why an engine can serve car, auto, and bike from the same map.

### The algorithm, tier by tier: why plain Dijkstra dies and what replaces it

Here is the scale story, told as three honest tiers.

Tier 1, about 1,000 edges, one neighborhood (Indiranagar alone).
Plain Dijkstra with a binary-heap priority queue is more than enough. You start
at the source node, repeatedly pop the closest-unsettled node from the heap,
relax its neighbors, and stop when you pop the destination. Over 1,000 edges this
settles in microseconds. A phone could do it. Data structures: an adjacency list
for the graph (each node holds an array of its outgoing edges) and a binary min-
heap for the frontier. Nothing clever needed. If Rapido only operated in one
locality, the story would end here.

Tier 2, about 100,000 nodes, one city (all of Bengaluru).
Dijkstra still returns the correct answer, but it is now doing too much work per
query. To find a path across the city it will settle a large fraction of the
city's nodes, because Dijkstra explores outward in all directions like a growing
circle, blind to where the destination is. That is fine once. It is not fine at
thousands of route requests per second during morning peak.

Two standard fixes, both real:
- A* search. Give Dijkstra a sense of direction. Add a heuristic, the straight-
  line distance from each node to the destination divided by max speed, and let
  the search prefer nodes that point toward the goal. The circle becomes an
  ellipse aimed at the destination. A* settles far fewer nodes. It is still exact
  as long as the heuristic never overestimates.
- Bidirectional search. Grow one frontier from the source and another from the
  destination at the same time, and stop when they meet in the middle. Two small
  circles beat one big circle because search cost grows faster than linearly with
  radius.

These help, but at country scale even A* explores too much.

Tier 3, 10 million plus nodes, all of India, times millions of route requests a
month.
Now the naive approaches genuinely fall over. A single cross-city query on a raw
10M-node graph can settle hundreds of thousands of nodes and take tens of
milliseconds. Multiply by Rapido's confirmed volume (about 60 million rides a
month, and every ride triggers not one route but a pickup route, a trip route,
and many reroutes along the way, so the real number of route computations is
several times higher) and a raw-Dijkstra fleet would need an absurd amount of
compute.

The survival trick is the same spine this ledger keeps finding: do the expensive
thinking offline, once, so the live query is cheap. For routing, the specific
technique is Contraction Hierarchies.

FACT (real, cited): Contraction Hierarchies (CH) were introduced in 2008 by
Robert Geisberger, Peter Sanders, Dominik Schultes, and Daniel Delling at
Karlsruhe. The idea: preprocess the whole graph once. Order the nodes by
"importance." Then contract them one at a time from least important to most,
and whenever removing a node would break a shortest path, insert a shortcut edge
that preserves that path's length. A shortcut can span a whole highway on-ramp,
merge, and off-ramp as a single edge with the summed weight. After preprocessing
you have the original graph plus a pile of shortcut edges arranged in a
hierarchy.

At query time you run a bidirectional Dijkstra that is only ever allowed to move
"upward" in the hierarchy, from less important to more important nodes. It rides
shortcuts across the country instead of crawling edge by edge. The published
result: on the Western European road network of more than 18 million nodes, CH
answers an exact shortest-path query in under 200 microseconds, visiting only a
few hundred nodes instead of millions. Preprocessing the whole continent takes
minutes to a couple of hours, done offline. Google Maps itself is documented to
use contraction-hierarchy-style techniques. OSRM is built on CH. GraphHopper
uses CH. This is not exotic; it is the industry default.

Walk Suresh's 11 km Manyata trip through it. Raw Dijkstra would crawl every
service road and gully between Indiranagar and Manyata, settling maybe 200,000
nodes. With CH, the query hops onto a shortcut that represents "Indiranagar to
the Outer Ring Road entrance," another shortcut "along ORR to Hebbal," another
"Hebbal to Manyata gate," and stitches the exact fine-grained path only at the
two ends near the source and destination. A few hundred nodes touched. Sub-
millisecond. That is the difference between a routing box serving 50 queries a
second and one serving 50,000.

Why CH and not just precompute every route? Because precomputing all pairs on a
10M-node graph is 10^14 routes, which is impossible to store. CH is the middle
path: precompute reusable shortcuts, not answers, so any of the 10^14 queries
becomes cheap without storing any of them. That trade, "cache the structure, not
the answers," is the whole game.

INFERENCE (clearly labeled): the exact engine Rapido runs is not fully public,
and their blog says they deliberately use multiple navigation providers (their
own SDK plus third parties) rather than one. But any engine that serves turn-by-
turn at their scale, in-house or bought, is doing CH-class preprocessing on an
OpenStreetMap-derived graph. There is no other way to hit the latency. So the CH
story above is the well-grounded "how this class of problem is solved" version,
not a claim about a specific Rapido code path.

### The other hard half: keeping the icon on the road (map matching)

The route is only half the live loop. The other half is figuring out where the
captain actually is, because raw GPS is a liar. A fix in Indiranagar's mid-rise
canyon can land 30 meters off, inside a building, on the wrong parallel road.

The fix is map matching: snap the noisy GPS trail onto the most likely sequence
of road edges. The standard method, and it is worth naming because this ledger
saw it before in the Swiggy teardown, is a Hidden Markov Model solved with the
Viterbi algorithm (Newson and Krumm, Microsoft Research, 2009). Hidden states
are "which road segment am I really on," observations are the GPS points, and
Viterbi finds the most likely road sequence given the whole recent trail, not
just the latest point. So one bad fix that jumps to a parallel lane gets
overruled by the five good fixes around it. The icon stays on the real road, and
the "are you off route?" check compares the matched position, not the raw dot, to
the planned line. That is what stops the app from crying "rerouting" every time
GPS twitches.

Between real GPS fixes, which arrive only every 1 to 3 seconds, the app
interpolates the icon along the route at 60 fps so the motion looks smooth. Same
"sparse truth, smooth fiction" trick as live order tracking: the truth updates a
few times a second, the animation fills the gaps.

### Serving, sharding, and the offline pipeline

FACT plus standard practice:
- The prepared graph (nodes, edges, shortcuts) is loaded into memory on the
  routing servers, typically memory-mapped so a fresh process is ready in
  seconds and the OS handles paging. A query never touches a disk database; it
  walks an in-memory graph.
- The graph is partitioned. You do not need all of India in one process to route
  inside Bengaluru. Shard by region or by map tile, route requests hit the shard
  that owns the area, and only long inter-city trips need cross-shard stitching.
  Rapido operates city by city, which lines up naturally with per-city or per-
  region graph shards.
- The map itself is an offline data pipeline. OpenStreetMap plus Rapido's own
  observed data get compiled into the routing graph on a schedule (roads change,
  new flyovers open, a lane becomes one-way). Edge weights, the live-ish travel
  times, come from the fleet's own GPS traces: with about 4 million captains
  riding, Rapido's own scooters are a live traffic-probe network measuring the
  real speed on 12th Main at 9 am. This is confirmed in spirit by the Google case
  study, which credits better location data (captain location visibility raised
  from 60% to 95% via their Fleet Analytics work) with improving matching and
  fulfilled orders.
- Reroutes are just fresh queries from the new point, and because each query is
  sub-millisecond, a reroute is essentially free. That is why the app can reroute
  within a second of a missed turn without the captain noticing a lag.

The clean summary: the graph build and the CH preprocessing are the slow offline
brain. The live captain request is a cheap, bounded, memory lookup that does not
grow with the size of India. Offline-think, online-lookup, one more time.

---

## 8. The retention and habit mechanic

Navigation is not a consumer engagement loop. There is no streak, no red badge.
Its retention power is indirect and it is stronger for it, because it moves the
one metric a marketplace lives on: completed rides, through two loops.

Loop one, captain retention (the supply side). A captain earns per completed
ride. Better routing means he reaches the pickup faster, takes the efficient
scooter path, wastes less fuel, and completes more rides per hour. More rupees
per hour is the only loyalty program that keeps a gig captain on your app instead
of the competitor's. Rapido's confirmed 60 to 95 percent jump in captain-location
visibility fed directly into "increasing the number of customer orders fulfilled
each day." Navigation quality is captain earnings, and captain earnings is
captain supply, and supply is the whole business.

Loop two, rider trust (the demand side). The rider's decision to not cancel is
made in the 3 minutes she watches the scooter icon approach. A smooth, accurate
icon and a trustworthy "arriving in 3 min" keep her from bailing to an auto.
Rapido has publicly framed reducing cancellations as a core goal, and a lost
pickup is very often a captain who could not find the lane. Kill the "captain got
lost" cancellation and you convert an accepted ride into a completed ride and a
rider who books again.

Which metric: retention and, through it, revenue. Not activation. The loop is the
liquidity flywheel, the same engine named in the Uber dispatch teardown: reliable
completion keeps captains earning and riders returning, each side making the
other side's experience better. Navigation is a quiet input to that flywheel, but
it is load-bearing.

Real observed example: the Google case study reports that raising captain
location visibility from 60% to 95% (a data-and-navigation improvement)
measurably increased fulfilled orders per day. That is the mechanic in one
sentence: better line on the map, more completed rides, more reason for both
sides to come back tomorrow.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. A routing engine is a shockingly good
mirror for exactly that shape, and it hands you two concrete lessons.

Lesson one, the big one: contract the graph offline so the runtime query cost is
decoupled from graph size. A road network and a shader node graph are both
directed graphs that a naive engine would traverse edge by edge every time. The
routing world learned that you must not pay the full-graph traversal at query
time; you precompute a hierarchy of shortcuts once (Contraction Hierarchies) so
the live query touches a few hundred nodes instead of millions, and stays sub-
millisecond no matter how big the map grows. Do the same to the shader DAG at
compile time. When the editor graph is compiled to shippable code, run the heavy
analysis then: constant-fold, dead-node eliminate, fuse chains of nodes into a
single "shortcut" pass the way CH fuses a highway on-ramp into one edge, bake
expensive-but-static sub-graphs into lookup textures (LUTs) or precomputed
uniforms. The goal is that a 500-node artist graph and a 50-node graph produce a
per-frame runtime cost that looks nearly the same, because the runtime is walking
a contracted, flattened form, not the artist's raw graph. Per-frame cost must be
decoupled from editor-graph complexity, exactly as CH decouples query cost from
map size. If your runtime frame time scales with how many nodes the artist
dragged in, you have shipped raw Dijkstra.

Lesson two, ship profiles, not one output. The single most India-specific idea in
Rapido's navigation is that a "two-wheeler profile" is just a re-weighting and
edge-filtering of one shared graph, so car, auto, and bike all come from the same
map with different rules. Rare.lab has the same opportunity across GPU targets. A
low-end mobile GPU, a high-end desktop, and a WebGL runtime are your "vehicles."
Do not author three graphs. Compile one editor graph through target profiles: the
mobile profile prunes the expensive nodes (drop the 16-tap blur to 4-tap, bake
more to LUTs, cap texture reads), the desktop profile keeps the full quality,
just as the two-wheeler profile drops the banned motorway edges. One graph, many
compiled outputs, each tuned to what the device can legally and physically
afford. That is how you keep a single artist workflow while still hitting frame
budget on the weakest device you support.

The through-line for both: the editor is the slow offline brain where all the
hard analysis happens; the embeddable runtime should be a cheap, bounded,
cache-friendly walk over a precompiled structure. Offline-think, online-lookup.
The routing engine has been living that principle at country scale for fifteen
years. Steal it.

---

## Sources

- Rapido Labs, "Rapido Navigation: Getting You There, Wherever You're Going" (Inder Singh): https://medium.com/rapido-labs/rapido-navigation-getting-you-there-wherever-youre-going-867e1993c80f
- Rapido Labs, "Improving Ride Dispatch with Data at Rapido" (Siddharth Panchanathan): https://medium.com/rapido-labs/improving-dispatch-with-data-6a307dab7ecc
- Google Maps Platform, "How Rapido is building customer and rider trust in India with Google Maps Platform": https://mapsplatform.google.com/resources/blog/how-rapido-is-building-customer-and-rider-trust-in-india-with-google-maps/
- Google Cloud, Rapido case study: https://cloud.google.com/customers/rapido-maps
- Geisberger, Sanders, Schultes, Delling, "Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks" (2008): https://link.springer.com/chapter/10.1007/978-3-540-68552-4_24
- "Exact Routing in Large Road Networks Using Contraction Hierarchies," Transportation Science (2012): https://pubsonline.informs.org/doi/10.1287/trsc.1110.0401
- Newson and Krumm, "Hidden Markov Map Matching Through Noise and Sparseness" (Microsoft Research, 2009): https://www.microsoft.com/en-us/research/publication/hidden-markov-map-matching-noise-sparseness/
- Google for Developers, "Get a two-wheeled vehicle route" (Routes API, two-wheeler mode for India): https://developers.google.com/maps/documentation/routes/route_two_wheel
- OpenStreetMap Wiki, Routing and Narrow Roads: https://wiki.openstreetmap.org/wiki/Routing/Narrow_Roads
- Project OSRM (open-source routing on OSM, contraction hierarchies): https://github.com/Project-OSRM/osrm-backend
- GraphHopper routing engine (CH, bike/two-wheeler profiles): https://github.com/graphhopper/graphhopper
