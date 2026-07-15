# Uber ETA prediction: the "arrives in 4 min" number

Date: 2026-07-15
Product: Uber
Feature: Trip and pickup ETA prediction (the routing engine plus the DeeprETA correction model)

---

## 1. The user

It is 5:40 on a Tuesday morning. Priya has a 8:15 flight out of Bengaluru
airport. She lives in Koramangala. The airport is 35 km north, past the whole
city, and she has done this drive enough times to know it can take 55 minutes at
6 AM or two hours if she leaves at 8. She opens Uber, types the airport, and
stares at one number before she does anything else: the little "12 min away" on
the car, and then the trip time once she picks the ride.

She is not reading a map. She is not thinking about roads. She is making one
decision: do I leave now, or do I have ten more minutes for coffee. That whole
decision rests on a single number Uber shows her. If that number is wrong by 15
minutes, she misses her flight.

## 2. The real problem

Here is the honest version of the pain. A rider does not care how Uber computes
the route. She cares that the promise matches reality. "Your driver arrives in 4
minutes" that turns into 11 minutes is not a small annoyance. It is a broken
promise at the exact moment she is deciding whether to trust the app at all.

And it cuts both ways. The driver on the other side is looking at the same
number to decide whether the pickup is worth it. The restaurant on Uber Eats is
timing when to start cooking off the same ETA. If the number is soft, everyone
downstream plans around a lie.

The naive answer is "distance divided by speed." That is useless in a real city.
The 35 km to the airport is not one speed. It is a stop-and-go crawl through
Koramangala, then a fast stretch on the elevated Hebbal flyover, then a toll
road where cars actually move. Distance over average speed would tell Priya "42
minutes" and be wrong by half an hour. The real problem is that travel time is a
sum of hundreds of little segment times, each changing minute to minute, and a
plain formula cannot hold that.

## 3. The feature in one sentence

Uber turns "where am I, where am I going, and when" into a trustworthy
arrival-time number by finding the fastest path on a live road graph and then
correcting that estimate with a machine-learning model trained on what actually
happened on millions of past trips.

## 4. Jobs to be done

What is Priya really hiring the ETA to do?

- "Tell me if I leave now, will I make my flight." A go or no-go decision.
- "Tell me how long I am about to be stuck in this car." Setting her own patience.
- "Do not embarrass me." She told her colleague she would land by 9. The ETA is
  the number she repeated to him.
- For the driver: "Is this pickup close enough to be worth accepting."
- For Uber the marketplace: "Match the right car to the right rider so nobody
  waits." A wrong ETA makes the whole matching engine dispatch the wrong car.

The ETA is not a nice-to-have label. It is the input to four different people's
decisions at once.

## 5. How it works for the user

Priya sees it as effortless. She types "Kempegowda International Airport." Before
she even picks a ride type, small numbers appear on the nearby car icons: "3
min," "7 min." She taps UberGo. Now a bigger number shows: "Trip about 58 min,
arrive 6:48 AM." She books. A driver accepts. The pickup ETA counts down live: 7
min, 6 min, 5 min, and the car crawls toward her on the map. Once she is in the
car, the destination ETA updates as they move, nudging a minute or two as
traffic on the flyover clears or clogs.

Three different ETAs, all feeling like one smooth thing:

1. Pickup ETA: how long until the driver reaches her.
2. Trip ETA: how long from pickup to airport.
3. Live ETA: the number recalculated every few seconds during the ride.

She never sees a "loading" spinner on these. They feel instant. That instant
feel is the entire engineering story.

## 6. The actual flow, step by step

1. Priya types the destination. The app sends her origin (a lat/long in
   Koramangala) and destination (the airport lat/long) to Uber's servers.
2. The routing engine finds the fastest path on the road graph and returns a
   base travel time: a sum of expected times across every road segment on that
   path. Call it 61 minutes.
3. That base ETA, plus context (time of day, is this a pickup or a dropoff, the
   origin and destination areas, live traffic), is handed to a second system,
   the ETA post-processing model.
4. The model does not recompute the route. It predicts the *residual*: "trips
   like this, at this hour, on this corridor, have historically run about 3
   minutes under what the router said." It returns minus 3.
5. Final ETA: 61 minus 3, so 58 minutes. That is the number Priya sees.
6. She books. The same machinery runs for the pickup leg (driver to Priya) and
   keeps rerunning every few seconds during the trip as her GPS moves and
   traffic data refreshes.

The rider experiences one number. Under it are two stacked systems: a graph
search, then a learned correction.

## 7. Under the hood, like the engineer

This is the heart of it. There are two halves, and they are genuinely different
problems, the same way search has a matching half and a ranking half. Here the
two halves are **routing** (find the path and a physics-based time) and
**correction** (learn what physics misses).

### Half one: the routing engine (the graph)

Uber built its own routing engine, internally called Gurafu, launched around
April 2015 to replace its earlier dependence on the open-source OSRM. (Fact,
from Uber engineering write-ups.) Computing an ETA here becomes a classic
computer-science problem: **find the shortest path in a directed, weighted
graph.**

The data structure is a **graph**. Nodes are intersections. Edges are road
segments. The weight on each edge is not distance, it is *time to traverse that
segment right now*. The Hebbal flyover edge might weigh 40 seconds at 6 AM and
6 minutes at 9 AM. Same road, different weight, because the weight is live.

Why a graph and not a table? Because roads connect to roads in a web, not a
list. One-way streets, turn restrictions, and the fact that the airport road
only connects to specific ramps are all naturally edges and missing-edges in a
graph. No other structure captures "you can get from here to there but only this
way" as cleanly.

**How the weights get set.** Uber blends aggregated historical speeds (what this
segment usually does at 6 AM on a Tuesday) with real-time speeds streamed in
from the phones of drivers currently on that road. Heavy traffic on a segment
raises that edge's weight, which makes the shortest-path search naturally route
around it. (Fact.)

**The algorithm.** A plain shortest-path search like Dijkstra explores outward
from the origin, settling the closest node, then the next, until it reaches the
destination. On a city graph that is fine. On a graph spanning a metro area with
millions of edges, run half a million times a second, Dijkstra is far too slow;
it would touch too many nodes per query.

Uber's team tried A* (Dijkstra with a distance heuristic to aim the search at
the destination) and found it good for short routes with live weight updates,
but for scale they settled on **Contraction Hierarchies**. (Fact, from Uber
engineering material.) The idea is a precomputation trick worth understanding:

- Rank every node by importance. A quiet residential intersection is low; a
  highway junction is high.
- "Contract" nodes from least important upward. When you remove a node, add
  **shortcut edges** between its neighbors that preserve the true shortest time
  through it. So a shortcut edge might represent "enter the flyover here, exit
  there, 40 seconds" as a single hop, hiding the ten little segments underneath.
- The result is a layered hierarchy where highways and major corridors sit at
  the top.

At query time, the search only ever goes "upward" into more important roads from
both ends and meets in the middle. It skips the thousands of tiny local streets
entirely. A query that would have touched hundreds of thousands of nodes now
touches a few thousand. That is how a 35 km cross-city route resolves in
milliseconds.

There is one more scale trick: Uber **partitions the map into cells**,
precomputes shortest paths inside each cell, and when a route crosses cells only
stitches together the **boundary nodes** between them. (Fact.) Priya's airport
trip crosses many cells; the engine mostly reasons about the handful of boundary
crossings, not every lane in between.

### Half two: the correction model (DeeprETA)

Here is the key insight Uber leaned into. The router's ETA is physically
reasonable but **systematically wrong in learnable ways**. It does not know that
this particular airport ramp backs up when a flight bank departs, that pickups at
this mall always take two extra minutes because the driver circles for the
rider, that rain on this corridor adds a predictable delay. So Uber trains a
model to predict the **residual**: actual time minus router time. (Fact.)

Why predict the residual instead of the whole ETA? Because the router already
does the hard geometric work well. Asking the model only to correct a mostly-good
number is a far easier learning job than predicting 58 minutes from scratch, and
it fails gracefully: if the model is unsure, a residual near zero just returns
the router's solid estimate.

The model is called DeeprETA, and the published architecture (arXiv:2206.02127,
"DeeprETA: An ETA Post-processing System at Scale") is unusually clear. Uber says
this post-processing model has the **highest queries-per-second of any model at
Uber.** That constraint, not accuracy alone, shaped every design choice.

**Feature encoding: discretize everything.** This is the surprising part. The
model takes continuous features (like the raw origin and destination
coordinates, the hour, the router's own estimate) and **bucketizes** them into
discrete bins before feeding them in. It embeds categorical features (like which
map region, is this a pickup or dropoff) as learned vectors. Uber reports that
discretizing and embedding *all* inputs beat feeding raw continuous numbers.
(Fact.)

- They use **quantile buckets**, not equal-width buckets, and found quantile
  buckets more accurate. The reasoning they give: quantile buckets maximize
  entropy. For a fixed number of buckets, splitting so each bucket holds an equal
  share of the data carries the most information (in bits) about the original
  value. An equal-width bucket over latitude would put half of Bengaluru in one
  bin; a quantile bucket splits dense areas finely and empty areas coarsely.
- For very high-cardinality things like fine-grained locations, they use
  **feature hashing**, and specifically *multiple* hash functions per feature.
  A single hash risks collisions (two unrelated locations landing in the same
  bucket). Hashing the same feature several ways lets the network combine the
  buckets and recover from any single collision. (Fact.) This is the classic
  hash-map collision problem solved by using more than one hash.

Why does discretizing help? Because the relationship between, say, latitude and
delay is bumpy and non-linear (this exact ramp is bad, the road 200m over is
fine). Buckets plus a learned embedding per bucket let the model assign each
little region its own correction, which a single continuous weight cannot.

**The network: a linear transformer.** After embedding, the features become a
short sequence of vectors, and DeeprETANet learns their interactions with
**self-attention**. Self-attention takes a sequence of vectors and produces a
re-weighted sequence, letting "origin region" and "hour of day" and "router ETA"
influence each other (airport corridor at 6 AM behaves unlike the same corridor
at 6 PM).

But standard self-attention costs O(K squared times d) where K is the number of
features and d the embedding size. At the highest QPS in the company, quadratic
is a non-starter. Uber uses a **linear transformer**, which reorders the math to
cost O(K times d squared). Linear attention wins whenever K is bigger than d.
The published network is deliberately tiny: two layers after the embedding, the
attention layer using key/query/value dimension of just 4, followed by a
fully-connected layer of size 2048. (Fact.) Small on purpose, because latency is
the product.

**The loss: asymmetric Huber.** Uber trains with an asymmetric Huber loss that
has two knobs. Delta controls robustness to outliers (it smoothly slides between
squared error and absolute error, so a few crazy GPS traces do not dominate).
Omega controls **asymmetry**: the cost of under-predicting versus
over-predicting. This matters because being told "4 minutes" and waiting 8 feels
worse than being told "8" and waiting 4. Omega lets Uber make the model lean
slightly toward not under-promising. (Fact.) A final **calibration layer**
adjusts bias per request segment, so pickups and dropoffs and Eats deliveries
each get their own correction.

**Serving.** The model runs on Uber's ML platform, Michelangelo, via its online
prediction service. The measured latency is a **median of 3.25 ms and a 95th
percentile of 4 ms.** (Fact, from the paper.) That is the budget the entire
"discretize and embed, don't multiply raw floats" design bought.

### The scale story, three tiers

**1,000 trips (a small town, one evening).** Honestly, you do not need any of
this. A plain Dijkstra on the road graph plus a lookup table of "this corridor
usually runs 10% over" would serve every request with room to spare. One server.
No model. The graph fits in memory and a query touches a few thousand nodes at
most. Nothing is on fire.

**100,000 trips (a busy metro, rush hour).** Now two things break. First, raw
Dijkstra per request starts costing real CPU when thousands of requests land per
second, so the **Contraction Hierarchies precomputation** earns its keep: shift
the heavy work offline into shortcuts, make each live query cheap. Second, the
"usually 10% over" rule of thumb is too blunt; delays are corridor-specific and
time-specific, so you need a real model. But a big model would blow the latency
budget, so this is exactly where **discretize-and-embed plus a linear
transformer** matters: accuracy without a fat matrix multiply on the hot path.

**10 million plus trips a day, ~500,000 ETA requests per second (Uber global).**
(Fact: Uber cites roughly half a million ETA requests per second.) Now every
shortcut compounds:

- The graph is **partitioned into cells** with precomputed internal paths, so a
  cross-city route reasons mostly about boundary nodes, not every lane. Sharding
  the map is what keeps a metro-spanning query bounded.
- Live traffic is a **streaming update** into edge weights, not a recompute; the
  graph is a shared, constantly-refreshed structure many queries read at once.
- The correction model is engineered so its 3.25 ms median holds at the highest
  QPS in the company. The feature encoding (embedding lookups instead of float
  math) is not a modeling nicety; it is the thing that keeps p95 at 4 ms when
  half a million requests hit per second.
- The whole two-stage split is itself a scale decision: the expensive geometric
  reasoning is amortized by the routing engine's precomputation, and the model
  stays cheap by only nudging a residual.

What breaks at each tier is latency and freshness, and every fix is the same
shape seen across this ledger: **move heavy work offline (contraction shortcuts,
precomputed cells), keep the live path a cheap lookup (linear transformer,
embedding lookups), and shard the big structure (map cells) so no single query
sees the whole thing.**

### Clearly-labeled inference

The exact number of buckets per feature, the embedding dimensions per field, and
the precise MAE improvement percentages are not all fully spelled out in the
public paper, so I am not quoting hard figures for those. The overall shape
(residual prediction, quantile bucketing, multiple feature hashing, linear
attention, asymmetric Huber, 3.25 ms median) is directly from Uber's own paper
and engineering posts and is treated as fact above. Anything about *specific
Bengaluru corridors* is my illustration, not Uber data.

## 8. The retention and habit mechanic

The ETA does not have a "share" button or a streak. Its habit mechanic is quieter
and stronger: **earned trust that removes hesitation.**

Every time Priya's "58 min" turns out to be 57, the app spends a little less of
her attention next time. She stops double-checking against Google Maps. She stops
padding an extra 20 minutes "just in case." The ETA becoming reliable is what lets
Uber become the default reflex instead of one option she verifies. That is a
retention loop built on accuracy, not on a notification.

Which metric does it move? Mostly **retention and revenue, through completed
trips.** A soft pickup ETA is a top reason riders cancel before the car arrives,
and cancellations are pure lost revenue plus a wasted driver dispatch. A soft
trip ETA erodes the trust that brings the rider back tomorrow. On the marketplace
side, the ETA is a direct input to matching: the dispatch system uses pickup ETAs
to decide which car to send, so a better ETA means less driver idle time and
shorter waits, which is efficiency that shows up as revenue. Uber's own framing
of ETA as its highest-QPS model is the tell: they treat this number as core
infrastructure, not a display label, because so many downstream decisions ride on
it.

A real observed example of the mechanic: on Uber Eats, the delivery ETA drives
when the restaurant starts cooking and whether the customer orders at all. An ETA
that reads "45 min" when the truth is 25 loses the order to hunger; one that reads
"25" and slips to 45 loses the customer for next time. The same number, tuned by
the same asymmetric loss, is quietly protecting both sides of that.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader code and runs it in an
embeddable runtime. The transferable lesson is the **two-stage split: an exact,
precomputed engine plus a cheap learned or heuristic corrector on the hot path.**

Uber does not run one giant model that outputs "58 minutes." It runs a
deterministic engine that does the heavy structural work offline (contraction
shortcuts, precomputed cells) and a deliberately tiny model that only nudges a
residual in 3.25 ms. Apply that shape to Rare.lab:

- **Do the expensive analysis at compile time, not at frame time.** When the node
  graph compiles, that is your "routing engine": constant-fold, precompute
  lookup tables, bake static branches, resolve which effects can never fire for a
  given material. The runtime should be the cheap lookup, the way Priya's live
  request is one keyed correction, not a fresh route search.
- **If you need runtime adaptivity (auto quality scaling, dynamic LOD, an AI
  effect that adjusts to the scene), make it a residual on top of a solid
  precomputed base, not a from-scratch decision.** A small corrector that says
  "drop this pass by 10%" over a known-good baseline degrades gracefully; a model
  that owns the whole output fails ugly when it is unsure. The base always ships a
  reasonable frame.
- **Copy the encoding discipline: buckets and lookups beat live math on the hot
  path.** DeeprETA replaced float multiplies with embedding lookups to hold p95
  at 4 ms. In a shader runtime the equivalent is precomputed textures, ramps, and
  gradient LUTs instead of transcendental math per pixel. When you are running
  the effect on a mid-range phone GPU at 60 fps, "look it up" is the move that
  keeps you inside the frame budget, exactly as it keeps Uber inside 4 ms.

One concrete action: for any runtime-adaptive feature in the Rare.lab runtime,
require it to be expressed as a bounded correction over a compile-time baseline,
and hard-cap its per-frame cost the way Uber caps ETA latency. The baseline
guarantees a shippable frame; the corrector only ever improves it. Never let the
adaptive layer be on the critical path to producing *a* frame.

---

## Sources

- Uber Engineering, "DeepETA: How Uber Predicts Arrival Times Using Deep Learning": https://www.uber.com/blog/deepeta-how-uber-predicts-arrival-times/
- Hu et al., "DeeprETA: An ETA Post-processing System at Scale" (arXiv:2206.02127): https://arxiv.org/abs/2206.02127
- Uber Engineering, "ETA Phone Home: How Uber Engineers an Efficient Route" (routing engine, Gurafu, Contraction Hierarchies): https://www.uber.com/blog/engineering-routing-engine/
- "How Uber Computes ETA at Half a Million Requests per Second," System Design newsletter: https://newsletter.systemdesign.one/p/uber-eta
- Charles Yuan, "Uber's ETA Prediction System (DeeprETA)," deMISTify / Medium: https://medium.com/demistify/ubers-eta-prediction-system-478026c96b95
- MarkTechPost, "Uber Explores Deep Learning To Develop DeepETA": https://www.marktechpost.com/2022/02/15/uber-explores-deep-learning-to-develop-deepeta-a-low-latency-deep-neural-network-architecture-for-eta-prediction/
