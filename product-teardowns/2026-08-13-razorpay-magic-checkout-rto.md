# Razorpay Magic Checkout and the RTO risk engine underneath it

Date: 2026-08-13
Product: Razorpay
Feature: Magic Checkout (one-click checkout: network address prefill + real-time COD return risk scoring)

A note on scope. This is a specific feature teardown, not a Razorpay tour. We already
covered Razorpay Smart Routing (Optimizer) on 2026-06-26 and UPI payments on 2026-08-01.
Magic Checkout is a different animal: it is the checkout page itself, plus the machine
that decides whether a shopper is allowed to pay Cash on Delivery. The interesting half
is not the pretty form. It is the model that predicts, in the moment the shopper taps
"Place Order," whether that COD parcel will come back unopened.

---

## 1. The user

Meet Priya. It is 11pm in Kanpur. She is lying in bed, scrolling a Shopify store that
sells running shoes, and she has decided on a pair for Rs 1,299. She taps "Buy Now."

Now the old, boring part of the internet usually begins. A form. Full name. Flat number.
Street. Landmark. City. State. A six digit pincode she has to remember. Phone number.
Email. Then a second page for payment. Priya is doing this one-thumbed, half asleep, on
a mid-range Android phone on a patchy 4G connection.

On the other side of this transaction is the merchant, a small direct-to-consumer (D2C)
footwear brand run by four people. They desperately want Priya to finish. They also
quietly dread her choosing Cash on Delivery, because in India roughly one in three COD
parcels comes back to the warehouse unopened. When that happens the brand eats the
forward shipping, the return shipping, the packaging, and the dead inventory that was
locked up for a week. That single Rs 1,299 sale can turn into a Rs 200 loss.

Two people, one checkout, opposite fears. Priya fears the form. The merchant fears the
return. Magic Checkout is built to kill both fears at once.

---

## 2. The real problem

Told like a friend would tell it, there are two problems bolted together.

Problem one is friction. Every extra field on a checkout form is a small cliff, and a
fraction of shoppers fall off each one. On mobile, in India, on a first-time store,
abandonment at checkout is brutal. Priya types her address wrong, gets frustrated, and
closes the tab. The shoes were in the cart. The sale evaporated at the last screen.

Problem two is trust, and it points the other way. Cash on Delivery is not a payment. It
is a promise to pay later, at the door. In India COD still dominates because a lot of
shoppers do not trust an unknown store enough to pay upfront, or simply prefer paying
when the box is in their hands. But COD is where Return to Origin (RTO) lives. The
industry numbers are stark: COD orders in India are returned roughly 30 to 40 percent of
the time, while prepaid orders come back only 5 to 10 percent of the time (multiple
Indian D2C logistics guides converge on this range). A large share of those returns are
not buyer's remorse. They are wrong addresses, fake orders, serial "order and refuse"
behavior, and pincodes where the last-mile courier just cannot deliver.

So the merchant is caught. Remove COD and you lose the shoppers who only buy COD. Keep
COD for everyone and you bleed money on returns. The honest answer is not "COD on" or
"COD off." It is "COD for Priya, but not for the account that has refused four parcels
this month." That is a per-order decision, made live, and it is a prediction problem.

---

## 3. The feature in one sentence

Magic Checkout is a one-tap checkout that prefills a returning shopper's saved address
and payment details from a shared 100 million-shopper network, and, at the same moment,
scores that order's Return to Origin risk in real time to decide whether Cash on Delivery
should be offered, discouraged, or blocked.

---

## 4. Jobs to be done

What is Priya really hiring this feature to do?

- "Let me buy these shoes in under ten seconds without typing my address again."
- "Do not make me create yet another account on a store I may never visit again."
- "Let me pay the way I like (COD), on a store I do not fully trust yet."

What is the merchant hiring it to do?

- "Get more of my add-to-carts across the finish line."
- "Stop shipping COD parcels that I already know will bounce back."
- "Do not insult my good customers by blocking their COD; only stop the risky ones."
- "Give me back some money when a parcel I was told was safe still comes back."

The feature has to serve both at once, and the two jobs are in tension. Fewer fields is
easy. Fewer fields without shipping more junk COD orders is the hard part.

---

## 5. How it works for the user

From Priya's seat, the magic is that the form is mostly gone.

She taps "Buy Now." A checkout sheet slides up. It asks for one thing: her phone number.
She types it and gets a one-time password (OTP) by SMS. The instant she is verified, her
saved address (from some other store she bought from months ago, on the same network)
drops into place. Name, flat, street, pincode, all filled. Razorpay's own line is that
this takes the whole thing down to about 9 seconds.

Then payment. If Priya is a low-risk shopper, she sees COD sitting there as a normal
option alongside UPI and cards. She picks COD, taps "Place Order," done.

Now run the same flow for a different shopper, an account that has refused three parcels
in the last month, ordering to a pincode with a bad delivery history. That shopper opens
the same sheet, enters the same kind of details, and reaches payment. But COD is greyed
out, or missing, or it now asks for a small partial prepaid amount to confirm intent. The
shopper never sees a scary "you look like a fraud" message. The risky option just quietly
is not there. This is the merchant's RTO rule firing, invisibly.

The shopper's visible experience is "fast form, normal payment." The whole intelligence
is hidden in which payment options render.

---

## 6. The actual flow, step by step

Walking Priya's Rs 1,299 sneaker order, tap by tap:

1. Product page, Priya taps "Buy Now." The merchant's site calls Razorpay to open a
   Magic Checkout session for this cart (items, amount, merchant id).
2. The checkout sheet renders. It asks for phone number only.
3. Priya enters her number. Razorpay sends an OTP. She enters it. She is now an
   identified shopper on the Razorpay network.
4. Server side, two things fire in parallel the moment she is identified:
   - Address lookup: her profile in the shared address book is fetched by her phone
     identity, and her saved Kanpur address is returned to prefill the form.
   - Risk scoring: the order context (shopper identity, address, pincode, order value,
     device, and network history) is fed to the RTO risk model, which returns a risk
     score and a recommended action.
5. The payment options render according to that action. Low risk: COD shown normally.
   High risk: COD hidden, or partial-COD, or a COD fee added, per the merchant's
   configured rules.
6. Priya picks COD, taps "Place Order." The order is confirmed to the merchant.
7. If the merchant has RTO Protection enabled and later the parcel still bounces despite
   being cleared as low risk, Razorpay reimburses that verified loss.

The key detail: steps 4a and 4b happen at the same time, off one identity resolution. The
same phone number that unlocks the fast address also unlocks the shopper's cross-merchant
history that feeds the risk model. Convenience and risk are two reads off one key.

---

## 7. Under the hood, like the engineer

This is the heart of it. There are two machines. One is an address network. One is a risk
model. They share a key (the shopper's phone identity) but they are completely different
engineering problems.

### Machine A: the shared address network (a lookup, not a search)

This part is deliberately boring, which is the point. When Priya verifies her phone, the
system does an O(1) keyed lookup into a shopper profile store. Think of it as a giant hash
map:

    key   = a stable shopper identity derived from the verified phone number
    value = { saved addresses[], saved payment prefs, past order behavior summary }

No search, no ranking, no scan of millions of rows. The phone identity is the primary key,
you go straight to the bucket, you read the record. That is why it is fast enough to feel
instant even on Priya's patchy 4G: the expensive part (typing) is replaced by a single
memory-cheap read.

The clever bit is not the data structure, it is that the address book is shared across
merchants. Razorpay states the network holds 100 million-plus registered shoppers and
serves 8 million-plus merchants. Priya saved her address once, months ago, buying a phone
case from a totally different store. Tonight, on the sneaker store she has never visited,
that address is already there. This is a classic network effect: coverage and accuracy of
the autofill improve as more merchants join, because each new merchant contributes more
shoppers and more confirmed deliveries to the shared book. The first merchant on such a
network gets little. The eight-millionth gets a nearly-complete address book for free.

Fact vs inference: the 100M shopper and 8M merchant figures and the "9 second" checkout
are Razorpay's published claims. The exact storage engine (hash map on which database,
which cache tier) is not published; the O(1)-keyed-read description is the well-grounded
"this is how this class of problem is always solved" version, clearly labeled as
inference. You do not scan for a saved address, you key into it.

### Machine B: the RTO risk model (matching then ranking, again, but for trust)

This is the real engineering, and it has a real pedigree. Razorpay's RTO intelligence
grew out of Thirdwatch, an AI fraud-and-RTO startup Razorpay acquired in August 2019
(founded 2016 in Gurugram by Adarsh Jain and Shashank Agarwal). Thirdwatch's engine,
called Mitra, analyzed over 200 parameters per transaction in real time to produce a
trust score, using device fingerprinting, location profiles, user behavior analysis, IP
verification, and account profiling. That engine is now the risk brain inside Magic
Checkout.

Structurally this is the same two-half pattern the whole ledger keeps finding, but pointed
at trust instead of relevance:

- The "matching" half is feature assembly. For Priya's order you gather the signals: her
  cross-merchant order history (how many past orders, how many delivered, how many
  refused), the pincode's delivery track record, the address quality (is it complete,
  does it geocode, is it a real deliverable place or "near the big temple"), order value
  versus category norm, device and IP signals (is this device tied to many refused
  orders), and time-of-day patterns. Most of these are precomputed running aggregates,
  read O(1) from a feature store, not calculated live. "Refusals in the last 30 days for
  this account" is a counter you keep updated, not a query you run at checkout.

- The "ranking" half is the model turning those features into one number: a probability
  this parcel comes back. Razorpay calls it a proprietary AI/ML RTO risk model. The exact
  architecture is not public. The well-grounded inference (labeled as such) is that this
  class of tabular risk problem is classically solved with gradient boosted trees
  (XGBoost or LightGBM), because the signals are mixed tabular features with sharp
  nonlinear cutoffs ("this pincode past 40 percent RTO," "this account refused 3 of its
  last 4"), and boosted trees carve exactly those cutoffs well. A public Indian-D2C
  writeup of the same problem describes precisely this: an XGBoost model over pincode RTO
  history, customer order history, COD value versus category average, and device/session
  signals, with orders above a score threshold held back.

Two engineering realities shape this model, and they are worth naming because they show up
everywhere in the ledger:

1. The label arrives late and lopsided. You do not learn whether Priya's parcel came back
   for a week or more after checkout. So the training label ("did this order RTO") lags
   the prediction by days. And the classes are imbalanced: most orders deliver fine, RTO
   is the minority class. This is the same censored-label, class-imbalance shape we saw in
   the Stripe Radar teardown (2026-07-04). You handle it with delayed-label training
   pipelines and with class weighting or focal-style loss so the rare RTO cases are not
   drowned out.

2. The threshold is the product, not the model. The model outputs a probability. The
   business decision is where you cut it. Razorpay exposes this as merchant-configurable
   rules: below the low-risk line, COD shows normally; in the medium band, nudge to
   prepaid or add a COD fee or ask for partial COD; above the high line, block COD
   entirely. Same lever as Radar's block threshold: move it one way and you insult good
   customers (a genuine buyer in a "bad" pincode loses COD unfairly), move it the other
   and you ship junk. The dial lives with the merchant because the right cut differs by
   category and margin.

The payoff Razorpay publishes: dynamically disabling COD for high-RTO-risk shoppers cuts
RTO by up to 50 percent, and single-page Magic Checkout brands report 21 to 26 percent RTO
reduction alongside 12 to 17 percent conversion improvement. And where the model clears an
order as safe but it still bounces, RTO Protection reimburses the merchant, which is
Razorpay effectively putting money behind its own predictions.

### The offline-think, online-lookup spine (again)

The same shape as Discover Weekly, YouTube, and every ranking teardown here. The expensive
learning (train the RTO model on the network's full history, recompute per-pincode and
per-account aggregates) runs offline, on a schedule. The live checkout path is cheap: one
identity resolution, one O(1) address read, one O(1) feature-vector read, one forward pass
through the model, one comparison to a threshold. Priya's checkout does not train anything
and does not scan anything. It reads precomputed numbers and does a little arithmetic. That
is what keeps it inside a checkout latency budget on a phone on 4G.

### The scale story at three tiers

Tier one, 1,000 orders (a brand-new store, first week). The model barely matters. There is
almost no history for these specific shoppers on this specific store. This is the cold-start
problem, and it is exactly what the shared network solves: even on day one, the merchant
inherits the network's view of Priya's account and her pincode from the other 8 million
merchants. Without the network, a new store's RTO model would be useless for months. With
it, the store gets a warm model on order number one. The network is not a nice-to-have here,
it is what makes the feature work at small scale at all.

Tier two, 100,000 orders (a growing D2C brand). Now per-pincode and per-account aggregates
for this merchant are meaningful on their own. The bottleneck shifts from "no data" to
"keeping the aggregates fresh." Every delivered or refused parcel has to update counters
(account refusals, pincode RTO rate). You do not recompute these from scratch per checkout;
you maintain them as running aggregates in a feature store and update them as outcomes
stream in. Reads stay O(1); the work moves to the write/update path, which you can batch.

Tier three, 10 million-plus orders across the network, lakhs of concurrent shoppers on a
sale day. Now two things break. First, the feature store is read on every checkout at high
QPS, so it must be sharded and cache-fronted; shard by shopper identity so each shopper's
history is self-routing (the same partition-by-key trick as the Notion and Stripe
teardowns). Second, a few hot entities dominate: a viral product, a single high-traffic
pincode, a popular device model. Those become hot keys on the read path, solved by caching
the hot lookups and, if a counter itself is contended, splitting it into sub-counters and
summing (the same hot-key fix from the Zepto and Instagram teardowns). Model inference
scales horizontally because each prediction is independent; you fan out stateless scoring
workers behind the checkout API. The training job is the one genuinely whole-corpus piece,
and it stays offline and periodic, never on the hot path.

The single most important scaling idea: the candidate the model scores is always exactly
one order. Unlike search, there is no candidate set of thousands to rank. So the live cost
is constant per checkout regardless of how big the network gets. The network getting bigger
makes the prediction better (more history), not slower. That is the ideal scaling curve.

---

## 8. The retention and habit mechanic

This feature runs two loops, one per side of the marketplace, and they reinforce each
other. It is a flywheel, not an engagement loop.

Shopper side: once Priya's address and preferences live in the network, every future Magic
Checkout store is a one-tap buy for her. The switching cost is not lock-in, it is pure
convenience compounding. The second store she buys from is easier than the first. By the
tenth, typing an address on a non-Magic store feels annoying. Her behavior trains her to
prefer stores that "just know her." This moves conversion (activation of a purchase) and,
over time, repeat purchase (retention).

Merchant side: the RTO model gets better the longer the merchant is on the network and the
more merchants join, so leaving means going back to shipping junk COD blind. The financial
proof (up to 50 percent less RTO, double-digit conversion lift) is a revenue and margin
mover, which is the stickiest kind of retention for a business buyer: you do not churn off
the thing that is visibly protecting your P&L.

The metrics moved: conversion (activation) via the fast form, RTO reduction (revenue and
margin protection) via the risk model, and network retention on both sides via the shared
address book. The clever part is that the same shared network powers both the convenience
loop and the risk loop, so every new shopper and every new merchant makes both loops
stronger at once. Real observed proof point: Razorpay's own footwear-brand case study and
its published 12 to 17 percent conversion and 21 to 26 percent RTO numbers are the loop
showing up in the P&L, not a hypothetical.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code,
plus an embeddable runtime. The lesson from Magic Checkout is about where you spend
computation and how a shared network makes a per-instance decision cheap.

Concrete lesson: precompute a shared "shader risk/cost profile" offline, keyed by node
graph signature, and read it O(1) at author time to make the expensive decision instantly.

Here is the mapping. Magic Checkout's insight is that the live decision (offer COD or not)
is a single cheap lookup plus a threshold, because all the heavy learning happened offline
over a shared network. Rare.lab has the same shape hiding in it. When a user wires up a
shader graph in the editor, the expensive question is "will this compile to something that
runs at 60fps on a mid-range phone GPU, or will it blow the fragment shader budget?" You do
not want to compile-and-profile on every node edit; that is the equivalent of running the
full RTO model synchronously on every keystroke.

So do what Razorpay did. Offline, across every graph every user in your network has ever
compiled and shipped, learn a cost model: hash each node and subgraph pattern to a
signature, and store its measured GPU cost (instruction count, texture reads, register
pressure, real measured frame time on reference hardware) as a precomputed aggregate in a
feature store. Then at author time, when the user drops a noise node into a loop, you do an
O(1) lookup of that subgraph's signature and instantly render a live "this will cost ~4ms on
a Pixel 6a, you are near budget" badge, exactly the way Magic Checkout instantly renders
"COD not available." No live compile, no live profile, just a keyed read of a number learned
offline over the whole network.

And take the network effect too: the first user's cost estimates are rough, but as more
users compile and ship graphs through Rare.lab, your shared cost profile gets denser and
your author-time predictions get sharper for everyone, including a brand-new user's first
graph (cold start solved by the network, just like a new merchant's first order). Bias to
performance means the badge has to be instant, and instant means the thinking is offline and
the author-time path is a lookup plus a threshold, never a computation.

One more borrowed idea: make the threshold a creator-facing dial, not a hardcoded constant.
Just as merchants set their own COD-risk cutoffs by category and margin, let a Rare.lab user
set their performance budget (target device, target frame time) so the same cost model warns
aggressively for a mobile web embed and loosely for a desktop demo. The model is shared; the
cut is theirs.

---

## Sources

- Razorpay, "Razorpay One-Click Checkout 2.0 | Magic Checkout" (product/engineering blog):
  https://razorpay.com/blog/razorpay-one-click-checkout-2-0-magic/
- Razorpay, "Magic Checkout's New Single-Page Checkout" (conversion and RTO numbers):
  https://razorpay.com/blog/magic-checkouts-new-single-page-checkout/
- Razorpay Docs, "RTO Overview | Magic Checkout":
  https://razorpay.com/docs/payments/magic-checkout/rto-analytics/overview/
- Razorpay Docs, "RTO Insights | Magic Checkout":
  https://razorpay.com/docs/payments/magic-checkout/rto-analytics/rto-insights/
- Razorpay, "Introducing RTO Analytics Dashboard":
  https://razorpay.com/blog/introducing-rto-analytics-dashboard/
- Razorpay, "Razorpay MagicX for Shopify Plus: Boost Conversions, Slash RTOs" (50% RTO,
  proprietary AI/ML model): https://razorpay.com/blog/razorpay-magicx-for-shopify-plus/
- Razorpay, "Thirdwatch has Merged with Magic Checkout":
  https://razorpay.com/blog/thirdwatch-has-merged-with-magic-checkout/
- Razorpay, "Razorpay Announces its First Acquisition - Thirdwatch" (Mitra, 200+
  parameters, real-time trust score, device fingerprinting):
  https://razorpay.com/blog/thirdwatch-acquisition-rto-fraud-ecommerce/
- Razorpay Tech, "Using Machine Learning to Detect Fraud: Introduction (Thirdwatch)":
  https://razorpay.com/blog/detect-fraud-using-ml-ai-thirdwatch/
- The Paypers, "Razorpay buys real-time fraud prevention platform Thirdwatch":
  https://thepaypers.com/mergers-aquisitions-and-investments/news/razorpay-buys-real-time-fraud-prevention-platform-thirdwatch
- HillTeck, "How to Reduce RTO in Ecommerce: The Complete Guide for Indian D2C Brands
  (2026)" (COD 30-40% vs prepaid 5-10% RTO, pincode risk):
  https://www.hillteck.com/blog/reduce-rto-ecommerce-india.html
- bePragma, "Using Data to Identify Pincode Clusters with High RTO Risk":
  https://www.bepragma.ai/blogs/using-data-to-identify-pincode-clusters-with-high-rto-risk
</content>
</invoke>
