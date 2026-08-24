# Uber Pool / UberX Share: matching two strangers into one car

Date: 2026-08-24
Product: Uber
Feature: Shared rides matching (Uber Pool, now UberX Share): combining multiple riders with different pickups and dropoffs into one vehicle route, cheaply, in real time.

A note on scope. This ledger already has three Uber matching pieces: surge
pricing (2026-06-14), batched dispatch DISCO (2026-07-02), and trip ETA
(2026-07-15). Those all solve the same shape of problem: hand one rider one
driver. Shared rides are a different animal. Now one car carries two or three
strangers going to different places at the same time, and the machine has to
decide who rides with whom without making anyone's trip miserable. That is not
a bipartite match anymore. It is a route-packing problem, and it is NP-hard.
This teardown is about that harder half.

---

## 1. The user

It is 6:10pm on a Friday. Rohan is standing outside his office in Koramangala,
Bangalore, and he needs to get to Kempegowda International Airport for a 9:30pm
flight. It is a long ride, about 35 km, and a solo UberX will cost him around
900 rupees at Friday-evening rates. He is not in a screaming hurry. His flight
is three hours out. He would happily trade ten extra minutes for a cheaper
fare.

Two kilometers away in HSR Layout, at 6:12pm, Aditi is also heading to the
airport for an evening flight. Same highway. Same direction. Neither of them
knows the other exists.

Rohan opens Uber, sees "UberX Share" sitting just below the regular UberX
option, and it quotes him 720 rupees instead of 900. That is a real dinner's
worth of savings for standing in the same spot and tapping a slightly different
button. He taps it.

## 2. The real problem

Here is the honest version, the way you would explain it to a friend.

A car going from Koramangala to the airport is mostly empty. One person, four
empty seats, 35 km of road. That is wasteful for everyone. The driver burns
fuel and time for one fare. The rider pays for the whole car alone. And the
road carries one more vehicle than it needed to.

The pain is on both sides. Rohan wants the trip to cost less. The driver wants
to earn more per hour on the road. The city wants fewer cars for the same
number of people. All three of those wishes point at the same fix: put more
than one paying passenger in the car when their paths happen to line up.

But "when their paths line up" is doing a lot of work in that sentence. If the
match is bad, everybody loses. Send the car 8 km backward to grab a second
rider and Rohan's cheap trip becomes a 25-minute detour, he rates it one star,
and he never taps Share again. The whole feature lives or dies on one question:
can the system find a co-rider whose route overlaps yours so well that adding
them barely costs you anything?

## 3. The feature in one sentence

UberX Share finds another rider whose route overlaps yours enough that one
driver can carry you both with only a small detour, and splits the savings so
you both pay less.

## 4. Jobs to be done

What is Rohan really hiring UberX Share to do?

- "Get me to the airport for less money, and I will accept a bit of extra time
  for it." The core trade is minutes for rupees.
- "Do not embarrass me or scare me. Small detour, reasonable co-rider, clear
  price up front." He wants the savings without a surprise 30-minute zigzag.
- "Tell me the price before I commit, not after." He does not want the fare to
  depend on whether a match is found while he is already in the car.

What is the driver hiring it to do? "Fill my empty seats so I earn more for the
same stretch of road." A driver doing airport runs all evening wants two fares
on one highway trip, not one.

What is Uber hiring it to do? "Move more people with fewer driver-hours."
Shared rides raise the number of paying passengers per vehicle-hour, which is
the single number that decides whether the marketplace is efficient or
bleeding money.

## 5. How it works for the user

Rohan sees "UberX Share" in the product list with a lower price and usually a
small note like "save up to 20%." He taps it and confirms. Uber shows him an
upfront fare of 720 rupees. Crucially, in the current version of the product,
that price is locked in whether or not a second rider is ever found. He is not
gambling on the match.

He gets matched to a driver. He may be asked to walk a short distance to a
better pickup point on the main road, because a corner the car does not have to
turn into saves everyone time. The app shows him the walk on a little map.

The car arrives. Aditi may already be in it, or the car may pick her up a few
minutes into the trip. Rohan sees on his screen that there is one co-rider and
roughly how the trip will go. The driver has turn-by-turn directions that
already fold in both stops in the right order. Rohan gets dropped at the
airport. He never negotiated anything. He just paid less.

## 6. The actual flow, step by step

1. Rohan opens the app. The app already knows his pickup (GPS) and he types the
   airport as the destination.
2. He selects UberX Share. The app sends a fare request to Uber's pricing
   service, which returns an upfront quote of 720 rupees. This quote is
   computed before any co-rider is known, based on his own route plus the
   expected discount for choosing Share.
3. He confirms. His request enters the shared-rides matching pool for the
   Bangalore region, tagged with pickup point, dropoff point, request time, and
   the number of seats he needs (1).
4. The matching engine, running on a few-second batch cycle, looks at Rohan's
   request alongside every other open request and every eligible vehicle in his
   part of the city. It asks two questions in order. First: which other
   requests could legally share a car with Rohan without breaking anyone's time
   limits? Second: of all the legal combinations, which assignment of riders to
   cars is best for the whole city right now?
5. It finds Aditi. Her HSR pickup and her airport dropoff both sit almost
   exactly on Rohan's path. Adding her costs Rohan about four extra minutes.
   The engine assigns both of them to Driver Suresh, who is near Koramangala
   with an empty car.
6. Suresh gets a route with an ordered stop list: pick up Rohan in Koramangala,
   pick up Aditi in HSR, drop Aditi at Terminal 1, drop Rohan at Terminal 1.
   The order is chosen to minimize total driving time, not the order they
   booked.
7. Rohan may get a "walk to this corner" nudge to shave a turn. He walks 80
   meters. The car arrives, he gets in.
8. Mid-trip, if a third rider appears on the same highway with a feasible
   insertion, the engine can add them to Suresh's car while it is already
   moving. This is en-route matching, and it is where shared rides earn their
   keep on long airport corridors.
9. Both riders are dropped. The fares were fixed at booking. The savings were
   real whether or not the match happened, but the match is what makes the unit
   economics work for Uber.

## 7. Under the hood, like the engineer

This is the heart of it. Shared-ride matching is search with the same two
halves this ledger keeps finding: **matching** (which requests can legally
share) and then **ranking** (which of the legal combinations to actually pick).
Both halves are harder here than in normal dispatch, because the thing being
assigned is not "a driver" but "a whole route with an order of stops."

### Why this is not just bipartite matching

Regular dispatch (the DISCO teardown, 2026-07-02) is a bipartite match: riders
on one side, drivers on the other, one edge per pair weighted by ETA, solved to
a min-cost assignment with the Hungarian algorithm or a min-cost flow. Every
rider gets exactly one driver and every driver at most one rider. Clean.

Shared rides break the "one rider per car" rule. Now a single car can be
assigned a *set* of riders, and the cost of that set is not the sum of
individual costs. It depends on the order you visit their four stops, and on
the detour each rider imposes on the others. The moment one resource (the car)
can be assigned a combination of demands (riders) and the combination has its
own cost, you have left bipartite matching and entered **set packing**, which is
NP-hard. You cannot just enumerate every subset of riders for every car at city
scale. So the entire engineering game is: shrink the set of combinations you
even consider down to a tiny, high-quality shortlist, then optimize over that.

### The matching half: the shareability graph

The founding idea is the **shareability network**, introduced by Santi, Resta,
Szell, Sobolevsky, Strogatz and Ratti in a 2014 PNAS paper on New York taxi
data. The trick is disarmingly simple.

Make a graph. Each **node is a trip request**. Draw an **edge between two
requests if, and only if, they can be shared** without either rider's delay
crossing a threshold. That threshold is a real product knob: a common value in
the research and in production is a maximum detour factor around 1.25, meaning a
shared trip may take at most 25% longer than the same rider's solo trip, plus a
cap on extra waiting.

Now Rohan is a node. Aditi is a node. There is an edge between them because
serving both in one car adds only about four minutes to each, comfortably under
the cap. A rider going from Whitefield to Electronic City at the same moment is
also a node, but there is no edge to Rohan, because their roads never touch and
sharing would nearly double someone's trip.

Once you have this graph, the pairing problem becomes a classic:
**maximum-weight matching** on the shareability graph, where each edge's weight
is how much total driving time the pairing saves. Maximum matching on a graph is
polynomial and well understood. The 2014 result was striking: with this method,
a large fraction of New York taxi trips could be shared in pairs with only a few
minutes of delay, cutting the number of trips needed by around 40%.

The key move to notice: the hard, combinatorial part (which pairs are even
possible) got pushed into building the graph, and the graph is built from cheap
pairwise feasibility checks. Concretely, testing whether Rohan and Aditi can
share is a small routing question: compute the best order of their four stops
(there are only a handful of legal orders) and check whether every rider's total
time stays under their cap. That is a constant-size calculation per pair.

### The ranking half: from pairs to the RTV graph and an ILP

Pairwise sharing is the easy case. Real UberX Share cars can hold three, and the
research pushes to capacity ten. For that you need the second landmark:
Alonso-Mora, Samaranayake, Wallar, Frazzoli and Rus, "On-demand high-capacity
ride-sharing via dynamic trip-vehicle assignment," PNAS 2017. This is the paper
whose skeleton every modern pooling system resembles. It builds the assignment
in three layers.

1. **The RV graph (request-vehicle).** Draw an edge between two requests if they
   *could* be shared (the shareability edge from above), and an edge between a
   request and a vehicle if that vehicle could serve that request alone within
   the time limits. This is the cheap pruning layer. It throws away the billions
   of impossible pairings before anything expensive runs.

2. **The RTV graph (request-trip-vehicle).** A "trip" is a set of requests that
   can be served together by one car. Using the RV graph as a guide, enumerate
   feasible trips: pairs first, then triples that are only allowed to form if all
   three of their pairs were already feasible (the cliques-only rule keeps the
   blowup in check), and so on up to the car's seat count. For each candidate
   trip, test whether some vehicle can actually serve it, by checking whether
   there is a legal ordering of its stops that keeps every rider inside their
   window. Add a trip-vehicle edge where it works. Rohan-plus-Aditi becomes one
   trip node; Suresh's car connects to it.

3. **The ILP assignment.** Now solve an integer linear program over the RTV
   graph: pick one trip per vehicle and cover as many requests as possible, so
   that total delay (or total vehicle-distance) is minimized, subject to each
   request being served at most once and each seat used at most once. This is the
   "ranking" half. It chooses, out of all the legal combinations, the single
   assignment that is best for the whole city in this batch.

The headline numbers from that paper give the scale intuition: on New York City
taxi data, **2,000 vehicles of capacity 10 (about 15% of the real taxi fleet),
or 3,000 of capacity 4, served 98% of demand with a mean waiting time of 2.8
minutes and a mean in-car delay of 3.5 minutes.** Same city, a fraction of the
cars, because the empty seats got used.

There is one more idea in that paper that matters in production: the algorithm is
**anytime optimal**. It builds the assignment greedily first so it always has a
valid answer, then spends whatever time is left improving it toward the optimum.
When the batch clock runs out, it returns the best answer it has. That property
is what lets a provably-hard optimization live inside a hard real-time deadline.
You never wait for optimal; you take the best feasible when the buzzer sounds.

### En-route matching: the insertion problem

Airport corridors are where pooling pays off, and the reason is en-route
matching. Suresh already has Rohan in the car, heading up the highway. A new
request, a third rider near the highway going the same way, arrives. The engine
does not rebuild the whole city. It runs a cheap, local **insertion feasibility
check**: given Suresh's current ordered stop list, try inserting the new rider's
pickup and dropoff at each pair of positions, and see if any insertion keeps
everyone (including the riders already aboard) inside their windows.

The data structure here is just an **ordered list of stops with cumulative
arrival times**. For a car with k stops remaining, there are on the order of k^2
places to slot a new pickup and dropoff, and each is checked by re-walking the
cumulative times. With k tiny (three riders means a handful of stops), that is a
few dozen arithmetic checks. Cheap enough to run against many idle-and-active
cars per new request. Insertion is the workhorse of live pooling because it
turns "re-optimize the fleet" into "test a few positions in one list."

### The data structures, named with their jobs

- **Graph (shareability / RV / RTV):** the whole method is graph-shaped because
  the core relation is "can A and B share." Nodes and edges are the natural way
  to hold "who is compatible with whom," and matching/covering on graphs is a
  solved science.
- **Spatial index (H3 hex cells or S2 cells, plus a time bucket):** you never
  test all pairs. You bucket requests by where and when, and only test pairs
  inside the same or neighboring cells in the same few-minute window. This is the
  same H3 machinery from the surge teardown, reused to prune the shareability
  graph from billions of pairs to thousands.
- **Ordered stop list with cumulative times:** the per-vehicle route. Insertion
  and feasibility checks read and rewrite this.
- **Priority/heap and min-cost flow / ILP solver:** the assignment layer. In
  practice production systems relax the exact ILP to min-cost flow or an
  auction, and cap the search with a time budget, exactly as with DISCO.
- **Hash maps everywhere:** request id to request, vehicle id to route, cell id
  to the list of requests in it. Constant-time lookups on the hot path.

### The scale story, three tiers

**1,000 open requests (a quiet suburb, off-peak).** All-pairs shareability is
about 500,000 pairs. That is nothing. A single machine builds the full RV graph,
enumerates trips exhaustively at capacity 3 or 4, and solves the ILP to true
optimum inside the batch window with time to spare. No pruning needed. Rohan
gets the mathematically best co-rider available.

**100,000 open requests (a whole metro at Friday peak).** All-pairs is now 5
billion pairs. You cannot compute that every few seconds. This is where it would
break, and here is what saves it. First, **spatial and temporal pruning**: bucket
by H3 cell and a short time window so you only ever test requests that are
physically near each other and booked close in time, which collapses 5 billion
candidate pairs to a few thousand real ones. Second, the **cliques-only rule** for
building bigger trips, so triples only form from already-feasible pairs and the
trip count does not explode. Third, the **anytime budget**: the ILP gets, say, a
couple of seconds, and returns the best assignment found so far. Rohan may not
get the globally perfect co-rider, but he gets a very good one, computed in time.

**10 million+ requests a day, globally, with lakhs concurrent on a peak
evening.** Two moves. First, **geo-shard by city or region**: Bangalore's pool
and Mumbai's pool never share a car, so they are solved on separate machines in
parallel, embarrassingly so. No global lock, linear scale with cities. Second,
inside a city the **hot corridor is the bottleneck**: at 6-8pm the airport
highway has far more shareable requests than anywhere else, so that cell's
combination count balloons. The defenses are a **cap on trips enumerated per
vehicle**, a **greedy/insertion fallback** when the ILP cannot finish, and
serving all ETAs from a **precomputed cache** (Uber's DeepETA, from the ETA
teardown) so route feasibility checks are memory reads, not fresh routing calls.
The batch window itself (wait a few seconds to gather requests before solving) is
both a quality lever, more requests means a denser shareability graph and better
matches, and a fairness lever, everyone in the batch is considered together
rather than first-come-first-served.

The through-line, again: **offline-think, online-lookup**, plus **anytime
optimization under a batch clock**. The expensive combinatorial search is bounded
by a deadline and fed by precomputed ETAs and a pruned graph, so the live path
stays inside a few seconds no matter how big the catalog of riders gets.

### Fact vs inference

Fact, from primary sources: the shareability-network method and its New York
results (Santi et al. 2014); the RV/RTV-graph plus anytime-optimal ILP and the
2,000-vehicles-serve-98%-of-demand numbers (Alonso-Mora et al. 2017); Uber's
public use of batched bipartite matching, min-cost assignment, and DeepETA in its
marketplace. Inference, clearly labeled: the exact internal solver Uber runs for
UberX Share, the precise detour and wait caps used in Bangalore today, and the
exact enumeration limits are not published. The 1.25 detour factor and the
few-second batch window are the well-grounded "this is how this class of problem
is solved" values that appear across the research and Uber's own descriptions,
not a leaked constant. Where I gave a mechanism I could not cite to Uber
directly, treat it as the standard solution to this problem, matched to what Uber
has said publicly.

## 8. The retention and habit mechanic

Shared rides do not build a habit through a dopamine loop the way a feed does.
The loop is **price**, and it feeds a **liquidity flywheel**.

The hook is money. UberX Share is simply the cheapest way to take an Uber, and
the current product guarantees the discount up front whether or not a co-rider is
found. So the price-sensitive rider learns a reflex: when I am not in a rush, tap
Share, save 20%. That reflex is the retention. Rohan, having saved 180 rupees on
one airport run with no downside, will look for the Share button next time by
default.

The flywheel is the deeper mechanic, and it moves marketplace efficiency and
revenue, not vanity engagement. More riders choosing Share means a **denser
shareability graph**, which means the matching engine finds better overlaps,
which means bigger real savings and more filled seats, which lets Uber price
Share even lower, which pulls in more Share riders. Each new shared rider makes
the next match better. This is the same liquidity flywheel from the DISCO
teardown, but pooling amplifies it, because the value of joining is not just "a
car exists" but "a car going almost exactly my way exists."

The metric it moves is **passengers per vehicle-hour**, the efficiency number
under everything. A real observed example: Uber has reported that in dense
markets a large share of trips on the network at peak are shared, and the entire
reason UberX Share can be priced below solo UberX is that two fares on one
vehicle-hour beats one. When the graph is dense enough, the discount pays for
itself out of the saved driver-time. When it is thin (a quiet Tuesday in a
low-density suburb), fewer matches form, the guaranteed discount costs Uber more
per trip, and that is exactly the market where Share is quietly de-emphasized.
The habit and the economics are the same lever pulled from two ends.

## 9. The lesson for Rare.lab

Rare.lab compiles a node-based shader graph to shippable GPU code and runs an
embeddable runtime. The pooling engine hands you a precise, performance-shaped
pattern to steal: **treat expensive global optimization as a bounded, anytime
search over a pruned candidate set, and make incremental edits a cheap local
insertion instead of a full re-solve.**

Concretely, two moves.

**First, borrow the shareability graph for pass fusion.** When Rare.lab compiles
a shader graph, a central performance question is which nodes can be **fused into
one GPU pass** instead of each writing to an intermediate texture. That is
exactly a shareability question: two nodes "can share a pass" if fusing them
keeps the combined register, varying, and sampler budget under the hardware cap,
just as two riders can share a car if the combined route stays under the detour
cap. Build a fusion-compatibility graph over the nodes, where an edge means "these
can legally live in one kernel," then run a covering/packing pass to choose the
set of fusions that minimizes total GPU passes. Prune the graph first with a
cheap feasibility check (do the budgets even fit?) exactly as the RV graph prunes
before the expensive RTV enumeration. You get fewer render passes and fewer
intermediate textures, which is the single biggest win on mobile and
integrated GPUs.

**Second, borrow insertion feasibility for incremental recompile.** When the user
drags one new node into a graph in the editor, do not recompile the whole thing.
Keep the compiled result as an ordered list of passes with a running resource
budget, the direct analog of the ordered stop list with cumulative times. Test
inserting the new node into existing passes at each feasible position, an
O(local) check against a few passes, and only fall back to a full re-solve if no
insertion fits. This keeps the editor interactive: a single node edit costs a few
budget checks, not a graph-wide recompile, so the canvas never stalls even on a
graph with hundreds of nodes.

And wrap both in the anytime discipline: give the optimizer a millisecond budget
tied to the frame or the keystroke, build a valid (unfused, safe) compilation
first, then improve toward the optimum until the clock runs out and ship the best
you have. Provably-hard optimization, living inside a hard real-time deadline,
because you never wait for optimal. That is the whole trick Uber uses to seat two
strangers in one car in three seconds, and it is exactly the trick a live shader
editor needs to stay responsive while doing real optimization underneath.

---

## Sources

- Santi, Resta, Szell, Sobolevsky, Strogatz, Ratti. "Quantifying the benefits of
  vehicle pooling with shareability networks." PNAS 2014. The origin of the
  shareability-network method and the New York taxi results.
  https://www.pnas.org/doi/10.1073/pnas.1403657111
- Alonso-Mora, Samaranayake, Wallar, Frazzoli, Rus. "On-demand high-capacity
  ride-sharing via dynamic trip-vehicle assignment." PNAS 2017, 114(3):462-467.
  The RV/RTV graph and anytime-optimal ILP. DOI 10.1073/pnas.1611675114.
  https://www.pnas.org/doi/abs/10.1073/pnas.1611675114
- PubMed record for the 2017 paper (abstract and metadata).
  https://pubmed.ncbi.nlm.nih.gov/28049820/
- Unofficial reference implementation of the Alonso-Mora et al. algorithm
  (RV-graph, RTV-graph, ILP assignment, rebalancing) on GitHub.
  https://github.com/MetaZuo/RideSharing
- Uber Engineering. "Reinforcement Learning for Modeling Marketplace Balance."
  On how Uber frames matching as a marketplace-efficiency optimization.
  https://www.uber.com/us/en/blog/reinforcement-learning-for-modeling-marketplace-balance/
- "Increasing Shareability in Ride-Pooling Systems," in Reengineering the Sharing
  Economy (Cambridge University Press). Survey of detour caps and shareability.
  https://www.cambridge.org/core/books/reengineering-the-sharing-economy/increasing-shareability-in-ridepooling-systems/43C6E5E26B6E0A54332A692A0F3DE2F4
- Secondary, for production color on batching windows, Hungarian vs
  Jonker-Volgenant, and greedy fallback under load (clearly a secondary
  explainer, not an Uber primary source):
  https://www.frugaltesting.com/blog/how-uber-prepares-its-ride-matching-app-for-high-demand
