# Zepto: dark-store inventory and order routing (the "deliver in 10 minutes" engine)

Date: 2026-06-17
Product: Zepto
Feature: Dark-store selection, real-time inventory, and order-to-rider routing

A note on sourcing up front. Zepto has not published a detailed engineering
blog the way Uber or Spotify have. So this teardown separates two things
cleanly. Facts are the publicly reported numbers (store sizes, picking times,
median delivery) and the named tools that secondary sources attribute to
Zepto. Inference is the "this is how this class of problem is solved" part,
labeled every time. The hard engineering here (atomic stock, geospatial
nearest-store, in-memory rider matching) is well understood across the
industry even where Zepto's exact code is private, so the inference is
grounded, not hand-waved.

---

## 1. The user

It is 9:40 on a Tuesday night in Powai, Mumbai. Riya is cooking and realizes
she is out of milk and has no onions left for the dal. The kirana shop
downstairs shut at 9. She does not want to get dressed, take the lift down,
and walk 400 meters hoping something is open. She opens Zepto, types "milk,"
adds Amul Taaza 500ml and a pack of onions, taps pay, and goes back to the
stove. The app says the order will arrive in 9 minutes. It shows up in 8.

She is not shopping. She is patching a tiny hole in her evening. The whole
interaction has to be shorter than the annoyance it removes, or it is not
worth opening the app at all.

## 2. The real problem

The pain is not "I want groceries." It is "I need this one thing right now
and every other option costs me more time or effort than it is worth." The
corner shop is closed or out of stock. A big online grocery order arrives
tomorrow, which is useless tonight. Driving to a 24-hour store means parking,
queueing, and twenty minutes gone.

Described like a friend: you just want the onion to appear before the dal
burns. The promise Zepto makes is that the gap between "I need it" and "I
have it" collapses to the time it takes to boil water.

That promise is brutally hard because it has to be kept while the item is
actually in stock, near you, and a rider is free. Get any of the three wrong
and the 10-minute promise becomes a 10-minute lie, which is worse than not
promising at all.

## 3. The feature in one sentence

When you place an order, Zepto instantly picks the one nearby dark store that
both has all your items in stock and can reach you fastest, reserves that
exact stock so nobody else can grab it, and assigns a rider, all before you
put your phone down.

## 4. Jobs to be done

- "When I run out of one thing at night, get it to me before the moment
  passes, without me leaving the kitchen."
- "Do not show me an item, let me pay, and then cancel because it was
  actually out of stock." (Post-payment cancellation is the cardinal sin of
  quick commerce.)
- "Make the wait predictable. 9 minutes should mean 9 minutes, not a hopeful
  guess." A wrong ETA erodes trust faster than a slightly longer correct one.
- "Let me reorder my usuals in two taps." Riya buys the same milk most weeks.

## 5. How it works for the user

Riya never sees a store. She sees a catalog that quietly is the inventory of
the single dark store assigned to her location right now. The Powai dark
store stocks roughly 2,500 to 6,000 SKUs, not the 50,000 a supermarket
carries, so the catalog she sees is already filtered to "things that can
reach you in minutes." If she had searched for a niche imported cheese, it
simply would not appear, because honesty about availability is the product.

She adds items, sees a live ETA, pays, and then watches a tracker: order
placed, packed, rider assigned, rider on the way, arriving. The packing step
is real and fast. Reported picking benchmarks inside Zepto stores are around
76 seconds from order to handoff, and the company has publicly cited a median
delivery time near 8 minutes 47 seconds (reported figures, not from a Zepto
engineering paper).

## 6. The actual flow, step by step

1. Riya opens the app. Her phone sends location (lat/lon, say 19.1176 N,
   72.9060 E for Powai).
2. The backend resolves that point to a serviceable dark store. If no store
   covers her polygon, she sees "we are not here yet." She is in range, so
   the Powai store is selected.
3. The catalog and prices she sees are that store's live inventory. Amul
   Taaza 500ml shows "in stock"; the onion pack shows "in stock."
4. She adds both, taps pay. Payment authorizes (UPI, ~2 seconds).
5. The system atomically reserves 1 unit of Amul Taaza and 1 onion pack from
   the Powai store's counters. If either had hit zero a millisecond earlier,
   she would have been told before charging, not after.
6. A pick task prints at the store. A picker walks a known route past the
   milk fridge and the produce bin, bags the two items in about a minute.
7. In parallel, the dispatch system finds the nearest free rider to the
   store and pre-assigns them so they are at the counter when the bag is
   ready.
8. Rider grabs the bag, the app streams their GPS, ETA updates live, bag
   reaches Riya's door in 8 minutes. Stock counters were already decremented
   at step 5, so the catalog never oversold.

## 7. Under the hood, like the engineer

There are three separate hard problems stitched together. Treat them one at
a time, because they fail at different scales for different reasons.

### Problem A: which store serves this address (a geospatial point-in-polygon)

Every dark store has a serviceability area, a polygon drawn around it (roughly
a 2 to 3 km radius in dense cities, but shaped by roads and rider reach, not a
clean circle). The question "which store serves 19.1176, 72.9060?" is a
point-in-polygon search.

- Data structure: the catalog of store polygons lives in a spatial index. The
  naive version checks the point against every polygon, which is O(number of
  stores). At a few hundred stores that is fine. At thousands across India it
  is wasteful to do on every app open. A spatial index (R-tree, or a geohash
  bucket map) narrows the candidate polygons to the handful near the point
  first, then does the exact polygon test on those few. This is the same
  matching-then-refining split that search engines use: cheap filter to a
  small candidate set, then exact check.
- Reported tool: secondary write-ups attribute Tile38 to Zepto for the
  real-time geospatial layer. Tile38 is an open-source in-memory geospatial
  store (tidwall/tile38) that speaks the Redis RESP protocol and answers
  NEARBY, WITHIN, and INTERSECTS over points, bounding boxes, geohashes, and
  GeoJSON polygons. Whether Zepto uses it for store selection, rider matching,
  or both is not confirmed by Zepto; treat the tool name as reported, the
  pattern as solid.

Concrete example: Riya's point falls inside the Powai polygon and outside the
neighboring Chandivali polygon. The index returns both as candidates because
their bounding boxes overlap her area, then the exact polygon test keeps
Powai. Sorting, of course, happens server-side. Her phone never downloads a
map of polygons to test locally.

### Problem B: reserving the last carton of milk without overselling (atomic decrement)

This is the heart of the whole thing, and it is a classic concurrency
problem. At 9:40pm in Powai, twenty people might tap "buy Amul Taaza" within
the same few seconds while the store has 18 cartons left. Two of them must be
told "out of stock" cleanly, and at payment time, never after.

The wrong way is "read stock, check it is greater than zero, then write
stock minus one." Between the read and the write, another order sneaks in.
This is the check-then-act race, and it is exactly how systems oversell.

The right way is to make check-and-decrement a single indivisible operation.
Two industry-standard patterns, both well documented:

- In-memory counter with an atomic script. Keep a counter per SKU per store,
  for example `stock:powai:amul_taaza_500` = 18, in Redis or a Redis-protocol
  store. Run a tiny Lua script on the server that checks `> 0` and `DECR`s in
  one atomic step, returning 1 for success or 0 for sold out. A single Redis
  node runs each script with no interleaving, so the race cannot happen, and
  it sustains on the order of 100,000 ops per second on commodity hardware
  (Redis docs and widely published flash-sale designs). Because it lives in
  memory, you pair it with append-only persistence (AOF, fsync every second)
  so a crash does not lose decrements.
- Database row lock with `SELECT ... FOR UPDATE` or `SKIP LOCKED`. Shopify
  published in 2026 that they moved high-contention inventory reservations
  onto MySQL using `SKIP LOCKED` and composite keys, precisely to get
  correctness under contention without a separate cache to keep in sync.

A subtlety quick commerce demands: reservation versus commit. When Riya pays,
the system reserves the unit (decrement, or write a reservation row with a
short TTL). If payment fails or she abandons, the reservation expires and the
carton returns to the pool. Redis even ships an official tutorial on exactly
this using WATCH/MULTI and an audit stream. So the milk is held for her for a
few seconds, not lost forever if her UPI hiccups.

Concrete walk-through: store counter is 18. Riya's Lua script runs, sees 18,
decrements to 17, returns 1, she is charged. A simultaneous order from
another flat runs a microsecond later, sees 17, decrements to 16. When the
counter is 0, the next script sees 0 and returns sold-out, and that user sees
"out of stock" before paying. Nobody gets a post-payment cancellation.

### Problem C: getting a rider to the door (in-memory nearest-rider matching)

Once the bag is being packed, you need the closest free rider. Rider
positions stream in constantly (every few seconds). The query is "free riders
within 1.5 km of the Powai store, pick the best one." That is a NEARBY query
over live points, ranked.

- Data structure: rider locations live in an in-memory geospatial index
  keyed by store proximity, updated continuously. Tile38 (or Redis GEO
  commands like GEOADD/GEOSEARCH) answers "nearest N riders to this point"
  without scanning every rider in India. You can also attach fields (a rider's
  current load, whether they are mid-delivery) and filter with WHERE in the
  same query, for example "nearby riders where status=free within 1500m."
- Two halves again, exactly like search ranking. Matching: the geospatial
  query returns the small candidate set of nearby free riders. Ranking: score
  those candidates on travel time, current load, direction of travel, and
  reliability, then assign. Distance is the filter; the score is the ranker.
  They are different jobs and live in different code.

Concrete example: three free riders sit within 1.2 km of the Powai store. The
match returns all three. The ranker prefers the one whose last drop was on the
same road as Riya's building, because real travel time, not straight-line
distance, is what the 10-minute clock measures.

### The scale story at three tiers

- 1,000 orders a day, one city, a handful of stores. A single Postgres row
  per SKU with `SELECT ... FOR UPDATE` is plenty. Point-in-polygon can even
  loop over all polygons. Nothing is hot. You could run this on one box.
- 100,000 orders a day, dozens of stores, evening peaks. Now contention
  bites. Many people hit the same popular SKUs (milk, eggs, bread) in the same
  store in the same minute, and a row lock on that one row serializes them and
  adds latency. What breaks: the hot row, and per-request polygon scans during
  the 8pm rush. What you do: move the hot counters into an in-memory atomic
  store (Redis/Tile38), put a spatial index in front of store selection,
  cache each store's catalog so most app-opens are reads, and add read
  replicas so browsing never touches the write path.
- 10 million plus orders, hundreds of stores nationwide, festival and sale
  spikes. The single in-memory node for a blockbuster SKU becomes the
  bottleneck, and one geospatial node cannot hold every rider in the country.
  What breaks: the single hot key and the single index. What you do:
  shard. Inventory is naturally sharded by store already (`stock:powai:*`
  lives on a different shard from `stock:koramangala:*`), so load spreads
  across cities for free. For a single store's contested SKU during a sale,
  split one counter into N sub-counters (`stock:powai:milk:0..9`) and have
  each order decrement a random shard, which removes the single-key
  bottleneck (the published flash-sale sharding trick). Geospatial state is
  sharded by region so the Mumbai index holds only Mumbai riders. Tile38
  leader/follower replication gives read scaling and failover. The order
  pipeline runs through queues so a spike becomes a slightly longer line, not
  a dropped order.

A second scale axis is the catalog itself. Demand forecasting decides what
each store stocks, since a 3,000-square-foot store cannot carry everything.
Secondary sources describe time-series models (ARIMA, Prophet, LSTM) on
historical, pincode-level order data plus a Min-Max replenishment rule: every
SKU has a min that triggers a refill and a max that caps overstock so produce
does not rot. Concrete example: the Powai store learns that milk and eggs
spike on weekday mornings and stocks deeper before 7am, while a slow-moving
SKU sits near its min. This is the offline-heavy, online-cheap pattern again:
the expensive forecasting runs in batch overnight, the live order path just
reads the resulting stock levels.

## 8. The retention and habit mechanic

The loop is "tiny need, instant fix, zero friction, repeat." Each time Riya
gets milk in 8 minutes, the app earns the right to be her default for the
next small need. Quick commerce deliberately expands the set of moments worth
opening the app: not just groceries but a phone charger, a paracetamol strip,
ice cream at 11pm. Every new "I can get that in 10 minutes" moment is another
hook.

Which metric it moves: retention and frequency, and through them revenue.
Quick commerce lives or dies on orders per user per month, not on order size.
The 10-minute promise plus never overselling is what converts a curious first
order into a weekly habit. A single post-payment cancellation at 9:40pm,
when Riya needed the onion now, can end the habit permanently, which is why
the atomic-inventory engineering in Problem B is not a backend nicety. It is
the retention feature.

Real observed mechanic: Zepto's "reorder your essentials" and saved-cart
nudges turn the repeat purchase (the same Amul Taaza every week) into a
two-tap action, and category tiles rotate to surface the night-time use cases
(ice cream, snacks) that manufacture new occasions. The habit is built by
making the next order require almost no thought.

## 9. The lesson for Rare.lab

The transferable idea is reservation under contention with an atomic
check-and-decrement on a single owner, and it maps directly onto a shader and
visual-effects runtime.

Rare.lab compiles node graphs to shippable code and ships an embeddable
runtime. The contested resource there is not milk, it is finite GPU budget:
texture memory, draw-call count, render passes per frame, compute slots. When
many effects on the same page or scene each want resources in the same frame,
you have exactly Zepto's oversell problem. Two effects both "see" enough VRAM,
both allocate, and you blow the budget, which on a GPU does not return a
polite "sold out." It drops frames or crashes the tab.

The concrete move: give each runtime instance a single in-memory resource
ledger (one owner, like the per-store stock counter) and make every allocation
an atomic check-and-decrement against it, not a check-then-allocate. Before an
effect runs, it atomically reserves its texture and draw-call budget for the
frame; if the budget is exhausted, it degrades (lower resolution, skip a pass)
instead of over-allocating, the way a sold-out SKU is shown before payment
rather than cancelled after. And shard it the way Zepto shards by store: keep
the budget ledger per render context so two independent canvases never
contend on one global counter, which removes the hot-key bottleneck before it
exists. Decide the expensive part offline, exactly like demand forecasting:
the compiler, at build time, can pre-compute each node graph's worst-case
resource cost so the runtime's per-frame decision is a cheap lookup and an
atomic decrement, not a live solve. Expensive thinking offline, cheap atomic
reservation on the hot path. That is the whole 10-minute trick, applied to a
frame budget.

---

## Sources

- Tile38, real-time geospatial and geofencing (in-memory, Redis RESP,
  NEARBY/WITHIN/INTERSECTS, leader/follower): https://github.com/tidwall/tile38
- Redis official tutorial, real-time inventory reservation with WATCH/MULTI
  and audit streams: https://redis.io/tutorials/inventory-reservation-in-real-time-with-redis/
- Redis INCR/DECR atomic counters: https://oneuptime.com/blog/post/2026-03-31-redis-incr-decr-atomic-counters/view
- Designing a flash-sale system that never oversells (Lua atomic
  check-and-decrement, inventory sharding, token list): https://medium.com/@umesh382.kushwaha/designing-a-flash-sale-system-that-never-oversells-from-1-user-to-1-million-users-without-8426db0f1ad0
- Flash sale system design (oversell, scale): https://singhajit.com/flash-sale-system-design/
- Shopify Engineering, scaling inventory reservations with MySQL SKIP LOCKED
  (2026): https://shopify.engineering/scaling-inventory-reservations
- Zepto delivery, the algorithmic brilliance behind 10-minute deliveries
  (reported Tile38 use, ARIMA/Prophet/LSTM forecasting, picking benchmarks):
  https://medium.com/@aryanakm01/zepto-delivery-the-algorithmic-brilliance-behind-10-minute-deliveries-daea75f0604f
- Deconstructed: Zepto's 10-minute delivery model (store sizes, median
  8m47s, 76s picking, Min-Max): https://www.42signals.com/blog/zepto-business-model-explained/
- Dark store management in quick commerce (real-time stock, 99% accuracy,
  oversell): https://kissflow.com/solutions/retail/dark-store-management-in-quick-commerce/
</content>
</invoke>
