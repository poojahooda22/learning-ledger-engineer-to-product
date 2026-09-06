# Razorpay Subscriptions and UPI AutoPay: the invisible clock that charges you every month without asking

Date: 2026-09-06
Product: Razorpay
Feature: Subscriptions and recurring payments (the plan-and-mandate model, the UPI AutoPay / e-mandate rail underneath, the scheduler that fires each debit, and the retry engine that fights involuntary churn)

A note on sourcing up front. Two layers sit under this feature and they are
documented very differently. The rail layer (UPI AutoPay, RBI e-mandates, the
pre-debit notification rule, the per-transaction caps) is public and precise:
NPCI launched UPI AutoPay on 22 July 2020, and the RBI e-mandate framework that
forced the whole change is a written circular with hard numbers. I lean on those
as confirmed fact. The product layer (Razorpay's subscription state machine, its
plan model, its retry behaviour) is documented in Razorpay's own developer docs,
which I treat as confirmed for the states and transitions they name. The
scheduler layer (how Razorpay actually stores tens of millions of due dates and
fires them without a thundering herd) is not published in engineering-blog
detail, so where I describe the queue, the sharding, and the batch-claim I label
it inference clearly and give the well-grounded "this is how this class of
problem is solved" version.

---

## 1. The user

It is the 1st of the month. Arjun is on the Mumbai local, standing, one hand on
the overhead rail, the other on his phone. He is not thinking about money at all.
He is watching a cricket highlight. His Netflix keeps playing, his Spotify keeps
streaming, his Hotstar renewed for the new season, and the 149 rupees for each
left his account while he was asleep last night. He never opened a single app to
approve any of it. He got three SMS messages he did not read.

That is the entire point. Arjun "subscribed" to these services exactly once,
months ago, and tapped his UPI PIN one time. Since then the money moves on its
own. The feature succeeds precisely when Arjun forgets it exists.

The other user in this story is the business. Priya runs a small SaaS tool for
dentists, 4,000 clinics paying 999 rupees a month. She does not want to chase
4,000 invoices. She wants the money to arrive on the 1st, to retry itself if a
card is short, and to tell her which customers fell off. She is hiring Razorpay
so she never has to think about collection at all.

Both users want the same thing from opposite sides: money that moves on a clock,
untouched by human hands.

---

## 2. The real problem

Recurring payment sounds trivial until you look at what actually has to be true.

The naive version, the one that existed in India until 2021, was simple and
dangerous: the merchant stored your card number and just charged it whenever they
wanted. Netflix kept your card on file and hit it every month. There was no fresh
consent, no cap, no warning. If Netflix decided to charge you twice, or a bad
merchant decided to charge 50,000 rupees instead of 500, nothing in the system
stopped them. Your only defence was to notice on your statement and fight it
later.

The RBI looked at that and said no. In its e-mandate framework (the circular
first issued in 2019, pushed to full enforcement on 1 October 2021 after two
extensions because the industry was not ready), the RBI banned the "just store
the card and charge it" model outright for recurring payments. From that date, a
recurring charge on an Indian card was only legal if it rode on a registered
e-mandate: the customer must authenticate the mandate once with an additional
factor (an OTP or a UPI PIN), the mandate must carry a maximum amount, and the
customer must get a pre-debit notification at least 24 hours before every single
charge with the option to cancel.

The day it went live, recurring card payments across India started failing in
bulk. Netflix, Google Play, Amazon Prime, news subscriptions, SaaS tools, all of
them saw auto-renewals decline because their old card-on-file flow was now
illegal and the new rails were half-built. TechCrunch covered it as a
country-wide disruption on 30 September 2021. Google Play had already stopped
onboarding new recurring customers back in May 2021 and suspended free trials in
India because trials depend on auto-renewal. Netflix started pushing UPI AutoPay
as its escape hatch.

So the real problem is not "charge a card every month." The real problem is:
charge a customer every month, legally, with a hard amount cap they agreed to, a
24-hour warning before each charge, an authentication they did exactly once,
across 60-plus banks and UPI apps that each fail in their own way, at a scale
where millions of charges all fall due on the same midnight, and do it without
double-charging anyone when your own network hiccups. That is a very different
problem, and it is mostly a scheduling and reliability problem, not a payments
problem.

---

## 3. The feature in one sentence

Razorpay Subscriptions lets a business define a plan once, collect a single
authenticated mandate from each customer, and then hands the recurring charging,
the pre-debit warnings, the retries, and the churn-recovery to an automated
scheduler that fires each debit on its due date over the legal UPI AutoPay and
e-mandate rails.

---

## 4. Jobs to be done

What Arjun (the payer) is really hiring it to do:

- "Let me keep my Netflix without doing anything every month."
- "Never let anyone charge me more than I agreed to. Warn me before you take
  money."
- "Let me kill it in one tap the day I am done, from my own UPI app, without
  begging the merchant."

What Priya (the business) is really hiring it to do:

- "Collect my 999 a month from 4,000 clinics without me lifting a finger."
- "If a payment fails because a card expired or an account was short, retry it
  intelligently and tell me, do not just drop the customer."
- "Keep me legal. I do not want to know what RBI circular 2021 says. Handle the
  mandate, the cap, the 24-hour notice."
- "Tell me who is active, who is failing, who churned."

The deep job on both sides is the same: turn a decision into a default. A
subscription converts "will you pay again this month?" from an active choice into
a passive non-choice. Silence means yes. That is the entire commercial magic and
the entire regulatory danger, which is why the RBI wrapped it in so many rules.

---

## 5. How it works for the user

For Arjun, the visible experience is almost nothing, and that is the design.

Setup, once: he taps "Subscribe" on Hotstar. A Razorpay page opens. He picks UPI
AutoPay, sees "Hotstar will be able to debit up to 1,499 rupees, monthly, until
you cancel." He approves in his UPI app (Google Pay, PhonePe, Paytm) with his UPI
PIN. Done. He is charged the first amount right then.

Every month after: he does nothing. He receives an SMS a day before ("Hotstar
will debit 299 on 01 Oct, to cancel visit..."). He ignores it. The next morning
the money is gone and the service keeps working. He sees a line on his bank
statement.

Cancel, once: he opens his UPI app, goes to "Autopay" or "Mandates," finds
Hotstar, taps "Pause" or "Cancel." No email to support, no retention call. The
merchant is simply told the mandate is dead.

For Priya the business, the visible experience is a dashboard. She creates a Plan
("Dentist Pro, 999, monthly"). She sends subscription links or embeds a button.
She watches subscriptions move through states: authenticated, active, then some
slip to pending and halted when charges fail. She gets webhooks on every event so
her own app can cut off access when someone truly churns.

---

## 6. The actual flow, step by step

Follow one real subscription: Priya's clinic customer, "Dr. Rao Dental," paying
999 rupees a month.

Registration (happens once):

1. Priya's server calls Razorpay to create a Plan: amount 99900 (paise), period
   monthly, interval 1. A plan is a reusable template.
2. Her server creates a Subscription against that plan for Dr. Rao, with a
   total_count (say 12 cycles) and optionally a start date. The subscription is
   born in the **created** state.
3. Dr. Rao is shown the Razorpay checkout. He picks UPI AutoPay. He approves the
   mandate in PhonePe with his UPI PIN. This is the one and only authentication
   (the "additional factor" the RBI demands). The mandate is registered on the
   NPCI rails and gets a Unique Mandate Number (UMN). It carries a max amount
   (say 999, or higher if variable) and a frequency (monthly).
4. On successful authentication, the subscription moves to **authenticated**.
   Razorpay immediately runs the first charge for cycle 1. The subscription moves
   to **active**.

Every cycle after that (happens on a clock, no human):

5. About 24 hours before the due date, Razorpay sends the mandatory pre-debit
   notification to Dr. Rao (SMS or app). This is not optional politeness, it is a
   legal precondition. On some rails the flow is literally: check mandate is
   still active, send the pre-debit notification, and only then are you allowed
   to execute the debit. Skip the notification and the debit is rejected.
6. On the due date, in an allowed time window, Razorpay executes the recurring
   debit against the UMN. No PIN from Dr. Rao. The amount must be within the
   mandate cap. NPCI carries an execution sequence number for the mandate:
   sequence 1 was the registration charge, sequence 2 the next, and so on.
7. If it succeeds, an invoice is marked paid, a webhook fires
   (subscription.charged), the next_charge_at moves forward one month, and Dr.
   Rao's subscription stays **active**.

When a charge fails (the interesting half):

8. Dr. Rao's account is short on the 1st. The debit is declined. The
   subscription moves to **pending**. Razorpay keeps retrying on its retry
   schedule while pending.
9. If a retry succeeds, back to **active**. Note: only future cycles are charged
   normally, Razorpay does not silently stack up missed months.
10. If all retries are exhausted, the subscription moves to **halted**. Invoices
    keep generating per the billing cycle, but no auto-charge is attempted.
    Priya's dashboard shows Dr. Rao as halted, and her webhook handler can revoke
    access.

Terminal states:

11. **cancelled**: Priya or Dr. Rao kills it. Once cancelled it cannot be
    restarted, a fresh mandate is needed.
12. **completed**: the subscription reaches its end (the 12th cycle, or the
    end_date). It finishes clean.
13. **expired**: a start date was set, but Dr. Rao never authenticated the
    mandate in time. The subscription dies before it ever charged.

That state set (created, authenticated, active, pending, halted, cancelled,
completed, expired) is the confirmed Razorpay subscription state machine. The
whole feature is that graph plus a clock that pushes subscriptions along its
edges.

---

## 7. Under the hood, like the engineer

Here is the mental reframe that makes this feature click: **a subscription
product is not a payments system, it is a scheduler with a payments system bolted
to the end of it.** The hard, interesting engineering is "fire the right event at
the right time for tens of millions of mandates without melting," not "send an
HTTP request to a bank."

Let me build it up from the data.

### The core objects and why those shapes

**Plan.** A small immutable record: id, amount, currency, period (daily, weekly,
monthly, yearly), interval (every 1 month, every 3 months). It is a template so
you store the price once and point thousands of subscriptions at it. Classic
normalization: one Plan row, many Subscription rows referencing it by plan_id. A
hash-map lookup by plan_id at read time.

**Subscription.** This is the live object and the important columns are the
time-and-state ones: id, plan_id, customer_id, status (the enum above),
current_cycle, total_count, and the two that drive everything,
**next_charge_at** (a timestamp) and a reference to the mandate/token. The status
is a small enum, cheap to index and filter on ("give me all active subscriptions
due today").

**Mandate / token.** The registration produces a durable handle: on UPI it is the
UMN (Unique Mandate Number), on cards it is a network token plus the e-mandate
registration id. This handle is what lets a later charge happen with no fresh
authentication. Store it once, reuse it every cycle. It carries the cap amount
and the frequency, because the rail itself will reject a debit that violates
either.

**Invoice / charge attempt.** Each cycle generates an invoice, and each debit
attempt is its own row with an idempotency key. This matters enormously and I
will come back to it.

### Matching is trivial here, scheduling is the whole game

Most features I have torn down (Amazon search, YouTube recommendations, Swiggy
ranking) split into "matching" then "ranking." Subscriptions do not. There is no
candidate set to search and no ranking. The query is dead simple:

```
SELECT * FROM subscriptions
WHERE status IN ('active','pending')
  AND next_charge_at <= NOW()
```

The entire difficulty is that this query has to run correctly, on time, without
duplicates, across a table that at Razorpay's scale holds tens of millions of
rows, where a huge fraction of them share the exact same next_charge_at (midnight
on the 1st of the month, because everyone's OTT bill renews then). The data
structure that matters is not an inverted index or a tree. It is a **time-ordered
queue**, and the algorithm is "claim the due items atomically and hand them to
workers."

### The two-phase timer per cycle

Because the RBI forces a pre-debit notification at least 24 hours before the
charge, each cycle is not one timed event, it is **two**:

- Event A at T minus 24h: send the pre-debit notification.
- Event B at T: execute the debit, but only if A succeeded and the mandate is
  still active.

So a scheduler holding N subscriptions is really holding up to 2N pending timers.
This is exactly the shape of a delay queue: "run this callback at this future
timestamp." The canonical data structure for "give me the next thing that is
due" is a **min-heap / priority queue keyed by fire time**, and that is the right
mental model. In production nobody runs a single in-memory heap for 40 million
timers on one box, so it gets externalized (below), but the abstraction is a
priority queue by due-time.

### Idempotency: the one that will double-charge Dr. Rao if you get it wrong

This is the sharp edge and it is why Stripe idempotency keys (an earlier teardown
in this ledger) are directly relevant. A recurring debit is a foreign state
mutation: money leaves a bank account, and you cannot ROLLBACK a bank. If the
scheduler fires the debit, the bank charges the account, but the network drops
the response, the scheduler does not know whether it succeeded. If it blindly
retries, Dr. Rao pays twice.

The fix is the same forward-only pattern: every charge attempt carries an
idempotency key derived from something stable and unique per cycle, for example
(subscription_id, cycle_number, execution_sequence). The first attempt writes a
"started" checkpoint before it calls the bank. A retry with the same key finds
the checkpoint and does not fire a second real debit, it returns the recorded
outcome. NPCI's own execution sequence number (1, 2, 3...) per mandate is a gift
here: it gives you a natural, rail-blessed unique key per cycle, so "cycle 5's
debit" is a single well-defined event on both your side and the bank's side.

### The retry engine (pending state) and why it is a state machine, not a loop

When cycle 5 fails, you do not just retry immediately in a tight loop. Failures
have reasons: insufficient funds (retry in a few days when salary lands), bank
downtime (retry in an hour), expired mandate (do not retry at all, ask for
re-registration). So the pending state is really a small policy engine: classify
the decline code, pick a backoff, respect a maximum number of attempts and a
maximum window, then give up and move to halted. Razorpay markets the smart
version of this as "Intelligent Revenue-Protect," which acts at three moments:
recover drop-offs at registration, retry intelligently at debit time, and
re-engage at churn. The engineering underneath is: a decline-code to
retry-strategy map, plus a scheduler that re-enqueues the next attempt at the
computed future time. It is the same two-phase timer, re-armed.

### The scale story, three tiers

**Tier 1, about 1,000 mandates (Priya's dentist SaaS).**
A single cron job every minute runs the SELECT above, gets a handful of due rows,
charges them inline, updates next_charge_at. A b-tree index on
(status, next_charge_at) makes the query instant. One worker, one database, no
queue. This works and you should not build more. Even the "everyone due at
midnight on the 1st" clustering is fine at 1,000 rows: the cron chews through
them in seconds. Nothing breaks here.

**Tier 2, about 100,000 mandates.**
Two things start to hurt. First, the thundering herd: monthly plans mean a large
share of the 100k all have next_charge_at at 00:00 on the 1st. At 00:00:00 your
single cron tries to fire tens of thousands of debits at once. The banks and
NPCI throttle you, and NPCI explicitly asks you to avoid peak windows (it
recommends not executing during roughly 10:00 to 13:00 and 17:00 to 21:30).
Second, one worker cannot keep up with the network latency of thousands of
sequential bank calls.

Fixes, all standard: (a) **spread the due times.** Do not schedule everyone at
00:00. Jitter each subscription's charge time across its due day (or across the
allowed windows), so load is a smear, not a spike. (b) **A worker pool with
atomic claim.** Instead of one cron, run many workers that each pull a batch of
due rows using `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 100`. SKIP LOCKED is the
key: two workers never grab the same subscription, and no worker waits on
another's lock. (c) **A real queue** in front of the bank calls so you can rate
limit and retry without holding a database transaction open.

**Tier 3, 10 million plus mandates (the Netflix-India, Hotstar, Spotify-India
reality, all renewing on the 1st).**
Now the single database and the single scan are the bottleneck, and the
"everyone due on the 1st" clustering is catastrophic if unmanaged: you might have
several million debits legally due in the same few hours, against dozens of banks
that each have their own rate limits and their own bad minutes.

What survives at this tier (this is the inference layer, grounded in how large
schedulers are built):

- **Shard the schedule store by mandate or customer id.** Each shard owns its own
  slice of subscriptions and its own due-index. Ten shards, ten parallel scans,
  no cross-shard coordination needed because each debit is self-contained. This
  is the same "partition by a self-routing key" trick that let Notion shard by
  workspace and Stripe shard by account.
- **Time-bucketed queues instead of a giant SELECT.** Rather than repeatedly
  scanning a 10M-row table for "who is due," push each timer into a bucket keyed
  by its fire minute. A Redis sorted set with score = fire timestamp, or a delay
  queue, turns "what is due now" into "pop the front of the queue," O(log n) per
  item instead of a full scan. The min-heap abstraction, externalized.
- **Per-bank rate limiters and backpressure.** Each downstream bank gets its own
  token bucket. If HDFC is slow tonight, its queue drains slower and the rest
  keep flowing. Debits that cannot go now get re-enqueued for a few minutes
  later, still inside the legal window. This is load shedding for money.
- **Idempotency becomes non-negotiable.** At 10M debits a night, network drops
  are not rare events, they are a constant. Without the (subscription, cycle,
  sequence) idempotency key, a small retry rate becomes thousands of double
  charges and a regulatory incident. The key is what makes "retry freely" safe.
- **Spread across days, not just hours.** Because the pre-debit notification and
  the retry windows already stretch each cycle over 24 to 72 hours, you can and
  should smear the 1st-of-month load across a longer band. The clock is your
  friend: a subscription due "in early October" does not have to fire at one
  instant.

The through-line at every tier is the same pattern this ledger keeps finding:
**push the expensive, clustered, whole-corpus work off the live path and spread
it over time.** For search that meant precomputing indexes offline. Here it means
the debit is not a live user request at all, it is a background event you are
free to schedule, jitter, batch, and retry whenever the rails and the law allow.
The user is asleep. Latency does not matter. Only correctness and throughput do.
That is a rare luxury, and the whole architecture exploits it.

---

## 8. The retention and habit mechanic

This is the purest retention feature in the entire ledger, because retention is
not a side effect here, it is the literal product.

The loop is: subscribe once, then do nothing, and keep paying. The genius and the
danger both live in the same fact, silence equals yes. A normal purchase asks
"buy again?" every single time and most people say no most of the time. A
subscription flips the default. Now the customer has to take an action to stop.
Inertia, which usually works against a business (people forget to come back), is
turned into the business's best friend (people forget to leave).

Which metric does it move? All three, but the sharp one is **retention through
reduced involuntary churn.** There are two kinds of churn. Voluntary churn is Dr.
Rao deciding to cancel, you cannot engineer that away and should not try.
Involuntary churn is Dr. Rao wanting to stay but his card expired or his account
was short on the 1st. That is a payments failure masquerading as a customer loss,
and it is enormous: a meaningful slice of subscription "cancellations" are really
just failed debits that were never retried. This is exactly the hole the October
2021 e-mandate rollout blew open in India, when auto-renewals across Netflix,
Google Play, and Amazon Prime failed in bulk and businesses lost paying customers
overnight not because those customers left, but because the charge could not go
through. The whole point of the retry engine (Razorpay's Revenue-Protect,
Stripe's Smart Retries, and every serious dunning system) is to win back that
involuntary churn: retry on the day the salary lands, retry when the bank is back
up, nudge the customer to fix a dead card. Every recovered debit is revenue and
retention that would otherwise have silently leaked.

A concrete observed example of the mechanic: when RBI capped no-OTP recurring
card debits at first 5,000 and later 15,000 rupees per transaction, subscriptions
priced just under the cap kept auto-renewing smoothly while anything above it
forced the customer to re-authenticate each time, and the re-authentication step
crushed conversion. Businesses responded by pricing plans to sit under the cap.
The cap literally reshaped pricing, because the frictionless silent renewal is
worth more than the extra rupees. That is how strong the "do nothing" habit loop
is: companies would rather charge less than break it.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. Bias toward scalability and
performance. Here is the concrete lesson from a recurring-payments scheduler.

**The whole subscription engine is a lesson in coalescing and spreading clustered
work off the hot path, gated by idempotency. Your runtime has the exact same
shape, and most VFX runtimes get it wrong.**

Think about what happens in a shader graph at runtime. A hundred parameters
change on the same frame (a resize, a theme swap, a data-driven binding update).
The naive runtime does what the naive scheduler does: it fires everything at
once. Every dependent node recompiles or re-evaluates on that one frame, you get
a stall, a dropped frame, the visual "thundering herd" at 00:00 on the 1st. The
subscription engine's answer maps straight over:

1. **Coalesce with idempotency keys.** The scheduler dedupes debits with a
   (subscription, cycle, sequence) key so a burst of retries becomes one real
   charge. Your runtime should dedupe invalidations with a stable key per
   (node, dirty-reason) so a burst of 100 parameter writes that all invalidate
   the same shader becomes exactly one recompile, not 100. Mark dirty, do not
   recompute; recompute once at the frame boundary. Same idea: make the
   expensive action idempotent under repeated triggers.

2. **Spread expensive work across time, because most of it is not latency
   critical.** The subscription debit is deliberately jittered across a 24-to-72
   hour window because the user is asleep and only throughput matters. In your
   runtime, most recompiles are not needed *this* frame either. Give the compiler
   a per-frame budget (say 4 milliseconds), push pending recompiles into a
   priority queue keyed by "how visible / how soon needed," and drain the queue
   across several frames instead of blowing the frame on frame zero. A slightly
   stale shader for two frames is invisible; a 40 millisecond stall is not. That
   is the exact "the user is asleep, spread the load" trade, applied to frames
   instead of billing days.

3. **Per-resource rate limiters with backpressure.** The scheduler gives each
   bank its own token bucket so one slow bank does not stall the rest. Your
   runtime should give the GPU upload path and the shader-compile path their own
   budgets, so one heavy 4K texture upload or one giant shader does not starve
   every other effect. Backpressure, not a free-for-all.

The deep principle, and it is the same one this ledger keeps landing on: separate
*deciding what work to do* from *when to actually do it*, put a queue between
them, make the work idempotent so you can retry and coalesce freely, and spread
the clustered load across the time axis you are lucky enough to have. Razorpay
has the billing calendar. You have the frame clock. Use it the same way.

---

## Sources

- NPCI, UPI AutoPay announcement (launched at the Global Fintech Festival,
  22 July 2020): https://www.npci.org.in/what-we-do/upi/upi-autopay
- Razorpay Docs, Subscriptions overview:
  https://razorpay.com/docs/payments/subscriptions/
- Razorpay Docs, Subscriptions States (the confirmed state machine: created,
  authenticated, active, pending, halted, cancelled, completed, expired):
  https://razorpay.com/docs/payments/subscriptions/states/
- Razorpay Docs, UPI AutoPay (S2S recurring integration, token flow):
  https://razorpay.com/docs/payments/payment-gateway/s2s-integration/recurring-payments/upi/
- Razorpay Blog, "Introducing UPI AutoPay on Razorpay Subscriptions":
  https://razorpay.com/blog/what-is-upi-autopay-recurring-payments-razorpay-subscriptions/
- Razorpay Blog, "UPI AutoPay with Intelligent Revenue-Protect":
  https://razorpay.com/blog/upi-autopay-with-intelligent-revenue-protect/
- Razorpay Blog, "What is a UPI Mandate":
  https://razorpay.com/blog/what-is-upi-mandate/
- TechCrunch, "India's stringent recurring payments rule goes into effect"
  (2021-09-30): https://techcrunch.com/2021/09/30/india-recurring-payments-rbi/
- Medianama, "Netflix starts accepting UPI AutoPay as RBI auto-debit regulations
  for cards loom" (2021-09):
  https://www.medianama.com/2021/09/223-netflix-upi-autopay-rbi-regulations/
- Medianama, "Google halts recurring payment options on Play Store" (2021-05):
  https://www.medianama.com/2021/05/223-google-recurring-payments-play-store/
- Business Standard, "RBI enhances limit for e-mandates on credit/debit cards to
  Rs 15,000" (2022-06-08):
  https://www.business-standard.com/article/finance/new-e-mandate-guidelines-rbi-enhances-limit-for-e-mandates-on-credit-debit-cards-to-rs-15-000-122060800417_1.html
- Business Standard, "RBI raises limit of e-mandates for recurring online
  transactions to Rs 1 lakh" (2023-12-08):
  https://www.business-standard.com/economy/interviews/rbi-raises-limit-of-e-mandates-for-recurring-online-transactions-to-1-lakh-123120801110_1.html
- PayU Docs, Pre-Debit Notification API and Recurring Payment Transaction API
  (the check-mandate, notify, then execute sequence):
  https://docs.payu.in/reference/pre_debit_notification_api
- NovoJuris, "RBI Guidelines on E-mandates for recurring transactions":
  https://www.novojuris.com/thought-leadership/rbi-guidelines-on-e-mandates-for-recurring-transaction.html
- PostgreSQL docs, SELECT ... FOR UPDATE SKIP LOCKED (the atomic batch-claim
  primitive for a due-item worker pool):
  https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
