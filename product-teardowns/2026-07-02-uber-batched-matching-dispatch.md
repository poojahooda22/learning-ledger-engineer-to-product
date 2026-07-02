# Uber: Batched matching and dispatch (DISCO)

Date: 2026-07-02
Product: Uber
Feature: Rider-to-driver matching, specifically the batched dispatch engine (DISCO) that waits a few seconds and then solves one global assignment instead of grabbing the nearest car per request.

---

## 1. The user

It is 6:40pm on a Friday in Koramangala, Bengaluru. Aditi just walked out of a
cafe with a friend. It is drizzling. She opens Uber, the pickup pin snaps to the
cafe's corner, she taps "Confirm Auto" and then just waits, phone in hand,
watching the little "Finding your ride" spinner. She is not thinking about
algorithms. She is thinking: how long until a car actually shows up, and please
do not let it be one of those rides where a driver accepts and then cancels.

Two hundred meters away, another rider, Rohan, is doing the exact same thing at
the same second outside a metro station. And circling both of them are two idle
auto drivers, one near the cafe, one near the metro. Four people, one moment,
one street. Who gets whom is the whole game.

## 2. The real problem

Here is the pain, said plainly. When you request a ride, the obvious thing for
the app to do is hand you the closest free driver right now. That feels fair and
fast. But "closest to me right now" is a trap when lots of people are requesting
at once.

Picture the four people above. If the app processes Aditi's request the instant
it lands, it gives her the nearest car, the one by the cafe. A half second later
Rohan requests, and now the only car left is the far one by the metro, which is
actually eight minutes from him. Total waiting across the two riders is high, and
Rohan is annoyed enough that he might cancel and try Rapido instead.

But if the app had looked at both riders and both drivers together, it could have
sent the cafe car to Rohan and the metro car to Aditi and cut everyone's wait.
The problem is that greedy, one-at-a-time, first-come-first-served matching
optimizes each request locally and loses globally. At city scale, on a rainy
Friday, that lost efficiency shows up as longer ETAs, more "no drivers available"
screens, and more cancellations, which is money walking out the door on both
sides of the marketplace.

## 3. The feature in one sentence

Instead of matching each rider to the nearest car the instant they ask, Uber's
dispatch engine collects all the open requests and available drivers in an area
over a short window of a few seconds and then solves one math problem that
assigns the whole group at once to minimize everyone's total wait.

## 4. Jobs to be done

What is the rider really hiring this feature to do?

- "Get me a car fast, and do not make me sit on a spinner." (Low, predictable
  ETA.)
- "Do not match me with a driver who is going to cancel because I am inconvenient
  for him." (Low cancellation, a match that actually sticks.)
- "Make the wait feel fair even when it is busy." (No sense that the person who
  tapped a half second later stole the good car.)

What the driver is hiring it to do, because Uber is a two-sided market:

- "Do not send me on a long deadhead pickup for a short fare." (Short pickup
  distance, better earnings per hour.)
- "Keep me busy." (High utilization, less idle time.)

And what Uber itself is hiring it to do: convert more requests into completed
trips, and keep both drivers and riders coming back so the marketplace stays
liquid.

## 5. How it works for the user

From Aditi's seat it looks like nothing. She taps confirm, sees "Finding your
ride" for a moment, and then a card slides up: a driver, a photo, a number plate,
and "Arriving in 3 min." What she does not see is that in the two or three
seconds she stared at the spinner, the system did not immediately reserve the
first car it found. It waited, on purpose, gathered her and Rohan and the two
drivers into the same batch, and solved for the pairing that was best for the
whole street, not just for whoever tapped first.

The tiny wait is the feature. A batch window of a couple of seconds is invisible
to a human but is enough time to turn a bad greedy guess into a good global
answer. When it works, Aditi never learns that she got the metro car instead of
the cafe car, and never learns that this saved Rohan five minutes.

## 6. The actual flow, step by step

1. Aditi taps "Confirm." Her phone sends a trip request (pickup location,
   product type, payment) to Uber's backend.
2. The request lands in the demand side of the marketplace for her area and is
   marked "waiting." It is not matched yet. It joins a short queue for the next
   batch tick.
3. Meanwhile every nearby driver's phone has been pinging its GPS location to
   Uber's Supply Service every few seconds (historically about every four to five
   seconds). The Supply Service holds a live picture of who is free, who is on a
   trip, and where each is.
4. Every couple of seconds the dispatch engine for that geographic area "closes"
   the current batch: it freezes the set of open requests (Aditi, Rohan, others)
   and the set of eligible drivers nearby.
5. For each rider-driver pair that is even plausible, the system computes a cost,
   mainly the predicted pickup ETA over real roads, plus penalties for things
   like wrong vehicle type or a driver likely to reject.
6. It solves the assignment: pick one driver per rider so the total cost across
   the whole batch is as low as possible.
7. Winners get offered. Aditi's phone shows "Arriving in 3 min." A driver's phone
   buzzes with the request. If a driver rejects or times out, that request rolls
   into the next batch and tries again.
8. The loop repeats every few seconds, forever, per area, across the city.

A real trace of the four people: batch closes, costs are computed as driver by
cafe to Aditi = 2 min, driver by cafe to Rohan = 3 min, driver by metro to Aditi
= 3 min, driver by metro to Rohan = 8 min. The greedy answer (serve Aditi first,
give her the 2 min cafe car) leaves Rohan the 8 min metro car, total 10 minutes.
The batch answer sends the cafe car to Rohan (3) and the metro car to Aditi (3),
total 6 minutes, and nobody waits more than 3. Same cars, same street, five
minutes of human waiting deleted by thinking about them together.

## 7. Under the hood, like the engineer

This is the heart of it. Matching is two halves, the same split that runs through
this whole ledger: first find the candidates (which drivers could serve which
riders), then choose the assignment (who actually gets whom). The candidate half
is a geography problem. The assignment half is a combinatorial optimization
problem. They need completely different tools.

### Half one: candidate generation is a spatial lookup, not a scan

You cannot score Aditi against every driver in Bengaluru. That would be lakhs of
pointless comparisons for a driver eight kilometers away who will never serve
her. So the first job is to shrink "every driver" down to "drivers who could
plausibly reach her soon."

Uber does this with a spatial index over the map. In the classic design of
Uber's real-time market platform (described by Matt Ranney), the Supply and
Demand services shard the world using Google's S2 library, which cuts the earth
into cells and gives each a stable 64-bit id. A driver's live location maps to
the cell that contains it. To find candidates for Aditi, you take her pickup
cell and gather the drivers in that cell plus the ring of neighboring cells,
expanding the ring until you have enough supply. Uber later built and
open-sourced H3, a hexagonal grid, which it leans on heavily for pricing and
marketplace features; hexagons are nice here because all six neighbors sit at an
equal distance, so a "k-ring" expansion grows evenly in every direction with no
diagonal distortion (the same property that made H3 the right tool for the surge
map in an earlier teardown).

The data structure that matters: the Supply Service is essentially a big
in-memory hash map from cell id to the set of drivers in that cell, kept fresh by
a stream of GPS pings. Candidate fetch for Aditi is then "read my cell and its
k-ring from the map," which costs the size of the neighborhood, not the size of
the city. That is the whole point. At 1,000 drivers or 10 million drivers, the
candidate set for one pickup is still a few dozen cars, because geography bounds
it. This is exactly the "cost tracks the query, not the catalog" trick that an
inverted index gives a search engine, just in two dimensions.

One important refinement is forward dispatch (also called en-route or dispatch by
route). The eligible drivers are not only the idle ones. A driver who is 90
seconds from dropping off a passenger 200 meters from Aditi may be a better match
than an idle car a full kilometer away. So the candidate set includes soon-to-be
free drivers, scored by where they will be when they are free, not where they are
now. This quietly expands effective supply without adding a single car to the
road.

### Half two: assignment is a min-cost bipartite matching

Now you have, say, 300 open requests in a region and 260 eligible drivers in the
same region for this batch tick. Build a bipartite graph: riders on one side,
drivers on the other. Draw an edge from a rider to a driver only if that driver
is a plausible candidate for that rider (from half one). Put a weight on each
edge equal to the cost of that pairing, dominated by the predicted pickup ETA,
adjusted by penalties (vehicle mismatch, a driver with a high reject rate for
this kind of trip, fairness terms so the same rider is not starved batch after
batch).

The task is to choose a set of edges so that each rider gets at most one driver,
each driver gets at most one rider, and the total weight is minimized. This is
the classic assignment problem. The textbook exact solver is the Hungarian
algorithm (Kuhn and Munkres), which runs in about O(n^3) for n on a side. It is a
graph algorithm, not a sort and not a nearest-neighbor query, and that is the
whole reason greedy is wrong: greedy walks the requests in arrival order and
grabs the local minimum edge each time, which can paint itself into a corner
(Aditi taking the cafe car and stranding Rohan). The Hungarian method considers
the whole graph and can afford to give Aditi her second-choice car because doing
so frees a much better car for Rohan. Note carefully: this is a real sort-free
optimization happening server-side in a dispatch service, never on the phone. The
phone's only job is to draw "Arriving in 3 min."

To make this concrete with the four people, the bipartite graph is two riders,
two drivers, four edges with weights 2, 3, 3, 8. The Hungarian algorithm returns
the perfect matching of weight 6 (the two 3-minute edges), not the greedy 10.

### Where inference starts

Fact: Uber has publicly described batch matching, geographic sharding of supply
and demand, ETA-driven scoring, and forward dispatch. The bipartite-assignment
framing and the batch window are well documented across Uber and Lyft engineering
and research writing. What is inference (the well-grounded "this is how this class
of problem is solved at this scale" version): the exact solver in production is
almost certainly not a naive O(n^3) Hungarian run over a whole city. At scale you
relax it. Common industrial choices are min-cost max-flow formulations, linear
programming relaxations, or auction and greedy-with-local-swap approximations
that get within a hair of optimal in a fraction of the time, plus caps on batch
size and edge count so the graph stays sparse. Reinforcement learning is also in
the mix (Uber and Lyft have both published on learning driver value functions so
that the edge cost reflects not just this trip but where assigning this driver
leaves the marketplace next). Treat the exact solver internals as "a min-cost
matching, approximated for speed," not as literally Kuhn-Munkres.

### The scale story at three tiers

Tier one, about 1,000 concurrent participants (a small city at 2am, or a new
market). One dispatch process holds the whole city in memory. Batches are tiny,
maybe 20 riders by 20 drivers. Even an exact Hungarian solve is microseconds.
Honestly greedy would mostly be fine here; batching barely helps because there is
no contention. Nothing breaks. The interesting engineering has not started yet.

Tier two, about 100,000 concurrent (a large metro at Friday peak, Bengaluru or
Mumbai in the rain). Now two things break. First, you cannot run one global
assignment over the entire city; an O(n^3) solve over 100k participants is
hopeless, and worse, a car in Whitefield has no business being matched to a rider
in Koramangala, so a global solve is not even meaningful. The fix is geographic
sharding: partition the city into regions (via S2 or H3 cells grouped into
dispatch zones), run an independent batch solve per region every few seconds, and
keep each region's graph small and dense. Second, the Supply Service state (who
is free, where) is now a firehose of GPS pings, so it moves to a streaming
pipeline (ingest pings through a log like Kafka, update per-cell aggregates in
something like Flink) and the location store is sharded by cell so no single box
owns the whole map. The batch window itself becomes a tuning knob: longer window,
better matches but more visible wait; shorter window, snappier but more greedy-
like. Uber tunes it per market and per condition.

Tier three, 10 million plus (global, all cities, or a single-city shock like New
Year's Eve). The architecture is now many regional dispatch services running the
same batch loop in parallel, coordinated only loosely, because the whole design
insight is that dispatch is embarrassingly shardable by geography: a match in
Bengaluru and a match in Delhi never touch. The hard remaining problem is the hot
cell, one tiny area (a stadium letting out, an airport at midnight) with thousands
of requests and too few drivers in one batch. Here you cap the batch size, sub-
shard the hot cell, lean harder on forward dispatch to conjure effective supply
from soon-to-be-free cars, drop the exact solver for an approximation, and let
surge pricing (a separate system, torn down earlier) bleed off demand and pull in
supply. ETAs on the edges are not recomputed from scratch per pair either; they
come from a routing engine and learned models (Uber's DeepETA post-processes a
routing-engine estimate with a transformer) served from fast caches, because
computing millions of true road-network ETAs per second inline would melt the
batch budget. The reported shape of that budget is telling: on the order of tens
of thousands of predictions produced within roughly 100 milliseconds per batch of
requests. Offline thinking (train the ETA and value models, precompute routing
tables), online lookup (read a cached ETA, solve a small sparse graph). Same
spine as the rest of this ledger.

## 8. The retention and habit mechanic

Matching does not have a flashy engagement loop like a Monday playlist or a
vanishing story. Its retention mechanic is quieter and, for a marketplace, more
powerful: liquidity.

The loop is a flywheel. Better matching means shorter waits and fewer
cancellations for riders, so more riders convert and come back. It also means
shorter pickup deadheads and higher utilization for drivers, so drivers earn more
per hour and stay online longer. More online drivers means shorter waits for
riders, which pulls in more riders, which keeps more drivers busy. Each turn of
batched matching tightens the loop by squeezing waste (idle driver minutes,
abandoned requests) out of the system. The metric it moves is not a vanity click;
it is the marketplace core: request-to-completed-trip conversion, cancellation
rate, and driver utilization, which together are Uber's word for liquidity.

The real observed example is the contrast riders feel without naming it. On the
nights matching is working, you open the app, tap, and a car is three minutes
away every time, so opening Uber becomes a reflex you do not think about. On the
nights liquidity collapses (a citywide surge, a supply crunch), you get the "no
cars available" screen, you cancel, you open a competitor, and the habit cracks.
The batched matching engine exists to make the good night the normal night. Trust
built from "it just works, fast, every time" is the stickiest retention there is,
because it is invisible until it fails.

## 9. The lesson for Rare.lab

Batch the burst, then optimize the batch as a whole. That is the transferable
idea, and it is a scalability idea, which is where Rare.lab lives.

In a node-based shader editor, edits arrive in bursts. A user drags a slider and
fires 40 parameter changes in a second; a paste drops in a subgraph that dirties
30 nodes at once. The greedy instinct is to react to each change immediately:
recompile the affected shader the instant a node goes dirty. That is the
"nearest driver per request" mistake. You do redundant work (recompiling five
times for five edits that landed in the same frame), and you make locally fine
choices that are globally wasteful (compiling variants in an order that thrashes
GPU pipeline state).

Do what DISCO does. Open a short batch window (a debounce of a frame or two, tens
of milliseconds, invisible to the user just like Uber's few seconds), collect all
the dirty nodes and pending recompiles, then solve the batch as one problem:
deduplicate redundant compiles, topologically order the recompile so each result
feeds the next with no repeated work, and schedule the GPU passes to minimize
expensive state changes across the whole batch rather than per effect. The window
length is your tuning knob, exactly like Uber's: longer means fewer, better-
ordered compiles but a hair more latency before the preview updates; shorter
means snappier but more thrash. Expose it, tune it per workload.

The same move applies in the embeddable runtime. Do not submit draw calls or
effect updates one at a time as they come; accumulate a frame's worth and solve
the assignment of effects to render passes to minimize context switches. And keep
the expensive thinking offline: precompile shader variants and cache them, so the
live frame is a lookup and a small, well-ordered batch, not a fresh solve. Match
cheap and wide (which nodes are dirty), then optimize narrow on the bounded batch
(the best compile and draw order). Batching a burst and optimizing it globally
beats optimizing each item locally, every time the system is under load, which is
exactly when it matters.

---

## Sources

- Matt Ranney, "Scaling Uber's Real-time Market Platform" (InfoQ talk and write-up) on the Supply and Demand services, DISCO, and Google S2 geographic sharding: https://www.infoq.com/presentations/uber-market-platform/
- Uber Engineering, "H3: Uber's Hexagonal Hierarchical Spatial Index": https://www.uber.com/en/blog/h3/
- Uber Engineering, "Engineering More Reliable Transportation with Machine Learning and AI at Uber": https://www.uber.com/us/en/blog/machine-learning/
- Uber Engineering, "DeepETA: How Uber Predicts Arrival Times Using Deep Learning": https://www.uber.com/en/blog/deepeta-how-uber-predicts-arrival-times/
- Uber Engineering, "Reinforcement Learning for Modeling Marketplace Balance": https://www.uber.com/blog/reinforcement-learning-for-modeling-marketplace-balance/
- "A Better Match for Drivers and Riders: Reinforcement Learning at Lyft" (arXiv 2310.13810), batch matching and driver value functions: https://arxiv.org/pdf/2310.13810
- Cornell INFO 2040 course blog, "Uber and Lyft's Batch-Matching Markets": https://blogs.cornell.edu/info2040/2019/10/17/uber-and-lyfts-batch-matching-markets/
- Background on the assignment problem and the Hungarian (Kuhn-Munkres) algorithm: https://en.wikipedia.org/wiki/Hungarian_algorithm
