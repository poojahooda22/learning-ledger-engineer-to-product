# Swiggy: restaurant serviceability (which restaurants can even deliver to your pin)

Date: 2026-09-28
Product: Swiggy
Feature: Serviceability. Given your exact GPS pin, decide which restaurants and
stores are eligible to deliver to you, before anything gets ranked.

This is the step that runs BEFORE the restaurant listing feed you already tore
down on 2026-08-30. That teardown was about ORDER: given a set of eligible
restaurants, which one sits at the top. This one is about the SET itself: out of
every restaurant Swiggy has in a city, which handful are even allowed to show up
for the little blue dot you just dropped on the map. Matching first, ranking
second. This is the matching half, and it is a geometry problem, not a
relevance problem.

---

## 1. The user

Ananya lives in Koramangala 5th Block in Bangalore. It is 9:20 on a Tuesday
night. She has been on calls since morning, there is nothing worth cooking in
the fridge, and she wants biryani. She opens Swiggy. Before she types a single
letter, before she even thinks the word "Meghana," the home screen has to fill
with restaurants. Not just any restaurants. Restaurants that will actually send
food to her building tonight, in a reasonable time, hot.

She is not thinking about any of this. She is thinking about biryani. The whole
job of serviceability is to make sure that in the two seconds between "open app"
and "see food," she never once sees a restaurant she cannot order from.

## 2. The real problem

Here is the pain, described like a friend would.

Imagine the app showed her every restaurant in Bangalore. She scrolls, she finds
a place in Whitefield that looks incredible, she builds a cart, she taps
checkout, and only THEN the app says "sorry, this restaurant does not deliver to
your location." That is a small betrayal. She did the work of choosing and got
punished for it. Do that twice and she stops trusting the app.

Now flip it. Imagine the app is too cautious and only shows her the three
restaurants on her own street. She misses Meghana Foods on Residency Road, which
absolutely would have delivered to her in 30 minutes. Swiggy just lost a real
order, and Ananya thinks "there is nothing good near me" when there was.

So serviceability sits on a knife edge. Show too much and you break trust at
checkout. Show too little and you lose orders and look empty. And the "right
answer" is not a neat circle around her house. A restaurant across the Bellandur
lake might be 1.5 km away as the crow flies but 20 minutes away by road because
the only bridge is far off. A restaurant on the far side of the Sarjapur flyover
might be close on the map but a nightmare for a two-wheeler to reach at night.
Distance on a map lies. What matters is whether a delivery partner can actually
get from that kitchen to Ananya's door fast enough for the food to arrive hot.

## 3. The feature in one sentence

Serviceability takes your exact drop location and, in a few milliseconds,
returns only the restaurants and stores that are allowed to deliver to that
point right now.

## 4. Jobs to be done

What is Ananya really hiring this feature to do?

- "Only show me places I can actually order from tonight, so I never hit a dead
  end at checkout."
- "Do it before I even notice, so the app feels instant and full, not empty."
- "Be honest about my specific spot, not a lazy circle. If the good biryani
  place two roads over can reach me, show it. If the place that looks close but
  is across the lake cannot, hide it."

And Swiggy is hiring it to do something too: protect the promise. Swiggy sells
fast delivery. Every restaurant it shows is an implicit promise "this will reach
you hot and soon." Serviceability is the gate that only lets through promises
Swiggy can keep.

## 5. How it works for the user

The visible experience is almost nothing, and that is the point.

Ananya opens the app. At the top, her address: "Home, Koramangala 5th Block."
The screen fills with restaurant cards. Meghana Foods is there. Truffles on St
Marks Road is there. A local darshini around the corner is there. A place she
loved in Indiranagar last month is NOT there tonight, because at 9:20pm the
delivery time to her would blow past the limit, so Swiggy quietly leaves it out.

If she changes her address to her parents' place in Jayanagar, the entire list
changes in a blink. Different pin, different eligible set. She never sees the
machinery. She just sees "food near me," and it is always food she can get.

The only time serviceability becomes visible is at the edges. Drop a pin in the
middle of a lake or on a highway with no buildings and Swiggy says "we do not
deliver here yet." Move to a brand new outer suburb and you might see "coming
soon." Those are serviceability saying no out loud. The rest of the time it says
yes silently.

## 6. The actual flow, step by step

Tap by tap, what happens the moment Ananya opens the app:

1. The app reads her selected address. Behind that friendly label "Home" sits a
   precise latitude and longitude, for example 12.9352, 77.6245. That pair of
   numbers is the real input. The label is just for humans.
2. The app sends that lat-long to Swiggy's serviceability service. Not the
   restaurant list, not a search query. Just "here is a point, what can serve
   it."
3. The service converts the point into a grid-cell key (more on this in the next
   section) and instantly looks up which delivery clusters cover that key.
4. It runs a precise geometry check to confirm the point really sits inside one
   of those cluster shapes, and picks the cluster.
5. From that cluster it does a "directional discovery" of restaurant clusters
   that are allowed to deliver INTO Ananya's cluster, filtered by delivery
   distance and time (Swiggy typically keeps this to restaurants within about 4
   to 5 km).
6. It returns that eligible set of restaurant IDs.
7. Only now does the listing and ranking system (the 2026-08-30 teardown) take
   over: it takes this already-safe set and sorts it by relevance, rating,
   delivery time, ads, and so on.
8. The cards render. Total time budget for the serviceability part: a few
   milliseconds, because it is one small thing standing in front of every single
   app open in the country.

Notice the split cleanly. Steps 3 to 6 decide WHO is eligible. Step 7 decides in
what ORDER. Two different halves, two different systems, and serviceability
never sorts anything. It just filters.

## 7. Under the hood, like the engineer

This is the heart. The core question: given one point and thousands of delivery
zone shapes, how do you find the right ones in milliseconds, for hundreds of
millions of requests, without melting a database?

### The naive version, and why it dies

The obvious approach: store every delivery zone as a polygon (a list of corner
points that trace the boundary of an area a set of restaurants serves). When a
request comes in, test Ananya's point against every polygon and keep the ones
that contain it. That test, "is this point inside this polygon," is a classic
computational geometry problem with a classic answer: RAY CASTING, also called
the crossing number or even-odd rule, known since at least 1962.

Ray casting is beautifully simple. Shoot an imaginary ray from Ananya's point
straight out to infinity in any direction. Count how many times that ray crosses
the polygon's boundary edges. Odd number of crossings means she is inside. Even
number means she is outside. Picture a ray leaving her point and crossing the
wiggly outline of the Koramangala delivery zone: cross once going in, you are
inside; cross a second time coming out, you are outside again. The cost of one
such test is proportional to the number of edges in the polygon, so O(n) in the
polygon's vertex count. (There is a sibling method, the winding number, which
sums up angles instead of counting crossings. It gives the same answer but is
usually slower because it leans on inverse trigonometry, so ray casting is the
workhorse.)

One test is cheap. The problem is the word "every." Swiggy has thousands of
delivery cluster polygons across hundreds of cities. Testing Ananya's point
against all of them on every app open is pure waste, because she is in
Bangalore and 95 percent of those polygons are in other cities she can never be
inside. Doing thousands of O(n) polygon tests per request, times hundreds of
millions of requests a day, is a fire you cannot afford to light. You need to
NOT look at almost all the polygons.

### The real trick: index space so you only test a handful

This is the same instinct as an inverted index in search (the 2026-06-23 Amazon
teardown) or the candidate-fetch step in any ranking system. Never run the
expensive precise check on the whole corpus. Use a cheap index to shrink
"thousands of polygons" down to "the two or three that could possibly contain
this point," THEN run the precise ray-casting test only on those.

For space, that cheap index is a SPATIAL INDEX. The family of tools here:

- GEOHASH: chop the world into a grid and give every cell a short string. Nearby
  places share a string prefix. The cell "tdr1y" sits next to "tdr1z." A point's
  geohash is computed by simple bit-interleaving of its latitude and longitude,
  which is fast and needs no database. Cells are rectangles.
- QUADTREE: recursively split a square into four smaller squares wherever you
  need more detail. Dense city center gets finely split, empty area stays
  coarse.
- GOOGLE S2: project the sphere onto a cube and lay a hierarchy of quadrilateral
  cells on it, ordered along a Hilbert curve so cells near each other in space
  are near each other in the ordering. DoorDash stores each delivery zone as a
  set of S2 cell IDs for exactly this reason.
- UBER H3: tile the world in hexagons across 16 resolution levels (0 the
  coarsest, 15 the finest, cells ranging from roughly 4,000,000 square km down to
  about 1 square meter; around resolution 7 a hexagon is a few square km, the
  scale of a delivery zone). Hexagons have one lovely property that squares do
  not: all six neighbors are equidistant from the center, which makes "expand
  outward evenly" queries clean. Uber built H3 originally to reason about supply
  and demand for surge (the 2026-06-14 teardown).

Swiggy's serviceability platform (described openly in their engineering blog by
Somsubhra Bairi) uses a GEOHASH-keyed cluster index held IN MEMORY. Here is the
exact shape of it in their own words, paraphrased:

> The serviceability engine pre-builds a GeoHash-keyed cluster index in memory.
> The system figures out the GeoHash index key that the customer's location
> belongs to and fetches a much reduced set of clusters overlapping and
> associated with that GeoHash key, then runs a point-in-polygon check on this
> reduced candidate set, which is much more efficient than running a
> point-in-polygon check across all the clusters.

Walk Ananya through it concretely:

1. Her point 12.9352, 77.6245 is converted to a geohash key, say "tdr1yk." This
   is a pure bit-math operation on her lat-long. No database. Nanoseconds.
2. That key is looked up in an in-memory hash map: geohash key maps to the small
   list of delivery cluster polygons that overlap that cell. In Bangalore, for
   her geohash cell, that might be three or four candidate clusters, not
   thousands.
3. Ray casting (the precise point-in-polygon test) runs ONLY on those three or
   four candidates. One of them, the Koramangala delivery cluster, contains her.
   Done.

Swiggy states the payoff plainly: this resolves a customer drop location to its
delivery cluster in O(1) time with ZERO live database queries. The precompute
(building the geohash-to-clusters map) is done ahead of time and held in RAM.
The live path is a hash lookup plus a handful of cheap geometry tests. This is
the offline-think, online-lookup spine that shows up in almost every teardown in
this ledger, from Discover Weekly to Google's index to Amazon search. Do the
heavy structuring once, offline. Make the live request a lookup.

### Why polygons and not a radius

A tempting shortcut: skip polygons, just draw a circle of radius 5 km around
each restaurant and call anyone inside it serviceable. Swiggy does use a distance
ceiling (about 4 to 5 km) as one filter, but a raw circle is wrong, and it is
wrong for a reason worth stating.

A circle measures distance as the crow flies. Delivery partners are not crows.
The real boundary of "can reach in time" is an ISOCHRONE, a drive-time polygon:
the set of all points reachable within, say, 20 minutes along the ACTUAL road
network. That shape is never a circle. It stretches out along fast roads like
the Sarjapur main road, where a rider covers ground quickly, and it pinches in
tight near barriers, like the edge of Bellandur lake where the nearest bridge is
far away. A house 1.5 km away across the lake can be a 20-minute ride, so it
falls OUTSIDE the isochrone even though it is inside a naive 5 km circle. Swiggy
draws its delivery clusters to respect this real geography, which is why they are
hand-shaped polygons and not circles, and why "directionality" matters in their
design: a cluster can serve strongly in one direction and weakly in another.

### The two halves again, spelled out

- MATCHING (serviceability): geometry. Point in polygon, accelerated by a
  spatial index. Answer: the set of eligible restaurant IDs. No notion of
  "better" or "worse." Just in or out.
- RANKING (the listing feed, 2026-08-30): relevance. Given the eligible set,
  sort by rating, delivery time, personalization, ads. Answer: the order of the
  cards.

Keeping them separate is what lets each scale on its own terms. Serviceability
is a cheap geometric gate. Ranking is an expensive relevance sort. You never
want the expensive sort touching restaurants that are not even eligible.

### The scale story, three tiers

What grows here is not a catalog of items to search. It is the number of
requests hitting a fixed set of zone shapes, and the number of zones as Swiggy
enters more cities. Watch what breaks at each tier.

TIER 1, about 1,000 restaurants, one city, low traffic (a Swiggy of year one).
A single database with a geo-capable extension (PostGIS on Postgres, or a
Redis Geo set) is genuinely fine. On each request you run a bounding-box query
to grab nearby zones, then ray-cast. Latency is a few milliseconds, the box
never sweats. Building an in-memory geohash index here would be
over-engineering. Ship the database query and move on. Nothing breaks.

TIER 2, about 100,000 restaurants across a few dozen cities, real traffic. Now
the naive "query the database on every app open" starts to hurt. The database
becomes the hot path for literally every session open in the country, read
traffic dwarfs everything, and a live geo-query per request adds latency and
load you cannot sustain during dinner peak. What breaks: the database round trip
itself. The fix is exactly Swiggy's: PRECOMPUTE the geohash-to-clusters map and
hold it IN MEMORY, so the live path does zero database queries. Add read
REPLICAS of the service, each carrying its own copy of the in-memory index, so
you scale horizontally by adding stateless replica nodes behind a load balancer.
The precise polygon shapes and their geohash buckets are rebuilt in the
background whenever ops changes a zone, and pushed to the replicas. The request
path is now a hash lookup plus a few ray casts. This is the tier where the whole
architecture earns its keep.

TIER 3, 10 million-plus orders, 700-plus cities (Swiggy's real world: over 3
billion orders delivered across 680-plus Indian cities as of 2024). Two new
walls appear. First, the number of polygons is now large enough that even the
per-city set matters, so you SHARD the index by city or region: a replica
serving Bangalore requests does not need Delhi's polygons in RAM, and geohash
prefixes make this partitioning natural because a geohash cell belongs to exactly
one region. Second, and more subtle, serviceability is now DYNAMIC, not static.
Whether a restaurant can serve Ananya at 9:20pm depends on live conditions: how
many delivery partners are near that kitchen right now, how backed up the
kitchen is, whether rain just spiked demand. So the static geometric gate ("are
you inside the polygon") gets a second, live gate stacked behind it ("given
current supply and load, can we actually promise a good delivery time"). The
static gate stays a cheap in-memory geohash-plus-polygon lookup. The dynamic
gate reads fast-changing signals (partner positions, live ETAs, the same DISCO
and ETA machinery from the Uber teardowns applied to Swiggy's own fleet) and can
temporarily shrink a zone during a crunch. The key engineering discipline: the
expensive, slow-changing structuring (drawing polygons, bucketing them by
geohash) stays offline and cached in RAM, and only the genuinely live signals
are computed per request. You never re-derive the map of the city on the hot
path. You look it up, then adjust it with a thin layer of live truth.

A concrete tier-3 failure and its fix: on a heavy monsoon evening in Bangalore,
delivery partners near Koramangala get scarce. If serviceability stayed purely
static, Swiggy would keep showing Ananya restaurants it can no longer reach in
time, and every one of those becomes a late order and a broken promise. The
dynamic gate catches this: it sees supply drop and quietly trims the eligible
set or widens estimated times, so the promise stays honest even when the city is
against it.

## 8. The retention and habit mechanic

Serviceability is invisible-papercut retention, the same family as Spotify
loudness normalization (2026-08-28), Netflix picture quality (2026-09-11), and
Zepto's 76-second pick (2026-09-09). Nobody ever opens Swiggy and thinks "thank
you, point-in-polygon test." They think "there is always good food near me and
it always actually shows up." That reflex is the retention.

The loop it protects: Ananya opens the app, sees a full screen of food she can
truly get, orders, it arrives hot and on time, so the next hungry night she
opens the app again without shopping around. The habit is "just open Swiggy." The
metric it moves is primarily RETENTION and ORDER FREQUENCY, and underneath that,
CONVERSION, because a serviceable, honest list is one she can actually buy from.

The way to see its value is to imagine it failing. Every time the app shows a
restaurant that turns out not to deliver to her, or an estimate that turns out
badly wrong because it ignored the lake between them, a little trust cracks. Do
it enough and "just open Swiggy" becomes "let me check Swiggy AND Zomato AND
maybe just cook." The whole silent geometry exists so that day never comes. The
real observed shape of this: Swiggy leans heavily on delivery-time promises and
on keeping the home screen full of genuinely orderable options, and the entire
serviceability platform exists so those two promises are almost always true at
once.

There is a second-order habit too, on Swiggy's own side. Because serviceability
is where "can we serve here" is decided, it is also where Swiggy learns where
demand exists that it cannot yet meet. A pin dropped in a not-yet-served suburb
is a data point: people here want us. Those denied requests quietly map the next
neighborhoods worth expanding into, which is how the serviceable area grows to
match real demand instead of guesswork.

## 9. The lesson for Rare.lab

Rare.lab is an AI shader and visual-effects product: a node-based editor that
compiles to shippable code, plus an embeddable runtime. The serviceability
lesson is about the runtime, and it is one specific move: do a cheap spatial cull
BEFORE any expensive per-pixel or per-effect work, using a precomputed spatial
index, exactly like Swiggy tests a geohash bucket before it ever ray-casts a
polygon.

Here is the mapping, one to one.

- Swiggy's problem: thousands of delivery polygons, but only two or three can
  possibly contain this point. Testing all of them is the fire.
- Rare.lab's problem: a scene can have hundreds of effect instances, light
  volumes, decals, and post-process regions, but for any given tile of the
  screen (or any given camera frustum) only a handful actually touch it.
  Running every effect's shader over every pixel is the same fire.

So build the same two-stage gate:

1. THE CHEAP SPATIAL INDEX (the geohash step). Precompute, offline in the
   compiler or at scene load, a spatial partition of the effects: a uniform grid,
   a bounding-volume hierarchy, or screen-space tiles (this is literally how
   modern tiled and clustered renderers cull lights: divide the screen or view
   frustum into cells, and for each cell store the short list of lights or
   effects that overlap it). Each effect gets a cheap bounding volume, its
   "delivery polygon." The index maps "this tile" to "these few effects," just
   as the geohash maps "this cell" to "these few clusters."

2. THE PRECISE TEST ONLY ON SURVIVORS (the ray-casting step). For a given tile,
   look up its short list, then run the real, expensive shader math only for
   those effects. Never run the precise per-pixel evaluation across the whole
   scene. Run it on the two or three effects the index says could possibly touch
   this tile.

Three specific carry-overs from how Swiggy did it:

- PRECOMPUTE THE INDEX, KEEP THE HOT PATH A LOOKUP. Swiggy builds the
  geohash-to-clusters map offline and holds it in RAM so the request is O(1) with
  zero database queries. Rare.lab's compiler should emit the spatial acceleration
  structure as a build artifact, so the runtime never rebuilds it at 60fps. Per
  frame, the runtime does a lookup and a cull, not a re-derivation. A scene that
  is static between frames should reuse last frame's structure entirely.

- USE THE RIGHT CELL SHAPE, AND KNOW WHY. Swiggy chose geohash cells; DoorDash
  chose S2; Uber built hexagonal H3 because equidistant neighbors make
  outward-expansion queries clean. Rare.lab should pick its partition to match
  its dominant query. If the dominant query is "which effects touch this screen
  tile," screen-space tiles win. If it is "which effects are within range of this
  point in world space," a hex or octree partition with even neighbors wins. The
  cell shape is a real engineering choice, not a default.

- ADD A CHEAP DYNAMIC GATE ON TOP OF THE STATIC ONE. Swiggy's tier-3 move was to
  stack a live supply-and-demand gate behind the static polygon gate, trimming
  eligibility when the city is under load, without recomputing the map. Rare.lab
  should do the same with a per-frame COMPUTE BUDGET: after the static cull hands
  you the effects that touch a tile, a live gate sheds the lowest-value ones when
  the frame is running hot, so a heavy scene degrades gracefully instead of
  dropping frames. The static structure decides what COULD draw. The live budget
  decides what actually draws this frame. Same two-layer shape as
  static-polygon-plus-live-supply.

One line to keep: the fastest way to render an effect is to prove, with a cheap
precomputed spatial index, that it does not touch this pixel, and only then spend
real GPU time on the few that do. That is Swiggy testing a geohash bucket before
it ever ray-casts a polygon, moved from the map of a city to the pixels of a
screen.

---

## Sources

Serviceability at Swiggy (primary, Swiggy engineering blog):
- Somsubhra Bairi, "What Serviceability means at Swiggy?", Swiggy Bytes:
  https://bytes.swiggy.com/what-serviceability-means-at-swiggy-c94c1aad352a
- Somsubhra Bairi, "Designing the Serviceability Platform at Swiggy for High
  Scale, Part 1", Swiggy Bytes:
  https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-1-751a631f0379
- Somsubhra Bairi, "Designing the Serviceability Platform at Swiggy for High
  Scale, Part 2", Swiggy Bytes:
  https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-2-ab20365fbc23

Geospatial delivery zones and indexing:
- DoorDash Engineering, "Scaling DoorDash's Geospatial Innovation with a
  Location-Based Delivery Simulator":
  https://doordash.engineering/2020/08/12/scaling-geospatial-innovation-with-a-location-simulator/
- Joud Wawad, "The Complete Guide to Location Indexing: Geohash, Quadtree,
  Google S2, and Uber H3":
  https://joudwawad.medium.com/location-indexing-complete-guide-36a143569555
- Uber H3, "Hexagonal hierarchical geospatial indexing system" (GitHub):
  https://github.com/uber/h3
- H3 documentation (resolution tables): https://h3geo.org/docs/

Point in polygon (the geometry):
- Wikipedia, "Point in polygon" (ray casting, crossing number, winding number):
  https://en.wikipedia.org/wiki/Point_in_polygon
- Wikipedia, "Even-odd rule": https://en.wikipedia.org/wiki/Even%E2%80%93odd_rule

Isochrones vs radius (why polygons, not circles):
- Radar, "What is an isochrone map?": https://radar.com/blog/what-is-an-isochrone-map
- Maptive, "Drive Time Polygons vs a Simple Radius Explained":
  https://www.maptive.com/drive-time-polygons-vs-a-simple-radius-explained/

Scale figures:
- Wikipedia, "Swiggy" (3 billion-plus orders, 680-plus cities as of 2024):
  https://en.wikipedia.org/wiki/Swiggy
