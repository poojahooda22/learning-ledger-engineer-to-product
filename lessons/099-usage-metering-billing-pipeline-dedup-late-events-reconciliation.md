# Day 99: Every API call, token and GPU second must become a line on an invoice, exactly once. How usage metering (Stripe Billing Meters, OpenMeter, ClickHouse) turns a firehose into a number you can charge for

**Date:** 2026-10-26
**Difficulty:** Advanced (builds on idempotency Day 12, exactly-once stream processing Day 40, double-entry ledgers Day 82, probabilistic counting Day 28)
**Topic:** Usage-based pricing means money is computed from events. A lost event is lost revenue. A duplicated event is an overcharge and a support ticket. Unlike a dashboard, a billing count has to be exactly right, auditable, and explainable to a customer who disputes it.
**Stack relevance:** Rare.lab will almost surely sell compute: AI generations, cloud compiles, bandwidth for shipped embeds. Section 9.
**A note on the pasted transcript:** the LeetCode mock interview is already taught in [Day 74](074-leetcode-online-judge-end-to-end.md) (sandbox Day 50, leaderboard Day 54). I did not repeat it. Section 0 reuses its interview recipe on today's topic, and Section 10 summarizes the interview in plain language again, as you asked.
**Honesty labels:** CONFIRMED = read in a search result excerpt today. Limit: direct page fetches failed (network blocked), so I never read a full primary article. Everything comes from search excerpts of official docs and vendor blogs. INFERENCE = my reasoning, labeled. MEMORY = standard knowledge I did not re-read today. Arithmetic is shown so you can check it.

---

## 0. The interview recipe applied to a metering pipeline

The LeetCode candidate went: functional requirements, non-functional requirements, entities, API, naive design, deep dives. Same recipe.

1. **Functional.** Services emit usage events ("customer 42 used 1,830 tokens"). The system aggregates per customer per billing period. It shows live usage, enforces limits or credits, and produces an invoice line.
2. **Non-functional.** Correct above all (no loss, no double count). Accepts a very high event rate without slowing the product. Late events handled. Every invoice line traceable to raw events. Availability matters for ingest (never block the product), consistency matters for the final bill.
3. **Entities.** Event, meter (what to count and how to aggregate), customer, billing period, invoice line, adjustment (credit or correction).
4. **API.** `POST /events {id, customer, meter, value, timestamp}`, `GET /usage?customer&meter&from&to`, a webhook for "usage threshold crossed", an invoice finalization job.
5. **Naive design.** Product calls `UPDATE usage SET total = total + 1830 WHERE customer = 42` in Postgres on every request.
6. **Deep dives.** Hot rows, duplicates from retries, late events, replay and reconciliation, real-time limits versus end-of-month billing.

Note the candidate in the transcript made the same first move: poll and recompute from the database. Then caching. Then a queue as a shock absorber. Metering follows the same path, with correctness added on top.

---

## 1. Named companies and the breaking numbers

- **Stripe Billing Meters (CONFIRMED via excerpts of official docs).** A meter defines what to measure and how to aggregate it over a billing period. A meter event carries an event name, a customer identifier, a numeric value, an optional timestamp, and an optional unique identifier used for idempotency (if you omit it Stripe generates one). The timestamp must be within the past 35 calendar days and no more than 5 minutes in the future. Events are processed asynchronously, so summaries and upcoming invoices can lag, and processing errors arrive later by webhook, not in the create response. A success response means "accepted", not "counted on an invoice".
- **Stripe rate limits (CONFIRMED via excerpts, with a conflict).** The classic meter event endpoint: 1,000 calls per second per account, and one concurrent call per customer per meter. The newer v2 meter event stream (requires a 15 minute session token): 10,000 events per second, with higher limits by arrangement. The excerpts disagree about the higher ceiling (200,000 per second on one page, 100,000 in a 2024 changelog excerpt). Treat that number as unverified. One third-party blog says the old endpoint is 100 per second, which conflicts with Stripe's own page, so I trust Stripe's page.
- **OpenMeter (now part of Kong Metering and Billing) (CONFIRMED via excerpts of its docs and a ClickHouse Cloud case-study post).** Events arrive in CloudEvents format through an API, are published to Kafka, then a custom Go consumer deduplicates and inserts into ClickHouse in batches. It scales by Kafka partition count. Their first stack was Kafka plus ksqlDB (one-minute tumbling-window aggregation) plus Postgres, and it hurt because ksqlDB had no clustering, forcing manual sharding. They moved to ClickHouse. They describe the target as "millions of billable events per second" that cannot be sampled or rolled up at ingest. They say ClickHouse performs best when writes are batched, up to about 100,000 records per batch. Their stated reason for exactly-once: a consumer can fail to acknowledge an offset after processing, so without dedup you either drop or double-insert usage.
- **Third-party practice (CONFIRMED as claims, weak authority):** community guides suggest deterministic idempotency keys, a lateness window (one example uses 48 hours) with watermarks, and sending only deltas to avoid drift. One author argues to keep your own usage ledger and build invoices yourself. Treat these as opinions, not proofs.

**The breaking number (INFERENCE arithmetic, labeled):** an AI product doing 5,000 requests per second, each producing one billing event, makes 5,000 increments per second. If the top 10 customers cause 60 percent of them, a single Postgres row per customer is updated about 300 times per second per big customer. Row-level locks serialize those updates. Above a few hundred updates per second on one row, request latency starts to include waiting on a billing counter. At the month boundary, the same table is also being read by an invoice job scanning millions of rows. The product slows down because of its own accounting.

---

## 2. Why the naive design dies

**Failure 1: The hot counter row.**
- `UPDATE usage SET total = total + n` on one row takes a lock. Concurrent requests queue behind it. This is the "10,000 writes per second to one row" problem from Day 1 and Day 16, now sitting in the middle of your request path.
- Analogy: one cashier who must write every sale in a single paper ledger before the customer can leave the shop.

**Failure 2: Retries create duplicates, and a counter cannot tell.**
- The client times out after the increment succeeded, and retries. The row now says 3,660 instead of 1,830. A plain counter has no memory of what it already counted.
- Analogy: tearing a ticket stub twice because the first tear was not visible.

**Failure 3: Late and out-of-order events.**
- A mobile app was offline for 3 days. Its events arrive after the invoice was finalized. Where do they go? A running total has no notion of "this event belongs to last month".
- Analogy: a receipt arriving after you closed the books.

**Failure 4: No audit trail.**
- Customer says "I did not use 1.2 million tokens". With only a total, you cannot show the 1.2 million events. Support cannot answer. Finance cannot reconcile.
- Analogy: a bank that only shows your balance and throws away the transactions.

**Failure 5: Coupling the product to the billing database.**
- If the billing DB is slow or down, every API call fails. Revenue accounting should never be able to take down the product.
- Analogy: the shop closes because the receipt printer jammed.

---

## 3. The architecture, top to bottom

```
Product services (API gateway, workers, GPU jobs)
   |   emit a usage event AFTER doing the work: {event_id, customer, meter, value, ts}
   |   fire and forget into a local buffer; never block the user request
   v
Ingest API (stateless, many copies behind a load balancer)
   |   validate shape, authenticate, reject absurd timestamps, ack fast ("accepted")
   |   job: be cheap and always up.   Analogy: a mailroom that stamps and receives, never opens
   v
Durable log (Kafka / Redpanda / Pub/Sub), partitioned by customer_id
   |   job: absorb spikes, keep order per customer, allow replay.   Analogy: a conveyor belt with a memory
   v
Consumer group (one worker per partition)
   |   dedup by event_id, batch, write
   |   job: turn at-least-once delivery into effectively-once storage
   v
Raw event store (append-only, ClickHouse or object storage + Parquet)
   |   job: the source of truth, immutable.   Analogy: the bank's transaction list
   v
Aggregates (per customer, per meter, per minute/hour/day buckets)
   |   job: cheap reads for live usage, limits and the invoice
   |   Analogy: the running subtotal on the till display
   v
   +--> Real-time path: usage dashboard, soft/hard limits, credit balance (seconds fresh, approximate OK)
   +--> Billing path: period close, apply late-event rules, reconcile against raw events, write invoice lines
                       (minutes to hours, must be exact)
   v
Invoice + payment (Stripe invoices, double-entry ledger from Day 82)
```

Key shape: **one write path (append events), two read paths (fast and approximate, slow and exact).** This is the same split as Netflix Top 10 (fresh sketch versus audited batch) and the same shock absorber as Day 9.

---

## 4. The transferable mechanisms (6)

1. **Idempotency key on every event.** The producer generates `event_id` once, at the moment the work happens, and reuses it on every retry. Stripe has exactly this field. The consumer keeps "seen ids" (a unique key in the store, or a Redis set with a TTL longer than the longest retry plus lateness window). A retry then writes zero new rows. The id must be deterministic from the work (for example `job_id + step`), not a random value made at send time, or a retry made by a different process looks new.
2. **Append-only log, aggregate later.** Never increment a counter in the request path. Append an event (cheap, no contention) and derive totals. A total is a view of the log, so it can always be rebuilt. This removes the hot row entirely.
3. **Partition by customer.** Events for one customer go to one partition, so order per customer is preserved and one consumer owns that customer's dedup state. The cost: a huge customer makes a hot partition (Day 16). Fix by sub-partitioning that customer by a salt and summing the pieces, since sums are commutative.
4. **Time-bucketed pre-aggregation.** Store per minute and per hour sums per (customer, meter). A month query adds 720 hourly rows, not 90 million raw events. Sums, counts, min and max merge safely. "Distinct users" does not merge exactly, so use a HyperLogLog sketch (Day 28) for the live view and recompute exactly for the bill.
5. **Watermark plus lateness window.** The system declares "events for hour H are final once the clock passes H plus 48 hours" (INFERENCE: choose from your real client-offline behavior; Stripe's 35 day acceptance window is a different, wider knob). Events inside the window update the bucket. Events after it become an explicit adjustment line on the next invoice, never a silent rewrite of a closed invoice.
6. **Reconciliation as a first-class job.** Daily: re-sum raw events and compare with the aggregate. A mismatch pages a human. Also compare product-side counts (requests served) with metering-side counts (events stored). Silent loss shows up as a ratio below 1.0. Billing bugs are found by diffing two independent counts, not by reading code.

Bonus: **separate "accepted" from "billed".** Stripe's own API returns success for acceptance and reports errors by webhook later. Your design should do the same and give customers a place to see rejected events.

---

## 5. The trade-offs accepted

- **Consistency versus availability, per data type.**
  - Event ingest: favor availability. If the log is up, accept. Buffer locally on the producer if the ingest API is down. Never fail the user's request because metering is unhealthy.
  - The invoice total: favor consistency. It is computed from the immutable log in a batch, can take minutes, and must be exactly reproducible.
  - Live usage display and spend alerts: eventual consistency is fine (seconds stale).
  - Hard limits (stop at the plan cap): the awkward middle. A strict real-time cap needs a synchronous check against a counter, which brings back the hot row. Common compromise (INFERENCE): a local token bucket per gateway with periodic reservation of quota from a central counter, accepting a small overshoot (say 1 to 2 percent of the cap) in exchange for no per-request coordination.
- **Cost versus latency.** Keeping every raw event is more storage than keeping totals. With 5,000 events per second at 200 bytes each, that is 1 MB per second, 86 GB per day, about 2.6 TB per month before compression (arithmetic from my own assumption). Columnar compression in ClickHouse or Parquet typically shrinks such data a lot, but I did not verify a ratio today. The raw log buys auditability and replay, and you can tier old months to cheap object storage.
- **Exactly once is a combination, not a feature.** At-least-once delivery plus idempotent writes equals effectively once. OpenMeter's own stated reason is the unacknowledged offset case above.
- **Rounding and units.** Decide the smallest unit (1 token, 1 GPU millisecond) and store integers. Floating point sums drift across millions of adds.

---

## 6. The systems-thinking lens

**Feedback loop: the retry duplicate spiral, and its slow cousin, the reconciliation drift.**

- Fast loop: the ingest API gets slow, clients time out, clients retry, retries add load, the API gets slower, more timeouts. Without idempotency, each retry also adds a duplicate and the bill inflates during the very incident you are fighting. Customers see overcharges exactly when trust is lowest.
- Slow loop: nobody reconciles, small losses accumulate (a dropped event here, a double count there), finance notices at quarter end, and the fix is manual spreadsheets. Drift compounds because there is no signal to correct it.
- **Senior fix breaks the loops, not just adds servers:** (a) idempotency key so a retry is harmless, (b) the producer buffers locally and backs off with jitter (Day 98), (c) the log decouples ingest speed from processing speed so a slow consumer does not slow the product, (d) a daily reconciliation job turns silent drift into a visible alert. Adding capacity helps the fast loop slightly and does nothing for the slow one.

---

## 7. Scale story at three tiers

**Tier 1: 1,000 events per day (an early product).**
- One Postgres table `usage_events(event_id primary key, customer_id, meter, value, ts)`. `INSERT ... ON CONFLICT DO NOTHING` gives idempotency for free. Totals via `SUM` at invoice time. Works fine.
- What breaks next: the primary key index and the month-end `SUM` start to hurt as rows grow.

**Tier 2: 100,000 events per day to a few million (a growing SaaS).**
- Still Postgres, but add hourly rollup rows maintained by a background job, partition the table by month so old partitions detach cheaply, and move the write off the request path into a queue.
- Real example of the pain: the OpenMeter team's first design (Kafka plus ksqlDB plus Postgres) worked until clustering limits forced manual sharding (CONFIRMED above).
- What breaks next: Postgres write throughput and rollup freshness at tens of thousands of events per second.

**Tier 3: 10 million events per day to millions per second (an AI API, a telecom-grade meter).**
- Kafka partitioned by customer, consumers dedup and batch-insert into a columnar store (ClickHouse), which loves large batches (up to about 100,000 rows per insert, per OpenMeter's post). Pre-aggregate into materialized hourly buckets. Hot customers get salted sub-partitions. Raw data tiers to object storage. Stripe's own ingest ladder (1,000 per second classic, 10,000 per second stream, higher by arrangement) shows the same step-ups as a public product.
- What breaks next: cross-region ingest (events must land in the customer's home region, Day 14), and exact distinct counts over billions.

---

## 8. A worked example, end to end

Customer "Acme" calls an image-generation API. Plan: $0.002 per 1,000 GPU milliseconds, billing month October.

1. A GPU worker finishes job `job_8841` after 1,830 ms. It builds the event `{event_id: "job_8841:gpu_ms", customer: "acme", meter: "gpu_ms", value: 1830, ts: 2026-10-26T09:14:03Z}`. The id is derived from the job, so any retry makes the same id.
2. It drops the event in a local in-memory buffer and returns the image to the user. Billing never blocks the user.
3. A flusher sends a batch of 500 events to the ingest API. The network drops the response. The flusher retries the same batch with the same ids.
4. Ingest validates (timestamp within bounds, value is a non-negative integer) and publishes to the log, partition = hash("acme") mod 64.
5. The consumer for that partition receives the batch twice. It checks ids against the store's unique key. 500 new on the first pass, 0 new on the second. Total stays correct.
6. The hourly bucket for (acme, gpu_ms, 09:00) goes from 4,200,000 to 4,201,830.
7. A dashboard reads that bucket and shows "4.2 million GPU ms today (about $8.40)". Seconds stale, fine.
8. Day 3 of November: a mobile client uploads an event timestamped 2026-10-30. Within the 48 hour lateness window of October close? The window closed on 2026-11-02, so this event is late. It becomes a +$0.003 adjustment line on the November invoice, labeled "late usage from October".
9. Nightly reconciliation: GPU workers report 41,220,118 jobs finished, metering holds 41,220,107 distinct job events. 11 missing. The job pages an engineer, who finds a crashed worker that lost its in-memory buffer. Fix: spill the buffer to local disk. This is why reconciliation exists.

---

## 9. Map to Rare.lab's stack

What Rare.lab already has that helps:
- **Content-addressed immutable scene JSON on R2 plus a manifest.** Immutable, hash-named objects are natural idempotency keys. A "compile this scene" event keyed by `scene_hash + target + runtime_version` is deterministic: the same compile requested twice is the same event id. Charging once for an identical compile is also a product feature (cache hit costs nothing, which you can show).
- **Supabase Postgres with RLS.** Tier 1 and early tier 2 metering fits here: a `usage_events` table with `event_id` as primary key, `ON CONFLICT DO NOTHING`, RLS so a customer can read only their own rows (instant "show me my usage" page). Partition by month early.
- **Cloudflare.** Workers can be the ingest API (stateless, global, cheap) and Queues can be the shock absorber, so no Kafka is needed at your scale.

Patterns to use today:
1. Derive event ids from the content hash plus action. Never random ids at send time.
2. Write usage as an append-only table. Do not keep a `credits_remaining` column you update on every request. Keep a ledger of grants and spends and sum it (Day 82), with a cached balance refreshed in the background.
3. Meter what costs you money: AI generation tokens or seconds, compile minutes, R2 egress bytes (via the embeddable runtime's CDN fetches), storage GB-days.
4. Pick the lateness window deliberately. The embeddable runtime may report usage from browsers that are offline. 48 hours plus an adjustment line is a sane default.

Where the next ceiling is:
- **The single Postgres write path.** Estimated ceiling: low thousands of simple inserts per second on a modest instance (INFERENCE, depends on indexes and instance size). That is far above early traffic, but a viral embed counting every frame render as an event would blow through it. Do not meter per frame. The shared WebGL context should aggregate locally (sum seconds rendered per session) and flush once per minute or on page hide. One event per session-minute instead of 60 per second per viewer is a 3,600x reduction.
- **Hard credit caps.** A strict "stop at zero" check per generation will hit a hot row per customer. Use the reservation idea from Section 5: reserve a chunk of credits per worker, spend locally, return the leftover.
- **Trust.** Before charging real money, ship a customer-visible usage page and a nightly reconciliation job. They cost little and prevent the support problems that kill usage-based pricing.

---

## 10. References, summarized in plain language

**The pasted LeetCode mock interview (YouTube, a Google engineer designing LeetCode).** Plain-language summary of what is inside:
- She starts with functional requirements (list problems, view one and code in any language, submit and get instant pass/fail, a live contest leaderboard) and says what is out of scope (login, payments, analytics).
- Non-functional: highly available, eventual consistency is OK for things like problem counts, low latency (a couple of seconds for results), isolation and security for running strangers' code, scale for a contest of 100,000 users.
- Entities: problem, user, submission, leaderboard. API: `GET /problems?page&limit`, `GET /problems/{id}?language`, `POST /problems/{id}/submit` with code as a string, `GET /leaderboard/{competition_id}`.
- Database: DynamoDB, because there are few relations and no joins. The interviewer pushes back that she needs a user id in the submissions table and discusses partition keys (competition id so contest queries stay together).
- Running code: a bad idea on the API server (malware, CPU hogs, crash risk). VMs are isolated but heavy. Containers are lighter, one pool per language, with a time limit to stop infinite loops. The interviewer notes VMs isolate better and mentions serverless as an alternative.
- Leaderboard: polling the database is wasteful, so cache it. Scale by adding servers, put a queue between the API and the code runners to smooth spikes, retry with exponential backoff. Tests are defined once in JSON and each language gets its own serialize/deserialize layer. Final addition: primary database with replicas that can be promoted.
- Her recommended resources: Hello Interview, and a YouTube channel the transcript spells "Shrians Jen" (the interviewer adds GKCS). I could not map those spellings to exact channels, so verify before relying on them. Full teaching of this: Day 74, Day 50 and Day 54.
- **How it connects to today:** the queue between API and runners is the same shock absorber as the metering log. Each submission run is also a billable unit if LeetCode charged per execution. The same idempotency issue appears: a retried submission must not run or count twice.

**Stripe docs on recording usage with meters.** Explains the event shape (name, customer, value, timestamp, identifier), why you should set the identifier yourself (retries become harmless), the 35 day past and 5 minute future timestamp rule, that processing is asynchronous so summaries lag, and that errors arrive by webhook. Use it as the model for your own event contract.

**Stripe v2 meter event stream.** Describes a higher-throughput endpoint (10,000 events per second, session token valid 15 minutes) versus the older 1,000 calls per second endpoint, and advises backoff on HTTP 429. Lesson: even Stripe tiers its ingest API by throughput, matching Section 7.

**OpenMeter on ClickHouse.** A real build story: API, then Kafka, then a Go consumer that deduplicates and inserts in batches, scaling by partition count; the earlier ksqlDB design hit a clustering wall. Read it to see the three-tier ladder lived out. Caveat: it is a vendor post.

**Third-party guides (DEV Community, community Elixir wrapper docs).** Practical checklists: deterministic keys, watermarks, delta pushes, "keep your own ledger". Useful as prompts for thinking, weak as proof.

**Where to look next on the wider web (not read today, flagged as leads only):** Stack Overflow questions tagged `stripe-billing` for real duplicate-usage bugs; r/SaaS and r/ExperiencedDevs threads on usage-based billing regrets; the Kleppmann book "Designing Data-Intensive Applications" chapters on stream processing and idempotence (MEMORY).

---

## 11. One-line takeaway

Billing is a counting problem with a legal consequence: append immutable events with a deterministic id, never increment a hot counter in the request path, count fast for the screen and exactly for the invoice, and reconcile every night so silent drift becomes a loud alert.

---

## Sources

- Stripe, Record usage for billing with the API: https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage-api (read via search excerpt)
- Stripe, Meter Event Streams v2: https://docs.stripe.com/api/v2/meter-event-streams (read via search excerpt)
- Stripe changelog, usage-based billing v2 meter events API: https://docs.stripe.com/changelog/acacia/2024-09-30/usage-based-billing-v2-meter-events-api (read via search excerpt)
- OpenMeter, How it works: https://openmeter.io/docs/metering/events/how-it-works (read via search excerpt)
- OpenMeter, How OpenMeter uses ClickHouse for usage metering: https://openmeter.io/blog/how-openmeter-uses-clickhouse-for-usage-metering (read via search excerpt)
- ClickHouse, OpenMeter real-time usage-based billing: https://clickhouse.com/blog/openmeter-real-time-usage-based-billing-powered-by-clickhouse-cloud (read via search excerpt)
- OpenMeter, Consistent Kafka consumer: https://www.plushcap.com/content/openmeter/blog/consistent-kafka-consumer (search listing only)
- Kong, Metering and billing events: https://developer.konghq.com/metering-and-billing/events/ (search listing only)
- DEV Community, Usage-based billing, a practical guide for engineers: https://dev.to/nkassetty/usage-based-billing-a-practical-guide-for-engineers-1ncc (read via search excerpt, opinion)
- AppMaster, Usage-based billing data model with Stripe: https://appmaster.io/blog/usage-based-billing-stripe-data-model (search listing and excerpt, opinion)
