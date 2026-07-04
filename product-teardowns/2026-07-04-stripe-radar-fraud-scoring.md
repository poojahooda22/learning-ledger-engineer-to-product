# Stripe Radar: scoring a payment for fraud in the 100 milliseconds before it clears

**Date:** 2026-07-04
**Product:** Stripe (Radar)
**Feature:** Real-time card-fraud risk scoring during a charge

---

## 1. The user

It is a Tuesday afternoon and Meera runs a small Shopify store that sells handmade
leather journals out of Jaipur. She does not think about payments at all. She
thinks about leather and about the parcel that has to reach Bangalore by Friday.
Stripe is the invisible thing between "customer taps Pay" and "money shows up."

Right now, three things are happening to Meera's store in the same second, and she
sees none of them.

A real customer, Arjun, is buying one journal for 1,200 rupees from his phone in
Bangalore. His card is his own. His payment should go through instantly.

At the same time, somewhere else, a script is hitting Meera's checkout with a list
of 4,000 stolen card numbers, charging 20 rupees each, just to find out which
cards are still alive. This is called card testing. Meera has never heard the term.
If it works, her Stripe account eats thousands of disputes next month, her fees go
up, and her processor may flag her as high risk.

And a third customer, Priya, is buying two journals for a gift. Her billing address
does not match her shipping address, and she is on a hotel wifi in another city. To
a dumb rule she looks a little like fraud. She is not. She is just traveling.

Meera needs all three handled correctly in the moment, with no fraud team, no
analyst, and no delay that makes Arjun or Priya give up and close the tab.

## 2. The real problem

Fraud is not one problem. It is a judgment call made against the clock, on almost
no information, where both kinds of mistake cost real money.

If Stripe waves through Arjun and the card testers alike, Meera pays. A disputed
charge (a chargeback) is not just the lost sale. It is the goods gone, a dispute
fee, and, if it happens enough, a card-network penalty program that can get a
business shut off. Fraud that gets through is a wound that bleeds for weeks,
because the "this was fraud" signal (the dispute) often arrives 30, 60, even 90
days after the payment.

But if Stripe gets nervous and blocks Priya because her addresses do not match,
that is also a real loss, and a quieter, meaner one. Meera never sees the sale that
did not happen. Priya just felt insulted and bought the journal somewhere else.
Every honest customer you block to stop one fraudster is a customer you paid to
acquire and then threw away at the last step.

So the real pain is a tightrope. Catch more fraud and you also block more good
customers. Block fewer good customers and you also let more fraud through. The two
errors pull against each other, and the right balance is different for Meera's
journal shop than it is for a company selling 90,000-rupee laptops.

And all of this has to be decided while Arjun is still looking at the spinner. A
card authorization does not wait. If the answer does not come back in a fraction of
a second, the whole checkout feels broken. The friend-level version: it is a
bouncer at a club who has to decide "in or out" for every single person in under a
blink, is fired if he lets in trouble, and is also fired if he turns away paying
regulars, and only finds out he was wrong two months later.

## 3. The feature in one sentence

Radar looks at every payment the instant it happens, scores how likely it is to be
fraud using patterns learned across millions of businesses, and blocks the risky
ones, all inside the roughly 100 milliseconds before the charge goes through.

## 4. Jobs to be done

What is Meera really hiring Radar to do?

- **Stop the bleeding she cannot see.** Kill the card-testing script and the stolen
  cards before they become next month's chargebacks.
- **Do not cost her good sales.** Let Arjun and Priya through. A blocked real
  customer is worse than invisible; it is a paying customer she insulted.
- **Require zero work.** Meera has no fraud analyst. The default has to be right out
  of the box, tuned by a network she is not smart enough (and should not have to be)
  to tune herself.
- **Be explainable when it matters.** When Radar does block something, or when Meera
  wants stricter rules for a big-ticket order, she needs to see why and adjust it.

Arjun and Priya are hiring it for the opposite thing: to be invisible. They never
want to meet it. The best outcome for a good customer is that Radar ran, decided
"fine," and got out of the way in a few milliseconds.

## 5. How it works for the user

From the outside, nothing happens, and that is the point.

Arjun taps Pay. The spinner turns for well under a second. He gets his receipt.
Radar scored him, decided he was low risk, and he never knew a fraud system
existed. That is the invisible 99 percent case.

The card-testing script gets a different experience. The first few tiny charges may
squeak through, but as the pattern becomes obvious (same store, hundreds of
attempts, many different cards, tiny amounts, failing address checks), Radar's risk
score for each new attempt climbs and Stripe starts returning declines. The attack
stops being profitable and moves on.

Priya, the traveler, is the hard middle. A good system lets her through on the
strength of everything else that looks normal (a real device, a card that has a
history, an amount that fits the store), even though one signal (address mismatch)
looks off. If a business has turned on stricter handling, a borderline payment like
hers can instead be sent to a review queue or challenged with an extra
verification step rather than silently killed.

Meera, the owner, sees Radar only in her dashboard: a risk score from 0 to 99 on
each payment, a reason for each block, and a rules screen where she can write things
like "review any payment over 50,000 rupees where the card country and IP country
disagree." She did not build the brain. She just nudges it.

## 6. The actual flow, step by step

Walk Arjun's single 1,200-rupee payment through the machine.

1. Arjun taps Pay in Meera's checkout. His browser sends the card details straight
   to Stripe (they never touch Meera's server), along with a bundle of context:
   device fingerprint, browser, IP address, the billing address he typed.
2. Stripe creates the payment and, before asking the bank for money, hands the whole
   thing to Radar.
3. Radar gathers the signals. Some come off this request directly (amount, card BIN,
   IP, email, device). Many more are looked up: has this card been seen anywhere on
   the Stripe network before, and how did those payments end? How many distinct cards
   has this IP touched in the last hour? How does this amount compare to Meera's
   normal order? These are precomputed running tallies, fetched fast, not calculated
   from scratch.
4. All of those signals become a row of numbers (a feature vector) and go into the
   model. The model returns one number: a risk score.
5. That score meets the rules. Stripe's built-in policy blocks the clearly-fraudulent
   band; Meera's own rules can block, allow, or send-to-review on top of that. Arjun
   scores low. Verdict: allow.
6. Only now does Stripe ask Arjun's bank to authorize the 1,200 rupees. The bank says
   yes. Arjun gets his receipt.
7. Weeks later, if a payment Radar allowed turns out to be fraud, the cardholder
   disputes it. That dispute flows back in as a label: "this one, that we called
   safe, was actually fraud." That label trains the next version of the model.

The whole live decision, steps 2 through 5, is designed to fit inside roughly 100
milliseconds, because it sits directly in the authorization path. The learning loop,
step 7, runs on a completely different clock: offline, over weeks, on the whole
history at once. Hold on to that split. It is the spine of the entire system.

## 7. Under the hood, like the engineer

This is the deep part. Radar is really three engines that run on three different
clocks, and almost every hard decision comes from keeping them apart.

Stripe describes Radar as three components: a rules engine, a manual-review
facility, and the machine-learning system that scores every payment. The ML system
is the interesting one, so start there.

### The two halves: features, then a model

Just like search is "matching then ranking," fraud scoring is "features then model."
The model is famous; the features are where the work actually lives. Stripe is blunt
about this: feature engineering is two jobs. First, figure out what signals have
predictive value (that takes deep knowledge of how fraud actually behaves). Second,
and this is the part people forget, make sure every one of those signals can be
computed both when you train offline and when you score a live payment in
production. A brilliant feature you cannot compute in 100ms is worthless.

**What a "feature" actually is, concretely.** Take Arjun's payment. Raw, it is a
card number, an IP, an email, an amount, a device fingerprint, a billing address.
Those raw values are useless to a model directly. Arjun's exact card number is one
of hundreds of millions; it has never been seen in exactly that form and never will
be again. So the useful features are derived:

- "How many distinct card numbers has this IP address tried in the last hour?" For
  Arjun: 1. For the card-testing script's IP: 380. That single number is a
  fraud-detector by itself.
- "Has this card succeeded on the Stripe network before, and how often did those end
  in disputes?" Arjun's card: seen, clean history. A freshly stolen card: never seen,
  or seen only in a burst of failures.
- "How far is this amount from this business's typical order?" 1,200 rupees at a
  journal shop: normal. 1,200 rupees of gift cards at 3am: not.

Each of these is a running aggregate. You do not scan history at request time to
compute "cards per IP this hour." You keep a counter that is updated as payments
flow, and at request time you just read it. This is the classic offline-think,
online-lookup move that shows up in almost every teardown in this ledger, from
Discover Weekly to Uber surge. The expensive counting happens continuously in the
background; the live path does a handful of fast reads.

### The data structures

- **Hash maps / key-value stores as counters.** The "distinct cards per IP per hour"
  and "disputes seen for this card" values live in a fast in-memory store keyed by
  IP, by card token, by device. At request time it is an O(1) lookup per signal, not
  a database scan. This is the feature store.
- **Embeddings for the high-cardinality stuff.** This is the key idea. IP, card,
  email, device fingerprint are high-cardinality: millions of possible values, most
  seen rarely. You cannot one-hot encode "card number" into millions of columns, and
  treating it as a plain category is close to useless because most values appear once.
  Stripe's answer, like modern recommendation systems, is to learn a dense embedding:
  map each high-cardinality entity to a short vector of, say, 32 numbers, where
  entities that behave similarly land near each other. A card that behaves like known
  good cards gets a vector near theirs. This turns "a value the model has literally
  never seen" into "a point in a space where nearby points have known behavior," which
  is how the model generalizes to Arjun's never-before-seen exact card.
- **The model itself.** Stripe's history here is well documented. The early Radar
  model was a "Wide and Deep" ensemble: an XGBoost gradient-boosted-tree model bolted
  together with a deep neural network. In mid-2022 Stripe replaced it with a single,
  pure deep neural network, and reported the DNN-only model trains faster, scales to
  far more data, and is easier to improve. Two reasons XGBoost had to go: it is not
  very parallelizable, so retraining was slow, and it was incompatible at scale with
  the newest ML techniques. When they threw 10x more training transactions at the new
  design, it kept getting better, where the old one had plateaued.
- **The multi-branch trick (ResNeXt).** The new network borrows an idea from an image
  architecture called ResNeXt. Instead of one deep stack, split the computation into
  many parallel branches, each a small network, and sum their outputs. Stripe found
  this "make it wider with many branches" shape made the model richer and more
  capable than just making one stack bigger. In plain terms: several specialists
  looking at the payment from different angles, then a vote, beats one generalist.

### Matching versus ranking, fraud edition

Fraud has no giant catalog to fetch candidates from, so the "matching" half is
cheap: there is exactly one payment to score. The whole difficulty moves into
"ranking," which here means producing one well-calibrated probability. And
calibration matters more than in search. A search engine only needs the right order.
Radar needs the number itself to mean something, because Meera's rule "block if score
> 75" only works if a 75 means the same thing on Tuesday as it did last month, and
roughly the same thing for a journal shop as for a laptop store. Stripe puts real
effort into score calibration so that the 0 to 99 dial is stable and comparable.

### The label problem, which is the deepest one

Here is what makes fraud genuinely harder than most ranking problems: you do not
find out if you were right for weeks, and for the cases you blocked, you never find
out at all.

The "this was fraud" label comes mostly from disputes and chargebacks, and those
land 30 to 90 days after the payment. So the model is always learning from a world
that is two months stale, while fraudsters change tactics this week. That delay is
why the retraining cadence and fresh signals matter so much.

Worse is the blocked-payment blind spot. When Radar blocks a payment, that payment
never happens, so you never learn whether it would truly have been fraud or was a
good customer you insulted. Your training data is censored: you only see outcomes
for payments you allowed. If you naively measure "precision and recall on payments
we let through," you are grading yourself on a rigged exam, because you removed the
hardest cases from the test. Stripe's fix is counterfactual evaluation: statistical
techniques to estimate what would have happened to the payments it blocked, so it
can compute an honest precision-recall curve. This is proprietary and, per Stripe's
own engineering talks, one of the harder pieces of the whole system. It is the fraud
analog of the off-policy evaluation problem in Netflix's artwork bandit from the
2026-06-19 teardown: you can only cheaply learn about the actions you actually took,
so you must correct for the ones you did not.

### The precision-recall dial, and why it is per-business

The single most important product decision is where to set the block threshold, and
the key insight is that there is no one right answer. Push the threshold down to
catch more fraud and you also block more Priyas. Push it up to spare the Priyas and
more card testers get through. Stripe learned a subtle version of this the hard way:
a model that looks better on aggregate metrics across all of Stripe can still spike
the block rate for one slice of small businesses, which is a disaster for those
specific merchants even while the global number improved. So "is this model better"
is not one number; you have to check that you did not quietly wreck some segment. A
laptop seller losing one 90,000-rupee sale to a false block cares very differently
than Meera losing one 1,200-rupee journal, and Radar lets the threshold and rules
bend per business.

### The network effect is the real moat

The reason Radar works out of the box for Meera, who has almost no fraud history of
her own, is that it does not learn only from Meera. It learns across millions of
businesses on Stripe, which collectively process over 1.9 trillion dollars a year,
across thousands of partner banks. The card testing Meera has never seen has been
seen a thousand times elsewhere on the network this week. A card that just got
flagged defrauding a store in Berlin is already suspicious when it hits Jaipur ten
minutes later. Stripe reports Radar reduces fraud on the order of tens of percent
(their newer figure is about 32 percent on average) while holding false positives
down, precisely because the shared signal is so much richer than any one store's
own data.

### The 2025 upgrade: a payments foundation model

The newest chapter is worth naming because it is the same idea taken further. In May
2025 Stripe unveiled a Payments Foundation Model: a transformer, trained
self-supervised on tens of billions of real transactions, with no fraud labels
needed. Instead of hand-crafting each feature, it learns a single dense "behavioral
embedding" for a payment, the way a language model embeds a sentence, capturing
temporal, geographic, and behavioral context in one vector. Plugged into fraud
detection, Stripe reported detection of card-testing attacks specifically jumped
from 59 percent to 97 percent with no increase in false positives. That is the
embedding idea from the features section, scaled up until the feature engineering
itself is largely learned.

### The scale story at three tiers

- **1,000 payments a day (Meera alone).** Trivial. A single Postgres table and a few
  hand-written rules ("block if more than 5 cards from one IP in an hour") would catch
  most abuse. At this size you do not even need machine learning; you need a couple of
  counters. The catch: Meera alone has almost no fraud examples to learn from, so a
  solo model would be blind. This tier only works because of the network.
- **100,000 payments a day (a mid-size platform).** Now you feel the two real walls.
  First, latency: you cannot compute "cards per IP this hour" by scanning a table on
  every request, so the aggregates move into an in-memory feature store that is
  updated as events stream in and read in O(1) at score time. Second, imbalance: fraud
  might be well under 1 percent of payments, so a model that predicts "not fraud"
  every time is 99 percent accurate and completely useless. You start measuring with
  precision-recall and AUC instead of accuracy, and you start weighting the rare
  fraud examples so the model bothers to learn them.
- **10 million-plus payments a day (Stripe).** Everything bends. The live scoring must
  stay under ~100ms, so the model forward pass is kept to single-digit milliseconds
  and everything expensive is precomputed: the DNN inference is a few matrix
  multiplies, and the heavy features are read, not calculated. Training cannot be a
  slow serial job, which is exactly why XGBoost was dropped for a parallelizable DNN
  that can chew through 10x the data. Counterfactual evaluation becomes mandatory
  because at this volume even a 0.1 percent wrongful block rate is thousands of
  insulted customers a day. And the per-segment problem appears: a global improvement
  that hurts one class of small merchants is not shippable, so evaluation fans out
  across business segments, not just the aggregate. The network is both the scaling
  problem and the scaling solution: more businesses means more load, but also means
  every new fraud pattern is seen somewhere first and shared everywhere fast.

## 8. The retention and habit mechanic

Radar is not an engagement loop. Nobody opens an app to enjoy fraud scoring. The
loop it drives is a trust-and-switching-cost flywheel, and it moves revenue and
retention, not daily activation.

The flywheel: more businesses on Stripe means more transactions, which means Radar
sees more fraud patterns, which means it blocks fraud more accurately for everyone,
which means merchants lose less money on Stripe than on a competitor, which attracts
more businesses. Each new merchant makes the fraud model better for every existing
merchant. That is a genuine data network effect, the same shape as a marketplace's
liquidity flywheel, and it is very hard for a new entrant to copy because they start
with no network and therefore a worse model on day one.

The retention mechanic is that this compounding is invisible and sticky. Meera never
logs in to admire Radar, but the day she considers switching processors to save a
little on fees, the real question is "will the new one catch the fraud Stripe was
quietly killing for me?" The longer she stays, the more her low chargeback rate is
partly Stripe's doing, and the scarier leaving becomes. Stripe's own framing of the
foundation model makes the flywheel explicit: better data and infrastructure compound
into an advantage competitors cannot easily close. A concrete observed example is the
card-testing number moving from 59 to 97 percent with the foundation model: existing
merchants got materially safer without lifting a finger, which is exactly the kind of
silent improvement that makes leaving feel like a downgrade.

## 9. The lesson for Rare.lab

The transferable lesson is the three-clocks split, and it maps almost directly onto a
shader and visual-effects engine.

Radar survives a 100ms budget by refusing to think hard on the hot path. The
expensive thinking (training the DNN, computing counterfactual evaluation, updating
the running aggregates) happens offline or continuously in the background. The live
per-payment path is reduced to two cheap moves: read precomputed features, run one
small forward pass. The score is calibrated and stable so the consuming rules do not
have to recompute anything.

For Rare.lab, put the same wall between "compile time" and "frame time." A node
graph an artist builds is your training-time, expensive-thinking phase: that is where
you can afford to do the heavy work of constant folding, dead-node elimination,
precomputing lookup tables and gradient ramps, baking noise into textures, and
choosing the cheapest equivalent instruction sequence. The compiled shader that runs
60 or 120 times a second is your hot path, and it must be the equivalent of "read a
precomputed feature, do a few multiplies." Anything that can be hoisted out of the
per-frame, per-pixel loop into the compile step must be, because a per-pixel cost is
paid millions of times per frame the way Radar's per-payment cost is paid millions of
times a day.

Two sharper corollaries from Radar:

- **Every feature you expose must be computable on the hot path, or it does not
  ship.** Stripe rejects predictive features they cannot compute in production in
  time. Rare.lab should reject any node that looks beautiful in the editor but cannot
  compile to something cheap enough for the target frame budget. Design the node
  library against the runtime cost, not just the visual result.
- **The high-cardinality embedding trick generalizes.** Radar turns "a value never
  seen before" into "a point near known-similar values." When Rare.lab needs runtime
  behavior over an unbounded space (per-object variation across thousands of
  instances, say), do not branch per case on the GPU; precompute a compact table or a
  small learned representation the shader can sample in constant time. Same spirit:
  move the unbounded thinking offline, leave a fast lookup on the frame.

---

## Sources

- Stripe, "How we built it: Stripe Radar" (engineering blog): https://stripe.dev/blog/how-we-built-it-stripe-radar
- Stripe, "A primer on machine learning for fraud detection": https://stripe.com/guides/primer-on-machine-learning-for-fraud-protection
- Stripe Radar product page (scale figures, block/false-positive claims): https://stripe.com/radar
- InfoQ / QCon, "Counterfactual Evaluation of Machine Learning Models" (Stripe talk on evaluating blocked payments): https://www.infoq.com/presentations/stripe-ml-models-fraud/
- ByteByteGo, "How Stripe Detects Fraudulent Transactions Within 100 ms": https://blog.bytebytego.com/p/how-stripe-detects-fraudulent-transactions
- TechCrunch, "Stripe unveils AI foundation model for payments, reveals deeper partnership with Nvidia" (May 2025): https://techcrunch.com/2025/05/07/stripe-unveils-ai-foundation-model-for-payments-reveals-deeper-partnership-with-nvidia/
- Stripe blog, "Updates to Stripe's advanced fraud detection": https://stripe.com/blog/advanced-fraud-detection-updates
- Stripe Documentation, "Risk evaluations": https://docs.stripe.com/radar/risk-evaluation
