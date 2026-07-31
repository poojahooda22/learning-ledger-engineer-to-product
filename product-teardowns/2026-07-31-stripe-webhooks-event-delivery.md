# Stripe Webhooks: the event delivery system that tells your server "the money moved"

Date: 2026-07-31
Product: Stripe
Feature: Webhooks (the event delivery system: the event object, signed POST, retries with backoff, at-least-once delivery)

---

## 1. The user

Meet Ravi. He runs a small subscription business, a website that sells a
monthly chai-and-coffee box called "Chai Point Home." He is not a payments
expert. He wired up Stripe Checkout in a weekend so customers could pay, and
it works: money lands in his Stripe account.

But Ravi has a problem he did not see coming. When someone pays, his own
server needs to *do* something. Mark the order paid. Reserve a box in the
warehouse. Send a "welcome, your first box ships Tuesday" email. Flip the
customer's account to "active" so they can log in.

Here is the catch. The customer paid on **Stripe's** servers, not Ravi's. The
card got charged inside Stripe's world. Ravi's little server on a cheap cloud
box has no idea it happened. How does Ravi's server find out that, thirty
seconds ago, a real human in Pune tapped "Pay" and the charge succeeded?

That is the exact moment this feature exists for.

---

## 2. The real problem

Think about it like a friend would explain it over chai.

Two computers that do not trust each other and do not share a database need to
stay in sync about one fact: *did the payment go through?*

The naive answer is: "Ravi's server should just ask Stripe." And sometimes it
does. But asking (polling) is wasteful and slow. If Ravi's server checks
Stripe every 10 seconds, "did anything happen? did anything happen? did
anything happen?", that is thousands of pointless calls a day, and the
customer still waits up to 10 seconds staring at a spinner. Multiply that
across two million businesses on Stripe and the polling alone would melt the
servers.

Worse, some events have **no request** for Ravi to hang a response on. A
subscription renews automatically on the 1st of next month at 3am. Nobody is
sitting at a browser. A card that was saved gets declined on renewal because
it expired. A dispute (chargeback) gets filed by a bank four days later. There
is no HTTP request from Ravi waiting for these. They just *happen*, on
Stripe's side, on Stripe's clock.

So the real problem is: **Stripe knows something changed, Ravi needs to know,
and there is no live request connecting them.** Stripe has to reach out and
knock on Ravi's door, reliably, even when Ravi's door is briefly locked (his
server is redeploying, or down for 40 seconds, or slow).

---

## 3. The feature in one sentence

A webhook is Stripe making an HTTP POST to a URL you registered, carrying a
signed "event" object that says exactly what changed, and retrying that POST
for up to three days until your server answers with a 2xx.

---

## 4. Jobs to be done

What is Ravi really hiring webhooks to do?

- "When a payment truly succeeds, tell my server so I can fulfill the order."
  The event `checkout.session.completed` or `payment_intent.succeeded`.
- "When a subscription renews next month at 3am while I sleep, still tell me."
  The event `invoice.paid`.
- "When a renewal card gets declined, tell me so I can email the customer and
  pause their box." The event `invoice.payment_failed`.
- "When a customer's bank files a chargeback days later, tell me so I can
  respond before I lose the money." The event `charge.dispute.created`.
- "Tell me reliably. If my server hiccups for a minute, do not silently drop
  the news. Try again."
- "Let me trust the message actually came from Stripe and not a scammer POSTing
  fake 'payment succeeded' events to my URL."

That last one matters. Ravi's webhook URL is public. Anyone can POST to it. If
a fraudster sends a fake `payment_intent.succeeded`, Ravi must not ship a free
box. So a real job is: **prove this message is genuinely from Stripe.**

---

## 5. How it works for the user

From Ravi's seat it is three things.

1. In the Stripe Dashboard he adds an endpoint: `https://chaipoint.in/stripe/hook`.
   He ticks the events he cares about, say `checkout.session.completed`,
   `invoice.paid`, `invoice.payment_failed`. Stripe hands him one secret string
   that starts with `whsec_...`. He pastes it into his server's config.
2. He writes a small handler at that URL. It reads the request body, checks the
   signature using that `whsec_` secret, and if it is genuine, does the work
   (mark paid, ship box) and returns HTTP 200.
3. He watches the Dashboard's webhook log. Every attempt shows up there: green
   for delivered, red for failed, with the exact request and response. If his
   endpoint fails too long, Stripe emails him and eventually disables it.

That is the whole surface. The depth is all underneath.

---

## 6. The actual flow, step by step

Walk one real event end to end. A customer named Priya buys Ravi's monthly box.

1. Priya finishes paying on Stripe Checkout. The card charge succeeds **inside
   Stripe**.
2. Stripe's core system records the state change and **creates an Event
   object**. It gets an id like `evt_1PgH2xK9...`, a `type` of
   `checkout.session.completed`, a `created` timestamp, an `api_version`, and a
   `data.object` that is the full Checkout Session (amount, customer id, email,
   line items).
3. Stripe looks up which of Priya-and-Ravi's registered endpoints subscribed to
   this event type. Ravi's `https://chaipoint.in/stripe/hook` is a match, since
   he ticked `checkout.session.completed`.
4. Stripe builds an HTTP POST to that URL. The body is the JSON event. It adds a
   `Stripe-Signature` header: a timestamp `t=` and a signature `v1=`.
5. Ravi's server receives the POST. Before trusting anything, it recomputes the
   signature (section 7) using the `whsec_` secret. Match. Genuine.
6. Ravi's handler reads `data.object`, sees Priya's email and the box she
   bought, marks the order paid, reserves stock, queues the welcome email, and
   returns **HTTP 200**.
7. Stripe sees the 200 and marks the delivery **succeeded**. Done. The whole
   round trip took maybe 300 milliseconds.

Now the unhappy path, which is the interesting one:

- At step 6, Ravi's server was mid-deploy and returned a 502 for 45 seconds.
- Stripe sees a non-2xx. It does **not** give up. It schedules a retry.
- Retries follow an exponential backoff: roughly immediately, then about 5
  minutes, then about 30 minutes, then a couple of hours, then every several
  hours, stretched across **up to three days**, roughly 16 attempts in live
  mode. (In a test sandbox Stripe retries only about 3 times over a few hours.)
- Ravi's deploy finishes in a minute. The next retry lands on a healthy server,
  gets a 200, and the event is delivered. Priya's box still ships. Ravi never
  even noticed.
- If Ravi's endpoint had stayed broken for the full three days, Stripe would
  disable the endpoint and email him.

The customer-facing lesson: a 45-second outage did not lose a real order,
because the delivery system is built to keep knocking.

---

## 7. Under the hood, like the engineer

This is the heart of it. A webhook system is a distributed messaging problem
wearing an HTTP costume. Let me build it up the way it is actually solved. I
will mark clearly what Stripe documents publicly versus what is well-grounded
inference about how this class of system is built, because Stripe has not
published a blow-by-blow of its webhook internals.

### 7a. The event object is an append-only log entry (documented)

Every meaningful state change in the Stripe API produces an **Event** object,
and Stripe stores it. You can list them at `GET /v1/events`. They are retained
in full for **30 days** (older than that, only a summary). Each event is
immutable once created and is rendered in the API version that was live when it
was created.

That is a **log**. An append-only, time-ordered, immutable journal of "things
that happened," keyed by `evt_...` id. This single design choice unlocks
everything else:

- Webhooks are just **one delivery mechanism** reading off that log.
- If webhooks fail entirely, Ravi can **poll** `GET /v1/events` and replay the
  last 30 days. The log is the source of truth; the POST is a convenience.
- Idempotency becomes possible, because every delivery carries a stable
  `evt_id` a consumer can dedupe on.

Data structure: think of it as an append-only log or journal, exactly the shape
Kafka gives you (offset-ordered, immutable records). The concrete record here
is `evt_1PgH2xK9 = { type: "checkout.session.completed", created: 1753900000,
data: { the Checkout Session } }`.

### 7b. Matching: from one event to the right endpoints (documented behavior, inferred structure)

When event `evt_1PgH2xK9` is created for Ravi's account, Stripe must answer:
*which endpoints should receive this?* Two filters:

1. Which endpoints belong to this Stripe account.
2. Of those, which subscribed to this event **type** (Ravi ticked
   `checkout.session.completed`; an endpoint that only listens for
   `charge.dispute.created` is skipped).

The natural structure is a hash map keyed by `(account_id, event_type)`
pointing at the set of matching endpoints. This is a lookup, order of a
handful of results, not a scan over millions. It is the same "matching vs
ranking" split you see in search, except here there is no ranking: a webhook
either matches an endpoint's subscription list or it does not. Matching is a
set-membership test (`is "checkout.session.completed" in this endpoint's
enabled_events?`), which is why `enabled_events` is stored as a set, not a
list you scan.

### 7c. Delivery is a durable queue with a delay timer (inference, standard for this class)

Here is the core engine. For each `(event, matching endpoint)` pair, create a
**delivery job**: "POST evt_1PgH2xK9 to https://chaipoint.in/stripe/hook."

Do not POST it inline on the thread that created the event. That would couple
Stripe's payment path to Ravi's flaky server. Instead push the job onto a
**durable queue**. Workers pull jobs and make the HTTP POST.

- On **2xx**: mark delivered, drop the job. Done.
- On **non-2xx, timeout, or connection error**: re-enqueue the job with a
  **future "not before" timestamp** computed by exponential backoff.

That delayed retry needs a time-ordered structure: a priority queue / min-heap
keyed by next-attempt-time, or a sorted set (a Redis ZSET keyed by the unix
timestamp of the next attempt), or a Kafka delay-topic ladder. A scheduler
pops jobs whose time has come and hands them back to workers. Concretely,
Ravi's failed delivery sits in that sorted structure with `score =
now + 30 minutes`; when wall-clock passes that score, it gets retried.

Stripe publicly runs very large Kafka deployments, so a Kafka-backed log plus
a delay mechanism is the well-grounded shape here, but the exact broker and
data structures are not published. Treat 7c as inference, not gospel.

### 7d. At-least-once, not exactly-once, and no ordering guarantee (documented)

Two facts Stripe states plainly, and they fall straight out of 7c:

- **At-least-once delivery.** Because a job is only removed after a confirmed
  2xx, a delivery that succeeded on Ravi's side but whose 200 got lost on the
  way back will be **retried**, and Ravi sees the same event twice. So Ravi
  **must dedupe by `evt_id`**. Store processed event ids; skip repeats. Real
  failure this prevents: shipping Priya two boxes for one payment.
- **No guaranteed order.** Parallel workers plus retries reorder things. A
  `customer.subscription.updated` can arrive before the
  `customer.subscription.created` that logically precedes it. So consumers must
  not assume order; when in doubt, treat the event as a "go look" nudge and
  fetch the current object from the API, whose state is authoritative.

Exactly-once delivery over an unreliable network is effectively impossible, so
the whole industry (Stripe, Kafka, SQS) settles on **at-least-once plus
idempotent consumers.** The dedupe lives on the receiver because only the
receiver knows what "already done" means for its business.

### 7e. Proving it is really Stripe: the signature (documented)

Ravi's URL is public, so every event is signed. The `Stripe-Signature` header
looks like `t=1753900000,v1=5257a869e7...`. Verification:

1. Take the raw request body **exactly as bytes**, before any JSON parsing
   reformats it.
2. Build the signed payload string: `t + "." + raw_body`, for example
   `1753900000.{"id":"evt_1PgH2xK9",...}`.
3. Compute `HMAC-SHA256(signed_payload, whsec_secret)`.
4. Compare, in **constant time**, against the `v1=` value.
5. Reject if the timestamp `t` is more than **5 minutes (300 seconds)** old.
   That window kills replay attacks: even if an attacker captures a real signed
   event, they cannot re-POST it an hour later.

Constant-time comparison matters: a naive `==` that returns early on the first
mismatched byte leaks, through timing, how many leading bytes were right, and a
patient attacker can forge a signature one byte at a time. HMAC uses a shared
secret (`whsec_`), so this proves authenticity and integrity, not encryption;
the body is not secret, it just must be unforgeable and untampered. This is why
Ravi must use the **raw** body: re-serialized JSON (keys reordered, spaces
changed) produces a different HMAC and every event would look forged.

### 7f. The scale story at three tiers

**Tier 1, about 1,000 deliveries a day (a small shop).** A single database
table `deliveries(event_id, endpoint, status, next_attempt_at)` and one worker
that POSTs synchronously is enough. Retries can be a cron job that scans for
rows where `status = failed AND next_attempt_at < now`. At a few thousand rows,
a plain indexed SQL scan is instant. Nothing sophisticated required.

**Tier 2, about 100,000 deliveries a day (a growing platform).** Now the
synchronous worker is a bottleneck, and one bad actor bites you: a single
merchant whose endpoint hangs for 30 seconds per request ties up your workers
and **delays everyone else's events.** This is head-of-line blocking. Fixes:
move to an async queue with a worker pool, add per-endpoint concurrency limits
and timeouts (give a slow endpoint 2 seconds, then move on), and replace the
"scan for failed rows" cron with a proper delay queue (a sorted set keyed by
next-attempt-time) so retries are popped in O(log n), not found by scanning a
growing table.

**Tier 3, 10 million-plus a day, billions a month (Stripe).** Corroborating
industry write-ups put Stripe at billions of webhook deliveries per month with
sub-second latency for the healthy path. At this size:

- **Partition the queue so noisy neighbors are isolated.** Shard delivery jobs
  by endpoint or account. One merchant's rebuild storm or dead endpoint fills
  its own partition and cannot starve the rest. Isolating head-of-line blocking
  is the single most important move at this tier.
- **Circuit-break dead endpoints.** An endpoint failing for three days gets
  auto-disabled and the owner emailed, so Stripe stops burning delivery
  capacity hammering a URL that will never answer.
- **Rate-limit per endpoint** so a burst (a flash sale firing 10,000
  `payment_intent.succeeded` in a minute) is smoothed, not fired all at once at
  a merchant who can absorb 50 per second.
- **Lean on the log for recovery.** Because events live 30 days in an
  append-only store, a merchant who was down for an hour can replay from
  `GET /v1/events` instead of Stripe holding infinite in-flight state. The
  durable log is what lets the live delivery path stay cheap and bounded.
- **Newer escape hatch: Event Destinations.** Stripe now also lets you route
  events to Amazon EventBridge and to "thin" event payloads, pushing the
  fan-out and buffering onto cloud infrastructure built for exactly this.

The pattern across tiers: what breaks next is always **one slow consumer
stalling the shared path.** Tier 1 ignores it, Tier 2 puts timeouts and a delay
queue on it, Tier 3 physically partitions it away.

---

## 8. The retention and habit mechanic

Webhooks are not a consumer dopamine loop like a Stories tray. The habit they
build is **developer and merchant lock-in**, and the metric they move is
**retention** (with a direct line to **revenue**).

The loop works like this. Ravi wires his order fulfillment, his dunning emails,
his access control, and his accounting to Stripe events. Once
`invoice.payment_failed` drives his "your card was declined, please update it"
email, and `customer.subscription.deleted` revokes access, and
`charge.dispute.created` triggers his evidence workflow, **Stripe is now the
heartbeat of his business.** Every critical backend behavior fires off a Stripe
event. Switching processors would mean re-plumbing all of it. That is the
stickiest kind of retention: not "I like it" but "my whole system is shaped
around it."

The feedback loop that keeps developers *trusting* it, and coming back, is the
Dashboard's delivery log plus the retry-and-notify behavior. When something
fails, Ravi sees the exact request and response, watches Stripe retry for three
days, and gets an email before an endpoint is disabled. That visible
reliability is what lets him build more on top, which deepens the lock-in.

The revenue tie is direct and real. A dropped `invoice.payment_failed` means
Ravi never sends the dunning email, the customer silently churns, and the
recurring revenue quietly dies. Stripe's own dunning and "Smart Retries" for
failed subscription charges are event-driven, and Stripe has said recovered
failed payments add up to real percentage points of merchant revenue. Reliable
event delivery is not a nicety; it is money that would otherwise leak.

---

## 9. The lesson for Rare.lab

Rare.lab compiles a node-based shader graph to shippable code and ships an
embeddable runtime. The moment you have an editor on one side and runtimes,
build pipelines, and collaborators on the other, you have Ravi's exact problem:
something changed here, and other machines that do not share your database need
to know, reliably, without polling.

The concrete lesson: **when a graph is published, do not push updates inline
and best-effort. Model it as an event delivery system.**

1. **Emit an immutable event to an append-only log.** On publish, write
   `graph.published = { graph_id, version, content_hash, created }` to a
   durable log, exactly like Stripe's Event object. That log, not the live push,
   is your source of truth. A runtime that missed an update can replay from it.
2. **Fan out through per-subscriber queues with backoff.** An embedded runtime
   on a flaky mobile connection is Ravi's flaky server. Retry the push with
   exponential backoff off a delay queue; do not drop the update because the
   first POST timed out.
3. **Partition by project so one heavy project cannot stall another.** This is
   the Tier 2-to-3 lesson applied directly to Rare.lab. If one artist triggers
   a rebuild storm publishing 500 graph versions in a minute, that must fill its
   own partition and never freeze the live preview of a different project. Head
   of line blocking is your enemy the instant you have more than one user.
4. **Make every runtime consumer idempotent, keyed by version.** Ship
   at-least-once, dedupe on `(graph_id, version)`, and never assume order. If
   `version 7` arrives before `version 6` due to a retry, the runtime keeps 7,
   because higher version wins. This is the "no ordering guarantee, fetch
   authoritative state" rule, and it means your update path can stay cheap and
   parallel instead of forcing a strict sequential channel.
5. **Sign the payloads.** An embedded runtime pulling a shader update from the
   network must verify it came from Rare.lab, HMAC-SHA256 over
   `timestamp + "." + body` with a per-project secret and a short timestamp
   tolerance, or an attacker on the same URL could push a malicious shader to
   every embed.

The one-line version: keep the expensive, authoritative record as an immutable
append-only log, and treat every push to a runtime as a cheap, retriable,
signed, idempotent, partitioned delivery off that log. That is how you notify
millions of embeds about a graph change without the notify path ever becoming
the bottleneck.

---

## Sources

- Stripe Docs, Receive Stripe events in your webhook endpoint (delivery, retries up to 3 days, endpoint auto-disable): https://docs.stripe.com/webhooks
- Stripe Docs, Check webhook signatures (Stripe-Signature header, signed payload = t + "." + body, HMAC-SHA256, 300s tolerance, constant-time compare): https://docs.stripe.com/webhooks/signature
- Stripe API Reference, List all events (30-day retention, types filter up to 20, event object shape): https://docs.stripe.com/api/events/list
- Stripe Support, Stripe event retention period (30 days full, summary after): https://support.stripe.com/questions/stripe-event-retention-period
- Svix, Stripe Webhooks Review (retry schedule, signature scheme deep dive): https://www.svix.com/resources/webhook-reviews/stripe-webhooks-review/
- WebhookWatch, Stripe Webhook Retry Policy Explained (non-2xx including 4xx triggers retries): https://www.webhookwatch.com/article/stripe-webhook-retry-policy-explained
- HookRelay, Stripe Webhook Retry Behavior Explained (approx 16 attempts over 3 days, backoff ladder): https://www.hookrelay.io/guides/stripe-webhook-retry
- DEV Community, At-least-once vs exactly-once delivery guarantees (why at-least-once plus idempotency is the industry norm): https://dev.to/letanure/at-least-once-vs-exactly-once-understanding-message-delivery-guarantees-4bhj
- Hookdeck, Webhook Infrastructure: What It Takes to Receive and Send Events Reliably at Scale (queues, retries, partitioning, Stripe scale context): https://hookdeck.com/webhooks/guides/webhook-infrastructure-guide
