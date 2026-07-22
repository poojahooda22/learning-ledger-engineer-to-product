# Zomato order batching and rider assignment: the moment your order gets a delivery partner

Date: 2026-07-22
Product: Zomato
Feature: The dispatch engine that assigns your order to a delivery partner and decides whether to club it with someone else's order (order batching)

## 1. The user

It is 1:15pm on a Wednesday in Indiranagar, Bengaluru. Arjun is between two
meetings. He opened Zomato ten minutes ago, ordered a chicken bowl and a lemonade
from a cloud kitchen 1.3 km away, and paid. The screen now says "Finding a delivery
partner." He locks his phone and goes back to work.

Arjun does not think about what happens in those next few seconds. To him the app
just goes from "Finding a delivery partner" to "Rehman is picking up your order,"
and a little bike icon starts crawling across a map. That silent jump is the single
most expensive decision in the whole food delivery business. Who should carry
Arjun's bowl, and should that same person also carry the biryani that a flat two
buildings away just ordered?

Arjun is not the only user here. There are three people whose day depends on this
one decision. Arjun, who wants hot food fast. Rehman the delivery partner, who wants
to earn as much as he can per hour of riding. And the restaurant, which wants the
food picked up the moment it is bagged so it does not sit under a heat lamp going
soggy. The dispatch engine has to make all three roughly happy at the same time.

## 2. The real problem

Here is the honest version. At 1:15pm in a big city, thousands of orders are being
placed every minute, and thousands of delivery partners are scattered across the
map. Some partners are idle. Some are mid delivery. Some are 30 seconds from freeing
up. The food for each order is not ready yet, and it will be ready at different times
that nobody knows exactly.

The naive answer is "give each order to the nearest free rider." That answer is a
trap, and it is worth seeing why like a friend would explain it.

Say Arjun's bowl is ready in 8 minutes. The nearest free rider, Suresh, is 200 m
from the restaurant right now. If you assign Suresh instantly, Suresh rides over,
parks, and stands around for 7 minutes waiting for a bowl that is not bagged. During
those 7 wasted minutes, three new orders came in near Suresh that he could have
handled. You just burned a rider. Multiply that by a city and you need far more
riders than the work actually requires, every single one of them paid, and delivery
either gets slow or gets expensive.

Now the second trap. Arjun's bowl and a biryani from the same food court are both
going to the same apartment cluster 1.3 km away. If two different riders carry them,
that is two bike trips for one direction of travel. If one rider carries both, that
is one trip, half the cost, and the rider earns for two orders on nearly the same
distance. But if you club them badly, say the biryani is ready 15 minutes after the
bowl, then Arjun's food waits 15 minutes getting cold while the rider hangs around
for the second bag. Bad clubbing makes everyone lose.

So the real problem has three hard parts stacked together:

1. Match orders to partners so that as few riders as possible do as much work as
   possible, without anyone's food going cold.
2. Decide when to club two or more orders onto one partner (batching), and only when
   the extra detour is small.
3. Do both in the few seconds a human is willing to stare at "Finding a delivery
   partner," while the map keeps changing under you.

This is not a search problem. It is an assignment problem, and it is one of the
genuinely hard ones in computer science.

## 3. The feature in one sentence

Zomato's dispatch engine takes the live stream of new orders and the live positions
of every delivery partner, and every few seconds it solves a matching problem that
decides who carries what, including whether to club several orders onto one partner.

## 4. Jobs to be done

What is Arjun really hiring this feature to do? Not "assign a rider." He is hiring
it to make his food show up hot, roughly when the app promised, without him having
to think. The 40 minute promise on the checkout screen is a contract, and dispatch
is the thing that keeps it.

What is Rehman the delivery partner hiring it to do? To keep him earning. A good
dispatch engine gives Rehman back to back orders with short gaps and clubs nearby
drops so his rupees per kilometer go up. A bad one leaves him idle or sends him
zig-zagging across town.

What is the restaurant hiring it to do? To get food out of the kitchen the instant
it is bagged, so counter space clears and nothing sits going cold.

What is Zomato itself hiring it to do? To move the most orders with the fewest paid
riders. Delivery cost is the biggest controllable line in the whole business.
Batching is the single biggest lever on it.

## 5. How it works for the user

From Arjun's side the whole thing is almost invisible, which is the point.

He pays. The screen says "Finding a delivery partner" for a few seconds. Then it
says a partner's name and shows a bike icon. If his order got clubbed with another,
he usually cannot tell, except that the bike icon might go to a different building
first before coming to his, and the ETA quietly accounts for it. He sees the partner
reach the restaurant, wait if the food is not ready, pick up, and ride to him. The
ETA number updates as this happens.

The only visible tells that batching happened at all: the route on the map has a
small dogleg, or the live ETA nudges up by a few minutes right after a partner is
assigned. Most users never notice. That invisibility is deliberate. The system is
allowed to trade a couple of Arjun's minutes for a much cheaper trip, as long as it
stays inside the promised window.

## 6. The actual flow, step by step

1. Arjun taps "Place order" and pays. His order becomes a live object in Zomato's
   system: pickup location (the restaurant), drop location (his flat), items, and a
   predicted food ready time from the food preparation time model.
2. The order does not get a rider the same millisecond. It joins a pool of unassigned
   orders for that area. The engine works on the whole pool at once, not one order at
   a time. This is the key move, and section 7 explains why.
3. Every few seconds a dispatch cycle runs. It looks at every unassigned order and
   every candidate partner nearby, scores every sensible order-to-partner pairing,
   and also considers clubbing pairs of orders.
4. It solves for the best overall set of assignments for that cycle, not the best for
   any single order. Some orders may be deliberately held back to the next cycle if
   holding them leads to a better club a few seconds later. This is called strategic
   delay.
5. The winning assignment is sent as an offer to the chosen partner, say Rehman. On
   Zomato this is typically a direct assignment rather than an open free-for-all.
6. Rehman's app lights up with the order (or the clubbed pair). He rides to the
   restaurant. If the food ready time was well predicted, he arrives right as it is
   bagged and waits close to zero minutes.
7. Arjun's screen flips to "Rehman is on the way," and the live tracking that was
   torn down in an earlier report takes over from here.

The entire interesting part is steps 2, 3, and 4. Everything Arjun sees is downstream
of a matching problem being solved on a server, in a loop, for a whole city.

## 7. Under the hood, like the engineer

This is the heart of it. I will build up the engine the way it actually evolved, from
the simple thing that breaks to the hard thing that works, because that is the
clearest way to see why each data structure is there. Where a detail is confirmed by
public engineering I will say so. Where I am describing how this class of problem is
solved in general, I will label it clearly as inference.

### The data, first

Two live sets change every second.

Orders. Each is a small record: order id, restaurant location as a latitude and
longitude, drop location, predicted food ready time, promised delivery deadline. At
a city's lunch peak there are thousands of these live at once. Think of a real one:
`order 8837121, pickup 12.9719,77.6412 (the cloud kitchen), drop 12.9782,77.6408
(Arjun's flat), ready in 8 min, deadline 1:55pm`.

Partners. Each is: partner id, current location (streaming in from their phone GPS
every few seconds), current state (idle, riding to pickup, at restaurant, riding to
drop), and the list of orders already on them. Rehman right now is
`partner 4471, at 12.9705,77.6431, idle, 0 orders`.

Both sets are indexed by geography so you can ask "which partners are near this
restaurant" without scanning the whole city. The standard tool is a geospatial grid
or geohash, the same idea used in the Uber surge teardown: chop the map into cells,
bucket partners into cells by a hash of their cell id, and to find partners near a
restaurant you look up the restaurant's cell and its neighbors. That turns "search
all 40,000 partners in Bengaluru" into "look in 9 cells and read a few dozen
partners." The lookup is a hash map keyed by cell, value a list of partner ids.

### Attempt one: greedy nearest free rider (breaks fast)

The first thing anyone builds: when an order comes in, find the nearest idle partner
and assign. Data structure is the geo grid plus a distance check. Simple, instant.

Why it breaks, concretely: it assigns Suresh to Arjun's bowl the moment the order
lands, and Suresh stands at the counter for 7 minutes because the bowl is not ready.
Greedy is myopic. It optimizes each order alone and ends up using far too many
riders. It also cannot batch, because batching is a decision about pairs of orders,
and greedy only ever looks at one. This works at a village scale of a few dozen
orders an hour and falls apart the moment volume and cost matter.

### Attempt two: turn it into a matching problem (this is the real idea)

Stop assigning orders the instant they arrive. Collect the unassigned orders for a
short window, a few seconds, then solve them all together. Now you have a set of
orders on one side and a set of candidate partners on the other. This is a bipartite
graph: orders on the left, partners on the right, and an edge between an order and a
partner if that partner could sensibly take that order.

Each edge carries a cost. The cost is not just distance. It bundles: how far the
partner is from the restaurant, how long until the food is ready (idle waiting is
bad), whether taking this order makes the partner late for something, and the
partner's direction of travel. A real edge: `order 8837121 to partner 4471, cost 6.2`
where the 6.2 is a blended penalty in minutes-ish units, low because Rehman is idle,
close, and the food is nearly ready.

Now the question becomes: pick a set of edges so that each order gets exactly one
partner and the total cost is as small as possible. This is the classic
minimum weight matching on a bipartite graph. The academic system FoodMatch, built
and evaluated by researchers at IIT Delhi on real order data from a large Indian food
delivery company, maps the vehicle assignment problem to exactly this: minimum weight
perfect matching on a bipartite graph. That is a confirmed, published framing of this
exact problem in the Indian context.

How do you actually solve minimum weight bipartite matching? The textbook answer is
the Hungarian algorithm, or equivalently a min-cost max-flow computation. Both give
the provably optimal assignment. The catch is cost. Building the full graph is
quadratic: with N orders and M partners you have up to N times M edges, and computing
each edge cost means computing a route on a road network, which is itself expensive.
At 2,000 live orders and 3,000 nearby partners that is up to 6 million edges per
cycle, every few seconds. That does not fit in the time budget.

FoodMatch's real contribution is here, and it is a lovely engineering move: do not
build the full graph. Use best-first search to build only the subgraph that is very
likely to contain the optimal matching. In plain terms, for each order you only wire
up edges to the handful of partners that could plausibly win, ranked by a cheap
proximity estimate, and you stop expanding once further partners cannot beat what you
have. This prunes the 6 million edges down to a small fraction and keeps the matching
optimal in practice while running fast enough for live workloads. This is the same
spirit as the pruning trick in the autocomplete teardown: never compute a candidate
that cannot possibly win.

### Attempt three: batching, which is a clustering problem hiding inside the matching

Everything so far assigns one order to one partner. The money is in clubbing. How do
you decide that Arjun's bowl and the neighbor's biryani should ride together?

The clean way to see it: before you match orders to partners, first decide which
orders should travel together. Group orders into small batches where a batch is a set
of orders that are close in pickup, close in drop, and close in time. FoodMatch
reduces this batching step to a graph clustering problem. Build a graph where nodes
are orders and an edge connects two orders whose combined trip has a small detour
penalty, then find tight clusters. Each cluster becomes a single unit that you then
feed into the matching as if it were one bigger order. So the pipeline is: cluster
orders into batches, then run the min weight matching between batches and partners.

The batching test is a threshold, and it is refreshingly concrete. Public accounts of
how Swiggy and similar players batch describe the same rule: when you consider adding
a second pickup to a partner's trip, compare the total route time with and without it,
and only club if the added time is under a threshold, commonly in the range of 8 to
10 minutes, and only if both customers still land inside their promised windows. If
Arjun's food goes cold or the neighbor breaks their deadline, no club. Zomato has
publicly framed how much batching is acceptable as a business decision, tuned with
configurable batching windows and scoring functions, not a fixed constant. Direction
matters too: FoodMatch anticipates where a moving partner is heading using angular
distance, so it prefers clubbing orders that lie along the way rather than behind the
partner.

The data structure for a batch is just a small ordered list: the sequence of stops.
For Arjun and the biryani it is `[pickup food court, drop Arjun, drop neighbor]` or
whatever ordering minimizes the route. Ordering the stops inside a batch of two or
three is a tiny traveling salesman problem, solved by brute force because the batch is
small. You never batch twenty orders onto one bike, so the hard version of TSP never
shows up. Batches stay at two, sometimes three.

### Attempt four: the full optimization with scoring and strategic delay

The production version wraps the matching in an objective that trades off competing
goals, and lets the engine choose to wait. DoorDash has published the clearest public
description of this shape for its DeepRed dispatch system, and it is a good confirmed
reference for how the class of problem is solved at scale. They write it as a
mixed integer program. The decision variables are binary: for each candidate
partner-to-order (or partner-to-batch) pairing, a 0 or 1 for "make this assignment or
not." The constraints enforce the obvious rules: each order assigned at most once, a
partner's capacity respected. The objective is a scoring function that balances
efficiency (fewer rider-kilometers) against quality (food on time), while accounting
for uncertainty in the machine learning estimates of food ready time, travel time,
and whether the partner accepts. They solve it with Gurobi, a commercial
mixed integer program solver, fast enough to run continuously. The two moves that a
plain matching cannot make: batching decisions live inside the same program, and the
program can choose strategic delay, holding an order to the next cycle because a
better club is about to become possible.

For Zomato specifically, the public engineering signal is Active Dispatch, which
Zomato describes as a Deep Q-Network based multi-agent reinforcement learning model.
Its job is subtly different and complementary: it does not just match right now, it
tells idle partners where to reposition, using predicted demand and predicted partner
supply, so that when orders land there is already a partner nearby. Read together with
Zomato's published food preparation time model (torn down earlier in this ledger),
you can see the full assignment brain: predict when food is ready, predict where
demand will be, reposition idle riders toward it, then run the matching so partners
arrive as the food is bagged. I am inferring the exact matching solver Zomato uses in
production; the batching-as-matching framing and the repositioning model are both
confirmed by published work.

### The scale story at three tiers

Tier one, 1,000 live orders in a small city at lunch, a few thousand partners.
Everything is easy. The geo grid finds nearby partners in microseconds. The bipartite
graph is small enough that even a full Hungarian solve finishes in well under the
cycle budget. A single server holds the whole city's live state in memory. Nothing is
stressed. Greedy nearest-rider would even limp along here, which is exactly why small
launches never discover the real problem.

Tier two, 100,000 live orders across a metro region at dinner peak, tens of thousands
of partners. Now the quadratic graph is the enemy. A naive all-orders-versus-all-
partners matching would be tens of billions of edges and would miss its few-second
deadline badly. Three things save it. First, geo-sharding: you do not solve one giant
national matching, you split the map into zones and solve each zone's matching
independently and in parallel, because a partner in Whitefield is never going to carry
an order in Jayanagar anyway. The map partition makes the problem embarrassingly
parallel. Second, candidate pruning: FoodMatch's best-first subgraph means each order
only ever wires to its handful of realistic partners, so edges grow roughly linearly,
not quadratically. Third, the cycle itself is a queue: orders wait a few seconds to be
batched together, which both enables clubbing and smooths bursts. The batching window
is not just a business lever, it is also a load-shedding valve.

Tier three, 10 lakh (1 million) plus orders a day nationally, with lakhs of orders in
the same dinner hour and hundreds of thousands of partners live. At this tier the
bottleneck moves off the matching math and onto the moving-parts problem. Partner
locations stream in constantly and must update the geo index without locking it; this
is a high-write streaming problem, handled by keeping the live index in memory, sharded
by region, with updates flowing through a stream processor rather than a database round
trip per GPS ping. Route-time estimates for edge costs get precomputed and cached
because you cannot run a fresh road-network route for millions of candidate edges per
cycle; you approximate with cached zone-to-zone travel times and correct with a cheap
model, the same precompute-then-cache pattern seen across this ledger. And the whole
thing runs as many independent zone solvers, so adding a city adds a shard, not load to
a central brain. What breaks if you ignore this: a single global optimizer becomes a
single point of contention, GPS writes stall reads, and the few-second cycle blows out,
which users feel as a spinning "Finding a delivery partner." The survival kit is the
familiar one: shard by geography, keep hot state in memory, precompute and cache the
expensive route costs, and let a queue (the batching window) absorb the bursts.

## 8. The retention and habit mechanic

The dispatch engine is not a feature Arjun taps, so its habit effect is indirect but
powerful. What it moves is reliability, and reliability is what makes food delivery a
habit instead of a gamble.

The loop is this. Arjun orders. The engine keeps the 40 minute promise, mostly because
it timed the pickup so the rider did not wait and clubbed the trip cheaply enough that
Zomato could afford to price the delivery low. Hot food, on time, cheap delivery fee.
Arjun's trust ticks up one notch. Next Wednesday at 1:15pm he does not comparison shop,
he just opens Zomato again. The metric this moves is retention, through on-time
delivery rate and through delivery fee, which batching keeps low.

There is a real, observed second-order effect worth naming. Because batching lowers the
cost per delivery, it is what makes low or zero delivery fees on membership programs
financially possible. Zomato Gold and similar subscriptions lean on the fact that the
engine can club orders in dense areas; without efficient batching, free delivery would
bleed money. So the invisible matching decision quietly funds the most visible
retention product on the app. And on the partner side, better clubbing raises rupees
per hour, which retains delivery partners, which keeps supply high, which keeps
everyone's food on time. It is a flywheel that starts with a matching problem solved
well.

## 9. The lesson for Rare.lab

The lesson from dispatch is: do not assign work the instant it arrives, collect a short
window and solve the whole batch together, because the best global answer is invisible
if you only ever look at one item.

For Rare.lab this maps almost directly onto how the node graph compiles and runs. The
naive engine handles each node or each draw the moment it is dirtied: a slider moves,
recompile that node, re-render. That is greedy nearest-rider. It is myopic and it
wastes the GPU exactly the way greedy wastes riders. The batched engine collects every
change that happened within one frame's window, a few milliseconds, then plans the
whole update at once. Within that window you can do the two things batching buys
Zomato. You can club work: if five nodes all sample the same noise texture this frame,
compute it once and share it, the way one rider carries two nearby drops. And you can
strategically delay: if a node's input is about to change again this frame, do not
recompile it twice, hold it to the end of the window and compile the final state once.

Concretely: give the runtime a per-frame dirty set, deduplicate and cluster the pending
node updates by shared inputs, order them by the dependency graph (a tiny topological
sort, the batch's stop ordering), then dispatch one coalesced set of GPU passes. Just
like Zomato, put a small time window and a scoring function between "work arrived" and
"work dispatched," and make both the window and the batching threshold configurable so
you can tune the tradeoff between latency and throughput per device. The 8 to 10 minute
batching threshold has a direct analog: a per-frame compute budget in milliseconds,
above which you shed or defer work to the next frame rather than drop the frame. The
engine that wins is not the one that reacts fastest to each change. It is the one that
waits a beat, sees the whole batch, and does the least total work that keeps the promise.

## Sources

- FoodMatch: Batching and Matching for Food Delivery in Dynamic Road Networks (Joshi, Singh, et al., IIT Delhi), arXiv: https://arxiv.org/abs/2008.12905
- FoodMatch, ACM Transactions on Spatial Algorithms and Systems: https://dl.acm.org/doi/10.1145/3494530
- FoodMatch / IEEE ICDE conference version: https://ieeexplore.ieee.org/document/9458617/
- Using ML and Optimization to Solve DoorDash's Dispatch Problem (DeepRed, MIP, Gurobi): https://careersatdoordash.com/blog/using-ml-and-optimization-to-solve-doordashs-dispatch-problem/
- Next-Generation Optimization for Dasher Dispatch at DoorDash: https://careersatdoordash.com/blog/next-generation-optimization-for-dasher-dispatch-at-doordash/
- Zomato Engineering, The Deep Tech Behind Estimating Food Preparation Time: https://blog.zomato.com/food-preparation-time
- Zomato Engineering, The elements of scalable machine learning: https://blog.zomato.com/elements-of-scalable-machine-learning
- Evolution of Food Delivery Dispatching (greedy to matching to optimization): https://ilyazinkovich.github.io/2020/06/16/delivery-dispatching-evolution.html
- Route Optimization for Food Delivery Apps: Lessons from Swiggy and UberEats (NextBillion.ai): https://nextbillion.ai/blog/route-optimization-for-food-delivery-apps
- Crowdsourced on-demand food delivery: an order batching and assignment algorithm (Transportation Research Part C): https://www.sciencedirect.com/science/article/pii/S0968090X2300044X
- Gigs with Guarantees: Achieving Fair Wage for Food Delivery Workers (fairness in assignment): https://arxiv.org/pdf/2205.03530
