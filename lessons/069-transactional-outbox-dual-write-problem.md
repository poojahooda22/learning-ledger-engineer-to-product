# Day 69 — How do you write a row to your database and publish an event about it as if they were one atomic operation, when the database and the message broker are two completely different systems that share no transaction?

**Date:** 2026-09-03
**Difficulty:** Expert
**Topic:** The transactional outbox pattern and the dual-write problem it solves. This ledger has already built every piece this lesson assembles, just aimed at other jobs: Day 17 covered tailing the write-ahead log for change data capture, Day 12 covered making a retried operation safe to run twice, Day 40 covered exactly-once semantics in a stream processor, Day 42 covered Kafka as a partitioned, ordered commit log, and Day 59 covered event sourcing and CQRS. Today's mechanism is the missing joint between "I wrote something to my database" and "everyone downstream who needs to know now reliably does," without ever opening a distributed transaction across two systems that were never designed to share one.
**Stack relevance:** Rare.lab's save path is a dual write waiting to happen. Saving a scene means two things have to become true together: a new content-addressed scene JSON object lands in Cloudflare R2, and the manifest row in Supabase Postgres is updated to point at it. Every collaborator's client, every embedded runtime pulling that scene at runtime, and any CDN edge cache holding the old manifest need to hear about that change reliably, not eventually-if-nothing-crashed. Today's mechanism, write the "this changed" fact in the same Postgres transaction as the manifest update, then let a change-data-capture process fan it out, is directly the shape Rare.lab needs the moment more than one thing (a collaborator's live view, a CDN purge, an embedded runtime's hot-reload) has to reliably learn about a save. The next ceiling: right now a save is one write with implicit trust that anyone who cares will eventually re-poll; the moment Rare.lab needs guaranteed, ordered, at-least-once delivery of "this scene changed" to N independent subscribers, a bare Postgres update stops being enough, and this lesson is the pattern that closes that gap without Rare.lab hand-rolling a distributed transaction.

---

## 1. The company and the breaking number

**Shopify, and the number hiding inside "80,000 checkouts a minute."** Shopify's own engineering blog, in its account of preparing the platform for Black Friday Cyber Monday, describes running five major scale tests between April and October, each one bigger than the last, culminating in a fourth test that hit 146 million requests per minute against the edge and more than 80,000 checkouts per minute. BFCM 2024 itself peaked at 284 million requests per minute at the edge, 80 million requests per minute reaching app servers, 12 terabytes of data moved every minute, and 10.5 trillion database queries across the weekend. Eighty thousand checkouts a minute is roughly 1,333 checkouts every second, and every single one of them is, underneath the product framing, the same two-part promise: commit the order to the database, and tell every other system that needs to know, inventory, fulfillment, the merchant's webhook endpoints, the analytics pipeline, the search index, that the order exists. Run the numbers on what a naive version of that promise costs: if committing the order and separately publishing an "OrderCreated" event fails to complete both halves even 0.1 percent of the time, a rate a struggling message broker under Black Friday load will absolutely produce, that is roughly 1.3 silently lost order events every second, close to 80 a minute, compounding for the entire peak window. Those are not failed orders. The customer paid, the database row exists, the money moved. They are invisible orders: paid for, but never told to fulfillment, sitting there until someone notices stock is short or a customer asks where their package is.

**Gunnar Morling's framing, and why the obvious fix isn't available.** The pattern this lesson teaches was named and popularized in a February 2019 Debezium blog post by Gunnar Morling, "Reliable Microservices Data Exchange With the Outbox Pattern," which lays out exactly this problem: a service needs to both update its own local data store and notify other services about that change, and the two operations have to succeed or fail together, but they touch two different systems, a relational database and a message broker, that do not share a transaction manager. The textbook answer to "make two systems commit together" is a distributed transaction, two-phase commit, XA. It is available, in principle, for a Postgres database and some message brokers. It is also almost never used at Shopify's scale, for a concrete reason the next section makes precise: 2PC ties the availability of the entire write path to the availability of every participant, and it holds locks on the database for the full round trip of coordinating with the broker, which is exactly the wrong trade to make on the one code path, checkout, that a flash sale cannot afford to slow down.

**Netflix's DBLog, a real production number on the infrastructure this pattern leans on.** The mechanism that makes the outbox pattern work without 2PC is change data capture, reading the database's own write-ahead log instead of writing to two systems directly. Netflix's own account of this, published on the Netflix TechBlog and as a peer-reviewed paper ("DBLog: A Watermark Based Change-Data-Capture Framework," Andreakis and Papapanagiotou), describes a CDC framework used in production by tens of microservices at Netflix, built specifically to solve a problem naive log-tailing has: you need both a full snapshot of existing rows and a live stream of new changes, without ever stalling the stream to take that snapshot. That is the same underlying machinery Debezium uses to turn an outbox table into a Kafka topic, and it is worth sitting with the fact that a company operating at Netflix's scale needed a dedicated, published, watermark-based framework just to make log-tailing safe. This was never going to be something a service bolts on with a five-line script.

---

## 2. Why the naive (demo) design dies

**The obvious version:** inside the checkout handler, after validating payment, run one SQL statement to insert the order row, then call the message broker's client library to publish an `OrderCreated` event carrying the order details. Two separate calls, one after the other, in the same function. This is exactly how most teams build it first, because for a demo, or for the first few hundred orders a day, it works close enough to every time that nobody notices the gap.

**Death one: the crash between the two calls loses the event, silently.** The database commit and the broker publish are two independent network calls to two independent systems. If the process crashes, is killed by an autoscaler, or the container is rescheduled in the half-second between "the database says the order committed" and "the broker says the event was received," the order exists and nobody downstream ever hears about it. There is no retry, because the code that would have retried no longer exists. This is not a rare edge case at Shopify's peak: at 1,333 checkouts a second, spread across thousands of app-server processes that autoscalers are actively adding and removing to handle the exact load the checkout is generating, a process getting recycled mid-request is a routine event, not a black swan.

**Death two: publishing first and committing second turns "eventually consistent" into "actively wrong."** Flip the order, publish the event first, then commit the database write, and a new failure mode appears: the database transaction can still fail, or the process can crash after the publish succeeds but before the commit lands, and now inventory, fulfillment, and the merchant's webhook all believe an order exists that the database will never actually contain. Downstream systems built on the assumption that an event means "this really happened" now have to independently verify every event against the source of truth, which defeats the entire point of publishing the event in the first place.

**Death three: wrapping the publish in a retry loop moves the failure into the request path itself, and that is worse.** The instinctive fix for death one is to retry the broker call until it succeeds before returning to the customer. Under normal conditions this looks like it closes the gap. Under the exact conditions a flash sale actually produces, a message broker under sustained write pressure returning slow acknowledgments or transient errors, that retry loop now sits synchronously inside the checkout request, holding a thread, a database connection, and the customer's open HTTP connection open for every retry attempt. Checkout latency, the one metric a flash sale cannot afford to move, starts climbing exactly when broker load is highest, which is exactly when checkout traffic is also highest. The mechanism meant to make delivery more reliable has coupled the availability of checkout to the availability of the broker, the same coupling 2PC would have created, just built by hand and hidden inside a `try/retry` block instead of declared up front.

---

## 3. The architecture

```
Client (checkout form submit)
  - job: send the order request once, wait for a single response
  - analogy: dropping a completed form in a mailbox, not standing at
    the counter until every downstream department signs off

        |
        v
Stateless app tier (the checkout service)
  - job: within ONE local database transaction, insert the order row
    AND insert a row into an "outbox" table describing the event to
    publish, then commit, then return to the customer immediately
  - analogy: writing the order in the ledger and writing a carbon-copy
    dispatch slip on the same page, in the same pen stroke, so there
    is no way to have one without the other

        |  (single local ACID transaction, same database, same commit)
        v
Postgres primary: `orders` table + `outbox` table
  - job: be the one and only source of truth; the outbox row is
    ordinary relational data, durable the instant the transaction
    commits, with nothing about it that has left the building yet
  - analogy: the restaurant kitchen's ticket rail, an order and its
    dispatch ticket printed on the same slip, still sitting in the
    kitchen, not yet handed to the runner

        |  (CDC reads the WAL, never queries the outbox table directly;
        |   this is Day 17's mechanism, aimed at this specific table)
        v
Change-data-capture connector (Debezium, tailing the write-ahead log,
built on the watermark-based approach Netflix's DBLog documents for
safely mixing a full backfill with a live tail)
  - job: notice new rows landing in the outbox table by reading the
    database's own replication stream, the same stream a read replica
    consumes, so watching for new events costs the primary nothing
    beyond ordinary replication
  - analogy: a runner who doesn't have to ask the kitchen "anything
    new?", who instead watches the ticket rail itself and picks up
    every slip the instant it's printed

        |  (Debezium's outbox event router transforms the row into a
        |   properly keyed, properly shaped event)
        v
Message broker (Kafka, partitioned by order ID or aggregate ID, the
same partitioned commit log Day 42 covers)
  - job: durably hold the event, ordered within a partition, available
    for every downstream consumer to read independently and at its
    own pace
  - analogy: a bulletin board pinned by category, so every department
    reads the notice in the order it was posted, on its own schedule

        |
        v
Independent consumers: inventory, fulfillment, merchant webhooks,
search index, analytics pipeline
  - job: each one processes the event exactly once from its own point
    of view, using an idempotency key (Day 12's mechanism) to make a
    redelivered event a safe no-op rather than a double-processed order
  - analogy: five separate departments each reading their own copy of
    the same bulletin, checking it off their own list so a re-posted
    notice doesn't get acted on twice

        |
        v
Outbox table cleanup (a scheduled job or a retention policy, deleting
or archiving rows already confirmed delivered)
  - job: keep the outbox table small, because it is a live, indexed,
    frequently-written table sitting on the primary, and an
    unbounded outbox table is its own future bottleneck
  - analogy: clearing yesterday's dispatched tickets off the rail so
    tomorrow's kitchen isn't sorting through a growing pile to find
    what's actually new
```

---

## 4. The transferable mechanisms

- **Make the two facts one fact, by writing them in the same local transaction.** The core move is refusing to treat "the order exists" and "an event about the order should be published" as two separate operations that need to be kept in sync. They are the same operation: one row, and one more row, in the same ACID transaction, on the same database. This converts a cross-system atomicity problem, which needs 2PC or gets punted on, into an ordinary single-database transaction, which Postgres has handled correctly for decades.

- **Read the change instead of writing it twice.** Change data capture tails the database's write-ahead log, the same durability mechanism the database already uses to survive its own crashes, and turns new rows into a stream. This is Day 17's mechanism, redeployed here: rather than the application code being responsible for telling both the database and the broker, only the database is told anything, and the broker learns about it by watching the database's own record of what already, definitely, happened.

- **Give every consumer an idempotency key, because "at least once" is the honest guarantee.** CDC-fed event delivery is at-least-once, not exactly-once: a connector restart, a rebalance, or a retry after a transient broker error can redeliver an event that already arrived. Day 12's mechanism, a stable key (the order ID, or a composite of order ID and event type) that a consumer checks before acting, is what turns a redelivered event into a safe no-op instead of a double-shipped order or a double-charged customer.

- **Partition by aggregate, not round-robin, to keep causal order where it matters.** Two events about the same order, `OrderCreated` then `OrderCancelled`, have to arrive at each consumer in that order, or a consumer could act on the cancellation before it even knows the order existed. Partitioning the Kafka topic by order ID guarantees every event for a given order lands in the same partition, and a single partition is strictly ordered, the same ordering guarantee Day 42 covers for why a partitioned log beats a single global ordering, applied here per-aggregate rather than globally.

- **Bound the outbox table, deliberately, from day one.** An outbox table that only ever grows is a slow-motion version of the exact problem this pattern exists to prevent: a hot, ever-larger table on the primary that eventually makes every insert into it, meaning every checkout, slower. A scheduled cleanup job, or a CDC connector that flags rows as processed and a separate reaper that deletes them, keeps the table's working size proportional to recent throughput, not total lifetime volume.

- **Decide the outbox row's shape once, and keep it boring.** A generic outbox schema, `id`, `aggregate_type`, `aggregate_id`, `event_type`, `payload` (JSON), `created_at`, lets the CDC connector's outbox event router (a real, documented Debezium single-message transform) reshape any row into a correctly-keyed Kafka message without bespoke code per event type. Keeping the payload itself schema-light, and letting each downstream consumer interpret it, avoids coupling the outbox table's shape to every consumer's evolving needs.

---

## 5. The trade-offs

**Consistency vs. availability, and which one gets which guarantee.** The order row itself gets full ACID consistency: the moment the transaction commits, it is durable and correct, no exceptions, because it never left Postgres. The downstream fact of "everyone was told" gets eventual consistency instead: CDC connector lag, typically tens to a few hundred milliseconds under healthy conditions, is real time during which the order exists but a consumer hasn't heard yet. This lesson's whole architecture is a deliberate choice to make the write path (checkout) strongly consistent and fast, and the fan-out path (telling everyone else) eventually consistent and decoupled, rather than trying to make both strongly consistent together the way 2PC attempts and pays for in availability and latency on every single write.

**Cost vs. latency, paid as CDC infrastructure instead of request-path latency.** Running Debezium, or an equivalent CDC connector, is a real, standing infrastructure cost: a process that has to stay healthy, watched for lag, and given enough resources to keep up with the WAL's write rate. The alternative, a synchronous dual write with retries, has no separate infrastructure cost, but it pays its cost directly in checkout latency and in the coupling described in death three above. Shopify's own architecture accepts the CDC infrastructure cost precisely because checkout latency is the one number a flash sale cannot spend, and a standing connector's operational cost is cheaper than losing orders during the ten minutes of the year that matter most.

**The outbox table's size is a cost the design has to actively manage, not something that resolves itself.** An unbounded outbox table trades a solved dual-write problem for a slow-growing-table problem, and the two look identical in a demo and very different at 1,333 inserts a second sustained for hours. This is the same shape of trade this ledger keeps returning to: a mechanism that fixes today's bottleneck introduces tomorrow's, and the senior move is choosing that trade-off deliberately, with a cleanup policy decided in advance, rather than discovering it during an incident.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is the same **retry death spiral** this ledger has named before, showing up in a new place: inside the request path of the write itself, not inside a downstream service under load. The naive fix for the dual-write problem, wrap the broker publish in a retry loop and hold the request open until it succeeds, looks like it is adding reliability. What it is actually doing is coupling checkout's availability to the broker's availability, and doing it at the exact moment that coupling is most dangerous: broker degradation under Black Friday write pressure is correlated with checkout traffic being at its highest, not independent of it. Once that coupling exists, a small broker slowdown makes checkout requests hold their connections and threads longer, which reduces the app tier's effective request capacity, which makes checkout itself slower and more likely to time out, which causes customers and client-side retry logic to resubmit, which adds more load to an already-struggling system. That is a metastable failure: the system doesn't fail because any single component crashed, it fails because a feedback loop the naive design accidentally built keeps making a bad situation worse in proportion to how hard the system is trying to recover.

The senior fix does not try to make the retry loop smarter, more patient, or better-backed-off. It removes the loop from the request path entirely. Writing the outbox row in the same local transaction as the order, and returning to the customer the instant that single-database commit succeeds, means checkout's latency and availability are now decoupled from the broker's health by construction, not by discipline. The broker can be down for five minutes and checkout does not notice, because nothing in the request path is waiting on it; the outbox table simply accumulates rows that CDC will fan out the moment the broker recovers. This is the same move Day 13's backpressure lesson makes for load in general, applied here to a specific coupling: don't try to make a synchronous dependency more resilient under load, remove the synchronous dependency, and let the two systems drift apart in time on purpose, with a durable record in between that makes catching back up automatic instead of manual.

---

## Sources

- [Reliable Microservices Data Exchange With the Outbox Pattern, debezium.io](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/): Gunnar Morling's February 2019 Debezium blog post, the canonical source naming and explaining the transactional outbox pattern, the dual-write problem it solves, and the outbox event router transform; direct fetch was blocked by this session's network egress policy, details drawn from search-indexed excerpts.
- [Revisiting the Outbox Pattern, morling.dev](https://www.morling.dev/blog/revisiting-the-outbox-pattern/): Gunnar Morling's follow-up post refining the pattern's guidance; referenced via search-indexed excerpt, direct fetch blocked by this session's egress policy.
- [DBLog: A Generic Change-Data-Capture Framework, Netflix TechBlog](https://netflixtechblog.com/dblog-a-generic-change-data-capture-framework-69351fb9099b): Netflix's own account of DBLog, its watermark-based approach to interleaving a full-table backfill with a live change stream without stalling, and its production use by tens of microservices at Netflix; the same paper is indexed on Semantic Scholar and ArXiv (Andreakis and Papapanagiotou, 2020) as an independent peer-reviewed corroboration of the design; accessed via search-indexed summary, direct fetch blocked by this session's egress policy.
- [How we prepare Shopify for BFCM (2025), shopify.engineering](https://shopify.engineering/bfcm-readiness-2025): source for the five pre-BFCM scale tests between April and October, the fourth test's 146 million requests per minute and 80,000-plus checkouts per minute figures, and the description of scale tests coordinated to avoid clashing with other major internet traffic events; accessed via search-indexed excerpt, direct fetch blocked by this session's egress policy.
- [How Shopify Prepares for Black Friday, ByteByteGo](https://blog.bytebytego.com/p/how-shopify-prepares-for-black-friday): secondary corroboration of Shopify's BFCM scale-testing figures and general architecture, cross-checked against the Shopify Engineering source above.
- [Shopify Merchants Achieve Record-Breaking $14.6 Billion in Black Friday-Cyber Monday Sales, shopify.com investor relations](https://www.shopify.com/investors/press-releases/shopify-merchants-achieve-record-breaking-14-6-billion-in-black-friday-cyber-monday-sales): source for the BFCM 2024 peak figures cited (284 million requests per minute at the edge, 80 million requests per minute at app servers, 12 terabytes per minute, 10.5 trillion database queries over the weekend); accessed via search-indexed excerpt.
- Day 12 (this ledger, idempotency and exactly-once delivery), Day 13 (backpressure and load shedding), Day 17 (write-ahead logs and change data capture), Day 40 (stream processing, exactly-once semantics, and watermarks), Day 42 (Kafka as a partitioned commit log), Day 59 (event sourcing and CQRS): the ledger's own prior lessons this one builds directly on, for the idempotency-key mechanism, the retry-death-spiral systems lens reused here, the WAL-tailing CDC mechanism, watermark-based stream correctness, partition-level ordering, and the broader event-driven architecture context the outbox pattern sits inside.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of debezium.io, morling.dev, netflixtechblog.com, and shopify.engineering, so every figure above is drawn from search-indexed excerpts of those pages rather than a full read of the original text. The Shopify BFCM figures (284 million requests per minute at the edge, 80 million at app servers, 80,000-plus checkouts per minute during scale testing, 10.5 trillion database queries) are corroborated across Shopify's own investor press release and independent secondary coverage, and are treated as solid. The DBLog production-usage figure ("tens of microservices at Netflix") comes from Netflix's own technology blog and is corroborated by the independently peer-reviewed paper indexed on Semantic Scholar and ArXiv, and is treated as solid. The 0.1 percent dual-write failure rate used in Section 1 is explicitly labeled illustrative math, not a measured Shopify statistic: it is a plausible failure rate for a synchronous cross-system call under peak load, used to make the scale of silent data loss concrete, not a reported number from any source above.

---
