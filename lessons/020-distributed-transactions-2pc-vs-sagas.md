# Day 20 — How do you keep a transaction atomic when it spans two different databases?

**Date:** 2026-07-01
**Difficulty:** Expert
**Topic:** Distributed transactions across shards and services — two-phase commit, sagas, TrueTime/Spanner external consistency
**Stack relevance:** Supabase Postgres (single-writer ACID today), any future cross-shard or cross-service write in Rare.lab (scene save + billing + asset write), the embeddable runtime's eventual multi-tenant backend

---

## 1. The company and the breaking number

**Uber, Fulfillment Platform re-architecture. One trip touches at least three independently-owned entities (trip, driver/supply, payment), and none of them share a database or a lock manager.**

Uber's own engineering write-up of the Fulfillment Platform rebuild describes the problem plainly: a single business action, matching a rider to a driver and completing a trip, has to update multiple entities that live in different services and different datastores. There is no single `BEGIN; UPDATE trip; UPDATE driver; UPDATE payment; COMMIT;` available, because "trip" and "driver" and "payment" are not rows in one database, they are owned by separate teams running separate services on separate storage. Uber's fix was to adopt the **saga pattern**: a coordinator triggers a "propose" step on every participating entity, and only if every participant agrees does it trigger "commit" on all of them; if any participant fails, it triggers "cancel" (a compensating action) on whichever participants already proposed.

**The breaking number to hold onto:** it is not a throughput number this time, it is a *structural* number: **the count of independently-owned datastores a single business transaction must touch, with zero shared lock manager between them.** The moment that count goes from 1 to 2, "just wrap it in a transaction" stops being an available option, no matter how fast either individual database is. Google hit the sharper version of the same problem at planetary scale: Spanner's engineering (OSDI 2012, "Spanner: Google's Globally-Distributed Database") states that TrueTime's clock uncertainty, ε, is generally kept under 10ms across thousands of spanserver machines, and every write transaction that needs external consistency pays a **commit wait of roughly 2×ε, on the order of 8ms**, purely so two transactions ordered on different continents agree on which one happened first. There is no faster fix for that 8ms: it is the price of not knowing, with certainty, what time it is on another machine, and the only way around it is a different consistency model, not a faster network.

---

## 2. Why the naive (demo) design dies

The one-database demo looks like this:

```
BEGIN;
  UPDATE wallets SET balance = balance - 500 WHERE user_id = 42;   -- debit
  UPDATE inventory SET qty = qty - 1 WHERE sku = 'X123';           -- reserve
COMMIT;
```

One database, one WAL, one commit. Either both rows change or neither does, because Postgres's own ACID transaction guarantees it. This works perfectly in every demo, right up until "wallets" and "inventory" are not tables in the same database anymore, they are two different services, each with its own Postgres, because the wallets team and the fulfillment team scaled independently and sharded their own data their own way. Now there is no single `COMMIT` that covers both.

**a. The two-commits-and-hope design (dual writes).**
The obvious naive fix: call service A, commit locally, then call service B, commit locally. This fails constantly in production for the boring reason that a business transaction is now two separate network calls with a gap between them. If the debit commits and the network call to reserve inventory times out (not fails, *times out*, meaning the caller genuinely does not know if it landed), the money is gone and the item was never reserved. There is no way, from the caller's side, to tell "reservation succeeded, reply lost" from "reservation never happened," the exact same ambiguity Day 6's idempotency-key lesson covers for a single retried call, except now it is baked permanently into two independent commits with no shared transaction boundary at all.

**b. The naive fix to the naive fix: a global lock (2PC), and why it blocks.**
The textbook answer is the two-phase commit protocol: a coordinator asks every participant to "prepare" (lock the row, write the intent, promise to commit or abort on command, but do not commit yet), waits for every participant to vote yes, then tells everyone to "commit." This genuinely restores atomicity. It also has a well-documented failure mode: **if the coordinator crashes after collecting every "yes" vote but before sending the final "commit," every participant is stuck.** They have already promised to commit and cannot unilaterally decide to abort (another participant might have already committed), so they hold their locks and wait, indefinitely, for a coordinator that might never come back. This is not a rare edge case; in a microservices environment where processes restart constantly (deploys, autoscaling, OOM kills), a crashed coordinator mid-protocol is a routine event, and every prepared-but-unresolved transaction blocks every other transaction waiting on the same rows.

**c. The lock-hold-time collapse under real network latency.**
Even when the coordinator never crashes, 2PC's cost is structural: every participant holds its row lock for the *entire* two-phase round trip, not just for its own local commit. If the cross-service round trip is 40 to 50ms (two services, two network hops, one coordinator decision in between), a single hot row, say one popular product's inventory count during a flash sale, can only be locked and unlocked roughly 20 to 25 times per second, because each lock is held for the full 40 to 50ms round trip instead of the sub-millisecond a local commit would take. That is Day 16's hot-key problem again, except now the cost per lock is the network, not the CPU.

The naive design's real mistake is treating "atomic across two databases" as a smaller version of "atomic within one database." It is not smaller, it is a different problem: the moment a network hop sits inside the boundary of "atomic," you have imported every failure mode of distributed systems (partial failure, unbounded delay, ambiguous outcomes) into what used to be a single, uninterruptible operation.

---

## 3. The architecture

Top to bottom, for the saga-based approach that Uber, and most production systems that need cross-service atomicity without a shared lock manager, converge on:

```
[Client]
   |  "complete this trip" / "check out this cart"
   v
[Stateless app tier / API gateway]
   accepts the request, hands it to a durable workflow, returns immediately
   analogy: a maitre d' who takes your order and walks away, not a waiter who
   stands at your table until the kitchen finishes every dish
   |
   v
[Saga orchestrator]           e.g. Uber's Cadence / Temporal / AWS Step Functions
   owns the sequence of steps and their compensations; itself durable, so a
   crash mid-saga resumes from the last completed step, it never re-starts blind
   analogy: a wedding planner with a written checklist, not a memory
   |
   +----------------> [Local transaction, Service A: debit wallet]
   |                      writes row + writes an "outbox" event, same local
   |                      ACID commit (Day 17's WAL/CDC lesson: no dual-write gap)
   |
   +----------------> [Local transaction, Service B: reserve inventory]
   |                      same pattern: local commit + outbox event, own DB, own shard
   |
   +----------------> [Local transaction, Service C: charge card / ledger entry]
   |                      idempotency key = saga step ID (Day 12), safe to retry
   v
[Event bus / queue]           Kafka, SNS/SQS, or the outbox relay
   each service's outbox is tailed and published; the orchestrator advances the
   saga only on confirmed events, never on a guess
   analogy: a shared bulletin board every department reads, not a phone tree
   |  (any step reports failure)
   v
[Compensating actions]        e.g. refund wallet, release inventory reservation
   run in reverse order of what already succeeded; each one is also a local,
   idempotent transaction in its own service
   analogy: undo, not rollback: you cannot un-ring a bell that already rang in
   the outside world (a card was charged, an email was sent), you issue a
   second, opposite action instead
```

**The key structural decision:** there is no cross-service lock anywhere in this diagram. Every box that touches a database does one local, fast, fully-ACID transaction and then tells the outside world "I did my part" via an event. Atomicity across the whole saga is achieved by choreography over time (do step, confirm, do next step, or undo previous steps), not by holding multiple locks simultaneously across a network.

---

## 4. The transferable mechanisms

### a. Saga: local ACID + compensating transactions instead of a global lock

Break one cross-service transaction into a sequence of local transactions, each fully committed on its own, plus a compensating action defined for each step that must be reversible if a later step fails. Per [AWS's prescriptive guidance on the saga pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/saga-pattern.html), there are two implementation styles: **choreography** (each service publishes an event, the next service reacts to it, no central brain, simple for 2 to 3 steps but hard to trace past that) and **orchestration** (a central coordinator, Uber's own choice for the Fulfillment Platform, explicitly drives propose/commit/cancel on every participant, easier to reason about and monitor as the step count grows). The core trade you are always making: you gave up "the whole thing is atomic at every instant" in exchange for "the whole thing is guaranteed to *end* in a consistent state, eventually, even if some step fails halfway."

### b. The transactional outbox: fixing the dual-write problem inside each local step

Section 2a's "two commits and hope" bug does not go away just because you switched to sagas, it reappears at every individual step: "commit the local DB row, then separately publish an event" is itself two operations with a gap. The fix, described across saga-pattern references including [microservices.io's Saga pattern page](https://microservices.io/patterns/data/saga.html), is the **outbox pattern**: write the business row and an "event to publish" row in the *same* local ACID transaction, then have a separate relay (Day 17's CDC-tailing pattern, reading the WAL) pick up new outbox rows and publish them. This guarantees the event is published if and only if the local commit actually happened, because they are the same atomic operation from the database's point of view.

### c. Idempotency keys per saga step

Every step, and every compensation, must be safe to run twice, because the orchestrator will retry a step it is not sure completed (crash after sending the command, before receiving confirmation, the identical ambiguity from Day 6's Stripe lesson). Key each step's operation by a stable saga-step ID, not by "just try again," so a duplicate delivery is a no-op instead of a double charge or a double refund.

### d. Timeouts and fencing tokens: bound how long anything can stay "in doubt"

2PC's fatal flaw in section 2b is a lock held with no expiry, waiting on a coordinator that may be gone forever. The transferable fix, whether you use 2PC, sagas, or anything in between: every "prepared but not yet resolved" state needs a timeout, and every resource that can be reclaimed after a timeout needs a **fencing token** (a monotonically increasing number attached to whoever currently holds the lock) so that if the original holder wakes up late and tries to finish its work, its stale token is rejected instead of silently overwriting whatever happened while it was presumed dead. This is the same primitive Day 11's consensus lesson uses for leader terms, applied here to "who is allowed to finally commit or abort this prepared transaction."

### e. External consistency via a trusted clock (the Spanner alternative)

Sagas trade strong consistency for availability and put the complexity in application code. Spanner takes the opposite trade: keep true ACID, cross-shard, strongly-consistent transactions, but pay for it with **TrueTime**, an API that returns not a single timestamp but an interval `[earliest, latest]` bounding the true time with a guaranteed uncertainty ε. Per [Google Cloud's own explanation of TrueTime and external consistency](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency), a committing transaction picks a timestamp and then simply waits out the remaining uncertainty window (**commit wait**) before letting any other transaction observe its effect, guaranteeing that if transaction 1 finishes before transaction 2 starts in real time, their timestamps reflect that order everywhere in the world. This requires real infrastructure most companies do not have (Google uses GPS and atomic clocks in every datacenter specifically to keep ε small), which is why it is presented here as the "what if you can actually afford it" alternative to sagas, not the default answer.

---

## 5. The trade-offs

### Consistency vs. availability, per approach

| Approach | Consistency | Availability under partial failure | Where the complexity lives |
|---|---|---|---|
| 2PC | Strong, all-or-nothing at commit time | Poor: a crashed coordinator blocks every prepared participant indefinitely | Infrastructure (a lock manager everyone trusts) |
| Saga | Eventual: intermediate states are visible to the outside world | High: no cross-service lock ever held, a stuck step just triggers compensation | Application code (every step needs a compensating action and must be idempotent) |
| Spanner / TrueTime | Strong, externally consistent, globally | High, but at a fixed latency cost (commit wait) on every write | Infrastructure (specialized clock hardware) plus a latency tax paid on every transaction |

[Jepsen's independent analyses of distributed databases](https://jepsen.io/analyses/yugabyte-db-1.3.1) are worth citing here precisely because they are adversarial, not vendor marketing: testing YugabyteDB 1.3.1 under injected clock skew and network partitions, Jepsen found no serializability violations even under severe clock skew, but did find linearizability violations under specific high-throughput concurrent workloads, a reminder that "we implemented 2PC/Raft/Spanner-style transactions" is a claim that has to be independently tested, not assumed correct because the paper describing it sounds rigorous. CockroachDB's own account of [two-plus years of nightly Jepsen runs](https://www.cockroachlabs.com/blog/jepsen-tests-lessons/) makes the same point from the vendor side: real bugs surfaced repeatedly, including ones that had been silently present since a much earlier release.

### Cost vs. latency

Sagas are cheap: no special hardware, ordinary databases, ordinary message queues, the cost is entirely engineering time spent writing compensating actions and reasoning about intermediate visible states. Spanner-style external consistency is the opposite: it works with completely ordinary application code (write a normal-looking ACID transaction, the database handles everything), but it costs real money in specialized clock infrastructure and a real, unavoidable latency tax on every write that needs cross-shard ordering. Uber picked sagas because trip and driver services are independently owned, independently scaled, and a rider blocked for 8ms per transaction is invisible, but a rider blocked because a coordinator crashed and never released a lock is an outage. Google picked TrueTime because Spanner's entire product promise is "acts like one database even though it spans continents," and paying a fixed, small, known latency cost is a better trade than exposing intermediate saga states to a banking-grade ledger product.

---

## 6. The systems-thinking lens

**The feedback loop that actually causes failure here: the blocked-lock cascade, a slow-motion cousin of the retry storm.**

Walk it through with 2PC under real production conditions:

1. A coordinator crashes after every participant has voted "yes" (prepared) but before it sends the final "commit."
2. Every participant is now holding its row lock, waiting for a decision that will never come from that coordinator instance.
3. Any other transaction that needs one of those locked rows queues up behind the stuck transaction, with no way to know it is waiting on something that is never coming.
4. The queue of waiters grows. Application-level timeouts on the *new* requests start firing (from the caller's point of view, the system just got slow), so clients retry, which adds more transactions to the queue behind the same stuck locks.
5. What looks like "the database got slow" is actually "one dead coordinator turned into an ever-growing pile of blocked transactions," and adding more application servers or more database capacity does nothing, because the bottleneck is a specific set of locked rows, not aggregate throughput.

This is structurally the same shape as Day 13's metastable failure and Day 19's cache-stampede loop: a failure that *looks* like it should be fixable by adding capacity, but the loop itself is what is consuming capacity, and more capacity just means more requests queue up behind the same stuck locks faster.

**The senior fix breaks the loop instead of adding capacity:**

1. **Don't hold cross-service locks in the first place.** This is the entire argument for sagas over 2PC in most microservice architectures: if there is no multi-service lock, there is nothing for a crashed coordinator to leave stuck.
2. **Bound every "in doubt" state with a timeout and a recovery path.** If you do use 2PC (or any protocol with a prepare phase), a prepared transaction that has not heard back within N seconds must have a defined recovery procedure (query a recovery coordinator, or a designated participant promotes itself and resolves the transaction), not an indefinite wait.
3. **Make every compensating action and every retried step idempotent**, so the recovery path itself can be retried safely without needing to reason about "did the compensation already run."
4. **Treat "in-doubt" as a first-class, monitored state**, not an edge case. Uber's own description of the Fulfillment rebuild frames the saga coordinator as explicitly tracking propose/commit/cancel state per participant; that state is queryable and alertable, so a stuck saga shows up as a metric, not as a silent, growing backlog someone discovers during an incident.

The general lesson, consistent with every prior lesson in this ledger that touches distributed failure: the fix is never "make the lock manager faster" or "add more coordinators." It is "stop the design from creating a state where one failure can block an unbounded number of unrelated requests behind it."

---

## Map to Rare.lab's stack

**Where you are today, and why this mostly doesn't bite yet:** Supabase Postgres is a single database with real ACID transactions, and R2's content-addressed scene JSON is immutable, so a "save scene" operation today is very likely a single local transaction (write the manifest row, the R2 object already exists by hash before the manifest ever points at it). You do not have the section 2 problem yet because you do not yet have two independently-owned, independently-committing datastores in the same business operation.

**Where the ceiling actually is:** the day a single user action needs to touch Supabase Postgres *and* a separate billing/usage-metering system *and* an R2 write, in a way where all three must agree or none should count (for example: "compile this shader, deduct one credit from the user's plan, store the compiled bundle"), you have exactly Uber's problem. Do not reach for a cross-service 2PC here; reach for the saga shape directly: (1) write the compiled bundle to R2 first, keyed by content hash, an operation that is naturally idempotent and has no compensating action needed because writing the same bytes twice is a no-op; (2) only after that succeeds, run one local Supabase transaction that both records the manifest pointer and decrements the credit, in the same ACID commit, on the same database, because RLS-scoped rows in one Postgres instance are exactly the case where you still get to use a real transaction instead of a saga; (3) if credit deduction fails (insufficient balance), the R2 object simply becomes an orphaned, content-addressed blob, garbage-collected later, no compensating "delete from R2" step required because nothing downstream ever pointed at it. The specific lesson for Rare.lab: **order your saga steps so the irreversible, side-effect-bearing action (charging money, sending a webhook) happens last, after every step that only touches your own idempotent, content-addressed storage has already succeeded.** That ordering is what lets you avoid writing a single compensating transaction at all for the common case, and only need the full saga machinery once you add a second mutable, non-content-addressed datastore to the write path.

---

## References and summaries

### Primary engineering source, the saga side

**Uber Engineering: "Uber's Fulfillment Platform: Ground-up Re-architecture to Accelerate Uber's Go/Get Strategy"**
https://www.uber.com/us/en/blog/fulfillment-platform-rearchitecture/
Uber's own account of rebuilding the system that matches riders to drivers. States directly that Saga provided application-layer transaction semantics for multi-datastore, multi-service operations across trip and supply entities, describing the coordinator triggering a "propose" step on every participant, then "commit" only if all succeed, or "cancel" (compensating action) otherwise. Also documents the scaling ceiling of their earlier architecture: cities were sharded across pods bounded by Ringpop's peer-to-peer ring size, which imposed a hard vertical limit whenever a single city's concurrent trip volume crossed a threshold, the concrete reason the rebuild happened at all.

### Primary engineering source, the strong-consistency side

**Google: "Spanner: Google's Globally-Distributed Database" (OSDI 2012)**
https://research.google.com/archive/spanner-osdi2012.pdf
The original Spanner paper. Introduces TrueTime, an API returning a time interval bounding true time within a guaranteed uncertainty ε (kept under 10ms across the fleet per the paper's own reported data), and commit wait, the mechanism by which a committing transaction waits out the remaining uncertainty before its effects become visible, guaranteeing external consistency (if transaction A finishes before transaction B starts in real time, everywhere in the world agrees A's timestamp is earlier). This is the mechanism referenced in section 1 and section 4e as the infrastructure-heavy alternative to sagas.

**Google Cloud: "Spanner: TrueTime and external consistency"**
https://docs.cloud.google.com/spanner/docs/true-time-external-consistency
The vendor's own current, plain-language explanation of the same TrueTime/commit-wait mechanism from the original paper, useful as a more approachable second read after the primary PDF.

### The saga pattern, vendor-neutral references

**AWS Prescriptive Guidance: "Saga pattern"**
https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/saga-pattern.html
A clean, vendor-neutral description of the saga pattern's two implementation styles (choreography vs. orchestration) and the requirement for compensating transactions at every step. Referenced in section 4a.

**Chris Richardson: "Pattern: Saga"** (microservices.io)
https://microservices.io/patterns/data/saga.html
One of the most-cited independent references for the saga pattern and the related transactional outbox pattern referenced in section 4b, written by the author of the "Microservices Patterns" book, outside any single vendor's product marketing.

### Two-phase commit's blocking failure mode

**Wikipedia: "Two-phase commit protocol"**
https://en.wikipedia.org/wiki/Two-phase_commit_protocol
The standard reference for the protocol itself and its documented blocking problem: participants that have voted to commit must wait for the coordinator's final decision, and a coordinator crash after votes are collected but before the decision is broadcast leaves every participant holding its locks with no way to proceed independently. This is the mechanism behind section 2b and the section 6 blocked-lock cascade.

### Independent, adversarial verification that these systems actually behave as claimed

**Jepsen: "YugaByte DB 1.3.1"** (Kyle Kingsbury, 2019)
https://jepsen.io/analyses/yugabyte-db-1.3.1
An independent, adversarial test of a distributed SQL database's transactional guarantees under injected clock skew and network partitions. Found no serializability violations even under severe clock skew, but did find linearizability violations under specific high-throughput concurrent workloads, cited in section 5 as a reminder that distributed-transaction correctness claims need independent verification, not just a trust in the design paper.

**Cockroach Labs: "Lessons learned from 2+ years of nightly Jepsen tests"**
https://www.cockroachlabs.com/blog/jepsen-tests-lessons/
CockroachDB's own vendor-side account of running Jepsen continuously against their distributed transaction implementation, including a bug present since an early release that was only caught years later. Useful as the "even the vendor who asked for adversarial testing kept finding bugs" counterpoint referenced in section 5.

### Video, comparative explainer

**YouTube: "Distributed Transactions Explained: 2 Phase Commit vs Saga Pattern"**
https://www.youtube.com/watch?v=DOFflggE_0Q
A focused, system-design-interview-style walkthrough comparing 2PC and the saga pattern side by side, useful for rehearsing the section 5 trade-off table out loud.

### Article, orchestration vs. choreography

**ByteByteGo: "Saga Pattern Demystified: Orchestration vs Choreography"**
https://blog.bytebytego.com/p/saga-pattern-demystified-orchestration
A clear breakdown of the two saga implementation styles referenced in section 4a, with diagrams contrasting the central-coordinator (orchestration) approach Uber chose against the event-driven, no-central-brain (choreography) alternative.
