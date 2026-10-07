# Day 83: How does Stripe call 100,000 customer servers reliably when half of them are slow, down or lying? Webhook delivery, retries, signing and per-endpoint isolation

**Date:** 2026-10-07
**Difficulty:** Advanced (at-least-once delivery, retry storms, head-of-line blocking, security of untrusted callbacks)
**Topic:** An outbound webhook platform. You hold an event ("payment succeeded"). Thousands of other people's servers want to hear about it. You do not control those servers.
**Stack relevance:** Rare.lab will receive webhooks (Stripe, Razorpay) and will likely send them (render finished, scene published). Section 7 maps both sides onto Supabase, Cloudflare and R2.

---

## 0. The framework (same recipe as Day 74)

1. Functional: customers register a URL and pick event types. When an event happens, POST it to the URL. Show delivery logs. Allow replay.
2. Non-functional: never silently lose an event, tolerate duplicates, one bad customer must not slow the others, let receivers prove the sender is real.
3. Entities: Endpoint (url, secret, status), Event (immutable fact), Attempt (one try), Subscription (endpoint wants event type X).
4. API: `POST /endpoints`, `GET /events`, `POST /events/{id}/resend`.
5. Naive design: inside the request that creates the payment, loop over the customer's URLs and call `http.post(url, body)`.
6. Deep dives: queue plus workers, retry schedule with jitter, per-endpoint isolation, signatures and replay defence, dead letters, circuit breaking.

---

## 1. The company and the breaking number

**Stripe, GitHub, Shopify, Razorpay: each sends webhooks to receivers it does not own.** Take a Black Friday style spike of **10,000 events per second**, each fanned out to an average of 3 customer endpoints, so **30,000 outbound HTTP calls per second to servers you cannot fix.**

The number that breaks the naive design is not 30,000. It is **the slowest customer**. If one endpoint takes 30 seconds to answer, a worker thread that calls it is frozen for 30 seconds. With 1,000 worker threads, you need only **34 such calls per second** to freeze the whole fleet (34 x 30 s = 1,020 busy threads). One sleepy customer server stops webhooks for everyone.

Analogy: a courier with 1,000 vans. One address has a gate guard who makes each van wait half an hour. Soon every van is parked outside that one house and no other parcel moves.

Real provider policies show how this shaped the products (all numbers from public docs or search excerpts, see section 8):
- **Shopify:** retries failed calls up to **8 times in 4 hours**, then **removes the subscription** if failures persist. A community answer cites a **5 second** response limit (the official page I found did not state it).
- **Stripe:** retries in live mode with exponential backoff for up to about **3 days** (via third party write ups, not opened at stripe.com).
- **GitHub:** per Svix's comparison, no automatic retry of failed repository webhooks. You must fetch redeliveries yourself.
- **Svix (a webhook sending vendor):** 8 attempts at roughly immediate, 5 s, 5 min, 30 min, 2 h, 5 h, 10 h, 10 h.

---

## 2. Why the naive design dies

**Naive: call the customer URL inside the code path that made the event.**

- **Coupling.** Payment creation now waits on a stranger's server. A customer outage becomes your checkout outage. Analogy: a shop that will not hand you your receipt until the delivery driver's boss signs for it.
- **Head-of-line blocking.** One shared queue and one worker pool. A slow endpoint holds workers (the 34 per second maths above). Healthy customers wait behind it.
- **Retry amplification.** Customer is down for 10 minutes. You retry every second. 100 events per second times 600 seconds means 60,000 failed calls you made yourself, then a flood the moment they recover, which knocks them over again. This is the retry death spiral (Day 13) pointed at someone else's server.
- **Lost events.** Server crashes after the payment commits but before the loop runs. Nobody ever tells the customer. There is no record that a delivery was owed.
- **Spoofing.** The receiver URL is public. Anyone can POST `{"type":"payment.succeeded"}` and trick a shop into shipping goods unless the receiver can verify who sent it.
- **Duplicates.** Receiver processes the event, then the network drops the 200. Sender sees a failure and retries. The receiver now ships two parcels.

---

## 3. The architecture, top to bottom

```
Business service (creates payment)
        |  one DB transaction: write payment + write Event row   <- "transactional outbox" (Day 17)
        v
Event store ........ immutable Event rows (id, type, payload, created_at)
        |
        v
Fan-out step ....... for each subscribed endpoint, create a Delivery job
        |
        v
Per-endpoint queues  (or one queue keyed by endpoint_id with fair scheduling)
        |
        v
Delivery workers ... stateless, sign, POST with short timeout, record Attempt
        |            \
        |             +--> Circuit breaker per endpoint (open = stop calling, park jobs)
        v
Customer server .... returns 2xx (success) or anything else (failure)
        |
        v
Retry scheduler .... delayed queue: next attempt = backoff + jitter
        |
        v
Dead letter store .. exhausted jobs, visible in dashboard, replayable
```

- **Event store:** the source of truth that the event happened. Job: make "owed a delivery" durable. Analogy: the carbon copy in the receipt book.
- **Fan-out:** one event becomes N delivery jobs. Job: let each endpoint succeed or fail on its own. Analogy: photocopying a letter once per recipient.
- **Per-endpoint queues:** Job: contain the damage from a slow customer. Analogy: separate checkout lanes per customer, so a long queue in lane 7 does not block lane 3.
- **Workers:** Job: sign, send, time out fast, log. Analogy: couriers who wait at most 5 seconds at any door.
- **Circuit breaker:** Job: stop hitting an endpoint that is clearly down. Analogy: the fuse box that trips.
- **Retry scheduler:** Job: bring failed jobs back later with growing gaps. Analogy: calling a busy friend again after 5 minutes, then 30, not every second.
- **Dead letters:** Job: never throw an event away silently. Analogy: the sorting office's "undeliverable" shelf, kept for a person to inspect.

**Worked example.** A Rs 4,999 order on a store. Event `payment.captured` (id `evt_91`) is written in the same transaction as the payment. Fan-out makes two jobs: shop's order server, shop's accounting server. Order server returns 200 in 80 ms: done. Accounting server returns 503: job rescheduled for about 5 s later, then 5 min, then 30 min. At 11 minutes the accounting server comes back and attempt 3 succeeds. The order server was never delayed by the accounting outage.

---

## 4. The transferable mechanisms

1. **Transactional outbox.** Write the event in the same database transaction as the business change. A separate relay reads the outbox and enqueues deliveries. Result: it is impossible to commit a payment and forget to tell anyone, and impossible to tell anyone about a payment that rolled back. This closes the "lost event" hole. (Day 17 covers change data capture, the machinery that often powers the relay.)

2. **At-least-once delivery plus an idempotency key on the receiver.** The sender cannot know if a timeout meant "never arrived" or "arrived, reply lost", so it must retry, so duplicates are guaranteed. Exactly-once over HTTP is not available (Day 12). The fix is shared: the event carries a stable id (Stripe `evt_...`, GitHub `X-GitHub-Delivery`, Standard Webhooks `webhook-id`), and the receiver does `INSERT INTO processed_events(id) ... ON CONFLICT DO NOTHING`, then acts only if the insert happened. Insert first, then act. A check-then-insert pattern races when two copies arrive together.

3. **Exponential backoff with jitter and a cap.** Gap grows geometrically (5 s, 5 min, 30 min, 2 h, 5 h, 10 h) so brief blips recover fast and long outages are not hammered. Jitter (random spread of plus or minus 20 to 50 percent) stops 10,000 jobs that failed together from retrying in the same second. The cap stops a wait stretching to days.

4. **Per-endpoint isolation (bulkheads) and a circuit breaker.** Give each endpoint its own concurrency limit, for example at most 10 in flight at once, so one slow customer can occupy 10 workers, never 1,000. After K consecutive failures, open the breaker: stop calling for a cool-off, keep jobs parked, probe with a single request. This is load shedding (Day 13) aimed at an unhealthy downstream. Shopify's removal of a persistently failing subscription is the slow, human-visible version of the same idea.

5. **Sign the payload, bind it to time.** Compute `HMAC-SHA256(secret, id + "." + timestamp + "." + raw_body)`. Receiver recomputes it over the raw bytes it received and compares in constant time. Reject timestamps older than about 5 minutes (300 seconds is the common default) so a captured request cannot be replayed next week. Publish a list of signatures so the secret can rotate with zero downtime (the Standard Webhooks design). Verify against the raw body, never the parsed and re-serialized JSON, or whitespace changes break the HMAC.

6. **Ack fast, process later (receiver side).** Receiver validates the signature, writes the raw event to its own durable queue or table, returns 200 in milliseconds, and does the real work asynchronously. A handler that does slow work inline will time out, trigger retries, and create its own duplicates. Persist before returning 200, or a crash between the 200 and the write loses the event while the sender thinks it succeeded.

7. **Do not trust order. Fetch current state.** Webhooks across retries arrive out of order (a `subscription.deleted` can land before the `subscription.updated` that preceded it). Treat the event as a doorbell: "something changed on object X". Then read X from the source API, or compare a version number. Plus a periodic **reconciliation job** that lists recent objects through the API to catch anything the retries gave up on.

---

## 5. The trade-offs accepted

- **Consistency vs availability per data type.**
  - Event store and outbox: strongly consistent with the business write. Must be CP. An event that "might" exist is useless.
  - Delivery to the customer: eventually consistent by nature. Seconds to days late is acceptable, never-delivered is not. Choose availability of the sender: never block checkout on a customer.
  - Delivery log shown in a dashboard: can be seconds stale.
- **Order vs throughput.** Strict per-object ordering means one in-flight message per object, a big throughput cost. Providers usually give up on ordering and tell receivers so. If you need order, partition by object id (Day 42 Kafka) so one object's events go down one lane.
- **Retry window length vs cost.** Retrying for 3 days (Stripe) is kinder to receivers with long outages but stores and schedules many pending jobs. Retrying for 4 hours (Shopify) is cheaper and pushes recovery work to the customer through the reconciliation API.
- **Timeout length.** A short timeout (5 to 10 s) protects your workers but marks slow-but-working handlers as failures, producing duplicates. You are choosing between worker safety and duplicate rate. Short timeout plus idempotent receivers is the stable choice.
- **Pull vs push.** Webhooks push, so latency is low but delivery is your problem. An events API the customer polls (Stripe also offers `GET /v1/events`) gives them control and guaranteed catch-up. Mature platforms offer both.
- **Cost.** Per-endpoint queues cost more than one shared queue. Fair scheduling inside a shared queue is the cheaper middle path.

---

## 6. The systems-thinking lens

**The loop: slow receiver, held workers, backlog, more retries.**

```
receiver slows ──> workers stay busy longer ──> queue depth grows
      ^                                              │
      │                                              v
 receiver drowned by    <──  retries + new events  <── delivery latency exceeds timeout
 retried traffic              aimed at same endpoint        (counted as failure, retried)
```

- This is a metastable failure (Day 13/Day 24 flavour): even after the original cause is gone, the backlog of retries keeps the receiver overloaded.
- Adding workers makes it worse. More workers means more parallel calls into the already drowning receiver.
- Senior fixes break the loop:
  1. **Bulkhead per endpoint** so the damage stays on one customer.
  2. **Circuit breaker** so a failing endpoint gets quiet time to recover instead of full-rate retries.
  3. **Jitter and backoff** so recovery arrives as a trickle, not a wall.
  4. **Concurrency cap on resumption.** After recovery, drain the backlog at a bounded rate (for example 5 per second) rather than all at once.
  5. **Short timeout** so no single call can hold a worker long.
  6. **Visible failure.** Email the customer after N failures and disable the endpoint after days, which turns a silent loop into a human decision.

---

## 7. Map to Rare.lab's stack (Supabase Postgres with RLS, R2 immutable scene JSON plus manifest, one shared WebGL context)

**Receiving (Stripe or Razorpay webhooks into Rare.lab). Do this now.**

| Pattern | Rare.lab touchpoint | Status | Action |
|---|---|---|---|
| Verify signature on raw body | Cloudflare Worker or Supabase Edge Function webhook route | Must have | Read `await request.text()`, compute HMAC, constant-time compare, reject stale timestamps. Never `request.json()` first. |
| Idempotent receive | Postgres | Must have | `webhook_events(event_id text primary key, payload jsonb, received_at)`; `insert ... on conflict do nothing returning`. Act only on a fresh insert. |
| Ack fast, process later | Edge function then queue or `pg_cron` / worker | Good fit | Insert the row, return 200, let a worker apply credits. Ties into the Day 82 credits ledger: webhook event id becomes the ledger entry id. |
| Reconciliation | Nightly job | Cheap insurance | List the last 48 h of Stripe events or payments and replay anything missing from `webhook_events`. |
| Do not trust order | Subscription state | Must have | On `customer.subscription.*`, refetch the subscription, do not apply the payload blindly. |

**Sending (Rare.lab telling customers "render done" or "scene published"). Later.**

| Pattern | Rare.lab touchpoint | Status | Action |
|---|---|---|---|
| Immutable event | Scene publish writes immutable JSON to R2 plus a manifest | **Already close** | The manifest flip is the natural outbox event. Write `events` row in the same Postgres transaction as the manifest pointer update. |
| Payload as pointer | R2 content-addressed scene | **Already doing this** | Send `{event_id, type, scene_hash, manifest_url}`, not the scene. Small payload, receiver fetches immutable bytes from the CDN. Retries cost almost nothing. |
| Per-endpoint isolation | Cloudflare Queues, one consumer concurrency limit | Future | Start with one queue, `max_concurrency` low, and a per-endpoint failure counter in a Postgres table. Add a breaker when the first customer endpoint hangs. |
| RLS | Endpoint secrets, delivery logs | Must have | Owner-only read of their own endpoints and attempts. Store secrets encrypted, show once. |

**Where the next ceiling is (inference, not measured):** for a young product the ceiling is not throughput, it is the first customer whose endpoint hangs. A single Worker making inline calls with no timeout will let that one customer stall your publish path. Set a 5 to 10 s timeout on every outbound `fetch` and send from a queue, not from the publish request, before you have a single external webhook consumer. Past that, the next limit is Postgres write rate for the attempts table (every try is a row), which you manage by keeping only recent attempts hot and archiving the rest to R2.

**One-line lesson for Rare.lab:** treat every webhook as an untrusted, duplicated, out-of-order doorbell on both sides: verify the signature on raw bytes, dedupe by event id, ack in milliseconds, and when you send, write the event in the same transaction as the change and isolate each customer's endpoint so one slow server cannot stall the rest.

---

## 8. Sources and what is actually in them

**Honest note on access:** docs.stripe.com and docs.github.com were blocked by the network proxy this session, so I could not open Stripe's or GitHub's own pages. Stripe and GitHub claims below come from search summaries of third party write ups and from Svix's comparison. Verify numbers against the official pages before quoting them. Shopify and Svix facts come from search results that quoted their own docs.

- [Shopify: Troubleshoot webhooks](https://shopify.dev/docs/apps/build/webhooks/troubleshooting). Official policy: failed calls are retried up to 8 times in 4 hours, then the subscription is removed. Monitoring page shows delivery counts and response time for 7 days.
- [Shopify Community: Webhooks retry after an error](https://community.shopify.dev/t/webhooks-retry-after-an-error/24625). Staff answer on warning emails before removal and the 5 second response expectation.
- [Shopify Engineering: Webhook best practices](https://shopify.engineering/17488672-webhook-best-practices). Older post (10 second timeout, 48 hour window cited), so policy has since tightened. Advice: respond fast, queue the work, reconcile through the API.
- [Svix docs: Retry schedule](https://docs.svix.com/retries) and [Svix: webhook best practices, retries](https://svix.com/resources/webhook-best-practices/retries). A sending vendor explains the 8 attempt schedule, only 2xx counts as success, timeouts count as failure, dead letter queue, jitter against thundering herd. Their open source server is a Rust API on Postgres with a pluggable queue (Redis or RabbitMQ), a useful reference design.
- [Standard Webhooks specification](https://www.standardwebhooks.com/) (via implementer docs such as [Linq](https://docs.linqapp.com/guides/webhooks) and [Glean](https://developers.glean.com/guides/triggers/webhook-delivery)). Three headers: `webhook-id` (stable across retries), `webhook-timestamp` (changes per attempt), `webhook-signature` (space separated list `v1,<base64>` so secrets rotate). Signed content is id, dot, timestamp, dot, raw body. 300 second tolerance is the library default.
- [Stripe webhooks in production, third party](https://www.alexcloudstar.com/blog/stripe-webhooks-production-2026/) and [Hookdeck: common outbound webhook mistakes](https://hookdeck.com/outpost/guides/common-outbound-webhook-mistakes.md). Practitioner stories: at-least-once, same event twice, no ordering guarantee, insert-first idempotency, persist before returning 200. Not primary sources, read them as field reports.
- [Stripe docs: Webhooks](https://docs.stripe.com/webhooks) and [GitHub docs: Best practices for using webhooks](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks). The primary sources for the two biggest senders. Not opened (blocked). Read these first when you implement.

**Confirmed vs inference.** Confirmed (from the sources above): Shopify 8 retries in 4 hours then removal; Svix schedule and dead letter design; Standard Webhooks header scheme; at-least-once and no ordering as stated by practitioners. Inference, labeled: the 34 calls per second arithmetic, the 30,000 calls per second scenario, the per-endpoint queue architecture (a synthesis, not any one company's published diagram), the bounded-drain numbers, and all of section 7.

**About the video in your prompt:** the "Design LeetCode" mock interview is already fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md), so I did not repeat it. Its recipe (requirements, entities, API, naive design, breaking number, fix one layer at a time, name the trade-off) is what sections 0 to 6 use here.

**Related lessons:** Day 6 (Stripe correctness), Day 9 (queue as shock absorber), Day 12 (idempotency), Day 13 (backpressure), Day 17 (WAL and CDC), Day 24 (fencing), Day 67 (noisy neighbor), Day 82 (ledger).
