# Day 59 — How does LMAX settle 6 million trades a second on a single thread, when one uncontended lock already costs more nanoseconds than the trade is allowed to take?

**Date:** 2026-08-20
**Difficulty:** Expert
**Topic:** Event sourcing and CQRS (Command Query Responsibility Segregation): why the "obvious" architecture, a table that holds current state and gets overwritten on every change, dies twice, once on throughput the moment updates need a lock or a database round trip inside a nanosecond-scale budget, and once on truth the moment anyone needs to know how the current state was arrived at, because an `UPDATE` statement destroys the previous value it replaces. How LMAX's Business Logic Processor solves the first death by keeping every order in memory on a single, lock-free thread and treating the append-only log of incoming orders as the only fact that ever gets written to disk, deriving all state by replaying it. How the second death, pure event sourcing without a read-optimized side, is solved by CQRS: splitting the write model (one ordered log, strongly consistent, cheap to append to) from one or more read models (denormalized, precomputed, allowed to lag) instead of forcing every read to replay history. And why naive, unbounded event sourcing has its own scaling collapse, a system whose replay time grows without limit as the log grows, and the two mechanisms, snapshotting and CQRS projections, that keep replay time bounded no matter how long the log gets.
**Stack relevance:** Rare.lab's node-based editor is, structurally, a sequence of graph edits: add a node, wire an input, change a parameter. Today, if the only thing that gets saved is the current graph, the system already has both of this lesson's failure modes waiting for it: no real audit trail of how a scene got the way it is, no cheap way to give a future collaborative multiplayer editor (Day 3's problem, arriving again here) a shared, ordered history to reconcile against, and no way to derive more than one shape (the live canvas, the compiled shippable code, an undo stack) from one source of truth without hand-building each one separately. The embeddable runtime's single shared WebGL context is also, unexpectedly, LMAX's Business Logic Processor in miniature: one hot, stateful resource that cannot tolerate uncoordinated concurrent writers, for exactly the same mechanical reason.

---

## 1. The company and the breaking number

**LMAX, London, 2010.** LMAX is a retail foreign-exchange and CFD trading venue: real customers submitting real buy and sell orders against a live, matching order book, in a market where the correct price a millisecond ago can be the wrong price now. The core of the system, the piece that decides whether an incoming order matches an existing one, updates an account's position, and confirms or rejects a trade, is called the Business Logic Processor. According to the architecture LMAX's engineers published, that processor runs the entire business logic for every trade, across every customer, in every market, on a single thread, in memory, and sustains **6,000,000 business logic operations per second on commodity hardware**.

That number is the breaking number this lesson turns on, and the reason it breaks things isn't intuition, it's arithmetic. Six million operations per second means the total time budget for one operation, start to finish, is:

```
1 second / 6,000,000 = 166.6 nanoseconds per order
```

Now hold that budget next to the actual cost of the primitives a completely ordinary, completely standard design would reach for to process one order safely. The widely cited "Latency Numbers Every Programmer Should Know" reference table gives concrete figures for exactly these primitives:

- An **uncontended mutex lock/unlock**: 25 nanoseconds. That's before the lock has protected a single instruction of real work, it has already spent 15% of the entire 166.6 nanosecond budget just acquiring and releasing itself, and that number is the *best* case, a lock with real cross-core contention (which a shared "current balance" row under concurrent trading load will have) costs substantially more, because the cache line backing that lock has to physically travel between CPU cores.
- A **round trip within the same datacenter**, the honest cost of "call the database" even when the database sits in the next rack: 500,000 nanoseconds. That is roughly **3,000 times** the entire per-order budget.
- Even the cheapest possible durable-storage operation, reading 1 MB sequentially from an SSD, is 1,000,000 nanoseconds, **6,000 times** over budget.

There is no version of "take a lock, then ask a database" that fits inside 166.6 nanoseconds, not by a small margin that clever engineering closes, but by three to four orders of magnitude that no amount of query optimization, connection pooling, or bigger hardware closes. This is the sense in which the number is a *breaking* number rather than merely a *big* number: it doesn't describe a system that is hard to build fast, it describes a system where the standard toolkit (locks guarding shared mutable state, a database consulted per write) is arithmetically disqualified before a single line of code is written.

LMAX's own published benchmark of the mechanism they built to replace that toolkit, the Disruptor, a lock-free ring buffer for passing work between stages of a pipeline, found that for a three-stage pipeline its mean latency was **three orders of magnitude lower** than an equivalent queue-based design, and it sustained roughly **eight times the throughput** at the same configuration. Note what that comparison actually is: not "our custom hardware beats a database," but "an ordinary multi-threaded, queue-and-lock design, the thing most engineers would build by default to move data between concurrent stages, is already three orders of magnitude too slow," before a database ever enters the picture. The naive design doesn't die at the database call. It dies one step earlier, at the lock.

---

## 2. Why the naive (demo) design dies

**Version one: a normalized `balances` table, updated in place, guarded by row locks.** This is what every payments tutorial teaches, and at demo scale, ten trades a second, it is completely correct: `BEGIN; SELECT balance FROM accounts WHERE id = ? FOR UPDATE; UPDATE accounts SET balance = balance - ? WHERE id = ?; COMMIT;`. Every step of that transaction is doing real, necessary work, checking the current balance, preventing a concurrent trade on the same account from racing it, persisting the new value durably. None of it is wasteful. All of it, individually, is standard advice. And the arithmetic in Section 1 already showed why it cannot reach 6,000,000 operations a second: the row lock alone costs more nanoseconds than the entire order is allowed to take, and the transaction commit, a durable write to disk with a network round trip to the database, costs thousands of times the budget on top of that.

**Version two, the "fix" that doesn't fix it: scale out the application tier.** Add more API servers, add a connection pool, shard the database by account range. This helps exactly nothing about the actual bottleneck, because two concurrent trades against the *same account* still have to serialize somewhere, there is no way to let them both proceed independently and still have a correct final balance. Scaling the app tier out just moves more concurrent traffic toward the same hot row, which is Day 16's hot-key problem wearing a different hat: the fix for too many app servers hitting one lock isn't more app servers, it's removing the requirement that a lock be held at all.

**Version three, the death that has nothing to do with throughput: the `UPDATE` statement destroys the evidence.** Suppose the throughput problem is somehow solved, faster hardware, a bigger database, whatever. A normalized `balances` table still only ever holds the *current* value. The moment a trade's `UPDATE` commits, the previous balance is gone, overwritten, not archived, just gone, unless something else was written to preserve it. For a trading venue this isn't a nice-to-have, it's the whole business: a regulator, an auditor, or a customer dispute needs to reconstruct exactly what the account balance was at 14:03:07 on a Tuesday and exactly which sequence of trades produced it. The standard patch, add a separate `audit_log` table and write to it alongside the real update, is a well-documented failure mode in its own right: it works exactly until one code path, a bugfix, a backfill script, an emergency hotfix under incident pressure, updates the balance without also remembering to write the log entry, and now the audit trail and the actual state have silently diverged, and nothing in the system can detect that they've diverged until someone goes looking. The state and the history of the state are two separate writes to two separate places, and two separate writes are never guaranteed to both happen, or happen in the same order, unless something makes that guarantee structural rather than a matter of remembering.

**Event sourcing is the direct answer to version three:** make the log itself the only thing that is ever durably written. Don't store current state and separately log how it changed, store *only* the sequence of changes, an append-only stream of events (`OrderPlaced`, `TradeMatched`, `PositionUpdated`), and derive current state, whenever it's needed, by replaying that stream from the beginning. Martin Fowler's original 2005 description of the pattern states it plainly: capture every change to application state as an event, and it becomes possible to reconstruct the state at any point in time by replaying the events up to that point. There is no longer a "state" and a "log of changes to the state" that can drift apart, because there is only ever the log, and current state is a *derived value*, not an independently stored fact. This also, incidentally, is exactly how LMAX's Business Logic Processor gets its speed: an append to an in-memory, single-threaded log costs nothing like a locked, durable database write, and the processor never has to ask "what is the current balance," it already knows, because it's holding the live, in-memory result of every event it has ever applied.

**But naive, unbounded event sourcing has a fourth death, and it's the one that actually determines whether this pattern survives past the demo:** if reconstructing state means replaying every event from the beginning of time, replay time grows without bound as the log grows, and eventually that growth crosses from "acceptable" to "an outage." This is documented, not hypothetical: one production financial system that stored every price tick as an event accumulated 3 terabytes of event history, and reconstructing an account balance from that history took query times measured in *minutes*. A separate incident, at a different company, involved a bug that corrupted a read projection; the fix required replaying two weeks of accumulated events to rebuild it correctly, and that replay took **18 hours**, blowing through the recovery SLA the team had committed to. Pure event sourcing, taken naively, doesn't fail at 1,000 events, and it doesn't fail at 100,000. It fails once the log has been running long enough, and grown large enough, that "replay everything" stops being a rounding error and starts being the outage.

---

## 3. The architecture

```
[Client: order ticket / graph edit / any state-changing intent]
   the thing a user actually did, expressed as an intention, not
   yet accepted as fact
   job: describe what should happen
   analogy: filling out a withdrawal slip at a bank counter, an
   instruction, not yet money that has moved

   |
   v

[Command gateway]
   validates the SHAPE of the request only (is this a well-formed
   order, does the field exist, is the format legal); does not
   consult or touch any business state yet
   job: reject garbage before it costs the write path anything
   analogy: the teller checking your slip is filled in correctly,
   before ever walking it back to the vault

   |
   v

[Aggregate router: shard by aggregate ID]
   every command is routed, by a stable hash of the entity it
   concerns (an account ID, an order-book symbol, a project ID),
   to exactly one partition; every command for that same entity
   always lands on the same partition, in the order it arrived
   job: guarantee a single writer per entity, so no two commands
   for the same account or order book are ever "concurrent" from
   the system's point of view
   analogy: every letter addressed to one specific mailbox always
   goes through the same sorting chute, in the order it was posted

   |
   v

[In-memory aggregate: single-threaded, lock-free, hydrated from a
 snapshot plus the events since that snapshot]
   holds the live, current state of exactly one entity in memory;
   applies business rules (does this account have enough balance,
   is this order valid against the current book) and decides
   accept or reject, entirely from memory, no lock, no network
   call, no database read, in the hot path
   job: make the actual business decision inside a nanosecond-
   scale budget, which is only possible because there is nothing
   else it has to wait on to make that decision
   analogy: a single bank teller, at one specific counter, who
   already has the ledger open on the desk and never has to phone
   anyone to check a balance

   |
   v

[Event store: append-only log, optimistic concurrency on write]
   the ONLY durable write in the entire flow; the aggregate emits
   one or more events (OrderPlaced, TradeMatched) and appends them
   to that entity's stream, guarded by an expected-version check
   (append only succeeds if the stream is still at the version the
   aggregate last read; if another writer got there first, in a
   design with more than one writer per entity, the append is
   rejected and retried) rather than a held lock
   job: be the single, permanent, provably-ordered source of truth
   analogy: a ship's log, written once per entry, in order, never
   erased, never edited, only ever added to

   |
   v

[Snapshot writer, async, off the hot path]
   periodically (every N events, or every T seconds) serializes
   the current in-memory aggregate state and writes it beside the
   log, tagged with the event version it corresponds to
   job: bound how far back a rebuild ever has to replay from, so
   recovery time stops growing as the log grows
   analogy: a museum photographing the assembled exhibit every so
   often, so restoring it after a fire means starting from the
   most recent photo, not rebuilding every piece from scratch

   |
   v

[Change feed: the log, tailed]
   the same durable log the write path just appended to, read from
   the outside, in order, by anything downstream; this is Day 17's
   write-ahead-log lesson wearing a new hat, the log the system
   already produces IS the change feed, nothing new is bolted on
   job: let every downstream consumer see every change, without
   adding one synchronous cost to the write path that produced it

   |
   v

[Projectors: one per read shape needed]
   each projector is a pure function from (an event, its own
   current projection state) to (a new projection state); it
   consumes the change feed and updates ONE denormalized read
   store shaped for exactly one query pattern (the live order
   book view, a trader's position statement, a compliance audit
   view, a real-time dashboard)
   job: pay the cost of shaping data for reads ONCE, asynchronously,
   instead of on every single read request
   analogy: a print shop running the same negative through several
   different presses, each producing a different finished format,
   a poster, a postcard, a proof sheet, from the one original

   |
   v

[Read stores: denormalized, one per projection, eventually
 consistent]
   ordinary databases, caches, or search indexes, each holding
   exactly the shape one query pattern needs, requiring no joins
   and no replay at query time
   job: answer reads in milliseconds, cheaply, at whatever scale
   read traffic demands, completely independent of write volume

   |
   v

[Query API]
   reads ONLY from projections, never from the event store and
   never from the live aggregate
   job: keep the read path and the write path from ever competing
   for the same resource
```

---

## 4. The transferable mechanisms

- **Append-only log as the sole source of truth.** Nothing is ever mutated in place; state is only ever added to, never overwritten. This is the same primitive as Day 17's write-ahead log, Day 23's content-addressed storage, and Day 58's shadow-copy-plus-atomic-swap: whenever "what actually happened" needs to be provably reconstructible, the system that keeps the log as the only fact, and derives everything else from it, cannot have the log and the derived state disagree, because there is only one of them.

- **Single-writer principle: shard by entity, not by request.** Every command for one entity (one account, one order book, one project) is routed to the same partition, so there is never more than one writer touching that entity's state at once, which means the write path never needs a lock to protect it. This is Day 10's consistent hashing applied to eliminate contention rather than just to distribute load: the goal isn't spreading writes evenly, it's guaranteeing that no two writers are ever fighting over the same piece of state in the first place.

- **Optimistic concurrency via compare-and-append, not held locks.** Where an entity does need protection against a genuinely concurrent writer, the mechanism is an expected-version check on append (append succeeds only if the stream is still where the writer thinks it is; otherwise it's rejected, and the writer re-reads and retries), not a lock held for the duration of the operation. A rejected append costs one wasted attempt. A held lock costs every other writer waiting behind it for as long as it's held, which is exactly the cost Section 1's arithmetic ruled out.

- **CQRS: split the write model from the read model, and give every distinct read pattern its own projection.** The write side stays a single, ordered, strongly consistent log per entity. The read side can be as many independent, denormalized, precomputed views as there are genuinely different query shapes, each one built once by a projector instead of assembled on every request. This directly sidesteps Day 46's distributed-join problem: instead of joining across shards at read time, the join, if there is one, gets done once, at write time, by a projector, and the read path only ever does a single-key lookup against an already-shaped table.

- **Snapshotting to bound recovery and replay time.** Left unchecked, replay time grows linearly with the size of the log, which is exactly the failure this lesson's fourth death describes. A periodic snapshot caps that growth: rebuilding an aggregate or a projection never means replaying from the beginning, only from the most recent snapshot forward. This is the same checkpoint discipline as Day 21's LSM-tree compaction and Day 40's stream-processing state checkpoints: bound the cost of recovery by periodically committing to a known-good point, so recovery cost is a function of "since the last checkpoint," never "since the beginning."

- **Idempotent, fully rebuildable projectors.** A projector must be safe to run twice over the same event, and a read store must be safe to throw away and rebuild entirely from the log at any time, with no loss, because that rebuildability is what makes the whole system self-healing: a corrupted or buggy projection is never a permanently lost piece of data, it's a temporary gap that a replay closes. This only holds if projector logic is a pure function of the event stream, with no side effects that aren't themselves safe to redo, the same discipline Day 12's idempotency-key lesson requires of any retried operation.

---

## 5. The trade-offs

**Consistency vs. availability, and the two halves of this design make opposite choices on purpose.** The write side, one entity's event stream, is CP: strongly consistent, strictly ordered, and an append either fully succeeds in its correct position or is rejected, there is no partial or ambiguous state. The read side, every projection, is AP: it will always answer, but it might answer with data that is a few events behind the true, current state, because the projector hasn't caught up yet. LMAX's own system draws this line concretely: the order-matching engine itself cannot tolerate two threads disagreeing about which order matched first, that has to be perfectly, immediately consistent. A trader's "recent activity" dashboard, or a compliance export, can lag by a few hundred milliseconds with zero real harm, nobody is placing a trade against a number on a dashboard. The mistake this design specifically avoids is applying the same consistency requirement to both halves, which either makes the read side unnecessarily slow (querying the strongly consistent write path for every dashboard refresh) or makes the write side unnecessarily fragile (trying to keep every downstream view instantly consistent with every write).

**Cost vs. latency, paid at different points in the pipeline.** The write path gets extremely cheap: an append, no joins, no read-modify-write, no lock. That cost doesn't disappear, it moves. Storage cost grows without bound unless events are archived or compacted, because the design's entire premise is "never delete evidence." And the cost of shaping data for reads gets paid up front, at design time and at write time (building and maintaining N separate projections), rather than deferred to query time. A system with one obvious, stable query pattern and no real audit requirement pays this cost for nothing, which is exactly why Fowler's own writing on CQRS is a warning as much as an endorsement: for most systems, splitting the model in two "adds risky complexity," and it's only worth paying for when the read shapes and the write shape have genuinely diverged enough that one normalized table can no longer serve both well. This lesson's whole architecture is a real answer to a real problem at LMAX's scale and audit requirements. It is not a default any system should reach for.

**The trade-off inside event sourcing itself: unlimited history vs. bounded operational cost.** Keeping every event forever is what makes the audit trail structurally trustworthy, but it's also what makes replay time grow without bound if nothing intervenes. Snapshotting is the tax paid to keep both: a small amount of extra storage and a small amount of extra write-side complexity, spent specifically to stop the audit trail's greatest strength (it remembers everything) from becoming its greatest operational liability (it has to re-read everything to answer a simple question).

---

## 6. The systems-thinking lens

The feedback loop worth naming here is a **replay storm**, and it's the event-sourced cousin of Day 58's lock convoy: a small, well-intentioned recovery action triggers a burst of load large enough to become the actual incident. Here's the shape of it. A projector has a bug, or a read store gets corrupted, or a new projection needs to be built for the first time. The fix, correctly, is "replay the log and rebuild the projection from scratch," and that's exactly what the architecture promises is safe to do. But replaying the *entire* history of a log that's been running for months generates a sudden, sustained burst of read load against the event store, exactly the load a system sized for steady, incremental "tail the new events" traffic was never provisioned for, and if the rebuild writes into the *same* read store that's still serving live queries, it also generates a burst of write load competing with live traffic for the same resource. The kitemetric account from Section 2, an 18-hour replay that blew an SLA, is this loop playing out for real: the recovery action itself became the reason recovery took so long, not a second, unrelated failure.

The senior fix is not "give the event store more read capacity so replays finish faster," that only raises the threshold at which the next replay storm happens, it doesn't remove the loop. The actual fix breaks the loop the same way Day 58's shadow-copy pattern does: **rebuild into a brand-new, isolated read store, never into the live one, and cut over with a single atomic pointer swap only once the rebuild has fully caught up.** A replay against a fresh, unused sink can run at whatever pace is safe without ever competing with live traffic for the same table, because nothing is reading from it yet. Pair that with partitioning the replay itself (rebuilding one entity's projection at a time, or one shard at a time, rather than the whole log in one pass) and with separating the event store's read capacity for live tailing from its read capacity for bulk historical replay (a different consumer group, a different set of replicas), so a large backward-looking replay physically cannot starve the small forward-looking tail that live projectors depend on. The mechanism is identical to Day 13's backpressure lesson in spirit: the thing generating a burst of load must never be allowed to compete, unthrottled, with the thing serving live traffic, on the same shared resource, at the same time.

---

## Map to Rare.lab's stack

Rare.lab already has half of this pattern in place, for a reason unrelated to this lesson: R2 scene storage is content-addressed and immutable, so a save writes a new object at a new address and flips a manifest pointer, never edits an existing object in place. That's Section 4's "append-only, never mutate" mechanism, already correctly applied, one level up, to whole scene versions.

The gap is one level down, inside a single scene: right now, if only the *current* graph gets persisted, Rare.lab has exactly this lesson's second and fourth deaths waiting for it. The second death: there's no structural way to answer "why does this shader look different than it did yesterday," because the sequence of edits that produced the current graph isn't preserved, only the result is, the same problem an `UPDATE`-only balances table has with an auditor. The fourth death doesn't apply yet, because there's no event log to grow unboundedly, but it's the shape of the mistake to avoid when one gets built: don't store *only* the full history with no snapshot, or a future "load this project" call inherits the same unbounded-replay risk LMAX's naive design would have had.

The concrete move: treat a project's graph not as a row that gets overwritten, but as an ordered stream of graph-edit commands (`NodeAdded`, `InputWired`, `ParameterChanged`, `NodeRemoved`), appended per project, with periodic snapshots of the fully materialized graph so loading a project never means replaying its entire history from node one. From that single stream, at least three genuinely different projections become nearly free instead of three separately hand-built systems: the **live canvas view** every open editor session renders from (rebuilt incrementally as new events arrive, exactly the projector role in Section 3), the **compiler's input**, an AST-shaped projection of the same events, structured for code generation rather than for rendering, and an **undo/redo and version-history view**, where undo stops being a special-cased inverse operation and becomes "the projection as of N events ago," the same mechanism Section 3 uses for point-in-time reconstruction generally. When real-time multiplayer editing (Day 3's Figma problem) eventually lands on top of this, the command stream is also the natural place multiple collaborators' edits get ordered and reconciled, instead of inventing a second, parallel ordering mechanism from scratch.

The embeddable runtime's single shared WebGL context deserves the same single-writer discipline Section 4 describes for LMAX's Business Logic Processor, for a mechanically identical reason: a WebGL context is one hot, stateful resource, and two uncoordinated callers issuing GL state changes against it concurrently produce the exact class of bug lock contention produces in a database, silent corruption of shared state that's expensive to diagnose after the fact. The browser's single-threaded event loop already provides this for free today, by accident. The lesson worth carrying forward deliberately, before a worker thread, an offscreen canvas, or a second concurrent render path changes that: route every mutation to the shared context through one ordered queue with a single writer, the same way LMAX routes every trade through one Business Logic Processor, rather than relying on JavaScript's current single-threadedness to keep providing that guarantee by coincidence.

The honest caveat from Section 5 applies here too: this is not a case for putting every Supabase table behind an event log. Account settings, billing state, simple admin tables, anywhere the read shape and the write shape are already the same shape, gain nothing from this and only pick up Fowler's "risky complexity" tax. The graph-edit history is the specific place at Rare.lab where the read shapes (canvas, compiler, undo) and the write shape (one ordered stream of edits) have genuinely diverged, which is exactly the condition Section 5 says makes CQRS worth its cost.

---

## Sources

- [The LMAX Architecture, Martin Fowler](https://martinfowler.com/articles/lmax.html): primary source for the Business Logic Processor's single-threaded, in-memory, event-sourced design and the 6,000,000 business-logic-operations-per-second figure on commodity hardware. Direct fetch of martinfowler.com was blocked by this session's network egress policy; relayed through search-indexed summaries rather than a first-hand read, and worth re-verifying directly.
- [Event Sourcing, Martin Fowler (eaaDev)](https://martinfowler.com/eaaDev/EventSourcing.html): primary source for the 2005 definition of event sourcing, capturing every state change as an event and deriving current or historical state by replay. Fetch blocked; relayed via search summary.
- [CQRS, Martin Fowler (bliki)](https://martinfowler.com/bliki/CQRS.html): source for the explicit warning that CQRS "adds risky complexity" for most systems and is only warranted when read and write models genuinely diverge. Fetch blocked; relayed via search summary.
- [CQRS Documents, Greg Young, 2010](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf): the original document naming and defining the CQRS pattern, separating command handling from query handling into distinct models. Not fetched directly; relayed via search summary.
- [LMAX Disruptor, GitHub repository](https://github.com/LMAX-Exchange/disruptor): fetched directly (partial: repository metadata and header confirmed; full README/technical-paper content was not retrievable in this session's fetch). Source for the Disruptor's identity as a high-performance inter-thread messaging library built on the single-writer principle and mechanical sympathy with CPU cache behavior.
- ["Understanding the LMAX Disruptor" and related technical summaries](https://itnext.io/understanding-the-lmax-disruptor-caaaa2721496), search-indexed: source for the benchmark figures that a three-stage Disruptor pipeline achieved roughly three orders of magnitude lower mean latency and roughly eight times the throughput of an equivalent queue-based design at the same configuration. Relayed via search rather than a first-hand read of LMAX's original technical paper; worth re-verifying against the primary PDF directly.
- [Latency Numbers Every Programmer Should Know, gist.github.com/jboner/2841832](https://gist.github.com/jboner/2841832): fetched directly. Source for the specific nanosecond figures used in Section 1's arithmetic: mutex lock/unlock (25 ns), main memory reference (100 ns), round trip within the same datacenter (500,000 ns), and read 1 MB sequentially from SSD (1,000,000 ns).
- [Event Sourcing Fails: 5 Real-World Lessons, Kite Metric](https://kitemetric.com/blogs/event-sourcing-fails-5-real-world-lessons): source for the two production incident figures in Section 2, a 3 TB event history producing minutes-long balance-reconstruction queries, and an 18-hour replay of two weeks of events after a projection-corrupting bug, missing the team's recovery SLA. Fetch blocked; relayed via search summary; company names were not given in the summarized source and are treated here as real but anonymized incidents, not attributed to a named company.
- [Snapshots in Event Sourcing, Kurrent (formerly EventStoreDB)](https://www.kurrent.io/blog/snapshots-in-event-sourcing/): source for the snapshot-plus-tail-replay mechanism used in Section 3 and Section 4 to bound recovery time. Fetch blocked; relayed via search summary.
- [Event sourcing, CQRS, stream processing and Apache Kafka: What's the connection?, Confluent](https://www.confluent.io/blog/event-sourcing-cqrs-stream-processing-apache-kafka-whats-connection/): source for the general pattern of building multiple independent materialized-view projections from one event log, used in Section 3's projector layer and the Rare.lab three-projections mapping. Fetch blocked; relayed via search summary.
- [EventStore/EventStore, GitHub repository](https://github.com/EventStore/EventStore): fetched directly. Confirms KurrentDB/EventStoreDB's positioning as a database purpose-built for event-sourced, event-native architectures with an integrated streaming engine; no specific throughput numbers were present in the fetched README.
- [Innovation at Mettle: Double entry and event sourcing, Mettle (NatWest-backed fintech)](https://www.mettle.co.uk/blog/innovation-at-mettle-double-entry-and-event-sourcing/): source for a named, real production example of event sourcing combined with double-entry accounting in a live banking system. Fetch blocked; relayed via search summary.

---

*Inference vs. fact, stated plainly: the LMAX 6,000,000-operations-per-second figure, the Disruptor's published benchmark comparison against queue-based designs, Fowler's original event sourcing and CQRS definitions, and the two production replay-time incidents are all documented claims from named primary or clearly identifiable secondary sources, but the majority of them were relayed through this session's web search rather than a first-hand read of the original page, because direct fetches to martinfowler.com, kitemetric.com, kurrent.io, and confluent.io were all blocked by this session's network egress policy; the "Latency Numbers Every Programmer Should Know" gist and the LMAX Disruptor and EventStore GitHub repositories were fetched directly and can be treated with higher confidence. The nanosecond-budget arithmetic in Section 1 (166.6 ns per order, and the multiples-of-budget comparisons against a lock, a datacenter round trip, and an SSD read) is this lesson's own derivation, built by dividing 1 second by the directly-sourced 6,000,000 figure and comparing it against the directly-fetched latency table, not a calculation LMAX or the gist's author published themselves. The architecture diagram's specific ten-layer framing, the "replay storm" name and its parallel to Day 58's lock convoy, and the entire Rare.lab mapping, including the WebGL-context-as-Business-Logic-Processor analogy and the three-projections-from-one-graph-stream proposal, are this lesson's own synthesis on top of the documented mechanics above, not a claim that LMAX, Fowler, or Confluent describe their systems in these exact terms.*
