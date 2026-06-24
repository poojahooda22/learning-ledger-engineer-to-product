# Swiggy live order tracking: the moving dot and the shrinking timer

Date: 2026-06-24
Product: Swiggy (food delivery, India)
Feature: Live order tracking (the live map with the moving rider and the "arriving in X min" timer)

---

## 1. The user

It is 9:15 pm on a Tuesday in Hyderabad. Priya has finished work, she is hungry,
and she just placed an order for a Paradise chicken biryani and a Coke from a
restaurant 2.6 km from her flat in Gachibowli. The moment she taps "Pay", her job
changes. She is no longer shopping. She is now waiting. And waiting is the worst
part of food delivery.

So she does what almost everyone does: she opens the order screen and stares at it.
A small map. A dot for the restaurant, a dot for her building, and somewhere in
between, a tiny scooter icon that is supposed to be her food. Above the map, one
line: "Arriving in 32 min." She will reopen this screen maybe fifteen times before
the doorbell rings.

This is one of the most-looked-at screens in all of Indian consumer tech. Swiggy
does roughly 3 to 4 million orders a day across food and quick commerce, and the
tracking screen is open for a big chunk of the 30-odd minutes each of those orders
takes. That is tens of millions of people, every single day, watching a dot move.

## 2. The real problem

Hunger makes time feel slow, and not knowing makes it worse. Before live tracking,
ordering food was a black box. You called the restaurant, someone said "20 minutes,
madam", and then you had no idea. Was the food being cooked? Had it left? Was the
delivery guy lost on your lane? Every minute past 20 felt like betrayal.

The pain is not really "I want my food faster." Swiggy cannot teleport biryani. The
pain is "I do not know what is happening and I cannot plan my next ten minutes."
Should Priya jump in the shower now or wait? Should she go down to the gate? Is it
worth calling the rider? The honest job of the tracking screen is to kill that
uncertainty. Show her the truth, keep it honest, and the wait stops feeling like a
wait.

There is a second, quieter problem underneath. The promise on the screen, "Arriving
in 32 min", is a brand promise made millions of times a day. If it says 32 and the
food shows up at 55, that is not a small miss. That is a broken promise, and it is
the single biggest driver of "where is my order" complaints and refunds. So the dot
on the map and the number above it are not decoration. They are Swiggy's credibility,
rendered live.

## 3. The feature in one sentence

Live order tracking shows the customer a real-time map of their assigned rider moving
along the road, plus a constantly updated arrival time, by streaming the rider's GPS
to a server, snapping it to actual roads, and re-predicting the ETA every few seconds.

## 4. Jobs to be done

What is Priya really hiring this screen to do?

- "Tell me the truth about where my food is, right now." Not a guess from when I
  ordered. The truth as of this second.
- "Let me plan the next ten minutes of my life." If it is 15 minutes away I will
  finish my call. If it is 3 minutes away I will go to the gate.
- "Let me stop refreshing my own anxiety." A number that updates on its own, that I
  trust, so I can put the phone down.
- "Give me a way to act if something is wrong." A call button to the rider, a chat,
  the rider's name, so a wrong turn on my street is fixable.
- "Reassure me that a real human has my food." The moving scooter is proof. Someone
  is on the way. It is not lost in a kitchen.

## 5. How it works for the user

After payment, Priya lands on the order tracking screen. At first it shows the
restaurant preparing the food, with a timer. Once a delivery executive (Swiggy calls
them DEs) is assigned, a scooter icon appears on the map and starts moving. The line
above changes from a prep estimate to a live "Arriving in X min" that ticks down,
sometimes ticks up, as conditions change.

She can see the rider's first name, a call button, and a chat button. As the rider
gets close, the app nudges her: "Tarun is arriving." The map smoothly animates the
scooter along the road, not in straight teleporting jumps. When the rider reaches her
gate, the screen flips to a delivery confirmation and an OTP or a tap-to-confirm.

The whole thing feels alive and continuous. That smoothness is engineered, and it
hides a surprising amount of machinery.

## 6. The actual flow, step by step

1. Priya taps "Pay". The order is created. The clock starts on what Swiggy internally
   treats as four separate legs of a journey, not one blob of time.
2. The system shows a pre-assignment estimate. No rider yet, so this number comes from
   a model that predicts cooking time and likely travel time using only what is known
   so far (restaurant, items, area, time of day).
3. A matching system assigns a nearby free DE, say Tarun, who is 600 m from the
   restaurant. The instant Tarun is assigned, the estimate gets sharper, because now
   there is a real person with a real location and real history.
4. Tarun's phone starts streaming his GPS location to Swiggy roughly every 3 to 5
   seconds while he is moving. This is the firehose that powers the dot.
5. Tarun rides to the restaurant (first mile), waits for the biryani to be packed
   (wait time), then rides to Priya (last mile). His position flows to the server the
   whole time.
6. On the server, each raw GPS point is cleaned and snapped to the actual road network,
   so the scooter rides on the road instead of cutting through buildings.
7. Priya's app holds an open connection to the server. New rider positions and new ETAs
   are pushed down to her phone. The app animates the scooter smoothly between the
   points it receives.
8. The ETA is recomputed on a fixed cadence using live signals: where Tarun actually
   is, how fast traffic is moving on his roads right now, whether the restaurant is
   running slow. The number above the map updates.
9. Tarun reaches the gate. Priya shares the OTP or the rider taps "Delivered". The
   tracking screen closes the loop. Order complete.

## 7. Under the hood, like the engineer

This feature is really three hard engineering problems bolted together: moving a
location firehose without melting the database, drawing a believable rider on a map,
and predicting a time that keeps its promise. Take them one at a time.

### Problem A: the location firehose (streaming, not storing)

Start with the obvious-but-wrong design. Every DE phone sends its GPS every few
seconds. Naive approach: write each ping as a row in a database, and have each
customer app poll "where is my rider?" every few seconds by reading that row.

Do the arithmetic at scale. Say 300,000 riders are active and online at peak, each
sending a ping every 4 seconds. That is 75,000 writes per second, every second, all
day, of data that is stale 4 seconds later and worthless tomorrow. Meanwhile millions
of customer apps poll for reads. You have built a system whose hottest table is full
of garbage you will never query again. The database becomes the bottleneck and it is
spending all its IO on writes nobody keeps.

The key insight, repeated across Swiggy, Zomato, Uber, and Blinkit writeups, is that
**a rider's live location is an event stream, not a database row.** You do not store
it. You move it. Once you frame it as streaming, the architecture gets simpler and
scales out.

The shape that emerges (described in multiple engineering writeups of this exact
class of system):

```
DE phone --GPS every 3-5s--> Ingestion API --> Kafka (location events topic)
   --> Location Processor service --> in-memory / Redis state ("latest position per order")
   --> WebSocket gateway --push--> Customer app (live map)
```

Why each piece:

- **Kafka as the shock absorber and fan-out.** Location pings are a classic high-volume
  event stream: append-only, time-ordered, read by several consumers (the live-tracking
  path, the ETA models, the assignment system, analytics). Kafka is built for exactly
  this. Producers (rider phones, via the ingestion layer) never wait on consumers. Swiggy
  is a public, large-scale Confluent/Kafka customer and runs location and order events
  through it. Partition the topic by something like rider id or city so order is preserved
  per rider and the load spreads across the cluster. At 75k pings/sec this is a Tuesday for
  Kafka; you scale by adding partitions and brokers.

- **A processor that keeps only the latest.** A consumer reads the stream and maintains,
  in memory or in Redis, just the current position and current ETA per active order. Old
  pings are not kept on the hot path. For an in-flight order the only interesting fact is
  "where is the rider right now", which is a single key. This is why a customer query is
  fast: it is a keyed memory lookup, not a scan.

- **Redis for the geo-questions.** "Which free riders are near this restaurant" is a
  spatial query. Redis has native geo support: GEOADD / GEORADIUS / GEOSEARCH store points
  in a sorted set keyed by a 52-bit geohash of latitude and longitude. Geohash is the trick
  that turns 2D location into 1D sortable integers: nearby points share a prefix, so a range
  scan over the sorted set returns a neighborhood. Swiggy's serviceability and assignment
  layers lean on geohash-bucketed, in-memory indexes for exactly this. Finding the nearest
  free rider becomes a bounded in-memory lookup instead of a full table scan over hundreds
  of thousands of riders.

- **WebSockets to push, not poll.** The customer app holds one long-lived WebSocket (or a
  similar push channel) to a gateway. When the rider moves, the server pushes the new point
  down. No polling. One open socket per active order beats millions of apps hammering an
  endpoint every 4 seconds. Inactive orders hold no socket.

The headline lesson: **the rider's GPS never needs to touch your primary database on the
live path.** It flows through a queue and memory and a socket, and only summarized, durable
facts (the route taken, delivery timestamps) get persisted later for analytics and pay.

### Problem B: drawing a believable rider (map matching)

Raw GPS is a liar. On a phone, in a city, with tall buildings and cheap chipsets, the points
jump around. A rider standing still at a red light can show GPS scatter of 20 to 40 meters.
Plot the raw points and the scooter teleports into a building, hops across a divided road,
or appears to drive through a park.

So the server does **map matching**: it snaps the noisy GPS trail onto the actual road
network. Swiggy uses GraphHopper's map-matching library on top of OpenStreetMap (OSM) road
data, and because Indian deliveries are overwhelmingly two-wheelers, they use the motorcycle
(two-wheeler) routing profile, which can use small roads and gullies a car cannot.

How map matching works, concretely. The road network is a **graph**: intersections are nodes,
road segments are edges, edge weights are travel cost. A noisy GPS trace is a sequence of
points that each could belong to several nearby road segments. The standard solution is a
**Hidden Markov Model**: the true road you are on is the hidden state, the GPS reading is the
noisy observation, and you find the most likely sequence of road segments using the Viterbi
algorithm. The probability of a match balances two things: how close the GPS point is to that
segment (emission), and how plausible the move from the previous segment to this one is given
the road graph and a shortest path between them (transition). The output is a clean trajectory
that rides on real roads. That is why Priya's scooter follows the road and turns at corners
instead of cutting diagonals.

Between the points the server actually receives, the **app interpolates**. It animates the
scooter smoothly from the last known snapped point to the new one along the road geometry,
at roughly the rider's speed, so the human eye sees continuous motion at, say, 60 fps even
though real updates arrive only every few seconds. This is the same split that Figma uses for
cursors: send sparse truth over the wire, interpolate to smooth fiction on the client. The
truth is cheap and occasional; the smoothness is local and free.

For distance and route, GraphHopper precomputes **contraction hierarchies** over the OSM graph.
Plain Dijkstra over a city graph with millions of edges is too slow to run per request at
Swiggy volume. Contraction hierarchies add shortcut edges offline so that a shortest-path query
at runtime explores a tiny fraction of the graph and returns in well under a millisecond. Same
pattern as everything else here: do the expensive whole-graph work offline, make the live query
a cheap lookup.

### Problem C: the honest timer (ETA as four models, not one)

Here is the part most people underestimate. "Arriving in 32 min" looks like one number. Swiggy
builds it as **four separate predictions for four separate legs of the order**, then sums and
refreshes them. Their data science team documented this in the "When is my order coming" work.
The four legs:

- **O2A (Order to Assignment):** time from the customer paying to a rider being assigned. Driven
  by how many free riders are around right now.
- **FM (First Mile):** time for the assigned rider to travel from wherever they are to the
  restaurant.
- **WT (Wait Time):** time the rider waits at the restaurant for the food to be cooked and packed.
- **LM (Last Mile):** time to ride from the restaurant to the customer's door.

Each leg is its own machine learning model because each is driven by different signals, and
mixing them into one regression would blur the things that actually move the number. The
documented inputs include:

- **Restaurant type.** A cloud kitchen that does only delivery is faster than a sit-down
  restaurant juggling dine-in customers. For Priya's Paradise outlet, the model has learned its
  typical biryani prep curve.
- **Order composition.** Number and variety of items. One biryani and a Coke is fast; a party
  order of ten dishes is slow and the kitchen serializes it.
- **Restaurant "stress" right now.** A near-real-time ratio of orders placed to orders prepared
  in the recent window. If the kitchen is buried at 9 pm dinner rush, wait time balloons, and the
  model sees it from the backlog signal rather than guessing.
- **Rider availability in the hyperlocal zone.** Few free riders means a longer O2A and a longer
  first mile.
- **First-mile and last-mile speed patterns** around this restaurant and this delivery area,
  learned historically (some lanes are always slow), and **near-real-time speed** derived from the
  live GPS pings of all the riders currently moving through those roads. Those same location events
  from Problem A are the traffic sensor. Swiggy does not need a third-party traffic feed for the
  roads its own riders are on; the rider fleet is the probe network.

Crucially, **the ETA is recomputed at fixed intervals, not frozen at order time.** As Tarun
actually moves, his real position replaces the predicted first mile. As the kitchen actually
finishes, the wait-time estimate collapses. This is why the number on Priya's screen sometimes
goes up: the model is being honest. A traffic jam appeared on Tarun's road, measured by the
slowdown of other Swiggy riders on it, so the truth changed, so the promise updates.

Why split into four models instead of one big one? Because the legs have different error
sources and different fixes. If deliveries in an area are consistently late, you can look at
which leg is wrong. Is it O2A (a rider supply problem, fixed by incentives) or WT (a slow
kitchen, fixed by ops) or LM (a routing or traffic problem)? One blended number hides the
diagnosis. Four numbers expose it. This is the same idea as Amazon splitting search into
matching and ranking, or YouTube splitting into candidate generation and ranking: decompose
the problem so each half can be measured and fixed on its own.

### The scale story at three tiers

**1,000 active orders (a single neighborhood, early Swiggy).** Almost anything works. You could
poll a Postgres table for rider positions and recompute a simple ETA on each read. A few thousand
pings a minute. No queue needed, no map matching strictly required. The naive design survives here,
which is exactly why early-stage teams build it and then hit a wall.

**100,000 active orders (a metro at dinner peak).** The naive design is now on fire. Polling a
database for live positions means millions of reads against rows that change every few seconds.
This is the tier where you must flip to streaming: pings go into Kafka, a processor keeps only the
latest position in Redis, and customers get pushed updates over WebSockets instead of polling. ETA
moves to precomputed-and-refreshed instead of computed-from-scratch on every read. Map matching
becomes a real service because raw GPS at this volume produces visibly broken maps and angry
"my rider is going the wrong way" complaints. Spatial queries move to geohash buckets in memory
because scanning a rider table to find nearby free riders is too slow.

**1 million plus events per second of location pings (national peak, the real Swiggy).** Now the
constraints are about horizontal scale and isolation. Kafka is partitioned by city or rider so no
single broker is hot. The WebSocket gateways are a fleet, sharded so a city's customers connect to
a city's gateways. Redis is sharded by geography. The ETA models run offline training on enormous
historical data, and only cheap inference runs live. Map matching and routing lean entirely on
offline-precomputed contraction hierarchies so each route query is sub-millisecond. The thing that
breaks at this tier if you are not careful is the **fan-out and the hot partition**: a viral moment,
a cricket final, a rainy Friday in Bengaluru, and one city's traffic spikes. The fix is the same
toolkit every report in this ledger keeps returning to: partition by geography so load spreads,
keep the hot path in memory, push all heavy computation offline, and never let the live request do
work that can be precomputed.

### Fact vs inference, labeled honestly

- **Fact (documented by Swiggy):** the four-leg ETA decomposition (O2A, FM, WT, LM), each as its
  own model refreshed at intervals; the use of restaurant stress, item composition, rider
  availability, and live GPS-derived speed as features; use of GraphHopper map matching on OSM with
  a two-wheeler routing profile; geohash-based in-memory indexing in the serviceability and
  assignment layers; Swiggy as a large-scale Kafka/Confluent user for event streaming.
- **Well-grounded inference (this is how this class of problem is solved, internals not all public):**
  the exact Kafka -> processor -> Redis -> WebSocket topology and the "latest position only" hot path;
  GraphHopper's HMM + Viterbi map matching and contraction-hierarchy routing (these are how
  GraphHopper itself works, publicly); the precise ping cadence of 3 to 5 seconds and the client-side
  interpolation for smooth animation; the exact peak pings-per-second figures, which I estimated from
  public order and rider counts. I have labeled these as inference rather than claiming them as
  Swiggy's confirmed internal design.

## 8. The retention and habit mechanic

The tracking screen is one of the strongest engagement surfaces Swiggy owns, and it works on a
simple loop: **uncertainty creates a compulsion to check, and a trustworthy, moving answer rewards
the check.** Priya reopens the app fifteen times per order not because Swiggy nagged her, but because
the screen reduces her anxiety every time she looks. That is a variable-reward loop in the cleanest
possible form. The reward (the dot is closer, the number is smaller) is real and earned, not a
manufactured notification.

Which metric does it move? Primarily **retention and trust**, with a direct line to **cost and
revenue**. The honest, self-correcting ETA is the thing that keeps the brand promise. When the
predicted time matches reality, "where is my order" support tickets drop, refund and re-delivery
costs drop, and customers keep ordering because the experience felt in control. Swiggy's own framing
is that every minute of the estimate is a brand promise made millions of times a day. A tracking
screen that tells the truth is cheaper to run (fewer support contacts, fewer refunds) and better at
keeping users than any loyalty gimmick.

There is a real, observable example of the loop in action: the screen actively earns the reopen by
changing state. It moves from "restaurant is preparing your order" to a moving scooter to "Tarun is
arriving", and it fires a nudge as the rider gets close. Each state change is a reason to look again,
and each look pays off with genuine progress. Compare that to the festival animations and home-screen
category nudges Swiggy and Zomato rotate to make the app feel alive: those manufacture novelty. The
tracking screen does not have to manufacture anything. The moving dot is intrinsically rewarding
because it answers the one question the hungry user actually has.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus an
embeddable runtime. The Swiggy tracking screen hands you one sharp, transferable principle:
**separate the sparse stream of truth from the smooth fiction the user sees, and never make the live
path do work you can precompute.**

Concretely, three moves:

1. **Stream sparse, interpolate dense.** Swiggy sends a rider position every 3 to 5 seconds and the
   client animates the scooter at 60 fps in between. For Rare.lab's embeddable runtime, do the same
   with expensive inputs. If a shader parameter is driven by something costly (an AI-generated value,
   a physics tick, a network event, an audio analysis), do not recompute it every frame. Update the
   keyframe of truth on a slow cadence and let the runtime interpolate it smoothly per frame on the
   GPU. The eye wants 60 fps; the source of truth almost never needs to. Decouple them and you cut
   the live cost by an order of magnitude while looking continuous.

2. **Precompute the hierarchy offline, keep the live query a lookup.** GraphHopper turns an
   impossible-per-request shortest path into a sub-millisecond lookup by building contraction
   hierarchies offline. In Rare.lab's compiler, push everything that does not depend on live input
   to compile time: constant-fold the node graph, bake static lighting and lookup tables, pre-sort
   and pre-link the execution order, hoist anything invariant out of the per-frame loop. The runtime's
   per-frame job should be a cheap traversal of a precompiled structure, not a re-evaluation of the
   whole node graph. That is what lets the same effect run on a cheap phone GPU, which is where your
   embed will actually live.

3. **Decompose the one number into the legs that can each be measured and fixed.** Swiggy did not
   ship one ETA model; they shipped four, one per leg, so they could see which leg was wrong. When
   Rare.lab reports a frame budget (say 16 ms for 60 fps), do not report one blob. Break it into legs:
   parse, compile, upload, per-frame compute, draw. When an embed janks on a customer's site, the
   per-leg breakdown tells you whether it is a compile-time cost, an upload stall, or a hot per-frame
   shader, which is the difference between a five-minute fix and a week of guessing. A single blurred
   metric hides the diagnosis; the decomposed one exposes it.

The throughline: the live, user-facing path should carry only the truth that genuinely changed this
instant, rendered smooth on top of a structure you built ahead of time. Swiggy moves a dot that way.
Rare.lab should move pixels the same way.

---

## Sources

- Swiggy Bytes, "How ML powers When is my Order coming, Part I" (the four-leg O2A/FM/WT/LM ETA
  decomposition and its features): https://bytes.swiggy.com/how-ml-powers-when-is-my-order-coming-part-i-4ef24eae70da
- Swiggy Bytes, "The OSM Distance Service Part 1: Evaluation Metrics and Routing Configurations"
  (GraphHopper map matching on OSM, two-wheeler routing profile):
  https://bytes.swiggy.com/the-osm-distance-service-part-1-evaluation-metrics-and-routing-configurations-6e8686ca814f
- Swiggy Bytes, "Designing the Serviceability Platform at Swiggy for High Scale, Part 1"
  (geohash-bucketed in-memory indexing): https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-1-751a631f0379
- Swiggy Bytes, "Architecture and Design Principles Behind the Swiggy Delivery Partners app"
  (background location, event-driven delivery flow, finite state machine):
  https://bytes.swiggy.com/architecture-and-design-principles-behind-the-swiggys-delivery-partners-app-4db1d87a048a
- Confluent customer case study, Swiggy (Kafka for location and order event streaming at scale):
  https://www.confluent.io/customers/swiggy/
- "Batching and Matching for Food Delivery in Dynamic Road Networks" (arXiv 2008.12905), road-network
  graph model for batching and assigning food-delivery orders: https://arxiv.org/abs/2008.12905
- DEV Community, "How Platforms Like Zomato, Swiggy, Uber, and Ola Update Rider's Location in Real
  Time" (the streaming-not-storage architecture): https://dev.to/rachit_avasthi/how-platforms-like-zomato-swiggy-uber-and-ola-update-riders-location-in-real-time-3ic5
- GraphHopper documentation, map matching (HMM + Viterbi) and contraction hierarchies routing:
  https://www.graphhopper.com/
- Swiggy Annual Report FY 2024-25 (scale: ~923M orders in the year, 690k+ delivery partners, 260k+
  restaurants, 1,100+ dark stores, 653 cities): https://www.swiggy.com/corporate/wp-content/uploads/2025/07/Swiggy-Annual-Report-FY-2024-25.pdf
