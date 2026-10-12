# Razorpay Route: splitting one payment across many sellers

Date: 2026-10-12
Product: Razorpay
Feature: Route (split payments and settlements for marketplaces, the one-to-many disbursement engine)

This teardown is about the plumbing under a marketplace. Not the checkout the
buyer sees. The part after the money lands: one payment from one buyer that has
to become many payouts to many sellers, held safely in between, clawed back
cleanly when there is a refund. Razorpay calls this product Route.

A note on neighbours in this ledger. We have already torn down three money-at-rest
pieces: the Stripe balance and payouts ledger (2026-07-23), Stripe Connect
(2026-09-27), and the Razorpay settlement and reconciliation spine inside UPI
payments (2026-08-01) and subscriptions (2026-09-06). This one does not re-derive
double-entry bookkeeping from scratch. It focuses on the new shape Route adds: the
fan-out (one credit becomes N obligations), the escrow hold (money parked on
purpose), and the reversal (money pulled back proportionally). Where it leans on
the older pieces it says so.

---

## 1. The user

There are two users here, and that is the whole point.

User one is Priya in Pune. It is a Sunday. Her bathroom tap has been dripping for
a week and the flat needs a deep clean before her parents visit. She opens a home
services app (think of a HomeClap style marketplace, the exact example Razorpay
uses in its own Route blog). She books a plumbing job for 500 rupees and a
full-home cleaning for 1,000 rupees. She pays once. One tap, 1,500 rupees, done.
She never thinks about who gets what.

User two is invisible to Priya but is the real customer of Route: the marketplace
itself, and behind it the two workers. Ramesh the plumber should get paid for the
tap. Sunita the cleaning partner should get paid for the clean. The platform keeps
its commission. The tax has to be set aside. Priya paid one number. That one number
now has to break into four correct pieces and reach four different places, on time,
without the platform ever touching money it is not allowed to hold.

The engineer building the marketplace is the person who feels the pain. Priya feels
nothing, which is the goal.

## 2. The real problem

Told like a friend would tell it.

You built a marketplace. Buyers pay you. You owe sellers. Sounds simple until you
try it. Three things bite you at once.

First, you are now holding other people's money. When Priya pays 1,500 rupees, 1,200
of that is really Ramesh's and Sunita's, not yours. In India, sitting on other
people's money while you run a platform is exactly what the RBI Payment Aggregator
rules police. You either become a licensed payment aggregator yourself (slow,
expensive, regulated) or you use someone who already is.

Second, paying sellers is not one payment, it is thousands of tiny payments. On a
busy Sunday a cleaning marketplace might take 5,000 orders. That is not 5,000
payouts. Each order might split three or four ways. That is 15,000 to 20,000 tiny
bank transfers a day, each to a different account, each needing the right amount
after commission and tax. Doing that by hand, or with a nightly spreadsheet and a
bank portal, falls apart the first week.

Third, money flows backward too. Priya cancels the clean. Now you owe her 1,000
rupees back. But you already told Sunita she is getting 800. If you already paid
Sunita, you are chasing a cleaning partner for a refund. That is a nightmare you do
not want to run a business on.

The honest version of the problem: a marketplace needs to split one inbound payment
into many outbound ones, hold the pieces safely until it is sure, and be able to
pull them back cleanly, all without becoming a bank.

## 3. The feature in one sentence

Route lets a platform split a single customer payment into many smaller transfers to
many seller accounts, control exactly when each piece settles, and reverse any piece
cleanly, all inside Razorpay's regulated escrow so the platform never has to hold or
chase the money itself.

## 4. Jobs to be done

What the marketplace is really hiring Route to do.

- "Take the one payment I collected and get the right slice to each seller, after my
  commission, without me writing a payout engine." The split job.
- "Hold the seller's money until I am sure the job is done, then release it." The
  escrow and on-hold job. Priya's 1,500 can sit parked; Sunita's 800 is released only
  after the clean is confirmed.
- "When a buyer refunds, pull the money back from the right sellers in the right
  amounts, automatically." The reversal job.
- "Keep me on the right side of the RBI, so I am not illegally holding funds." The
  compliance job. This is the quiet one that makes the whole product exist.
- "Give each of my sellers their own view and a predictable payout date so they trust
  me and stay." The two-sided trust job.

Notice there is no search here, and no ranking. Every other heavy teardown in this
ledger (Amazon search, YouTube recommendations, Swiggy listing) was a matching half
plus a ranking half. Route is a different animal. It is a fan-out plus a ledger plus
a scheduler. The engineering difficulty is correctness and timing, not relevance.
The closest cousin in this ledger is the subscriptions teardown (2026-09-06), which
we called "a scheduler with payments bolted on." Route is "a ledger with a fan-out
bolted on."

## 5. How it works for the user

For Priya: nothing. She pays 1,500 rupees once, the way she pays for anything. One
card charge or one UPI debit. She sees "Payment successful." The split is invisible.
That invisibility is the feature.

For the marketplace engineer, the visible experience is an API and a dashboard. They
set up each seller once as a Linked Account (Ramesh and Sunita each get an account id
that looks like `acc_GfnmleeViP9Y4a`). When Priya's payment is captured, the platform
calls Route and says, in effect: of these 1,500 rupees, send 400 to Ramesh, send 800
to Sunita, keep the remaining 300 for me. Razorpay records the split, holds the money
in escrow, and later settles each piece to each seller's bank account on a schedule.
Each seller can log into a dedicated Linked Account dashboard and see what they earned
and when it will land.

For the sellers: a predictable deposit. Sunita sees 800 rupees hit her bank account
two working days after the job, every time, without having to invoice anyone.

## 6. The actual flow, step by step

Walking Priya's 1,500 rupee order end to end.

1. Priya checks out. She pays 1,500 rupees through Razorpay Checkout. This half is
   ordinary payments (the Stripe Checkout teardown 2026-08-09 and UPI payments
   2026-08-01 cover the mechanics). The payment gets captured. It now has a
   `payment_id` like `pay_29QQoUBi66xm2f`. The money sits in the platform's Razorpay
   escrow balance.

2. The platform splits it. The backend calls `POST /v1/payments/pay_29QQoUBi66xm2f/transfers`
   with a `transfers` array. Two items:
   - `{ account: acc_Ramesh, amount: 40000, currency: INR, notes: {...}, linked_account_notes: [...] }`
   - `{ account: acc_Sunita, amount: 80000, currency: INR, notes: {...}, linked_account_notes: [...] }`

   Amounts are in paise, so 40000 paise is 400 rupees and 80000 is 800. The platform
   does not create a transfer for its own 300 rupee commission. That simply stays in
   the platform account. Razorpay enforces one hard rule here: the sum of all
   transfers plus Razorpay's fees cannot exceed the captured amount. 400 plus 800 is
   1,200, leaving 300 for the platform and fees. The books balance.

3. Each transfer gets its own id. Razorpay returns two transfer entities, each with a
   `transfer_id` (like `trf_E9utHWhVgfQOphk`), each pointing back at the same
   `payment_id`, each starting with `settlement_status: pending`. That shared
   `payment_id` is the join key that ties the whole split together. It plays the same
   role the 12 digit RRN played in the UPI teardown: the one id that lets you line up
   the inbound credit with every outbound leg later.

4. Optional hold. The platform is not sure Sunita will finish the clean well, so it
   parks her money. It calls `PATCH /v1/transfers/trf_SunitaLeg` with
   `{ on_hold: true, on_hold_until: 1735689600 }` (a Unix timestamp). Sunita's leg
   moves to `settlement_status: on_hold`. The money is hers in the books, but it will
   not leave escrow until the hold lifts. Ramesh's tap job was simple, so his leg stays
   on the normal track.

5. Settlement. By default, domestic settlement is T plus 2 working days, where T is the
   day the payment was captured. On the second working day, Razorpay batches up every
   due transfer and pushes the money out to each seller's bank account. Sunita's leg,
   if the hold has lifted, flips to `settlement_status: settled`. If the platform paid
   for the Instant Settlements add-on, it can move funds on a T plus 0 basis, 24x7, for
   a fee, up to a daily cap.

6. The refund path (the backward flow). Priya cancels the clean the next morning. The
   platform issues a refund to Priya (a normal payment refund call) and reverses
   Sunita's leg. If Sunita's 800 was still on hold or still pending, the reversal is
   clean: the money never left escrow, so Razorpay just pulls it back into the platform
   balance to fund Priya's refund. The transfer's status becomes `reversed`, or
   `partially_reversed` if only part came back, and an `amount_reversed` field records
   how much. This is exactly why the hold exists: it keeps the seller's money
   reversible until the platform is sure.

Every one of these steps happens server side, in Razorpay's books, over seconds to
days. None of it happens on Priya's phone. Her phone's job ended at "Payment
successful."

## 7. Under the hood, like the engineer

This is the heart. What are the data structures, what is the algorithm, and what
breaks as you scale.

### The shape of the problem

Route is three classic problems stacked:

1. A fan-out. One inbound credit becomes N outbound obligations.
2. A ledger. Those obligations, plus the hold and the reversal, have to be recorded so
   that money is never created or destroyed by accident.
3. A scheduler. Each obligation has a release time (now, T plus 2, or a held-until
   date), and thousands of them come due in the same window.

We have torn down each of these before in other products. Route is where they meet.

### The data structures in play

Linked Accounts are a hash map. The key is the `account_id` (`acc_Ramesh`). The value
is everything Razorpay needs to pay that seller: the verified bank account, the KYC
state, the settlement schedule. When a transfer says "send 400 to `acc_Ramesh`," that
is an O(1) keyed lookup into the address book, not a search. This is the same move the
UPI teardown called the NPCI mapper, and the same move Magic Checkout used for saved
addresses: expensive setup once (KYC a seller), cheap keyed read forever.

A transfer is a row. The important columns, confirmed from the Route API:
`transfer_id`, `payment_id` (the join key), `account` (the linked account),
`amount` (paise), `settlement_status` (one of `pending`, `on_hold`, `settled`),
`on_hold`, `on_hold_until`, `amount_reversed`, and a status that can read `reversed`
or `partially_reversed`. The set of transfers sharing one `payment_id` is the split.

The ledger underneath is an append-only, double-entry record. This part is inference,
clearly labeled, because Razorpay has not published its internal ledger schema. But the
behavior Route exposes (balances that always add up, reversals that are exact, a
reconciliation that can replay history) is the signature of the same immutable
double-entry ledger we saw in the Stripe balance teardown (2026-07-23) and the
Modern Treasury writeups. The grounded version: Priya's 1,500 rupee capture is one
credit into the platform's escrow account. Each transfer is a pair of entries moving
value from the platform's escrow sub-ledger to a seller's sub-ledger. Nothing is ever
updated in place. A reversal is not a deletion, it is two more entries that undo the
first two. You never edit the past; you append a correction. That is why
`amount_reversed` is a field and not just a smaller number: the history stays whole.

The settlement queue is a scheduler keyed by time. Thousands of transfers all say
"release me on the second working day." The right abstraction, as in the subscriptions
teardown, is a time-ordered queue, a min-heap keyed by release timestamp, or its
database cousin, an index on `(settlement_status, settle_at)`. A worker pops everything
due and pushes it to the banks. A held transfer is simply one whose release time is
`on_hold_until`, or infinity until someone lifts the hold.

### The invariant that rules everything

Conservation of money. For any payment, the sum of its transfers plus Razorpay's fees
plus whatever the platform retains must equal the captured amount, exactly, forever.
400 plus 800 plus 300 equals 1,500. Not 1,499, not 1,501. This is the invariant Route
enforces when it rejects a transfer set that adds up to more than the payment. It is
also the invariant that makes reversals safe: because every rupee is tracked as a pair
of entries, pulling Sunita's 800 back is an exact operation, not an estimate.

### Atomicity: the split must be all or nothing

Here is the subtle engineering. When the platform posts "400 to Ramesh and 800 to
Sunita," you cannot have a world where Ramesh's leg is recorded and Sunita's is lost
because a server died mid-write. If that happened, 800 rupees of Priya's money would be
in limbo: not the platform's, not Sunita's, not refundable cleanly. So the split has to
be atomic. Either both legs land in the ledger or neither does. In practice this is a
single database transaction that writes all legs of one `payment_id` together, or a
state machine that treats the split as one unit of work. (Inference, labeled: the exact
mechanism is not public, but the all-or-nothing behavior is observable and is the only
safe design.)

### Idempotency: retries must not double-pay

The platform's server calls the transfers API. The network hiccups. The server does not
get a reply. Did the split happen? If it blindly retries, it could create the split
twice and try to pay Ramesh 800 instead of 400. This is the exact ambiguous-reply
problem the Stripe idempotency teardown (2026-06-20) was built around. The fix is the
same: the create call is keyed so that a repeat with the same key collapses to one
split. A bank transfer is a foreign mutation you cannot roll back, so you never let a
retry become a second payout.

### Where the sorting and the heavy work happen

Server side, always, and off the hot path. Priya's phone did one thing: send 1,500
rupees. The split math, the hold, the batching of thousands of payouts into bank files,
the reconciliation, all of it happens in Razorpay's backend, most of it asynchronously.
This is the same through-line as every teardown in this ledger: the live, user-facing
path stays cheap and fast; the expensive, clustered work (here, settling lakhs of
transfers) is pushed off to batch cycles where latency does not matter because the
seller is not staring at a loading spinner.

### The scale story, three tiers

What grows here is not a catalog of items. It is three things at once: the number of
transfers created per day, the number of distinct Linked Accounts (sellers), and the
clustering of settlements into the same T plus 2 window.

Tier one, 1,000 transfers a day. A new marketplace: 50 sellers, a few hundred orders,
each splitting two or three ways. One Postgres table of transfers. A single escrow
balance row for the platform. A nightly cron that reads everything due and fires the
payouts inline. Sharding here would be over-engineering. The real-world shape: the
HomeClap style startup in month one. Ship the simple thing.

Tier two, 100,000 transfers a day. Now you are a real marketplace: thousands of sellers,
lakhs of tiny transfers. Three things break.
- The settlement batch. Lakhs of payouts all come due in the same T plus 2 window and
  all have to go to banks at once. Fix: do not pay them one API call at a time. Batch
  them into bank rail files (NEFT batches) and hand them off, and run a pool of
  settlement workers that claim batches with a `SELECT ... FOR UPDATE SKIP LOCKED` so
  two workers never grab the same transfer. (The rail detail is general-knowledge
  inference for Route specifically; Razorpay's public docs confirm T plus 2 and the
  Instant and Same-Day add-ons.)
- Each seller's balance. You cannot recompute Sunita's balance by scanning every
  transfer she ever had on every read. Fix: keep it as a running aggregate, updated on
  the write path, read O(1). Same move as the Magic Checkout feature store.
- Hot escrow row contention. Many concurrent splits all touch the platform's one escrow
  balance row and fight for the lock. This is the first place a single row becomes a
  bottleneck.

Tier three, 10 million plus transfers a day. Razorpay-wide scale. The company reports
180 billion dollars plus in annualized transaction volume, its UPI Switch is built for
10,000 transactions per second, and its Optimizer router is built for 5,000 online TPS
(for comparison, India as a whole averages roughly 300 online payment transactions per
second). On a festive sale day, millions of splits land in hours. What breaks and the
fixes (fixes are grounded inference, following the same patterns the rest of this
ledger documented):
- The single ledger database breaks. Fix: shard the ledger by merchant or account id,
  so each marketplace's books live on their own partition and route to themselves. This
  is the exact self-routing trick from the Notion-by-workspace (2026-06-25) and
  Stripe-by-account teardowns.
- The hot Linked Account. One seller goes viral (a cloud kitchen that everyone orders
  from on Diwali). Its row becomes a hot key, every order wants to credit it at once.
  Fix: sub-shard the counter and sum the parts, the same hot-key fix Zepto's last-carton
  decrement (2026-06-17) and Instagram's counters used.
- The settlement window stampede. Millions due at once would melt the banks. Fix: the
  settlement is fully off the hot path, batched into rail cycles, and spread. Per-bank
  token buckets and backpressure so one slow bank does not stall every other payout,
  the exact per-bank rate-limit pattern from the subscriptions retry engine.
- Partial failure recovery. At this volume some payouts will fail mid-batch. Because the
  ledger is immutable and append-only, Razorpay can re-derive the true state of every
  balance by replaying the entries, and reconcile against the banks' settlement files
  nightly. You never trust a mutable per-row number as the source of truth; you trust
  the log and rebuild. Same spine as the UPI URCS reconciliation and the Stripe ledger.

The one-line scaling idea: the number of transfers and sellers grows without bound, but
the cost of any single split stays constant (a fixed, small set of ledger writes and a
scheduled release), because all the expensive clustered work (settling, reconciling) is
pushed off the live path into batched, shardable, re-derivable cycles.

### The escrow and compliance layer

One more real constraint, because it shapes the architecture. Route funds sit in
Razorpay's escrow, which is how a marketplace avoids being a payment aggregator itself.
Razorpay also offers Direct Settlement as a Route add-on, where the money skips the
escrow, and under the RBI Payment Aggregator guidelines from September 2025 a business
must meet turnover and payer-to-payee transparency criteria to use certain Route modes.
The engineering and the regulation are not separable here. The escrow hold is not just a
feature, it is the legal reason Route can exist, and it is the technical reason reversal
is clean.

## 8. The retention and habit mechanic

Route is not a consumer dopamine loop. There is no streak, no animation, no nudge. Its
retention is B2B, and it is one of the strongest kinds there is: deep integration lock-in
plus a two-sided trust flywheel. The metric it moves is revenue and merchant retention,
not daily active users.

The lock-in. A marketplace that wires Route into its payout logic has pushed Razorpay
deep into its spine. Every seller's KYC lives there. Every commission rule, every hold,
every reversal runs through it. Reconciliation (the thing finance teams lose sleep over)
is handled. Ripping Route out means rebuilding a payout engine, re-KYC-ing every seller,
and rebuilding reconciliation from zero. The switching cost is enormous, which is exactly
why platforms stay. This is the same shape as the Razorpay subscriptions teardown: once
the money plumbing is load-bearing, nobody rips it out.

The two-sided trust flywheel, which is the more interesting half. Reliable, predictable
payouts retain the supply side: Sunita gets her 800 rupees exactly two working days after
every job, with her own dashboard to see it, so she keeps taking jobs on this platform
instead of a rival. Clean, automatic reversals retain the demand side: Priya gets her
refund without the platform having to chase Sunita for money, so the platform can promise
easy refunds without going broke, so Priya trusts it and books again. More sellers staying
means more supply, which means more buyers served, which means more orders, which means
more sellers want in. The same liquidity flywheel we named in the Uber and Rapido
teardowns, here powered by the boring reliability of money arriving on time and leaving
cleanly.

A real observed example of the mechanic: Razorpay markets Route specifically to
marketplaces and platforms as the one-to-many disbursement backbone, and its own case
example (HomeClap splitting a 500 rupee plumbing job and a 1,000 rupee cleaning job across
different providers after commission and tax) is the loop in miniature. The habit being
built is not the buyer's. It is the platform's, and the sellers'. They come back because
the money is boring, and boring money is the highest compliment a payments product can
earn.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader code and ships it to an embeddable
runtime. When a creator hits publish, one authored version has to fan out to many targets:
many embeds on many sites, many CDN edge nodes, many device classes. That is the exact
shape of Route. One inbound thing becomes many outbound obligations. So steal Route's
spine.

1. Separate capture from settlement. Route does not block Priya's payment on paying the
   sellers; it captures instantly and settles the legs later, on a schedule. Do the same:
   when a creator publishes, record the publish as one atomic, immutable event
   instantly, and let the fan-out to edge nodes and embeds happen asynchronously in
   batches. Never make the author wait on a global rollout to confirm their publish
   succeeded. The author's hot path should be as cheap as Priya's "Payment successful."

2. Make every fan-out push idempotent, keyed by `(graph_id, version, target)`. A rollout
   to 10,000 edge nodes will have retries. A retried push must be a no-op, never a double
   apply, the same way a retried transfer collapses to one. This is the difference between
   a flaky rollout and a correct one.

3. Make "on-hold" a first-class state, and use it as a canary gate. Route holds a
   seller's money until the platform is sure the job is done. Rare.lab should stage a new
   shader version to edge nodes but hold it inactive until a canary confirms it compiles
   and renders inside the frame budget on real devices. If the canary blows the budget
   (a shader that stalls at 40ms on a Pixel 6a), do not activate. The hold is your safety.

4. Make rollback a reversal, tracked per target. When the canary fails, you do not patch
   nodes in place. You reverse that target back to its last-good version, and you track it
   per target, the way `amount_reversed` tracks each transfer. Some nodes may already be
   live and need reverting while others never activated; both must be exact, not
   estimated.

5. Hold the conservation invariant. The set of shader versions live across all runtimes
   must always be derivable from the immutable publish ledger, never from mutable
   per-node state. That is what lets you reconcile after any partial rollout failure:
   replay the log, compare to what each node reports, fix the drift. Route survives
   partial bank failures this way; your runtime should survive partial rollout failures
   the same way.

6. Scale it the way Route scales. Shard the rollout ledger by project or graph id so each
   creator's rollouts route to themselves. Batch edge pushes per region off the hot path.
   Put per-node backpressure in front of each CDN POP so one slow edge does not stall the
   global rollout, exactly as Route puts per-bank token buckets in front of each bank.

One line: treat publish as a capture and rollout as a settlement, fan out idempotently,
stage behind an on-hold canary gate, roll back as an exact per-target reversal, and keep
the whole live set re-derivable from an immutable publish ledger so that no partial
failure can ever leave a creator's effect half-shipped and unaccountable.

---

## Sources

- Razorpay Route product page: https://razorpay.com/route/
- Razorpay Route documentation (About Route): https://razorpay.com/docs/route/
- Razorpay Route docs (payments): https://razorpay.com/docs/payments/route/
- Razorpay Linked Accounts docs: https://razorpay.com/docs/payments/route/linked-account/
- Razorpay Route Transfers API (create transfers from a payment): https://razorpay.com/docs/api/payments/route/transfers
- Razorpay Route direct transfers API: https://razorpay.com/docs/api/payments/route/transfers/direct-transfers/
- Razorpay modify settlement hold API (on_hold, on_hold_until): https://razorpay.com/docs/api/payments/route/modify-settlement-hold.md
- Razorpay Route transfers entity (settlement_status, amount_reversed, reversed/partially_reversed): https://razorpay.com/docs/us/api/payments/route/transfers-entity
- Razorpay blog, "Effortlessly split and transfer your payments with Razorpay Route" (HomeClap example): https://razorpay.com/blog/effortlessly-split-and-transfer-your-payments-with-razorpay-route/
- Razorpay Settlements docs (T plus 2 default, working days): https://razorpay.com/docs/payments/settlements.md
- Razorpay blog, Instant Settlements (T plus 0, 24x7, daily cap): https://razorpay.com/blog/instant-settlement-payment-gateway/
- Razorpay newsroom, UPI Switch with Airtel Payments Bank (10,000 TPS): https://razorpay.com/newsroom/
- Razorpay newsroom, Optimizer (5,000 online TPS, India ~300 TPS average): https://razorpay.com/newsroom/built-to-save-over-7000-cr-in-payment-failures-razorpay-launches-optimizer-indias-first-ai-powered-payments-routing-solution/
- Modern Treasury, "How to Scale a Ledger" (immutable, append-only double-entry at scale, used as grounded inference): https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-i
- Prior teardowns referenced: product-teardowns/2026-07-23-stripe-balance-payouts-ledger.md, product-teardowns/2026-09-27-stripe-connect-ledger-payouts.md, product-teardowns/2026-08-01-razorpay-upi-payments.md, product-teardowns/2026-09-06-razorpay-subscriptions-upi-autopay.md, product-teardowns/2026-06-20-stripe-idempotency-keys.md
