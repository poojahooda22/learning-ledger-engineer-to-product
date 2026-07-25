# Amazon delivery date promise (the "Get it by Tuesday" line and the countdown timer)

Date: 2026-07-25
Product: Amazon
Feature: The delivery date promise on the product page (the green "Get it by" date, the "Order within 1 hr 13 mins" countdown, and the Promised Delivery Date behind them)

---

## 1. The user

Priya is standing in her Bengaluru kitchen at 11:47 on a Tuesday morning. The
electric kettle died this morning mid-chai. She opens the Amazon app on her
phone, types "electric kettle," and taps the Prestige 1.5L listing. She is not
reading the product description. Her eyes go straight to one line in green:

"Get it by 10 PM today. Order within 1 hr 13 mins."

That single line decides whether she buys. She has people coming over at 7 PM
and she wants tea. If the line said "Arriving Thursday," she would close the app
and drive to the shop down the road. Because it says "today, before 10 PM," she
taps Buy.

She is not a logistics person. She does not know or care that a warehouse in
Soukya Road has one kettle left, that a delivery station in Whitefield has a van
slot open, or that a model somewhere just decided her order can make it. She
sees one promise and a ticking clock.

## 2. The real problem

Here is the honest version, the way you would explain it to a friend.

Online you cannot touch the thing you are buying, so the one question in your
head is "when will it actually show up." A shop answers that by handing you the
item. Amazon has to answer it with a sentence written before anyone has picked,
packed, or driven anything.

And it is a promise, not a guess. If Amazon says "today by 10 PM" and the kettle
shows up Thursday, Priya does not shrug. She feels lied to. She trusts the next
promise less. She buys less. On the flip side, if the kettle could genuinely
arrive today but the page timidly says "Thursday," she walks away and Amazon
loses a sale it could have won.

So the feature lives on a knife edge. Say a date too early and you break trust
when you miss it. Say a date too late and you lose the sale to the shop down the
road. The two mistakes pull in opposite directions, and Amazon has to pick a
date for hundreds of millions of item-and-address combinations, live, in the
time it takes a page to load.

## 3. The feature in one sentence

The delivery promise is a per-item, per-address, per-moment prediction of the
soonest date Amazon is confident it can actually deliver, shown as a firm
"Get it by" date with a countdown telling you how long the current answer holds.

## 4. Jobs to be done

What is Priya really hiring this line to do?

- "Tell me if I get tea tonight or not." A yes/no on her actual need, today.
- "Give me a date you will keep, so I do not have to keep checking." Certainty,
  not optimism.
- "Let me compare this seller to that seller by speed, not just price." The date
  is a sortable feature of the product, like price or rating.
- "Tell me how long I have to decide." The countdown removes the "I will buy it
  later" that quietly kills the sale.

Note that none of these jobs is "predict the average delivery time." Priya does
not want the average. She wants a date Amazon will hit. That distinction is the
whole engineering story below.

## 5. How it works for the user

The visible experience is almost nothing, which is the point.

- On the product page, under the price, a green line: "FREE delivery Tuesday,
  29 July" or "Get it by 10 PM today."
- Right next to it, when a faster option has a deadline, a live countdown:
  "Order within 1 hr 13 mins." The number ticks down in real time.
- Change your pincode (Priya taps "Deliver to 560066") and the date and the
  countdown both change instantly, because the answer depends on where she is.
- Add Prime and a faster, often free, date appears. Remove it and the date slips.
- At checkout the promise is restated as the Promised Delivery Date. That is the
  date Amazon will measure itself (and the seller) against later.

That last point matters. The green line is not marketing copy. It is a
commitment the company scores itself on. Amazon calls the internal number the
Promised Delivery Date (PDD), and a seller's On-Time Delivery Rate (OTDR) is
measured against it. The line Priya reads and the metric the operation is graded
on are the same number.

## 6. The actual flow, step by step

1. Priya opens the kettle listing. The app sends the product id (the ASIN),
   her pincode (560066), her Prime status, and the current time to Amazon.
2. Amazon figures out which warehouses actually have this kettle in stock and
   can serve Whitefield. Say two: a large fulfillment center on Soukya Road and
   a smaller same-day site closer in.
3. For each of those source-to-Priya lanes, Amazon looks up how long the whole
   chain takes: time left to pick and pack today, the cutoff for the outbound
   truck, the drive time to her area, the local delivery window.
4. It turns each lane into a date, and not a single number but a spread: "this
   lane usually lands same day, but 1 time in 20 it slips to tomorrow."
5. It picks the soonest date it is confident enough to promise, at a chosen
   safety level. Same-day by 10 PM wins.
6. It computes the order cutoff: the latest moment Priya can tap Buy and still
   make that promise. Right now that is 1 PM, so it renders "Order within
   1 hr 13 mins" and starts the clock.
7. Priya taps Buy at 11:52. The promise is locked as her PDD. The same-day slot
   she just consumed is decremented, so the next shopper in Whitefield may see a
   slightly later date if the slots are nearly full.
8. Later, the actual delivery scan is compared to the PDD. On time or not on
   time. That truth feeds tomorrow's model.

The key thing: steps 2 through 6 feel instant because almost all the heavy
thinking already happened offline, before Priya ever opened the app. More on
that now.

## 7. Under the hood, like the engineer

This is the heart of it. The promise is two problems wearing one green line,
and the ledger has seen both shapes before.

### It is match, then rank. Again.

Same split as Amazon's own search page, just with warehouses instead of
listings.

Matching (which lanes are even possible). Out of 350-plus US fulfillment
centers and 1,000-plus logistics facilities worldwide, only a handful hold this
exact kettle AND can legally and physically serve pincode 560066 today. You do
not loop over every warehouse. You keep an index: for each item, the set of
nodes that stock it; for each destination zip, the set of nodes that serve it.
Intersect the two and you get the candidate lanes, cheap, in the size of the
answer, not the size of the network. This is the same inverted-index move as
posting lists in search: turn "search everything" into "look up a key."

Ranking (which possible date to actually promise). Now you have, say, four
feasible lanes, each producing a delivery-time estimate. You pick one date. This
is where the real modeling lives, and it is not "pick the fastest." It is "pick
the fastest date I am confident I can keep."

### The transit time is a distribution, not a number

A single lane, Soukya Road to Whitefield, does not take "6 hours." It takes 5
hours most days, 7 on a rainy Friday, 9 once a month when a van breaks down.
The honest object is a probability distribution over arrival times, not a scalar.

Amazon models this zip-to-zip. Predictions are specific to the exact origin and
destination pair, learned from billions of historical shipments on that lane and
lanes like it (fact, per Amazon's published delivery-day-prediction descriptions
and the AWS write-up on predicting shipment delivery time). Think of a giant hash
map keyed by (origin zip, destination zip, service level, time of day, day of
week) whose value is a learned distribution of transit times, plus features for
weather, carrier, and current network load.

### Quantile regression: promising the p90, not the mean

Here is the single most important idea in the whole feature.

If you promise the average delivery time, you are late half the time by
definition. Half of all promises broken is a disaster. So Amazon does not
predict the mean. It predicts a high quantile of the distribution, say the 90th
or 95th percentile, and promises that.

Promise the p90 and you keep the promise 9 times out of 10. Promise the p95 and
you keep it 19 times out of 20, at the cost of quoting a slightly later date.

This is done with quantile regression and its relatives: quantile regression
forests, gradient boosted trees, XGBoost, neural networks, or an ensemble of
these (fact, named in Amazon's delivery-promise-optimization patent material and
the academic work on the problem). The reason quantile methods win here is that
the product decision is asymmetric, and only a quantile loss lets you encode
that asymmetry:

- Being late (over-promising) costs a lot: broken trust, refunds, a seller's
  OTDR drops, future sales fall.
- Being needlessly slow (under-promising) also costs: a lost sale today.

A standard model minimizing average error treats a 1-day-early miss and a
1-day-late miss as equally bad. They are not. Quantile loss (pinball loss)
weights the two sides differently, so the chosen quantile q becomes the product
dial: crank q toward 1.0 to almost never be late (conservative dates), lower it
to quote faster dates and accept more misses. Amazon's own research frames it
exactly this way: lowering a 24-hour delivery promise from 95 percent
reliability to 94 percent unlocks a large reduction in the inventory you must
hold to back it, a trade worth making on purpose (fact, Amazon Science work on
learning quantile functions). One percentage point of promise is real money in
warehouses.

This is the same lesson the ledger already logged for Zomato food-prep time
(quantile loss makes q the dial between an idle rider and cold food) and for
Uber ETA (asymmetric Huber loss leans against under-promising). Delivery date is
that idea stretched from minutes to days, across a whole continent of lanes.

### Learn the whole curve at once

A refinement from Amazon Science: instead of training a separate model for the
p90 and another for the p95 and another for the p50, learn the entire quantile
function in one shot (a spline-based quantile function, inference on exact
production use, but published as Amazon research). Why: you often want to read
several points off the same curve (show the customer a p90 date, but let the
operations team reason about the p50 for planning), and separately-trained
quantile models can cross over each other in nonsense ways (a "p90" date earlier
than the "p50"). One learned curve is cheaper to serve and cannot self-
contradict.

### Where the sorting happens, and when the thinking happens

The phone sorts nothing. The phone renders a date. Everything, the candidate
lanes, the transit distributions, the quantile models, is computed server-side,
and almost all of it is computed offline, ahead of time.

This is the ledger's spine one more time: offline-think, online-lookup. The
expensive parts (training the zip-to-zip quantile models on billions of
shipments, precomputing transit distributions per lane, precomputing per-node
capacity for the day) run in batch. The live page render is close to a keyed
lookup: take (item, destination zip, Prime, now), fetch the precomputed
candidate lanes and their promise dates, apply the current remaining capacity,
pick the winner, emit the date and the cutoff (inference on the exact serving
architecture, but this is the only shape that survives the latency budget).

### The countdown timer is a cutoff calculation, not a sales gimmick

"Order within 1 hr 13 mins" is not a fake-urgency trick. It is the real answer
to "how late can you tap Buy and still make the promised date." Work backward
from the promise: to arrive by 10 PM today, the van must leave the delivery
station by, say, 3 PM; to make that van, the item must be picked and packed by
1 PM; so the order cutoff is 1 PM, and at 11:47 that is 1 hr 13 mins away. When
the clock hits zero the promise flips to the next feasible date. Amazon says the
timer also accounts for the fact that the date can become unavailable before you
order, because inventory or delivery capacity changed under you (fact, Amazon's
own same-day delivery help text). The clock is honest about a slot that is
genuinely draining.

### The scale story at three tiers

Tier 1, about 1,000 items, one city, one warehouse. You barely need a model. A
lookup table of "this warehouse, this pincode, ships in 1 day" is fine. A
nightly average of past deliveries per lane covers you. Nothing breaks. This is
why a small D2C store can hardcode "delivered in 3 to 5 days" and be done. The
real problem is invisible at this size, which is exactly why small operations
never discover it.

Tier 2, about 100,000 items, several warehouses, a region. Now two things break.
First, the average lies: some lanes are reliable, some are not, and a single
"3 to 5 days" either over-promises the bad lanes or under-promises the good ones.
You switch from an average to a per-lane distribution and start promising a
quantile. Second, capacity is now contested: the same-day van has a finite number
of slots, and two shoppers can race for the last one. You cannot let the promise
be a stale read. You start tracking remaining capacity per node per day and
decrementing it as orders land, which is the classic hot-resource problem the
ledger logged for Zepto's last carton of milk and Stripe's balance: a single
counter everyone wants to write.

Tier 3, 10 million-plus items, the whole country, a network that ships 20 to 25
million packages a day (fact, 2025 estimates), 9 billion-plus items delivered
same-day or next-day in 2024 (fact). Now the number of (item, destination zip,
moment) combinations is effectively unbounded, and you cannot compute a promise
from scratch on the hot path. Survival moves, all of which the ledger has seen:

- Precompute and cache. Transit distributions and per-lane promise dates are
  built offline and cached; the live request is a lookup plus a light capacity
  adjustment. Head lanes (the busy zips) stay hot in cache.
- Shard by geography. The promise for a Bengaluru pincode is computed against
  Bengaluru-area nodes; a Delhi request never touches it. Add a city, add a
  shard, not central load. Same partition logic as Swiggy sharding by city.
- Split the hot capacity counter. The last-slot problem on a popular same-day
  node under a sale is the celebrity counter again. You reserve-then-confirm the
  slot atomically and split one hot counter into sub-counters so a Prime Day
  rush does not serialize on a single row (inference, but it is the standard fix
  and the exact pattern from the Zepto and Razorpay teardowns).
- Keep the model offline. Training on billions of shipments, rebuilding the
  zip-to-zip curves, and re-forecasting node capacity all run in batch. The
  online path never trains.

What SPEEDY adds. For the hardest tier, sub-same-day (promise a narrow window
today, not just a date), Amazon published SPEEDY: it builds multiple different
"views" of the order-to-delivery data and fuses them with a deep view
interaction network, letting Amazon promise a narrower, faster slot for more
than 60 percent of eligible orders (fact, Amazon Science). Narrower promises are
worth more to the customer and harder to keep, so you only earn them with a
better model, not by loosening the quantile.

## 8. The retention and habit mechanic

The immediate metric this feature moves is conversion, which is revenue. The
green "Get it by today" line plus the ticking countdown is one of the strongest
buy-now levers Amazon has. It answers Priya's real question (tea tonight or not)
and then puts a deadline on the answer. The countdown converts "I will decide
later" into "I decide now," because later the date might slip. That is honest
scarcity, and it works.

The longer loop is trust compounding into Prime. Every promise Amazon keeps
trains Priya to believe the next one. Once she believes the promise, she stops
comparison-shopping on delivery speed, because she has learned that Amazon's date
is real and the shop down the road is a gamble. That learned reflex is exactly
what a Prime membership monetizes: pay once a year, and every "Get it by
tomorrow" line becomes free and fast. The promise engine is what makes the Prime
subscription feel worth it on every single order. The 9 billion-plus same-day
and next-day items in 2024 are the loop running at full speed: fast reliable
promise, kept, so you come back, so you subscribe, so you buy more.

The mechanic is defensive as much as it is growth. A missed promise is a trust
withdrawal, and trust is the switching cost that keeps Priya out of the
competitor's app. This is the same shape the ledger logged for Stripe (reliable
payouts as an artery you do not switch) and Netflix Open Connect (speed as a
switching cost): the feature retains by being boringly reliable, not by being
flashy.

## 9. The lesson for Rare.lab

Do not promise a single number. Promise a quantile, and expose the quantile as a
product dial.

Rare.lab compiles node graphs into shippable shader code and ships an embeddable
runtime. The user's real question is Priya's question in disguise: "will this
effect hold 60fps on my players' phones." The temptation is to answer with one
number, an average or a single benchmark on your dev machine. That is the "promise
the mean" mistake, and you will be late (a dropped frame) on half the devices in
the wild.

Concretely, borrow the whole promise engine:

- Model frame time as a distribution per device-capability bucket, not a scalar.
  Build the buckets offline (GPU family, memory, feature set), the same
  capability-geohash idea from the Swiggy and Instagram lessons. For each
  (effect, bucket) learn the distribution of per-frame GPU cost from real
  measurements, not one dev-machine sample.
- Promise the p95, not the mean. When the editor tells a creator "this runs at
  60fps on mid-tier Android," that badge should mean the p95 frame time fits the
  budget, so 19 devices in 20 actually hold the frame. Use a quantile
  (pinball) loss when you train the cost predictor, because the cost is
  asymmetric exactly like Amazon's: a dropped frame (over-promised complexity)
  destroys trust far more than shipping a slightly simpler effect
  (under-promised). Bias the dial toward safety.
- Make the quantile a knob the creator sets. "Aggressive" (p80, prettier, riskier)
  versus "Safe" (p99, plainer, rock-solid), the same q-dial Amazon turns between
  faster dates and fewer misses. Different games want different points on that curve.
- Precompute the promise, look it up at runtime. Do the expensive compile-and-
  measure offline per (effect, device bucket) and cache the verdict, so the
  runtime "can this device afford this effect right now" check is a keyed lookup,
  not a live benchmark on the player's phone. Offline-think, online-lookup.
- Treat the per-frame GPU budget as contested capacity. When several effects run
  at once, reserve-then-commit each one's slice of the frame's millisecond budget
  atomically, and when the frame is nearly full, quote the next effect a lower
  quality tier, exactly like Amazon draining a same-day slot and quoting the next
  shopper a later date. The frame budget is your last carton of milk.

The one-line version: predict a distribution, promise a safe quantile of it,
precompute the promise offline, and treat the frame budget like a slot that can
sell out. A promise you keep 19 times out of 20 builds the trust that makes
people ship on your runtime, the same way a kept "Get it by today" is what turns
a shopper into a Prime member.

---

## Sources

- Amazon Science, SPEEDY: Framework for sharpening promise time estimates in sub-same day delivery: https://www.amazon.science/publications/speedy-framework-for-sharpening-promise-time-estimates-in-sub-same-day-delivery
- Amazon Science, Improving forecasting by learning quantile functions: https://www.amazon.science/blog/improving-forecasting-by-learning-quantile-functions
- Quantile Regression for Delivery Promise Optimization (paper): https://www.academia.edu/124686682/QUANTILE_REGRESSION_FOR_DELIVERY_PROMISE_OPTIMIZATION
- INFORMS M&SOM, Real-Time Delivery Time Forecasting and Promising in Online Retailing: When Will Your Package Arrive?: https://pubsonline.informs.org/doi/10.1287/msom.2022.1081
- AWS Blog, How to Predict Shipments' Time of Delivery with Cloud-based Machine Learning Models: https://aws.amazon.com/blogs/industries/how-to-predict-shipments-time-of-delivery-with-cloud-based-machine-learning-models/
- Veeqo, Delivery Day Prediction on Amazon: What It Is and How It Works: https://www.veeqo.com/blog/delivery-day-prediction-on-amazon-what-it-is-how-it-works-and-why-it-matters
- Amazon Customer Service, Order with Prime FREE Same-Day Delivery (countdown and cutoff): https://www.amazon.com/gp/help/customer/display.html?nodeId=GFUT24ALVAVC6VFD
- US Patent 11,334,845, System and method for generating notification of an order delivery: https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/11334845
- Capital One Shopping, Amazon Logistics Statistics 2026 (daily package volume, network scale): https://capitaloneshopping.com/research/amazon-logistics-statistics/
