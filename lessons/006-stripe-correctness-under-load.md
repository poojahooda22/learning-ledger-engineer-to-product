# Day 6 -- How does Stripe stay correct when 10,000 retries hit one row?

**Date:** 2026-06-16
**Topic:** Financial correctness at scale: idempotency, atomic phases, immutable ledgers, DocDB sharding
**Difficulty:** Advanced
**Prereqs:** HTTP and REST basics, what a database transaction is, Day 3 (Figma -- distributed state), Day 5 (YouTube -- sharding concepts)

---

## 1. The company and the breaking number

**Stripe.** More than $1 trillion in payments processed annually. More than 5 million database queries per second across its production fleet. More than 2,000 database shards running the internal document store called DocDB.

The specific breaking number: **one payment row receiving 10,000 concurrent write attempts per second on a naive single-Postgres setup causes row-lock serialization that reduces effective throughput to hundreds of writes per second and queues requests until they time out.**

But the more dangerous breaking number is this: **a card-network call that times out at 30 seconds causes the merchant server to retry -- and a non-idempotent API charges the card a second time.**

A Stripe engineer named Brandur Leach wrote the canonical piece on this in 2017. He started with this observation:

> "Consider a simple payment flow: validate the card, charge the card, email the receipt. What happens if the server crashes after step 2 but before step 3? What if the charge request succeeds at the card network but the response never reaches your server because the network dropped the packet? The client retries. You just charged the card twice."

That is the problem. It is not about raw throughput. It is about **correctness under retries**. And retries happen constantly in distributed systems: network blips, server restarts, deploy-time pod evictions, client timeouts. A payment system that cannot survive a retry is not a payment system; it is a liability.

Real example: Kavya buys a subscription for 999 INR on a small SaaS product built on Stripe. Her Mumbai network drops the response packet after Stripe has successfully authorized her card. The SaaS server's SDK retries. Without idempotency, Kavya's card gets charged twice. Her bank flags it as a suspicious duplicate. Stripe processes a dispute. Everyone loses.

---

## 2. Why the naive version collapses

**The naive design:** A single Postgres instance. A `payments` table with an `amount` column and a `status` column. A synchronous call to the card network inside the HTTP handler. The handler flow:

```
1. SELECT * FROM cards WHERE id = $1 FOR UPDATE  -- lock the card row
2. POST https://visa.gateway.com/authorize        -- charge the card
3. UPDATE payments SET status = 'captured'        -- mark as done
4. Send email receipt
```

Four reasons this collapses at Stripe's scale:

**Problem 1: SELECT FOR UPDATE serializes everything.**
Row-level locking in Postgres means only one transaction at a time can touch that payment row. A merchant running a flash sale -- say, Zepto running a midnight grocery promo -- gets 10,000 buy requests per second. They all hit Postgres simultaneously. Postgres queues them. Each request holds its lock for 30+ seconds (the card network call). Queue depth explodes. Connections pile up. Postgres runs out of connection slots (default: 100). Every request starts returning "connection refused." The database is not CPU-bound or disk-bound; it is **lock-bound**.

**Problem 2: The network timeout causes double charges.**
The card network call (step 2) takes 200ms to 2 seconds in the best case. Under load it can hit 30 seconds and time out. When it times out, the HTTP handler throws an exception. The `UPDATE payments` in step 3 never runs. The merchant server sees an HTTP 500 (or a connection reset), assumes the charge failed, and retries. But Visa already processed the charge. The retry creates a second charge. The card gets debited twice.

**Problem 3: One database is one failure.**
A disk failure, a kernel panic, or a Postgres vacuum getting stuck can take the entire payments database offline. Every merchant on the platform stops accepting payments simultaneously. At $1 trillion per year, downtime costs Stripe roughly $1.9 million per minute.

**Problem 4: Mutations overwrite state, losing history.**
`UPDATE payments SET status = 'captured'` is a destructive write. If a bug sets status to 'captured' for a payment that was actually declined, there is no audit trail to diagnose it. Financial regulators require an immutable record of every state transition. Overwriting rows violates that requirement.

---

## 3. The real architecture, layer by layer

Read this top to bottom. Every box has one job, named here explicitly.

```
+------------------------------------------------------------------------+
|                  MERCHANT SERVER (Stripe customer)                     |
|                                                                        |
|  Job: generate a globally unique idempotency key per user action.     |
|  The key is a UUID (v4) tied to ONE checkout attempt.                 |
|  If the user clicks "Pay" twice, the server generates ONE key and      |
|  sends it with BOTH requests.                                          |
|                                                                        |
|  HTTP header: Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000  |
|  Body: { amount: 99900, currency: "inr", card: "card_abc123" }        |
|                                                                        |
|  Analogy: a check with a unique serial number. If you deposit the     |
|  same check twice, the bank rejects the second attempt. The number    |
|  is the check; the number is also the idempotency key.                |
+------------------------------------------------------------------------+
         |
         | HTTPS (TLS 1.3, certificate pinned in Stripe.js)
         v
+------------------------------------------------------------------------+
|              RATE LIMITER (4-layer token bucket via Redis)             |
|                                                                        |
|  Job: shed load before it reaches application servers.                |
|                                                                        |
|  Layer 1 -- per-user rate limit:                                      |
|    Every Stripe API key gets a token bucket. Default: 100 req/sec.   |
|    Each request consumes one token. Tokens refill at 100/sec.        |
|    If bucket is empty: HTTP 429, Retry-After header.                  |
|                                                                        |
|  Layer 2 -- concurrent request limiter:                               |
|    Heavy operations (creating a subscription with 10 line items)      |
|    are limited by in-flight count, not just rate. A merchant cannot   |
|    have more than 25 such requests in-flight simultaneously.          |
|                                                                        |
|  Layer 3 -- load shedder (panic mode):                               |
|    When the system is under critical load, load shedders drop         |
|    low-priority traffic (analytics calls, test-mode requests)         |
|    entirely to protect live payment traffic.                           |
|                                                                        |
|  Layer 4 -- per-endpoint rate limit:                                  |
|    Endpoint-specific limits. Creating a customer: 25/sec.            |
|    Listing all charges: 5/sec. Charging a card: 100/sec.             |
|                                                                        |
|  Storage: Redis with consistent hashing. Each merchant's rate-limit   |
|  entry hashes to a specific Redis node. Hot merchants (high-volume     |
|  sellers) do not share a node with low-volume merchants, preventing   |
|  a "noisy neighbor" from slowing everyone's token bucket reads.       |
|                                                                        |
|  Source: Stripe Engineering Blog, "Scaling your API with rate         |
|  limiters," 2017 (stripe.com/blog/rate-limiters)                     |
|                                                                        |
|  Analogy: a nightclub door with multiple policies. Strict per-person  |
|  limit (Layer 1). Capacity cap inside (Layer 2). VIP bypass during   |
|  regular nights, VIP-only during fire-alarm situations (Layer 3).    |
|  Separate lines for different ticket types (Layer 4).                 |
+------------------------------------------------------------------------+
         |
         | Request passes rate limiter
         v
+------------------------------------------------------------------------+
|                STATELESS API SERVERS (Stripe API fleet)                |
|                                                                        |
|  Job: parse the request, authenticate the API key, route to the       |
|  idempotency layer. No local state -- any server can handle any       |
|  request. Deployed on AWS EC2 across 3+ availability zones.           |
|                                                                        |
|  Analogy: ticket windows at a train station. Any window can process   |
|  any ticket. You go to whichever is free. The window does not         |
|  remember you after you leave.                                         |
+------------------------------------------------------------------------+
         |
         v
+------------------------------------------------------------------------+
|                   IDEMPOTENCY LAYER (DocDB table)                      |
|                                                                        |
|  Job: guarantee exactly one outcome per unique (api_key, idempotency  |
|  key) pair, even if the same request arrives 100 times.               |
|                                                                        |
|  The idempotency_keys table (simplified schema):                      |
|                                                                        |
|    key          TEXT  -- the UUID the merchant sent                   |
|    api_key_id   UUID  -- which merchant's key this belongs to         |
|    status       ENUM  -- 'in_flight', 'finished'                      |
|    recovery_pt  TEXT  -- which atomic phase we have reached           |
|    response     JSONB -- the full HTTP response, saved for replays    |
|    locked_at    TIMESTAMP                                              |
|    expires_at   TIMESTAMP  -- 24 hours from creation                  |
|                                                                        |
|  Three outcomes when a request arrives:                               |
|                                                                        |
|  Case 1: Key does not exist.                                          |
|    INSERT a new row with status='in_flight', recovery_pt='started'.  |
|    This INSERT uses a UNIQUE constraint on (api_key_id, key).         |
|    Only one server wins the race; all others get a unique-violation   |
|    error and return HTTP 409 "idempotency conflict."                  |
|    The winning server proceeds to payment processing.                 |
|                                                                        |
|  Case 2: Key exists, status='finished'.                              |
|    Return the saved response body immediately. Zero card network      |
|    calls. Zero new database writes. The merchant cannot tell the      |
|    difference between the original and the replay.                    |
|                                                                        |
|  Case 3: Key exists, status='in_flight'.                             |
|    Another server is already processing this request (or the previous |
|    server crashed mid-flight). Return HTTP 409 and tell the merchant  |
|    to wait and retry after a short pause. The merchant's SDK does     |
|    this automatically with exponential backoff.                       |
|                                                                        |
|  Analogy: a theater box office with a claim-check system. You hand   |
|  in your ticket stub. If it has never been presented, they stamp it  |
|  "in progress" and you wait. If it was already processed, they hand  |
|  you the pre-printed receipt. Two people presenting the same stub     |
|  is a conflict -- only one wins.                                       |
+------------------------------------------------------------------------+
         |
         | Key is new -- begin payment processing
         v
+------------------------------------------------------------------------+
|              ATOMIC PHASES + RECOVERY POINTS                           |
|                                                                        |
|  Job: break the multi-step payment into isolated phases so a crash    |
|  mid-flow can resume exactly where it stopped, without repeating      |
|  completed steps.                                                      |
|                                                                        |
|  A payment has three atomic phases. Each phase is a single database   |
|  transaction. If the server crashes DURING a phase, the transaction   |
|  rolls back (nothing happens). If the server crashes AFTER a phase,  |
|  the recovery point is saved and the next retry skips to that point.  |
|                                                                        |
|  PHASE 1: Validate + Reserve (recovery_pt = 'validated')             |
|  Inside one DB transaction:                                           |
|    - Load the card record.                                            |
|    - Run fraud model score (Stripe Radar, ML model on 100+ signals). |
|    - Create a pending ledger entry: "amount reserved."                |
|    - Update idempotency_keys: recovery_pt = 'validated'.             |
|  If crash here: retry restarts Phase 1 (the transaction rolled back, |
|  nothing was written). Safe to redo.                                  |
|                                                                        |
|  PHASE 2: Charge the card network (the dangerous step)               |
|    - POST to Visa / Mastercard / Rupay authorization API.             |
|    - This call is NOT inside a DB transaction (external side effect). |
|    - It can time out. It can return an ambiguous response.            |
|  If crash here: the recovery point is still 'validated'.             |
|  The next retry re-enters Phase 2 and re-calls the card network.     |
|  The card network call itself is idempotent: Stripe sends a unique    |
|  "merchant reference ID" on each card network call. Visa deduplicates |
|  on their side within a short window. So re-calling is safe.         |
|                                                                        |
|  PHASE 3: Confirm (recovery_pt = 'finished')                         |
|  Inside one DB transaction:                                           |
|    - Write the final ledger entry: debit the buyer, credit the seller.|
|    - Mark the payment as 'captured' in the payments table.           |
|    - Save the full HTTP response in idempotency_keys.response.       |
|    - Update recovery_pt = 'finished'.                                 |
|  If crash here and the transaction commits: idempotency key shows    |
|  'finished', future retries return the saved response. Correct.      |
|  If crash here and the transaction rolls back: recovery point is      |
|  still 'validated', next retry re-runs Phase 2 and Phase 3.          |
|  Visa's dedup handles the repeated card-network call.                |
|                                                                        |
|  Analogy: a moving company with a checklist. Phase 1 is wrapping     |
|  furniture (reversible). Phase 2 is loading the truck (external,     |
|  hard to undo). Phase 3 is signing the delivery receipt (final       |
|  record). If the driver calls in sick after Phase 1, the next driver  |
|  does NOT re-wrap the furniture -- they resume at Phase 2.           |
+------------------------------------------------------------------------+
         |
         v
+------------------------------------------------------------------------+
|              DOCDB (Stripe's document database, 2000+ shards)         |
|                                                                        |
|  Job: store all payment and customer data at massive scale with       |
|  99.999% uptime and zero-downtime migrations.                         |
|                                                                        |
|  What DocDB is:                                                       |
|    A proprietary Database-as-a-Service Stripe built on top of         |
|    MongoDB. Developers at Stripe use it via an API; they never        |
|    manage connection strings, shard counts, or failovers themselves.  |
|                                                                        |
|  Scale numbers (from Stripe's blog and QCon SF 2025 talk):           |
|    - 5 million+ queries per second                                    |
|    - 2000+ shards                                                     |
|    - 5000+ collections                                                |
|    - Petabytes of financial data                                       |
|    - 5.5 nines of uptime (99.999%)                                    |
|    - In 2023, migrated petabytes between shards without downtime      |
|                                                                        |
|  Three layers inside DocDB:                                           |
|                                                                        |
|  Layer 1: Go Proxy                                                    |
|    Every DB query from an API server goes through a fleet of proxy   |
|    servers written in Go. The proxy handles: routing to the correct   |
|    shard, retrying on shard leader failover, connection pooling,      |
|    admission control (reject queries if the shard is saturated), and  |
|    access control (which API service can write to which collection).  |
|    Merchants cannot tell this proxy exists.                           |
|                                                                        |
|  Layer 2: Chunk Metadata Service                                      |
|    A control plane that knows: "collection X, key range Y to Z lives  |
|    on shard 1247." Every Go proxy reads from this service to know     |
|    where to route. When a shard is rebalanced (data moved to a new   |
|    shard to reduce hotspots), the Chunk Metadata Service is updated   |
|    atomically. The proxy picks up the change within seconds.          |
|                                                                        |
|  Layer 3: MongoDB storage nodes (one primary + 2 secondaries per shard)|
|    Each shard is a MongoDB replica set. Writes go to the primary;    |
|    reads can go to secondaries (with slight staleness accepted for   |
|    non-critical reads like analytics). Primary failover is automatic: |
|    MongoDB's Raft-based election takes 10-30 seconds. During that    |
|    window, the Go proxy queues or rejects writes to that shard.      |
|                                                                        |
|  Sharding key: Payments are sharded by merchant_id (or a hash of     |
|  it). All of one merchant's payments live on the same shard. This    |
|  allows range queries ("show me all of Swiggy's payments this month") |
|  without cross-shard scatter. The tradeoff: a viral merchant whose   |
|  payments all land on one shard can create a hot shard. The Data     |
|  Movement Platform detects load imbalance and splits the hot shard.  |
|                                                                        |
|  The Data Movement Platform (zero-downtime resharding):               |
|    Problem: once a shard is full or hot, you need to move data to a  |
|    new shard without taking the payment system offline even for 1ms.  |
|    Solution: bidirectional replication + version gating.             |
|      Step 1: Start copying data from old shard to new shard (reads   |
|              still served from old shard, old shard still accepts     |
|              writes).                                                  |
|      Step 2: Enable bidirectional sync: new writes replicate to both  |
|              old and new shard. New shard catches up.                 |
|      Step 3: Version gate: flip the Chunk Metadata Service to point  |
|              the key range to the new shard. All proxies pick up the  |
|              change within seconds (version is checked on each query).|
|      Step 4: Drain and disable the old shard.                        |
|    Result: no downtime. Stripe migrated petabytes this way in 2023.  |
+------------------------------------------------------------------------+
         |
         v
+------------------------------------------------------------------------+
|              IMMUTABLE DOUBLE-ENTRY LEDGER                             |
|                                                                        |
|  Job: maintain a tamper-proof record of every financial movement.     |
|                                                                        |
|  What a double-entry ledger is:                                       |
|    Every financial event creates exactly two entries: a debit and a   |
|    credit. They must sum to zero. If Kavya pays 999 INR to an online  |
|    store:                                                              |
|      Debit:  Kavya's account         999 INR  (money leaves)         |
|      Credit: Online store's balance  999 INR  (money arrives)        |
|    The ledger is always balanced. Accountants call this "double-entry  |
|    bookkeeping." It predates computing -- Luca Pacioli described it   |
|    in 1494. Stripe implements it in software.                         |
|                                                                        |
|  Why append-only:                                                     |
|    Stripe never does UPDATE ledger SET amount = 999 WHERE id = 42.   |
|    Instead, it does INSERT INTO ledger (entry_type, amount, ...).    |
|    The "balance" is the SUM of all relevant rows.                     |
|                                                                        |
|    Why this matters:                                                   |
|    - Any tampering is visible. If a row is deleted or changed, the    |
|      debits and credits no longer sum to zero.                        |
|    - Disputes are auditable. Every state change has a timestamp and  |
|      a reason code.                                                    |
|    - Rollbacks are safe. Reversing a charge means adding two new      |
|      rows (debit the store, credit Kavya). The original charge stays  |
|      in history.                                                       |
|    - Regulators (RBI, PCI DSS, SOX) require this record.             |
|                                                                        |
|  Balance caching:                                                      |
|    Summing millions of ledger rows on every balance check is slow.   |
|    Stripe caches running balances in a separate table and increments  |
|    them atomically with each new ledger entry (inside the same DB    |
|    transaction). If the cache is out of sync, it can be recomputed   |
|    from the ledger -- the ledger is always the source of truth.      |
+------------------------------------------------------------------------+
         |
         v
+------------------------------------------------------------------------+
|              ASYNC QUEUE (webhooks + downstream events)                |
|                                                                        |
|  Job: decouple the fast synchronous payment path from slow, failure-  |
|  prone downstream actions.                                             |
|                                                                        |
|  After Phase 3 commits, Stripe enqueues an event:                    |
|    { type: "payment_intent.succeeded", payment_id: "pi_abc123" }     |
|                                                                        |
|  The queue worker delivers this to the merchant's webhook endpoint.   |
|  The worker retries with exponential backoff if the merchant's server |
|  is down: try after 1s, 5s, 30s, 2m, 10m, 1h, 24h, 72h.           |
|  The merchant's webhook handler must be idempotent (see "transferable |
|  mechanisms" below) because the worker delivers at-least-once.        |
|                                                                        |
|  Analogy: a postal courier. The package is sent once (Phase 3),      |
|  but delivery is a separate job. The courier retries if nobody        |
|  answers the door. The sender does not stand at the door waiting.    |
+------------------------------------------------------------------------+
```

---

## 4. The transferable mechanisms

These six primitives appear in Stripe's architecture and are directly reusable in any system that handles mutations.

**Mechanism 1: The idempotency key**

Definition: a unique identifier that ties a client intent to exactly one server outcome, regardless of how many times the request arrives.

How it works:
- Client generates a UUID per user action (not per HTTP request -- the same UUID is reused on retries).
- Server stores the UUID in a table with a UNIQUE constraint.
- On duplicate arrival: look up and return the cached response.
- TTL: 24 hours. After that, the key expires and a new action would generate a new key.

The rule: any API that creates or modifies state should accept an idempotency key. READ endpoints do not need them (reads are already safe to repeat).

Real example: Stripe's create-payment-intent endpoint accepts `Idempotency-Key` as an HTTP header. AWS uses `ClientToken` on EC2 RunInstances. Twilio uses `X-Twilio-Idempotency-Token` on message sends.

**Mechanism 2: Atomic phases with recovery points**

Definition: decompose a multi-step operation into N single-transaction phases; after each phase commits, save a "recovery point" so the next retry knows which phase to start from.

Why it matters: you cannot wrap the whole operation (validate + call card network + write ledger) in one database transaction, because the card network call is an external side effect that can take 30 seconds. Long transactions hold locks and destroy throughput. So you break the operation apart. Each phase is short. Between phases, the only external call happens (Phase 2). If it fails, you know exactly which phase to re-enter.

The key rule: each phase must be safe to re-run. Phase 1 (validation) is a pure read + write that can be repeated. Phase 3 (confirm) is guarded by the UNIQUE idempotency key, so it cannot create a duplicate. The card network call in Phase 2 is made idempotent by the merchant reference ID.

**Mechanism 3: The immutable append-only ledger**

Definition: never overwrite financial data; insert new rows for every state change; compute balances as a SUM.

Why it matters:
- Every change is auditable.
- Rollbacks are additive, not destructive.
- Concurrent writes cannot race to overwrite the same row (each INSERT gets its own row and its own primary key).
- Bugs can be diagnosed by replaying the append log.

This pattern is not just for money. Event sourcing (used in gaming, collaboration tools, audit systems) applies the same idea: store events, not state.

**Mechanism 4: Optimistic locking with version numbers**

Definition: add a `version` column to a row. When you read it, note the version. When you write it, require `WHERE id = $1 AND version = $old_version`. If another process wrote first, your update affects 0 rows -- you detect the conflict and retry.

When to use it: for writes that are USUALLY non-conflicting (two merchants rarely touch the same row). Optimistic locking avoids taking a lock upfront, which dramatically improves throughput.

When NOT to use it: for writes that frequently conflict (thousands of writes per second to one row). In that case, use a different data model (see YouTube lesson: Bigtable's native add-to-cell avoids a lock entirely).

In Stripe's context: subscription renewal status, payout schedule fields, and customer metadata use optimistic locking. The ledger itself avoids updates entirely (append-only), sidestepping both lock types.

**Mechanism 5: Exponential backoff with jitter**

Definition: on retry, wait 2^n seconds before the next attempt, multiplied by a random factor between 0.5 and 1.5.

Why jitter matters: if 10,000 clients all time out at T=0 and all retry at T=1s with pure exponential backoff (no jitter), they all hit the server simultaneously at T=1s. The server gets 10,000 requests at once, overloads again, and they all time out again at T=2s. The system never recovers. This is called a retry storm (more on this in section 6).

Jitter spreads the retries across a window. With jitter, the 10,000 clients retry between T=0.8s and T=2.1s. The server handles 5,000 requests at T=1s and 5,000 at T=1.5s. It recovers.

Stripe documents this pattern explicitly. AWS Exponential Backoff guide and Google SRE Book both require jitter on all retry logic.

**Mechanism 6: The circuit breaker**

Definition: track error rates to an external dependency. If they exceed a threshold (e.g. 50% errors in the last 10 seconds), stop calling that dependency for a fixed window (e.g. 30 seconds). Return a default or queued response instead. After the window, allow a small number of test calls through. If they succeed, close the circuit (normal mode). If not, stay open another window.

States: Closed (normal, all calls pass) -- Open (dependency is failing, all calls blocked) -- Half-Open (testing recovery, small fraction of calls pass).

Why it matters: without a circuit breaker, when Visa's API starts timing out at 30 seconds, your servers spin for 30 seconds on every charge attempt. 1,000 concurrent payment attempts = 1,000 threads blocked for 30 seconds each. Your server runs out of threads. Other API endpoints (refunds, customer lookups, not-related to Visa) also stop responding because the thread pool is exhausted. One failing dependency takes down everything. The circuit breaker prevents this cascade: after 10 failed Visa calls, stop trying Visa for 30 seconds and tell the merchant "payment pending, we will retry."

---

## 5. The trade-offs

**CAP theorem applied concretely to payments:**

Payments are the hardest case because BOTH double-charging (incorrect) and missing a legitimate charge (also incorrect) are catastrophic. Different data types get different CAP choices:

| Data type | Choice | Reason |
|---|---|---|
| Ledger entries (debit/credit) | CP -- Consistency over Availability | A double-charge is worse than a brief "service unavailable" |
| Payment status (captured/pending) | CP | Wrong status drives wrong downstream behavior |
| Customer metadata (name, email) | AP -- Availability over Consistency | Showing stale name for 30 seconds is fine |
| Analytics dashboards | AP, eventual | Creator sees hourly view count; 5-min lag is fine |
| Webhook delivery | AP, at-least-once | Merchant needs to receive the event; duplicate delivery is handled by their idempotent handler |

**Consistency vs availability example:**

During a DocDB shard leader election (primary fails, election takes 15 seconds), Stripe returns HTTP 503 for write requests to that shard rather than accepting writes to a secondary (which might diverge from the primary and cause split-brain). Merchants see a brief error. The Stripe SDK retries with backoff. No payment is lost. This is the CP choice: prefer returning an error over accepting a potentially inconsistent write.

**Cost vs latency:**

- The idempotency key lookup adds one synchronous DB read to every payment request. At Stripe's latency targets (payment confirmation in under 2 seconds), this adds roughly 5ms. The trade is: 5ms latency for double-charge prevention. Clearly worth it.
- DocDB's 2000+ shards mean petabytes of capacity, thousands of servers, and enormous operational complexity. The trade is: infrastructure cost for 99.999% uptime and zero-downtime migrations. At $1 trillion/year in payments, a single hour of downtime costs Stripe roughly $114 million in uncaptured payment volume. The infra cost is negligible by comparison.

---

## 6. The systems-thinking lens

**The feedback loop: the retry death spiral (metastable failure)**

A metastable failure is a system that enters a bad state it cannot exit without external intervention, even after the original trigger is gone.

Here is how one manifests in payments:

```
Trigger: Visa's API slows from 200ms to 8 seconds during a data
         center event.

Step 1:  Stripe's payment handler waits 8 seconds per call.
         Thread pool fills up (1,000 threads waiting on Visa).

Step 2:  New payment requests start timing out because all threads
         are busy. Merchants' SDKs retry these timeouts.

Step 3:  Retry traffic adds to the queue. Now 2,000 requests are
         waiting. Thread pool is more saturated.

Step 4:  Retry traffic causes more timeouts, which cause more retries.
         Load doubles every 30 seconds.

Step 5:  Visa's event ends after 5 minutes. But now Stripe has
         20,000 queued retry requests. Even with Visa responding at
         200ms again, the backlog takes hours to drain. During that
         drain, new requests continue to time out because the queue
         is full.

Step 6:  The original 5-minute Visa event causes 4+ hours of degraded
         payment reliability for Stripe. This is metastability: the
         system cannot recover without the backlog draining, but the
         backlog drains slowly because new traffic keeps arriving.
```

**The senior fix: break the feedback loop, not just add capacity**

Adding more Stripe servers does not help -- they will all also queue on Visa.

Three mechanisms that break the loop:

**Fix 1: Circuit breaker (cuts the inflow).**
After detecting that Visa is slow (10 calls in a row taking 8+ seconds), the circuit breaker opens. All new payment requests immediately return "payment pending, we will retry automatically." No thread waits on Visa. Thread pool stays empty. When Visa recovers, a few test calls get through, circuit closes, and payments resume. The backlog is small (30 seconds of requests) rather than hours-long.

**Fix 2: Idempotency key (prevents amplification).**
Every retry from the merchant SDK carries the original idempotency key. Even if some retries slip through before the circuit breaker opens, the idempotency layer ensures they do not create new work -- they either return a cached "pending" response or queue behind the in-flight request. Retries stop amplifying server load.

**Fix 3: Jitter on client backoff (spaces out the surge).**
When the circuit closes and Visa starts accepting traffic, Stripe does not immediately let 20,000 queued requests through at once. Each client's SDK is configured with exponential backoff and jitter. The first retry wave is small and spread over a 60-second window. The server catches up gradually, not in a thundering herd.

Together these three fixes are not about capacity -- they are about **preventing the feedback loop from forming in the first place**. The loop is: more load causes more timeouts, more timeouts cause more retries, more retries cause more load. Circuit breaker severs the "more retries" link. Idempotency severs the "more load per retry" link. Jitter severs the "simultaneous surge" link.

This is the senior infrastructure insight: the system failed because of a feedback loop, and the fix targets the loop, not the symptoms.

---

## 7. Stripe at three scale tiers

**Tier 1: 1,000 payments per day (early startup)**

Single Postgres instance. One API server. No idempotency table. Payments are synchronous and sequential. Works fine: at 1,000/day you average 0.01 payments/second. The chance of two concurrent requests hitting the same card row approaches zero. No retries in normal operation. Cost: one $20/month VPS. Nothing breaks.

**Tier 2: 100,000 payments per day (growing business)**

You notice occasional double charges when the card network has a slow day and the SDK retries. Add idempotency keys: 1 extra DB row per payment, 5ms extra latency. Worth it.
Postgres single instance starts struggling with 50+ concurrent write transactions. Add read replicas for non-critical reads (customer lookup, payment history). Add PgBouncer as a connection pool: fewer connections to Postgres, better utilization.
Rate limiting becomes essential: a buggy merchant SDK starts retrying 1,000 times per second. Add a Redis-backed token bucket per API key. Cost grows, but stays manageable.

**Tier 3: Stripe's actual scale ($1 trillion per year)**

Single Postgres is impossible. Even with read replicas, 5 million QPS would require thousands of Postgres instances with careful orchestration. Stripe built DocDB: a sharding layer that routes queries to one of 2,000+ MongoDB shards transparently. The Go proxy handles failover, admission control, and version-aware routing. Zero-downtime shard splits handle hot merchants.

The immutable ledger is non-negotiable at this scale: with petabytes of payment history, the ability to recompute any balance from raw events (rather than trusting a single mutable field) is the only way to diagnose bugs, pass audits, and survive corruption in one shard.

Circuit breakers, atomic phases, and recovery points become load-bearing infrastructure rather than nice-to-haves. At 5 million QPS, even a 0.001% retry rate generates 50 extra requests per second. Without the mechanisms above, those 50 requests can cascade.

**What breaks at Tier 3 that did not at Tier 2:**

- Single Postgres: cannot handle 5M QPS. Needs sharding.
- Synchronous card network calls with no circuit breaker: one Visa outage takes down the entire platform.
- Optimistic locking on hot merchant rows: 10,000 concurrent writes to one merchant's row cause too many version conflicts. Needs a queue or a partitioning strategy.
- At-least-once webhook delivery without idempotency on the merchant side: duplicate deliveries become daily events at scale.

---

## 8. Map to Rare.lab's stack

Rare.lab: node-based shader editor, scene JSON in Cloudflare R2 (content-addressed, immutable), Supabase Postgres with RLS, embeddable WebGL runtime.

**What Rare.lab already does correctly:**

Content-addressed immutable scene JSON in R2 is the same pattern as Stripe's immutable ledger. A shader node graph is written as a new object with a new hash on every save; nothing is overwritten. This is correct. It means any past state can be replayed from history. Keep this.

**The next ceiling: concurrent node graph edits**

If two users edit the same shader node simultaneously (a team workflow), Supabase will execute two concurrent UPDATEs on the same scene row. Last write wins. One user's changes silently disappear.

Apply the lessons from today:

Option A (lower effort, simpler): Add a `version` column to the scene table. Each client reads the current version. On save: `UPDATE scenes SET content = $new, version = version + 1 WHERE id = $1 AND version = $expected`. If another client wrote first, your UPDATE returns 0 rows. Return HTTP 409 to the editor client. The editor shows "conflict detected, reload to merge." This is optimistic locking. It prevents silent data loss with one column and one changed query.

Option B (higher effort, better UX): Add idempotency keys to the scene-save endpoint. Each "save" action gets a UUID. If the network drops during an autosave and the browser retries, Supabase returns the cached response rather than applying the save twice. This prevents duplicate saves corrupting a scene graph.

**The further ceiling: Supabase single Postgres**

Supabase is a managed Postgres. At current Rare.lab scale, one Postgres instance is correct. But watch for: shader preview renders that write output images to the scenes table (large JSONB writes), WebGL compilation results stored as rows, or a scenario where a viral shader gets thousands of remixes per minute updating the same parent record. These are the same hot-row problems Stripe faced. The mitigations are the same: read replicas for non-write traffic, connection pooling (PgBouncer, which Supabase includes), and partition mutable counters ("remix count") into a separate table with a Bigtable-style aggregation pattern.

The immutable R2 layer Rare.lab already has is the right foundation. The risk is the Supabase row that POINTS to the R2 object -- that pointer row is mutable and can be corrupted by concurrent writes. Protect it with optimistic locking first.

---

## 9. References with summaries

**Primary sources from Stripe engineering:**

**Stripe Blog: "Designing robust and predictable APIs with idempotency" (2017)**
URL: https://stripe.com/blog/idempotency
The original Stripe piece on idempotency. Written by Brandur Leach (then at Stripe). Explains the idempotency key header, the 24-hour TTL, how servers correlate keys with outcomes, and why keys are not a permanent archive but a near-term safety net. The most concise statement of the problem: "Network requests fail. Clients retry. Without safeguards, retries cause duplicate side effects." Must-read. Paywall-free.

**Stripe Blog: "Scaling your API with rate limiters" (2017)**
URL: https://stripe.com/blog/rate-limiters
Stripe's own explanation of their 4-layer rate limiting approach: request rate limiter (token bucket), concurrent request limiter, fleet load shedder, and worker queue limiter. Explains the rationale for each layer, when to use each, and how they interact. The piece also explains the "panic mode" where load shedders cut low-priority traffic entirely during critical events. Honest about failure modes. Free to read.

**Stripe Blog: "How Stripe's document databases supported 99.999% uptime with zero-downtime data migrations"**
URL: https://stripe.com/blog/how-stripes-document-databases-supported-99.999-uptime-with-zero-downtime-data-migrations
The canonical source on DocDB. Stripe engineers explain the sharding architecture, the Go proxy, the Chunk Metadata Service, and the Data Movement Platform. The key engineering insight: moving data between shards while serving live traffic requires bidirectional replication and version-gated cutover. The piece describes migrating petabytes in 2023 without a single second of downtime. Primary source, directly from the engineers who built it.

**QCon San Francisco 2025: "Stripe's DocDB: How Zero-Downtime Data Movement Powers Trillion-Dollar Payment Processing"**
URL: https://qconsf.com/presentation/nov2025/stripes-docdb-how-zero-downtime-data-movement-powers-trillion-dollar-payment
A 45-minute conference talk by Stripe engineers at QCon SF 2025. Covers the same DocDB material as the blog post but with more architectural depth: the exact sequence of steps in a shard migration, how version gating works at the proxy level, how the Chunk Metadata Service handles races during cutover. Slides and video available. Best for visual learners who want to see the state machine.

**Brandur Leach: "Implementing Stripe-like Idempotency Keys in Postgres" (2017)**
URL: https://brandur.org/idempotency-keys
Brandur was a Stripe engineer. This post is the deepest public description of atomic phases and recovery points in a payment context. He walks through the exact table schema for idempotency keys, the three outcomes (new / in-flight / finished), and how to implement each phase as a separate transaction with a recovery point. Includes real Postgres SQL. This is the piece that made "atomic phases" a known pattern in the industry. Free to read.

**Secondary sources and explainers:**

**ByteByteGo: "How Stripe Scaled to 5 Million Database Queries Per Second"**
URL: https://blog.bytebytego.com/p/how-stripe-scaled-to-5-million-database
Alex Xu's explainer synthesizing the public Stripe DocDB blog post into a visual summary. Good for a quick refresher on the three DocDB layers (proxy, chunk metadata, MongoDB shards) and the data movement sequence. Newsletter format, free tier available. Note: this is secondary analysis, not a primary Stripe source.

**Modern Treasury: "Designing the Ledgers API with Optimistic Locking"**
URL: https://www.moderntreasury.com/journal/designing-ledgers-with-optimistic-locking
Modern Treasury is a fintech company that builds ledger infrastructure for banks and payment processors. This post explains their implementation of optimistic locking on a double-entry ledger in Postgres. It explains the version column approach, the exact SQL for a conflict-safe update, and when to choose optimistic vs pessimistic locking. Concrete SQL examples throughout. Useful for understanding how the immutable-ledger + version-column combination works in practice.

**Martin Richards: "Scaling a Double-Entry Ledger for a Massive PSP"**
URL: https://www.martinrichards.me/post/ledger_p2_scaling_double_entry_ledger_massive_psp/
A deep-dive technical post by a payments engineer describing how to scale a double-entry ledger to millions of transactions per day. Covers: append-only INSERT vs UPDATE, materialized balance views, the SUM query performance problem at scale, and the transition from a single Postgres to a partitioned setup. Real SQL, real numbers. Secondary source but technically rigorous.

**Encore Blog: "Distributed Systems Horror Stories: The Thundering Herd Problem"**
URL: https://encore.dev/blog/thundering-herd-problem
A readable explainer of the thundering herd pattern with concrete examples. Explains how simultaneous retries cause server overload, why random jitter solves it, and includes an animation of the request distribution with and without jitter. Good reference for understanding why jitter is not optional. Free to read.

**Google SRE Book: Chapter 22 "Addressing Cascading Failures"**
URL: https://sre.google/sre-book/addressing-cascading-failures/
The canonical text on cascading failure and feedback loops in production systems. Chapter 22 covers: the retry amplification problem, load shedding, circuit breakers, graceful degradation, and why adding capacity does not fix a metastable failure. Written by Google SREs who operated systems at 10x Stripe's scale. The chapter on "bad queue management" directly describes the retry death spiral from this lesson. Free to read online (Google published the full book).

**AWS Architecture Blog: "Exponential Backoff and Jitter" (2015)**
URL: https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
The AWS post that popularized jitter in distributed systems. It includes empirical measurements: a simulation showing that pure exponential backoff (without jitter) under load results in synchronized retry waves that never resolve, while jitter-added backoff converges quickly. Includes Python simulation code. Short, rigorous, and the original source that most subsequent jitter discussions cite.

**Reddit r/ExperiencedDevs: "How do you handle idempotency in your payment flows?" (popular thread)**
URL: https://www.reddit.com/r/ExperiencedDevs/search/?q=idempotency+payments
Not a single post but the subreddit contains multiple high-signal threads on idempotency, double-charges, and ledger design. Engineers from Stripe, PayPal, Adyen, and smaller fintechs discuss real production failures and the solutions they used. Good for understanding which problems are universal vs Stripe-specific. Use the search above and filter by top posts of all time.

**Stack Overflow: "How does SELECT FOR UPDATE cause deadlocks at high concurrency?"**
URL: https://stackoverflow.com/questions/20763622/postgresql-select-for-update-and-deadlocks
A practical thread explaining why SELECT FOR UPDATE in Postgres causes deadlocks and throughput collapse under concurrent write load. The accepted answer explains lock queuing, the interaction with connection limits, and when to switch to optimistic locking. Directly relevant to Problem 1 in section 2 of this lesson.
