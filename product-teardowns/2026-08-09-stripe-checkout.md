# Stripe Checkout: the hosted pay page, the payment-method ranker, and the PaymentIntent state machine

Date: 2026-08-09
Product: Stripe
Feature: Checkout (the hosted payment page: which payment methods appear and in what order, plus the PaymentIntent state machine and 3D Secure authentication underneath)

---

## 1. The user

Priya runs a two-person indie software studio out of Berlin. She just launched a
small design tool and wired up billing with Stripe Checkout because she did not
want to build a payment form, store card numbers, or think about European
banking rules. It is Friday night. Her first paying customer, Marco, is sitting
in Milan on his laptop. He clicked "Subscribe, 49 euro per year," and now he is
looking at a clean Stripe-hosted page with his email prefilled, a card field,
and a big "Pay" button.

Priya is not on the page. She is at dinner. The whole thing has to work without
her. If Marco sees a broken form, an American-looking payment page with no
European methods, or a scary bank redirect that dumps him back to nowhere, she
loses her first sale and never knows why.

---

## 2. The real problem

Taking a card online is deceptively ugly under the hood, and three ugly things
all land at once.

First, the page has to guess what Marco wants to pay with. An Italian shopper
expects to see his card, maybe a wallet like Apple Pay, maybe a local method. A
Dutch shopper expects iDEAL. A German shopper often wants a bank-debit option. If
Priya hand-codes "show Visa and Mastercard" she quietly kills conversion in half
of Europe. Stripe's own experiments found that showing even one payment method
that is not relevant to where the shopper is can cut conversion by up to 15
percent. The wrong menu is not neutral. It actively costs money.

Second, European law now forces an extra step. Since 14 September 2019, under
PSD2, most online card payments in Europe must pass Strong Customer
Authentication (SCA), which in practice means 3D Secure: the bank may pop up and
ask Marco to confirm with a code or a fingerprint. That step can happen or not,
can be a silent check or a full challenge, and Priya's code has no way to know in
advance which one Marco's bank will demand.

Third, a payment is not a function call that returns true or false. It is a
conversation with a bank that can pause in the middle for that authentication,
can take seconds, and can drop the network connection right at the worst moment.
"Did Marco pay?" does not have an instant answer.

Priya wants none of this in her code. She wants "give me money for this
subscription" and a page that just handles Milan, Berlin, Amsterdam, and the
bank pop-up on its own.

---

## 3. The feature in one sentence

Stripe Checkout is a hosted payment page that decides which payment methods to
show each shopper and in what order, then drives a bank conversation (including
the 3D Secure pause) to a definite yes or no, all tracked by one object called a
PaymentIntent that moves through a fixed set of states.

---

## 4. Jobs to be done

- "Show my shopper the payment methods he actually uses, in the order most
  likely to get him to finish, without me writing per-country logic."
- "Handle the European bank authentication pop-up correctly so I do not get
  declined for missing SCA."
- "Never double-charge my customer if the network hiccups, and never tell me a
  payment succeeded when it did not."
- "Give me a page that looks trustworthy on a phone in Milan and a laptop in
  Berlin without me designing anything."
- "Tell me, reliably and after the fact, whether the money actually arrived."

---

## 5. How it works for the user

Marco clicks "Subscribe." His browser lands on a Stripe URL
(checkout.stripe.com/...). The page already knows the amount (49 euro), the
currency, and roughly where Marco is. It shows his card field first because he is
in Italy and cards dominate there, with a wallet button on top if his device
supports one. He types his card number. He clicks "Pay."

The button spins. His bank decides Marco looks legitimate tonight and waves him
through silently. Two seconds later the page flips to a green checkmark and
bounces him back to Priya's "Thanks, you are subscribed" page.

If instead his bank wanted proof, a small window would have appeared asking Marco
to approve the payment in his banking app or type a one-time code. He approves.
Same green checkmark. He never had to understand any of it.

---

## 6. The actual flow, step by step

1. Priya's server (one API call) creates a PaymentIntent: amount 4900, currency
   eur, and "figure out the payment methods automatically." Stripe hands back a
   client secret and a hosted page URL.
2. Marco is redirected to the Stripe page. Stripe looks at the amount, currency,
   Marco's country and device, and Priya's account settings, then renders the
   ordered list of eligible payment methods. Card first for Italy.
3. Marco enters his card and taps Pay. The card details go straight to Stripe,
   never to Priya's server. Priya's servers never touch a raw card number, which
   is most of what keeps her out of PCI scope.
4. Stripe confirms the PaymentIntent. The bank is contacted. The intent moves to
   requires_action because the bank wants 3D Secure.
5. The Stripe page runs the authentication. Frictionless: the bank checks its own
   data and says yes with no pop-up. Challenge: a modal or bank redirect asks
   Marco for a code or a biometric.
6. Authentication done, Stripe submits the charge to the card network. The intent
   moves to processing, then to succeeded.
7. Marco is redirected to Priya's return URL. Separately, Stripe fires a webhook
   (payment_intent.succeeded) to Priya's server. That webhook, not the redirect,
   is the source of truth she uses to switch on Marco's subscription. (Webhooks
   got their own teardown on 2026-07-31.)

---

## 7. Under the hood, like the engineer

Two engines are stacked here. One decides what to show (a matching-then-ranking
problem over payment methods). One decides what actually happened (a state
machine over a slow, interruptible bank call). They are worth pulling apart
because they fail in completely different ways.

### Engine A: which methods to show, and in what order (match, then rank)

This is the ledger's recurring two-half shape, applied to payment methods instead
of products or videos.

**The matching half is cheap and rule-bound.** Start from the full catalog of
methods Stripe supports (cards, Apple Pay, Google Pay, iDEAL, SEPA debit, Klarna,
Bancontact, UPI, and dozens more). Filter to the ones that are actually eligible
for THIS checkout. The filter keys are hard facts: the currency (iDEAL only
settles in euro), the amount (some buy-now-pay-later methods have floors and
caps), the shopper's country (iDEAL is a Netherlands method), the merchant's
enabled set (Priya toggled some on), and device capability (Apple Pay only shows
inside Safari or iOS with a provisioned card). This is a fast boolean filter over
a few dozen candidates, not a search over millions. Think of it as an in-memory
predicate list: keep a method only if every gate passes. For Marco in Milan
paying 49 euro on Chrome, the eligible set comes out to something like {card,
Apple Pay if device supports it, maybe Klarna}. iDEAL and Bancontact are filtered
out because Marco is not in the Netherlands or Belgium.

**The ranking half is where the AI lives.** Once you have the handful of eligible
methods, order them so the shopper finishes. Stripe's Optimized Checkout Suite
uses machine-learning models, trained on billions of transactions, to pick the
order for every single checkout session. The models read on-session signals
(device type, browser locale, which methods are even available on this device)
and network-level signals (which methods shoppers at similar businesses actually
complete with). So a first-time German shopper on an Android phone might see card
plus a bank-debit option high up, while an Italian shopper sees card and a wallet
first. The payoff is real and measured: dynamic payment-method ordering, together
with saved credentials and localized pricing, drives an average 11.9 percent
revenue lift for businesses on the suite, and getting the menu wrong (one
irrelevant method shown) can cost up to 15 percent of conversion. A sister model,
Adaptive Pricing, predicts the shopper's true currency preference and lifted
cross-border revenue 17.8 percent.

The important engineering point: this ranking is NOT computed from scratch while
Marco waits. The heavy training over billions of transactions is offline. The
live page does a cheap thing: filter the eligible set, then apply a
precomputed/served model score to order a list of about ten items. It is the same
offline-think, online-lookup spine as Discover Weekly and YouTube recommendations,
just with payment methods as the items. The data structures are humble: an array
of candidate methods, each tagged with a feature vector, sorted server-side by a
model score. Sorting ten things is free. The intelligence is in the score, and
the score was learned last night on someone else's cluster.

Concrete walk: Marco's eligible array is [card, applepay, klarna]. The ranker
scores them for "Italian, mobile Safari, 49 euro, similar merchants" and returns
[applepay, card, klarna]. Apple Pay floats to the top because on an iPhone in
Italy it converts best. That reorder is the whole feature, and it is one sort.

### Engine B: the PaymentIntent state machine (the correctness core)

A card payment cannot be a single blocking call, because it can pause for bank
authentication and because the network can die mid-flight. Stripe models the
whole attempt as one object, the PaymentIntent, that lives in exactly one state
at a time and moves along fixed transitions. The published states are:

- requires_payment_method: created, no usable method attached yet (or the last
  one failed, so it comes back here to try again).
- requires_confirmation: a method is attached, waiting for the confirm call.
- requires_action: the bank wants more, almost always 3D Secure. The payment is
  paused, waiting on the human.
- processing: submitted to the network, outcome not yet known (common for slower
  methods like bank debits).
- requires_capture: only when the merchant does a two-step "authorize now,
  capture later" flow (think a hotel hold). Money is reserved, not yet taken.
- succeeded: money is captured. Terminal.
- canceled: abandoned or timed out. Terminal.

An individual attempt can also fail, which drops the intent back to
requires_payment_method so the shopper can try a different card rather than
starting over.

Why a state machine and not a boolean? Because the requires_action pause is a
real suspend point. When Marco's bank wants 3DS, the server cannot just block and
wait. It writes the intent to requires_action, sends a next_action instruction
down to Stripe.js in the browser, and lets the client drive the authentication.
The browser calls handleNextAction (or confirmPayment), which either shows the
challenge in an iframe modal or redirects Marco to his bank. When the bank is
done, the shopper returns to a return_url and the iframe posts a message up to
the page to say authentication is complete; the intent then advances to
processing and on to succeeded. The state machine is what lets the flow survive a
pause that might last thirty seconds while Marco fishes his phone out of his
pocket.

**The 3D Secure sub-flow.** 3DS is itself a conversation between three parties.
Stripe talks to the card network's Directory Server, which routes to the
issuer's Access Control Server (the ACS, the bank's authentication brain). The
ACS picks one of two paths. Frictionless: it already has enough signal (device,
past behavior, the data Stripe collected during checkout) to trust the shopper,
and it authenticates with no pop-up at all. Challenge: it wants a code or a
biometric and shows the modal. The merchant can express a preference with
request_three_d_secure set to "any" or "challenge," but the issuer decides. The
prize for passing 3DS is the liability shift: once the bank authenticates the
shopper, fraudulent-chargeback liability moves from Priya to the bank. So 3DS is
both a legal requirement in Europe and a fraud shield, which is why Stripe leans
on frictionless as much as issuers allow (fewer pop-ups, same liability
protection, higher conversion).

**Not double-charging.** The confirm step carries the same idempotency spine
covered in the 2026-06-20 idempotency teardown. If Marco's Pay tap fires twice,
or the network drops the reply and the client retries, the PaymentIntent id plus
an idempotency key make the second attempt a no-op that returns the first result.
The state machine is forward-only for money: once succeeded, re-confirming does
not charge again. "Charge succeeded but the reply got lost" and "charge never
happened" are exactly the ambiguity the state machine plus idempotency exist to
resolve.

### The scale story at three tiers

**1,000 payments a day (Priya today).** One row per PaymentIntent in Postgres.
The state transitions are ordinary UPDATEs guarded by the current state. The
requires_action round-trip to the browser is just a couple of extra HTTP calls.
No sharding, no queues. A single database and a synchronous confirm handle this
comfortably. The payment-method ranker can even be a simple rules table at this
size and nobody would notice.

**100,000 payments a day (Priya's tool takes off).** Now three things bite. One,
slow methods and 3DS mean many intents sit in processing or requires_action for
seconds, so you cannot hold a request thread open per payment; the flow becomes
event-driven, and the webhook (payment_intent.succeeded) becomes the real
"it happened" signal instead of the synchronous response. Two, retries multiply,
so idempotency keys and a unique index on (account, key) become mandatory to stop
concurrent double-confirms. Three, reads (dashboards, "did it succeed" polling)
start to threaten the write path, so analytics move off the transactional
database onto read replicas and a separate store.

**10 million plus a day (Black Friday).** This is real Stripe scale. Over the four
days of Black Friday to Cyber Monday 2023, businesses on Stripe processed more
than 300 million transactions worth over 18.6 billion dollars, and the API peaked
at 27,395 requests per second while holding 99.999 percent uptime; Stripe crossed
1 trillion dollars in total payment volume in 2023. Surviving this rests on the
patterns the ledger keeps returning to. Shard the PaymentIntents by merchant
account so each intent is self-routing and no global lock is needed; a payment for
Priya never contends with a payment for a giant retailer. Push all heavy analytics
off the hot path: Stripe's real-time BFCM transaction dashboard runs on Apache
Pinot, a separate columnar OLAP store, precisely so that counting 18.6 billion
dollars of volume never touches the systems taking the next payment. Serve the
payment-method ranking from precomputed model scores rather than training in the
request. And absorb the hottest merchant (a flash sale on one store) with the same
hot-key defenses used elsewhere: the per-account shard is the unit you replicate
and cache, and a single contested resource gets split so one viral store cannot
stall the queue for everyone.

Where public detail runs out, this is labeled inference: Stripe has not published
the exact model architecture behind dynamic payment-method ordering or the precise
sharding key for PaymentIntents. What is confirmed is the state names, the 3DS
Directory-Server-to-ACS flow, the conversion numbers, the BFCM scale figures, and
the use of Apache Pinot for the live dashboard. The "shard by account, precompute
the ranker, offload analytics" description is the well-grounded way this class of
system is built, matched to the numbers Stripe has published, and it is marked as
inference where it goes past the docs.

---

## 8. The retention and habit mechanic

Checkout's loop is not an engagement hook like a daily feed. It is a
switching-cost flywheel built on removed friction, and it moves revenue first,
retention second.

The direct move is revenue: the 11.9 percent average lift from optimized checkout
is money that shows up on the merchant's own dashboard. Priya sees more of her
"Subscribe" clicks turn into paid subscriptions without changing anything, because
Stripe reordered the methods for her. That visible lift is what keeps Priya on
Stripe. Every new payment method Stripe adds (a new wallet, a new local method in
a new country) shows up in her Checkout automatically, ordered by the same ranker,
with zero code from her. The longer she stays, the more her global conversion
quietly depends on Stripe's model, and the more painful it is to rebuild all of
that herself. That is the retention mechanic: the value compounds on Stripe's side
of the boundary, so leaving means giving up a conversion machine she never built.

There is also a two-sided data flywheel. Every checkout across millions of
businesses teaches the ranking and the fraud and authentication models which
methods and which 3DS paths convert. A brand-new merchant like Priya, with almost
no history of her own, still gets a well-ordered payment menu on day one because
the network already learned it from similar businesses. More merchants means
better ordering means higher conversion means more merchants. The observed proof
is the published 11.9 percent and 17.8 percent lifts: those are network effects
cashed out as conversion.

---

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader code and runs it in an
embeddable runtime. Two concrete lessons transfer.

First, split "what to offer" into a cheap wide filter and a narrow learned rank,
and precompute the learned half. When a user drops a node onto the canvas and
Rare.lab suggests the next node, the plausible-next-node menu, or a starting
effect template, do not run an expensive model per keystroke. Filter first on hard
rules (which nodes can even connect to this output type, which effects the target
platform can compile to, which shaders fit the current pass) to get a small
eligible set, then order that short set by a model score trained offline on what
graphs real users actually build and ship. That is exactly Stripe's
match-then-rank over payment methods: the live path is a boolean filter plus a
sort over ten items, and the intelligence sits in a score computed last night. It
keeps the editor instant no matter how large the node catalog grows, because live
cost tracks the eligible set, not the catalog.

Second, model the compile-and-render pipeline as an explicit, resumable state
machine with an idempotency key, exactly like the PaymentIntent. Compiling a big
graph to code and warming a shader can be slow and can involve an external step (a
remote build, a texture upload, a GPU warm-up) that may pause or drop the
connection, the same way 3DS pauses a payment at requires_action. If Rare.lab
treats each compile as one object moving through states (queued, compiling,
awaiting_asset, linking, ready, failed) keyed by an idempotency token, then a lost
connection or a double-clicked "Publish" resumes or no-ops instead of kicking off
a second expensive compile or, worse, shipping two conflicting builds. Fast and
learned at the front, forgiving and forward-only at the back. That combination is
what lets Stripe take a payment from a phone in Milan without Priya watching, and
it is the same combination that will let Rare.lab publish an effect from a
designer's laptop without a human babysitting the build.

---

## Sources

- Stripe Documentation, The Payment Intents API: https://docs.stripe.com/payments/payment-intents
- Stripe Documentation, Authenticate with 3D Secure (authentication flow): https://docs.stripe.com/payments/3d-secure/authentication-flow
- Stripe Documentation, Strong Customer Authentication: https://docs.stripe.com/strong-customer-authentication
- Stripe guides, Designing card payment flows for SCA: https://stripe.com/guides/sca-payment-flows
- Stripe Documentation, Dynamic payment methods: https://docs.stripe.com/payments/payment-methods/dynamic-payment-methods
- Stripe blog, How Stripe is using AI to create personalized checkout experiences: https://stripe.com/blog/stripe-ai-personalized-checkout-experiences
- Stripe guide, Optimizing payments at scale (AI across the payment lifecycle): https://stripe.com/guides/optimizing-payments-at-scale
- Stripe Documentation, Finalize payments on the server (handleNextAction): https://docs.stripe.com/payments/finalize-payments-on-the-server
- Stripe Documentation, stripe.handleNextAction (Stripe.js): https://docs.stripe.com/js/payment_intents/handle_next_action
- Stripe Newsroom, Businesses process more than $18.6 billion over Black Friday and Cyber Monday 2023: https://stripe.com/newsroom/news/bfcm2023
- StarTree, Stripe's journey to $18.6B of transactions during BFCM with Apache Pinot: https://startree.ai/user-stories/stripe-journey-to-18-b-of-transactions-with-apache-pinot/
- Stripe blog, Surprising findings from our analysis of 3DS transactions in the US: https://stripe.com/au/blog/surprising-findings-from-our-analysis-of-3ds-transactions-in-the-us
