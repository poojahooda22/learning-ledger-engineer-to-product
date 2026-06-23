# Day 12 — How does Stripe charge you exactly once even when the network fails?

**Date:** 2026-06-23
**Difficulty:** Advanced+ — Idempotency & Exactly-Once Delivery
**Stack relevance:** Stripe payments, AWS SQS FIFO, Kafka EOS, Supabase edge functions, any booking or payment flow

---

## 1. Named Company + The Breaking Number

**Stripe. 1 in 1,000 HTTP requests times out mid-flight. At 1,000 payment requests/second, that is 86,400 potential duplicate charges per day.**

Stripe processes billions of dollars daily. Every charge is a POST to their API. HTTP is unreliable: the internet drops packets, TLS connections hang, and load balancers reset idle connections. When a POST /charge request times out *after* the server has already charged the card but *before* the 200 OK reaches the client, what does the client do? It retries. Stripe charges the card twice.

**The breaking number: a naive retry-on-timeout design produces a duplicate charge rate equal to your timeout rate. At 1,000 requests/second with 0.1% timeouts, that is 86,400 duplicate charges per day.**

This is the idempotency problem. It is not a theoretical edge case. It is the default behavior of every distributed system. Every message queue, every event bus, every HTTP API delivers messages "at least once" by default, which means sometimes more than once.

---

## 2. Why the Naive Design Dies

The naive version is a stateless POST handler:

```
Client → POST /v1/charges → App server → Payment processor → Postgres → Return 200
```

**Collapse point 1: timeout after processing.**
The server debits the card. The DB write succeeds. The 200 response is routing back through the load balancer. The client's timeout fires at 500ms. The client sees "connection timeout" and retries. The second request debits the same card again. The server has no memory of the first request. Two charges.

**Collapse point 2: load balancer failover mid-request.**
The request lands on Server A. Server A processes the charge, then crashes. The load balancer routes the retry to Server B. Server B sees no record of the original charge and processes it as new. Duplicate charge.

**Collapse point 3: the queue delivers twice.**
Even if you move to a message queue (Kafka, SQS), the queue guarantees "at-least-once" delivery by default. A consumer crashes after processing but before committing its offset. The broker retries the message to a different consumer. Both consumers charged the card. Duplicate.

**The core physics:** networks partition. Servers crash. A timeout means "I do not know if it worked," not "it failed." A retry that does not know whether the first attempt succeeded will always risk running the operation twice.

---

## 3. The Architecture

Top-to-bottom flow:

```
+--------------------------------------------+
|  CLIENT                                    |
|  Generates idempotency key UUID ONCE.      |
|  Attaches the same key to every retry.     |
|  Never generates a new key on retry.       |
+--------------------+-----------------------+
                     |  POST /v1/charges
                     |  Idempotency-Key: abc-123-def
                     v
+--------------------------------------------+
|  LOAD BALANCER / EDGE                      |
|  Passes key through. No dedup here.        |
+--------------------+-----------------------+
                     v
+--------------------------------------------+
|  STATELESS APP TIER                        |
|  Step 1: atomic check-and-insert the key   |
|  Step 2: run the operation                 |
|  Step 3: store the result against the key  |
+----------+-------------------+-------------+
           |                   |
           v                   v
+------------------+  +------------------------+
|  IDEMPOTENCY     |  |  OUTBOX TABLE          |
|  KEY STORE       |  |  (same Postgres txn)   |
|  Postgres unique |  |  Pending downstream    |
|  index or Redis  |  |  events (webhooks,     |
|  NX with TTL     |  |  Kafka messages)       |
+------------------+  +-----------+------------+
                                   |
                                   v
                      +------------------------+
                      |  MESSAGE QUEUE         |
                      |  Kafka EOS             |
                      |  or SQS FIFO           |
                      |  Dedup window: 5 min   |
                      +-----------+------------+
                                  |
                                  v
                      +------------------------+
                      |  DOWNSTREAM WORKER     |
                      |  Checks its own DB     |
                      |  for this key before   |
                      |  charging the card.    |
                      +------------------------+
```

**Each layer's single job:**

- **Client**: prints the receipt number before going to the store. One UUID per logical operation. Same UUID on every retry. Never generate a new UUID on retry.
- **App tier (check-and-insert)**: the gatekeeper. `INSERT ... ON CONFLICT DO NOTHING`. Exactly one request wins. All duplicates see the row already exists and get the cached result.
- **Idempotency key store**: stores `{key, status: processing|complete, result, expires_at: now+24h}`. Postgres with a unique index gives ACID. Redis with `SET NX EX` gives speed but without ACID across the key and the business record.
- **Outbox table**: business record + downstream event written in ONE Postgres transaction. A relay process reads the outbox and publishes to Kafka. Prevents "charged the card but the webhook publish failed."
- **Message queue (Kafka EOS or SQS FIFO)**: SQS FIFO deduplicates on `MessageDeduplicationId` within a 5-minute window. Kafka EOS uses producer epoch + sequence numbers. Second layer of dedup downstream.
- **Downstream worker**: before touching the card network, queries: "have I already processed a charge with this key?" If yes, return the cached result. This is the idempotent consumer pattern.

---

## 4. The Transferable Mechanisms

**Mechanism 1: Idempotency key (UUID per logical operation)**

The client, not the server, generates a UUID before the first attempt. Every retry of the same logical operation sends the same UUID in a header (`Idempotency-Key: uuid`). The server uses it to detect "I've already processed this." The key insight: the client knows this is a retry; the server does not. The UUID bridges that gap.

Stripe requires idempotency keys for all mutating API calls. AWS requires a ClientToken for EC2 instance launches. Postgres `INSERT ... ON CONFLICT` is the server-side primitive.

**Mechanism 2: Atomic check-and-insert**

```sql
INSERT INTO idempotency_keys (key, status, created_at)
VALUES ('abc-123-def', 'processing', now())
ON CONFLICT (key) DO NOTHING
RETURNING id;
```

If the insert returns a row, this is the first request. Process it. If it returns nothing, this is a duplicate. Fetch the stored result and return it without re-running the operation. Postgres's unique index makes this a single atomic operation. Redis `SET key NX` is the equivalent for non-ACID stores.

**Mechanism 3: The outbox pattern**

Never write to the DB and then publish to a queue as two separate operations. If the DB write succeeds and the Kafka publish fails, your DB and your event stream diverge. Instead:

1. Write the business record AND a `pending_event` row to Postgres in one ACID transaction.
2. A relay process polls the `pending_events` table, publishes to Kafka, then marks the row as sent.

The relay can retry safely because the event has not yet left your system. Kafka deduplicates at the consumer side. This pattern eliminates the split between "what happened" (DB) and "what was broadcast" (queue) without 2PC.

**Mechanism 4: At-least-once + idempotent consumer = effectively exactly-once**

No distributed queue can guarantee exactly-once delivery at the transport layer without a performance penalty. The practical solution: the queue delivers at-least-once, and every consumer is idempotent. "Idempotent consumer" means: applying the same message twice produces the same result as applying it once. A consumer that checks "did I already process charge X?" before charging achieves this. This is the Kafka production pattern and the only safe way to build payment consumers.

**Mechanism 5: Fencing tokens**

When a worker holds a distributed lock and tries to write to a resource (a DB row, a file in R2), it can be preempted. A newer worker gets the lock with a higher epoch number called a fencing token. The resource rejects writes from workers with older tokens. This prevents a zombie worker (thought dead, then resumes) from overwriting newer state. Kafka uses generation IDs for consumer groups. etcd and ZooKeeper use monotonically increasing tokens for distributed locks.

**Mechanism 6: Two-phase commit (and why Stripe does not use it)**

2PC is the classic protocol for "commit atomically across two systems." Phase 1: coordinator asks all participants to prepare. Phase 2: if all say yes, commit. The failure: if the coordinator crashes between phases, participants block indefinitely. Stripe uses the outbox pattern + idempotent consumers instead. Same safety guarantee, no blocking failure mode. 2PC is mostly seen inside single distributed databases (Postgres distributed transactions, XA) and is costly at scale.

---

## 5. The Trade-offs

**Consistency vs availability (CAP applied to payments)**

Stripe chooses consistency over availability for payment operations. If the idempotency key store is down, Stripe returns an error rather than risk a double charge. An unavailable Stripe is recoverable (retry later). A double charge is not (chargebacks, customer trust, regulatory risk).

| Data type | Choice | Why |
|---|---|---|
| Idempotency key record | Strong consistency (Postgres unique index) | Must not double-charge |
| Dashboard reads (payment list) | Eventual consistency (read replica) | Stale by 100ms is fine |
| Webhook delivery status | At-least-once + idempotent receiver | Retry is cheaper than missing |
| Analytics, leaderboards | Eventual consistency | Approximation is acceptable |

**Cost vs latency**

Storing idempotency keys in Postgres adds two extra round trips per request (check + update result). At 1,000 requests/second that is 2,000 extra DB operations/second. Significant but manageable with connection pooling (PgBouncer, Supabase's built-in pooler).

Redis is faster (sub-millisecond) but does not give ACID across the key and the business record. If Redis crashes after you store the key but before the Postgres write, the key is gone and the next retry will double-process. Stripe stores keys in Postgres and accepts the latency. The correctness guarantee is worth it.

**Idempotency key TTL**

Stripe expires keys after 24 hours. 1,000 req/sec * 86,400 sec * ~200 bytes/record = ~17 GB/day without expiry. A 24-hour TTL keeps the live table under 17 GB. Use a Postgres `expires_at` index and a scheduled job (or pg_cron in Supabase) to delete expired rows.

---

## 6. The Systems-Thinking Lens

**The feedback loop that kills systems: the retry death spiral**

The loop:
1. Server slows down (GC pause, lock contention, hot shard).
2. Clients time out. Clients retry.
3. Retries add more load to the already-slow server.
4. Server slows further. More timeouts. More retries.
5. Exponential growth in total request rate. Server dies.

This is a positive feedback loop in the bad sense: load causes failures, which cause more load.

Adding idempotency keys alone does not break this loop. They make retries *safe* (no duplicate side effects), but they do not stop the thundering herd. You also need:

**Exponential backoff with full jitter**

```
delay = random(0, min(cap, base * 2^attempt))
```

The jitter is not optional. Without it, 10,000 clients that all timed out at the same second retry at the same second (the thundering herd). Jitter spreads retries across time, reducing peak server load by an order of magnitude. AWS's benchmark shows full jitter reduces retry peak load by ~7x vs pure exponential backoff.

**Circuit breaker**

If more than X% of requests in the last 30 seconds have failed, stop retrying and fail fast. The circuit "trips open" and returns errors immediately rather than forwarding them to an already-saturated server. When the error rate drops below threshold, the circuit resets. Netflix Hystrix, Resilience4j, and Cloudflare's edge workers all implement this pattern.

**Queue as shock absorber**

Instead of the client retrying directly to the server, the operation is placed in a queue. The queue absorbs the spike. Workers drain it at controlled throughput. The payment processor never sees more than its steady-state capacity. This is covered in detail in Lesson 009 (queue as shock absorber) and is the structural fix: the queue decouples arrival rate from processing rate.

**The senior engineer's diagnosis:** the retry storm is not a capacity problem. Adding more servers increases the steady-state limit but does not dampen the feedback loop. The loop breakers are backoff (reduce retry rate), circuit breakers (stop retrying when system is sick), and queues (decouple arrival from processing). Idempotency keys make it safe to retry at all.

---

## 7. Map to Rare.lab's Stack

Rare.lab uses Supabase Postgres with RLS, Cloudflare R2 for content-addressed scene JSON, and an embeddable WebGL runtime.

**What you already do right:**
- R2 with content-addressed keys (SHA-256 of content) is idempotent by design. Writing the same file twice to the same key produces the same result. No duplicate problem at the storage layer.
- Supabase Postgres has ACID transactions and unique constraints. You can implement idempotency key checks today with one table and one upsert.

**Where your next ceiling is:**

1. **Shader compilation jobs**: if a user triggers a compilation and the edge function times out, does the client retry? If so, does the job run twice? Add an idempotency key (hash of user ID + shader content + timestamp) stored in a Supabase `compilation_jobs` table before dispatching the worker. The worker checks this key before starting.

2. **Export and publish events**: when a scene export completes, if you write the result to Postgres and then call a webhook or emit to a Supabase Realtime channel in sequence (not in a transaction), you have the split problem. Write the completion record and a `pending_notification` row together in one transaction. A relay sends the notification.

3. **Stripe billing**: when Rare.lab charges a subscription, pass an idempotency key derived from `user_id + billing_period_start`. Prevents double-charges on network hiccups at month boundaries. Stripe's SDK makes this one line: `stripe.charges.create({...}, {idempotencyKey: key})`.

---

## References — Summarized in Plain Language

**1. Stripe Engineering: Designing Robust and Predictable APIs with Idempotency**
https://stripe.com/blog/idempotency

*What is inside:* Stripe's own engineers explain why they invented the `Idempotency-Key` header and how they store keys in Postgres. They walk through every phase of a request (key created, key locked, operation executed, response stored) and what happens if the server crashes at each phase. This is the source document for the entire pattern. If you read one thing from this list, read this. 15-minute read.

**2. Brandur Leach: Implementing Stripe-like Idempotency Keys in Postgres**
https://brandur.org/idempotency-keys

*What is inside:* A former Stripe engineer goes deeper than the official blog post. Covers the exact Postgres schema, the `atomic_phase` enum (tracking which phase of a multi-step operation completed), and the recovery path when a server crashes mid-transaction. Includes working Go code. If you implement this in Supabase, this is the blueprint. Also explains why some operations are not safe to retry even with idempotency keys (those with visible side effects outside your DB).

**3. Brandur Leach: Transactionally Staged Job Drains in Postgres (Outbox Pattern)**
https://brandur.org/job-drain

*What is inside:* Explains the outbox pattern with working Postgres SQL. Writes a job to a `jobs` table in the same transaction as the business record, then has a background worker drain it. Solves "DB write succeeded but queue publish failed" without 2PC. Directly applicable to Supabase using pg_cron or a Deno background task as the relay.

**4. AWS Builders Library: Making Retries Safe with Idempotent APIs**
https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/

*What is inside:* Amazon's internal engineering playbook, made public. Covers how Amazon implements idempotency across EC2 (ClientToken), S3 (Content-MD5), and DynamoDB (conditional writes). Shows the client-side token generation strategy, server-side storage, and TTL decisions at Amazon's scale (millions of API calls per second). These are battle-tested patterns used inside one of the world's largest distributed systems.

**5. AWS Architecture Blog: Exponential Backoff and Jitter**
https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/

*What is inside:* The definitive short explanation (15-minute read) of why retrying immediately makes things worse and why "full jitter" breaks the thundering herd. Includes simulation graphs showing that full jitter reduces server load by 7x vs pure exponential backoff. Python code examples are directly portable to JavaScript. Read this before you write any retry loop.

**6. Kafka Documentation: Delivery Semantics and Exactly-Once Semantics**
https://kafka.apache.org/documentation/#semantics

*What is inside:* The official Kafka docs explain the three delivery modes: at-most-once (messages can be lost), at-least-once (messages can be duplicated), and exactly-once (EOS). EOS uses idempotent producers (sequence numbers per partition) and transactional writes. The key nuance: Kafka EOS guarantees the message appears in the log exactly once, but if your consumer writes to an external DB, you still need an idempotent consumer. Kafka EOS alone is not sufficient for payment processing.

**7. Martin Kleppmann: Designing Data-Intensive Applications, Chapter 11 — Stream Processing**
https://dataintensive.net/

*What is inside:* Chapter 11 is the clearest explanation of exactly-once semantics in distributed systems in print. Explains why exactly-once at the transport layer is physically expensive, why at-least-once + idempotent consumer is the practical answer, and how Kafka's EOS works under the hood. The whole book is the best single resource for system design at this level. Chapters 8 (distributed systems problems) and 9 (consistency and consensus) are also directly relevant to this lesson. Available as a physical book or e-book.

**8. Google SRE Book: Chapter 22 — Cascading Failures**
https://sre.google/sre-book/cascading-failures/

*What is inside:* Google's Site Reliability Engineers document the retry death spiral from real incidents, with actual numbers. Covers how a 10% server degradation turns into a 100% outage within 60 seconds via retry amplification. Explains circuit breakers, load shedding, and why throwing more capacity at a feedback loop does not fix it. The book is free to read online. This chapter is a 20-minute read and will change how you think about failure modes.

**9. "Idempotency is not just for APIs" — High Scalability Blog**
https://highscalability.com/

*What is inside:* High Scalability aggregates architecture write-ups from engineers at large companies. Searching "idempotency" on the site yields real case studies from companies like Uber (double-booking prevention), Airbnb (payment retry logic), and PayPal (order deduplication). Each post explains the specific failure the team hit, what broke, and the exact idempotency mechanism they added. Less theoretical than a textbook, more useful than a blog post — these are postmortems from production.

**10. "Saga Pattern for Distributed Transactions" — Chris Richardson's Microservices Patterns**
https://microservices.io/patterns/data/saga.html

*What is inside:* The Saga pattern is the alternative to 2PC for multi-service distributed transactions. Instead of a single atomic commit across services, you define a sequence of local transactions, each with a compensating transaction (rollback action). If step 3 of 5 fails, steps 1 and 2 are rolled back via compensating transactions. Stripe uses a variant of this for complex payment flows that touch multiple internal services. The linked page is the canonical pattern description with diagrams. The full book by Richardson is a practical guide to implementing sagas with choreography vs orchestration.

---

*Filed under: distributed systems, idempotency, payments, exactly-once delivery, retry safety*
