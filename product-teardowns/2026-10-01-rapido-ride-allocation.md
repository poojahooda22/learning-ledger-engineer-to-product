# Rapido: ride allocation (which captain actually gets your request)

Date: 2026-10-01
Product: Rapido
Feature: Ride allocation and dispatch. You tap Book, and in a few seconds a
specific bike captain is on his way to you. This teardown is about the decision
in between: out of the hundreds of captains near you, which ONE (or which few,
in which order) gets offered your ride, and why.

This is a different Rapido feature from the one already in the ledger. On
2026-07-03 we tore down Rapido captain navigation: the turn-by-turn engine that
guides the captain once he has your ride. That is what happens AFTER allocation.
This is what happens BEFORE: the matching brain that picks him in the first
place.

It also sits next to three earlier dispatch teardowns, and the point of this one
is what makes it different. Uber DISCO (2026-07-02) was batched matching as an
assignment problem. Uber surge (2026-06-14) and Swiggy serviceability
(2026-09-28) were about the spatial index (H3 hexagons, point in polygon). Zomato
dispatch (2026-07-22) clubbed food orders. This teardown's spine is a single
sharp idea none of those centered: **the straight-line distance to a captain is
a lie, and almost everything hard about bike-taxi dispatch is the work of
replacing that lie with the real driving time to you and the real chance he says
yes.**

---

## 1. The user

It is 9:10 on a Tuesday night. Arjun is standing outside Indiranagar metro
station in Bengaluru. He has a 20 rupee idli dinner in a bag and he wants to get
the 4 km home to Domlur before the drizzle turns into rain. A cab is 180 rupees
and ten minutes away. An auto driver quotes 120 and shrugs. Arjun opens Rapido,
the pickup pin is already on the metro gate, he taps Book Bike.

He is not thinking about dispatch. He is thinking about rain. What he wants is
simple: a bike, soon, cheap, and he wants the little "captain assigned" card to
appear fast so he can stop refreshing and put his phone away.

The whole allocation machine exists to make the next eight seconds feel like
nothing happened. He taps, he waits, a face and a bike number appear, a blue dot
starts moving toward him. Behind that calm, the system just searched a few
hundred moving captains, scored them, and offered the ride to the right ones in
the right order, all inside the time it takes him to shift the dinner bag to his
other hand.

## 2. The real problem

Here is the pain, described like a friend would.

Rapido runs on bikes, and the bike market is brutal on two sides at once.

On Arjun's side: a bike ride is cheap and short. The average Rapido trip is a few
kilometers. If allocation takes 40 seconds, Arjun just cancels and opens Uber or
Ola, because the whole trip home is only 12 minutes. The time you spend finding
the captain is a tax on a tiny fare, and riders have no patience for it. Rapido's
own engineers have said the matching has to resolve in roughly two to three
seconds of compute or riders start abandoning the request.

On the captain's side: Yusuf is sitting on his bike 300 meters away, but across a
divided road, in traffic, and the next legal U-turn is 1.5 km down. As the crow
flies he is the closest captain to Arjun. In real life he is seven minutes away
and he knows it, so when his phone buzzes with Arjun's ride he just ignores it.
Now Arjun has waited, gotten nothing, and the app shows the dreaded line: "2
captains did not accept your ride." Every rejected offer is dead time stacked on
an already tiny trip.

So the real problem is a squeeze. You must pick a captain fast, the fare is too
small to waste a second on, and the obvious cheap answer (just ping whoever is
nearest in a straight line) is wrong often enough to break the whole thing. The
nearest dot on the map is routinely not the captain who will reach Arjun fastest,
and not the captain who will say yes.

## 3. The feature in one sentence

Given a rider's pickup point and hundreds of moving captains nearby, allocation
finds the small set of captains who can actually reach that point fastest and are
most likely to accept, and offers them the ride in that order, in a couple of
seconds.

## 4. Jobs to be done

What is Arjun really hiring this feature to do?

- "Get me a bike before I lose patience." Speed of assignment is the product,
  not a detail of it.
- "Do not make me the one who notices the system guessed wrong." No assigning a
  captain who is actually stuck across the road and will cancel.
- "Make the price and the wait feel honest." The ETA on the card should be the
  real ETA.

What is Yusuf the captain hiring it to do?

- "Only buzz me for rides I can actually do well." A ping for a pickup he cannot
  reach cleanly is noise that costs him attention in traffic.
- "Keep my dead kilometers low." Every kilometer he rides empty to reach a pickup
  is fuel he spends and money he does not make. Rapido captains largely keep the
  fare (the platform runs a subscription / low-take model rather than a fat
  per-ride commission), so dead kilometers come straight out of Yusuf's pocket,
  which makes this his number one complaint.

Allocation serves both at once, and the two jobs pull against each other. The
captain who is best for Arjun's wait is often not the one with the least dead
distance for Yusuf. The dispatch brain lives in that tension.

## 5. How it works for the user

Arjun taps Book Bike. A card slides up that says "Finding your captain" with a
little radar animation. Within a few seconds it flips to a captain card: a name,
a photo, a rating, a bike registration number, and an ETA like "Yusuf is 3 mins
away." A blue dot appears on the map and starts crawling toward the metro gate.

What Arjun never sees is that his request may have been offered to one captain
who did not respond in time, then a second, then accepted by a third, all inside
those few seconds. He also never sees the version of events where the system
quietly refused to offer the ride to the nearest-looking captain because it knew
the road did not allow it.

The captain's phone, meanwhile, buzzes with an offer card: pickup point, rough
drop direction, fare, and a countdown. Rapido recently added voice readouts that
speak the ride details aloud in the captain's own language, so Yusuf can decide
without taking his eyes off the road. He taps Accept, and from that instant the
navigation teardown from 2026-07-03 takes over.

## 6. The actual flow, step by step

1. Arjun's pickup is a lat long (the metro gate, 12.9784, 77.6408). His app also
   sends his likely drop.
2. The request hits the dispatch service. First job: find candidate captains near
   that point. Not all captains in Bengaluru. The handful close enough to matter.
3. The service looks up which captains are live and free in the cells around
   Arjun's location, pulling their latest reported positions.
4. For each candidate, estimate the real driving time from where he is to Arjun's
   pin. Not the straight-line distance. The road time.
5. Score and sort the candidates by a blend of that driving-time ETA and how
   likely each captain is to accept, plus fairness and dead-distance factors.
6. Offer the ride. Depending on supply, this is either a sequential ping down the
   sorted list (offer to the top captain, wait a couple of seconds, if no accept
   move to the next) or a batched assignment that pairs several waiting riders to
   several captains at once.
7. First captain to accept wins the ride. The others' offers are withdrawn.
8. Arjun gets the captain card. A persistent connection (a WebSocket) now streams
   Yusuf's moving position to Arjun's map and keeps both apps in sync on trip
   state.
9. If nobody accepts, the app widens the search, retries, and eventually shows
   "captains did not accept, searching again." That visible line is the dispatch
   loop leaking into the UI.

The sort in step 5 happens on the server, over live data, never on Arjun's phone.
His phone just renders the winner.

## 7. Under the hood, like the engineer

This is the heart. Allocation is two halves, the same split that runs through the
whole ledger: **matching** (find the candidates) and **ranking** (order them).
Mixing them up is the classic mistake. Keep them apart and each becomes tractable.

### Half one: matching (find the candidate captains)

The first question is dumb and physical: who is near Arjun right now?

**The naive way, and Rapido's real starting point.** Rapido's early dispatch was,
by their own account, a radial system. A rider requests a ride, the system draws
a circle of a fixed radius (around 2 km) around the pickup, collects every
captain inside that circle, computes the crow-fly distance from each to the
rider, and pings them in order of that straight-line distance. Simple, and for a
young app in one city, completely fine.

It has two problems that both grow with scale. First, "collect every captain
inside the circle" means you need a fast way to answer "which captains are inside
this circle" without scanning every captain in the city on every request.
Second, crow-fly distance is the lie from section 2: it ranks Yusuf-across-the-
divider as the best captain when he is actually the worst.

**The data structure that fixes the first problem: a spatial index.** Instead of
scanning all captains, you partition the map into cells and keep a map from each
cell to the list of captains currently in it. Rapido moved its dispatch onto
Uber's open-source H3 grid. H3 tiles the Earth into hexagons, each with a unique
64-bit integer ID, in 16 nested resolution levels. Hexagons are the clever part:
every one of a hexagon's six neighbors is the same distance from its center,
which squares and triangles cannot promise, so "expand the search ring outward"
is clean and symmetric.

Concretely, you keep a hash map: `h3_cell_id -> [captain_id, captain_id, ...]`.
To find candidates for Arjun, convert his lat long to its H3 cell (pure bit math,
nanoseconds, no database), then grab that cell plus its ring of neighbors (H3
calls this a k-ring). At a resolution where a cell is a few hundred meters
across, a k-ring of 1 or 2 covers Arjun's neighborhood and hands you a few dozen
to a few hundred candidate captains instead of every captain in Bengaluru. This
is the exact move from Swiggy serviceability (2026-09-28) and Uber surge
(2026-06-14), reused here: a cheap cell lookup shrinks the problem before any
expensive work runs. It is also, structurally, an inverted index, the same idea
as Amazon search (2026-06-23): the term is "this cell," the posting list is "the
captains in it."

**The captains are moving, so the index is a live stream, not a table.** Yusuf's
bike reports its GPS every few seconds. Those pings are a firehose, not rows you
UPDATE one at a time. The standard shape, and the one Rapido's stack points to,
is: pings flow in over a stream (Kafka is Rapido's event bus, carrying millions
of events a day), a processor keeps only the latest position per captain in a
fast in-memory geo store (Redis geo, keyed by H3 cell), and the dispatch service
reads that store. This is the same architecture as Swiggy live tracking
(2026-06-24) and Uber live driver location (2026-08-08): location is an event
stream collapsed to a latest-value store, never a slow write to a disk table on
every ping.

**The ugly real-world problem under the index: you cannot match a captain you
cannot locate.** Captains switch off GPS to save battery, or the signal dies in a
basement, and then their dot is stale or gone. Rapido has said flatly that it
improved its ability to capture captain locations in real time from as low as 60%
to close to 95%, by fusing GPS with other phone sensors instead of trusting GPS
alone. That jump from 60 to 95 is not a footnote. If 40% of your supply is
invisible, your spatial index is lying about who is available, and the best
captain for Arjun may simply never be considered. Matching quality is capped by
location quality.

### Half two: ranking (order the candidates)

Now you have, say, 80 candidate captains near Arjun. This is where the lie gets
corrected.

**Driving time, not straight-line distance.** Rapido's engineers have said the
single most crucial component of a smart dispatch system is reliable driving-time
estimates, and that they build these from historical ride data. The road network
is a directed weighted graph: intersections are nodes, road segments are edges,
and each edge is weighted by how long it really takes to traverse right now. The
shortest path through that graph is a classic problem (Dijkstra with a min-heap,
sped up in practice with precomputed shortcuts, which is the navigation engine
from 2026-07-03). But the sharp insight is that the raw graph estimate is not
enough. You correct it with what actually happened on this road at this hour in
the past. Yusuf being across a divider with no U-turn for 1.5 km is invisible to
crow-fly distance and obvious to a driving-time model trained on real trips. This
is the same two-step as Uber ETA (2026-07-15): a routing estimate plus a learned
correction from history.

Rapido put this plainly in a customer story: "a Captain may be closest to the
customer but on the opposite side of the road in heavy traffic and unable to make
a U-turn for several kilometers." That one sentence is the whole reason ranking
exists as a separate half.

**Acceptance, not just reachability.** A captain who can reach Arjun in 3 minutes
but will decline the ride is worse than one who takes 4 minutes and accepts,
because a decline costs everyone the ping timeout. So the ranking score folds in
a predicted probability that this captain accepts this ride. That probability is
learned from the captain's own history: does he take short rides, rides in this
direction, rides at this hour, rides toward areas where he can get a return fare.
Rapido's dispatch writeup lists exactly the signals you would expect them to
optimize: ETA, matching time, the distance the captain has to drive to reach the
customer (dead kilometers), and cancellations from both the demand and supply
side. Those are the columns of the ranking objective.

So the per-candidate score is roughly:

`score = f(predicted driving-time ETA to pickup, predicted acceptance
probability, captain dead distance, fairness/rotation)`

and you sort descending. This is pure offline-think / online-lookup, the ledger's
backbone: the ETA model and the acceptance model are trained offline on the data
lake (Rapido's platform lands events in Kafka, then into a warehouse with bronze
and silver layers for exactly this kind of modeling), and the live path just
scores 80 candidates against the loaded models and sorts. The expensive learning
is batch and cold. The hot path is a lookup and a sort.

### Offering the ride: sequential ping vs batched assignment

You have a sorted list. Now, who do you actually buzz?

**Sequential broadcast.** The simplest honest answer, and the one Arjun sees
leak into the UI, is: offer to the top captain, give him a few seconds, if he
does not accept, offer to the next. This is a priority queue drained in order.
Its fingerprint is the message "x of y captains did not accept your ride," which
is literally the system telling you how far down the sorted list it had to walk.
It is simple and it is fair to the rider (best captain first), but every decline
adds seconds, which is poison on a tiny fare.

**Batched assignment.** At density, one-at-a-time is leaving money on the table.
If five riders request near Koramangala in the same two-second window and there
are eight free captains, you should not greedily give rider one his nearest and
then scramble for rider two. You should solve all five at once for the globally
best pairing, because the nearest captain to rider one might be far better for
rider three. This is the assignment problem on a bipartite graph (riders on one
side, captains on the other, edge cost = the ranking score), solvable optimally
with the Hungarian algorithm in O(n^3), or greedily when n gets large and the
clock is tight. This is the DISCO idea from Uber (2026-07-02): collect requests
into a short time window, then match the batch, trading a sliver of latency for a
much better overall pairing. The two-to-three-second budget is the width of that
window.

Rapido operates where both modes matter: dense metro cores where batching wins,
and thin Tier-2 streets at midnight where a single sequential ping is all the
supply allows.

### Demand forecasting: moving the supply before the request exists

The best way to win the two-second match is to have already positioned a captain
nearby. Rapido runs hyperlocal demand forecasting: models that ingest weather,
local events, and neighborhood-level ride history and produce demand maps updated
in near real time, so captains can be nudged toward where the next requests will
come from. A cricket match letting out at Chinnaswamy Stadium, or the first drops
of rain in Indiranagar, both spike demand in a known cell minutes before the
requests land. Getting supply there early shrinks dead kilometers (the captain is
already close) and shrinks Arjun's wait. Forecasting is the preload for
allocation, the same spirit as everything precomputed offline in this ledger.

### The scale story: 1,000, 100,000, 10 million plus

What grows here is the rate of requests and the size and churn of the live
captain pool. The catalog is not books, it is moving dots.

**1,000 captains, one small city, a few hundred rides a day.** The radial system
is genuinely correct. Keep all live captains in an in-memory list, and on each
request do a linear scan for those within 2 km, crow-fly sort, ping nearest
first. An H3 index and a learned acceptance model would be over-engineering. A
linear scan of 1,000 captains is microseconds. Ship it and go home. Nothing
breaks.

**100,000 captains, a dozen cities, hundreds of thousands of rides a day.** Two
things break. First, scanning all live captains per request, times thousands of
concurrent requests at dinner peak, makes the scan the hot path for the whole
country. Fix: the H3 spatial index, so each request touches a few cells, not the
whole fleet, plus the Redis geo store for live positions and read replicas for
scale. Second, and sharper, crow-fly ranking now visibly hurts: declines and
cancellations climb because the "nearest" captain keeps being the across-the-
divider captain. Fix: replace crow-fly with driving-time ETA from historical
data, and add the acceptance model. This is the tier where the architecture
earns its keep, and it is exactly the journey Rapido describes: from the radial
crow-fly system to a data-driven one. The rider-facing win is that "finding your
captain" stops timing out.

**10 million plus rides a day, 100 to 400 cities, 15,000-plus ride requests a
minute.** Now four new walls, and they are the interesting ones.

- Shard by geography. A dispatch node in Bengaluru should not hold Delhi's
  captains in memory. H3 cell IDs partition naturally by prefix, and city
  boundaries partition the fleet, so each region is an independent shard with its
  own index and its own models. This is the same shard-by-tenant discipline as
  Notion-by-workspace (2026-06-25) and Stripe-by-account (2026-09-01), applied to
  the map.
- The thundering herd. At 6:30 pm in the rain, requests in one cell spike an
  order of magnitude in seconds. Demand forecasting is the pressure valve,
  because pre-positioning supply spreads the load before it arrives, and a
  short batching window naturally groups the burst into one assignment solve
  instead of thousands of frantic sequential pings.
- Hot-captain contention. Yusuf is the single best candidate for three riders who
  all request at once. If all three offers go out and he accepts the first, the
  other two assignments must be instantly and atomically invalidated, or two
  riders both see "Yusuf assigned" and one gets betrayed. This is the atomic
  check-and-claim problem from Zepto's last carton of milk (2026-06-17) and
  District's last concert seat (2026-08-21), moved onto a captain: a captain is a
  contested resource, and a claim must be a single atomic win, not a race.
- Dead and stale supply. At 4 million captains, a meaningful slice have stale GPS,
  dead apps, or have gone offline mid-session every minute. The index must prune
  them aggressively (the 60-to-95 percent location fight), because a candidate
  list full of ghosts wastes the ping budget exactly when a real rider is
  waiting.

What never changes across all three tiers: Arjun's live path stays a cell
lookup, a score, a sort, and a ping. All the tier-three pain is pushed offline
(training the ETA and acceptance models, forecasting demand, rebuilding shard
maps) or onto infrastructure (sharding, Redis, Kafka, WebSockets). The decision
the rider waits on stays cheap. That is the offline-think / online-lookup spine
one more time.

**Honest labeling of fact vs inference.** Confirmed by Rapido's own engineering
and customer material: the radial crow-fly starting system with a roughly 2 km
circle and ping-in-order; the move to H3; driving-time estimates built from
historical ride data as the crucial component; the ETA / matching-time / dead-
distance / cancellation metrics; the 60-to-95 percent location-capture jump; the
road-topology (U-turn) reasoning; WebSockets for live comms; Kafka plus a bronze
or silver warehouse; hyperlocal demand forecasting; voice order readouts; and the
scale numbers (around 45 million rides a month, roughly 1.5 million a day rising
past 3 million a day across categories by 2025, 4 million captains, 100-plus
cities, over 1 billion cumulative rides, 15,000-plus ride requests a minute,
allocation in under 15 to 20 seconds with a 2-to-3-second compute budget).
Clearly labeled inference (standard ways this class of problem is solved, not
confirmed Rapido internals): the specific hash-map-cell-to-captain-list and
k-ring mechanics; Redis geo as the latest-position store; the Hungarian algorithm
and the sequential-vs-batched offer choice; and the exact learned shape of the
acceptance model. Rapido has not published that interior in full, and I am
giving the well-grounded version.

## 8. The retention and habit mechanic

Allocation builds retention on both sides of the market at once, and the loop is
a flywheel, not a daily nudge.

**Rider side.** The habit Rapido wants is "just open Rapido" replacing any
thought about which app to use. That reflex only forms if allocation is fast and
right almost every time. The loop is: open, get a captain in a few seconds, reach
home, repeat tomorrow. The thing that kills the reflex is the "captains did not
accept your ride" screen, because it converts a 12-minute trip into a 5-minute
wait and sends Arjun to a competitor. So allocation speed and acceptance accuracy
are not features that support retention, they ARE the retention. This is invisible
-craft trust, the same shape as Spotify loudness (2026-08-28) and Swiggy
serviceability (2026-09-28): nobody thinks "thank you, driving-time model," they
think "I always get a bike fast," and that thought is the moat.

**Captain side, and this is the sharper one.** Because Rapido runs a low-take /
subscription model where captains largely keep the fare, dead kilometers are the
captain's biggest cost and his biggest reason to quit the platform for a rival.
Every improvement that reduces dead kilometers (better driving-time matching,
demand forecasting that pre-positions him near real demand) directly raises
Yusuf's take-home per hour. Rapido states its matching and routing gains reduced
dead kilometers and improved captain earning efficiency. Higher captain earnings
means captains stay, which means denser supply, which means faster allocation for
Arjun, which means more riders, which means more rides for captains. The
allocation engine is the pump of that flywheel.

**Which metric it moves.** Both retention metrics, rider and captain, and under
them the real currency of a marketplace: liquidity. The underlying activation
metric is conversion from "tapped Book" to "captain assigned" inside the patience
window, and the underlying revenue metric is rides completed per captain-hour
(throughput), which the dead-kilometer reduction lifts directly. A real observed
example of the loop working: the location-capture fix (60 to 95 percent) and the
routing work drove Rapido's direction-related support calls from 5 percent of
calls to under 1 percent, which freed captains from being on the phone and put
them back on rides, feeding earnings and retention.

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor that compiles visual effects to shippable code,
plus an embeddable runtime. The one idea to steal from Rapido dispatch is the
one that runs through this whole teardown: **the cheap-looking estimate is a lie,
and the scalable fix is to rank on measured real cost, computed offline, with a
cheap spatial cull in front of it.**

Concrete and actionable:

1. **Crow-fly cost is your trap too.** The naive way to decide which effect or
   pass to run, or at what quality, on a given device is a static cost estimate
   (node count, texture size, a cost number baked at compile time). That is
   crow-fly distance. The real cost is the measured GPU milliseconds this effect
   takes on this device class, on this pipeline-state path, which depends on road
   topology you cannot see from the graph: the U-turn is a shader stall, a
   bandwidth-bound texture fetch, a pipeline rebind. Build the runtime's
   scheduling decisions on measured cost from profiling telemetry, not on the
   static estimate. This is Rapido replacing crow-fly with driving time learned
   from history.

2. **Two halves, cheap cull before expensive score.** Do not run the expensive
   cost model over every effect every frame. Put an H3-style cheap cull first (a
   spatial or visibility partition that answers "which few effects could touch
   this screen tile," the exact move from Swiggy serviceability applied to pixels
   in the 2026-09-28 lesson), and only run the measured-cost ranking on the
   survivors. Matching then ranking, never the expensive step on the whole set.

3. **Offline-think, online-lookup.** Train the cost and acceptance models offline
   from telemetry, as Rapido trains its ETA and acceptance models offline from
   the ride lake. The runtime's hot path, at 60 frames a second, should be a
   lookup and a sort of a handful of candidates against a loaded model, never a
   recompute. The compiler is your batch pipeline; emit the cost model as a build
   artifact.

4. **Predict the decline.** Rapido does not offer a ride to a captain who will
   reject it, because the rejection costs the whole budget. Rare.lab should not
   dispatch an effect to a device that will "reject" it by blowing the frame
   budget. Predict, per device class, which effects or quality tiers will miss the
   frame deadline, and degrade them gracefully before you try, instead of
   shipping the work and dropping frames. That predicted-acceptance gate is the
   difference between a smooth runtime and the "2 captains did not accept"
   experience rendered as jank.

5. **Batch the window.** When several pieces of work could be scheduled in the
   same frame, do not greedily dispatch each first-come. Collect them into the
   frame's budget window and solve for the globally best set to run, the way
   batched assignment beats sequential pinging under density. One good schedule
   beats five greedy ones.

One line: Rapido got fast, fair dispatch by refusing to trust straight-line
distance, ranking captains on learned real driving time and real acceptance after
a cheap hexagon cull, and keeping the live decision a lookup while all the
learning sits offline. Build Rare.lab's runtime scheduler the same way: cheap
spatial cull, then rank on measured cost and predicted feasibility from offline-
trained models, and never let a static estimate decide what the GPU actually does.

---

## Sources

Primary (Rapido's own engineering and customer material):

- Rapido Labs, "Improving Ride Dispatch with Data at Rapido." The radial crow-fly
  starting system (~2 km circle, ping in order), driving-time estimates from
  historical data as the crucial component, and the ETA / matching-time / dead-
  distance / cancellation metrics. https://medium.com/rapido-labs/improving-dispatch-with-data-6a307dab7ecc
- Rapido Labs, "Data Platform @ Rapido (Part I): cheap, efficient and scalable
  analytics." Kafka event bus, millions of events a day, at-least-once, bronze /
  silver warehouse layers. https://medium.com/rapido-labs/data-platform-rapido-part-i-cheap-efficient-and-scalable-analytics-52662111b2d2
- Google Cloud, "Rapido" customer story. Scale (45M rides/month, 4M captains, 100
  cities, 1B+ cumulative rides), the 60%-to-95% captain-location-capture
  improvement, the road-topology / U-turn matching reasoning, two-wheeler routing
  and ETA, and the 5%-to-under-1% direction-call drop. https://cloud.google.com/customers/rapido-maps

Secondary (press and analysis corroborating the engine and numbers):

- "From Coordinates to Hexagons: The Design Behind Rapido's Dispatch." Radial to
  H3, 64-bit hexagon cell IDs, equidistant neighbors, WebSockets for live comms.
  https://medium.com/@iwasifirshad/from-coordinates-to-hexagons-the-design-behind-rapidos-dispatch-4e7f699e955e
- CIO Tech Outlook, "Rapido Tech Boosts Ride Matching, Captain Earnings."
  15,000+ ride requests/min, allocation in under 15 to 20 seconds, matching on
  distance plus ETA plus acceptance rate, demand forecasting from weather /
  events / ride history, dead-kilometer reduction. https://www.ciotechoutlook.com/news/rapido-tech-boosts-ride-matching-captain-earnings-nid-14645-cid-160.html
- Digital Terminal, "Rapido Enhances Ride Experience and Earnings with Advanced
  Matching and Forecasting Engine." Hyperlocal demand maps updated in near real
  time, 2-to-3-second matching budget. https://digitalterminal.in/startup/rapido-enhances-ride-experience-and-earnings-with-advanced-matching-and-forecasting-engine
- YourStory, "Rapido set to expand to 500 cities" (2025). ~3.3M rides/day and
  city-expansion scale. https://yourstory.com/2025/01/rapido-expand-500-cities
- Explorist, "Case Study: Rapido - 'x/y captains didn't accept your ride.'" The
  sequential-ping-in-order model made visible in the UI. https://explorist.in/case-study-rapido-x-y-captains-didnt-accept-your-ridecase-study-rapido/

Reference (standard methods for this class of problem):

- Uber Engineering, "H3: A Hexagonal Hierarchical Spatial Index," and the H3
  documentation at https://h3geo.org . Hexagon cells, 64-bit IDs, k-ring
  neighbor expansion.
- The assignment problem and the Hungarian algorithm (Kuhn, 1955) for optimal
  bipartite matching, as referenced in the Uber DISCO teardown (2026-07-02).
