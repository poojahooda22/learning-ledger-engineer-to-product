# Day 92: Millions of people want to buy and sell the same stock at the same instant, and every one of them must be treated fairly, in order, with no lost trade. How does an exchange matching engine work? One thread, an order book, a sequencer, and replay

**Date:** 2026-10-18
**Difficulty:** Advanced (the OPPOSITE of most lessons: here the answer is to NOT scale out the hot path)
**Topic:** Earlier lessons covered the pieces around this one: event sourcing (Day 59), Raft consensus (Day 11), idempotency (Day 12), the double-entry ledger (Day 82), real-time bidding (Day 56), hot keys (Day 16), tail latency (Day 76). Today's lesson is the one place where the senior answer is "keep the critical path on a single thread", and it explains why that beats sharding and locks.
**Stack relevance:** Rare.lab has one shared WebGL context, which is also a single-threaded, strictly ordered resource. Section 9 maps the exchange pattern onto the scene editor and the compile queue.

---

## 0. The framework (same six-step recipe as Day 74 and Day 91)

The LeetCode mock interview you pasted is already taught in [Day 74](074-leetcode-online-judge-end-to-end.md). Today is a new topic that reuses its recipe. Memorize the recipe, not the product:

1. **Functional requirements (and out of scope).** Place a limit order (buy 100 shares at no more than 50.00). Place a market order (buy now at any price). Cancel an order. See the order book (best prices and sizes). Get fills (confirmations) back. Out of scope: login, KYC, clearing and settlement (the money actually moving, which takes a day or two), margin, tax.
2. **Non-functional requirements.** Fairness (same price, first in line wins). Determinism (same inputs, same outputs, every time). Durability (an accepted order is never lost, even on power cut). Low and flat latency (microseconds, and the p99.99 matters more than the average). High availability (a failover in seconds, with zero lost or duplicated trades).
3. **Core entities.** Order, Order book (one per symbol), Price level, Trade (a fill), Sequence number, Journal.
4. **API.** Binary protocols in real exchanges (FIX, ITCH, OUCH). In an interview, say: `POST /orders {symbol, side, type, price, qty, clientOrderId}`, `DELETE /orders/{id}`, `GET /book/{symbol}`. Note `clientOrderId`: it is the idempotency key from Day 12.
5. **Naive high-level design.** Client, API server, SQL database. Insert the order row, then run `SELECT ... WHERE side='SELL' AND price <= 50.00 ORDER BY price, created_at LIMIT 1 FOR UPDATE`, match, update both rows.
6. **Deep dives.** Break the naive design with a number, then fix each break and say the trade-off out loud.

The interviewee's best move in the transcript was "do not run user code on the API server; isolate it and put a queue in front". Today's equivalent: "do not let many threads fight over the book with locks; put ONE thread in front of it and a queue before that thread".

---

## 1. Named company and the breaking number

**LMAX Exchange** (a London foreign-exchange trading venue). Its engineers built a design that Martin Fowler documented in 2011. The headline from that article, confirmed in a search today: the core "Business Logic Processor" ran about **6 million orders per second on ONE thread**, in a benchmark on a 3 GHz dual-socket quad-core Nehalem server with 32 GB RAM. That is a benchmark, not production traffic, and the article is the primary source. The design is surrounded by a lock-free queue called the **Disruptor**.

The breaking number for a naive design is the same one in your transcript's style: how many orders hit **one hot row set**?

- A busy stock or futures contract can see **100,000 or more order events per second** in a spike (inference: public exchange feeds routinely show bursts of that order on the most active symbols; exact figures vary by venue and I did not verify a specific one today).
- Every one of those events must touch the SAME small structure: the best bid and best ask of one symbol. It is a hot key (Day 16) with no way to spread the heat.
- A SQL database doing `SELECT ... FOR UPDATE` on those rows manages maybe **a few thousand lock-protected transactions per second per hot row** (inference, typical for row-lock contention on Postgres or MySQL; Day 29 and Day 64 show the same wall). It serializes at a millisecond scale. The exchange needs **microseconds**: roughly 1,000 times faster.

So the problem is not "too many users". It is "too many users all fighting over the same two numbers, and the order they arrive in IS the product".

---

## 2. Why the naive design dies

Picture one SQL database and a pool of API servers.

**Failure 1: Lock contention on the hot rows.**
- 50 API threads all run `FOR UPDATE` on the best-ask rows of ACME stock. 49 wait while 1 works.
- Analogy: one cashier window, 50 people, and each person must fill in a paper form that takes a full minute while the others watch.
- Adding more API servers makes it WORSE (more waiters), not better. Throughput is capped by the single hot row, not by CPU.

**Failure 2: Unfairness hidden inside the locks.**
- The rule of an exchange is **price-time priority**: best price first, and among equal prices, first-in wins. Who is "first"?
- With many app servers, `created_at` comes from different clocks, and a request that sat in a network buffer arrives "later" than one that physically came after. Two traders can each be sure they were first. Regulators care. Money is attached.
- Analogy: two people both swear they were first in the queue at a counter, and the only evidence is two watches that disagree by 3 ms.

**Failure 3: Latency that is not flat.**
- Disk fsync, garbage collection, lock handoff and a network hop to the database add random jitter. A trader whose order sometimes takes 40 ms instead of 400 microseconds is being treated differently from the others. Fairness includes tail latency (Day 76).

**Failure 4: Crash in the middle.**
- Server dies after writing the buyer's row and before writing the seller's row. Now one side believes it traded. A multi-row transaction can fix this, but it makes Failures 1 and 3 worse.

The correct response is to stop treating this like a CRUD app. The matching step is a tiny, pure function over in-memory state. Make it so fast and so alone that no lock is needed.

---

## 3. The architecture, top to bottom

```
 Traders (firms, brokers, apps)
        |  FIX / binary over TCP, co-located cross-connects
        v
 [1] GATEWAYS / ORDER ENTRY    many, stateless, scale OUT
        |  validate, authenticate, risk checks, assign clientOrderId
        v
 [2] SEQUENCER                 ONE per shard of symbols
        |  stamps every message with the next sequence number N
        |  writes it to the JOURNAL (and replicates it) BEFORE anything acts on it
        v
 [3] MATCHING ENGINE           ONE thread per symbol group, in memory
        |  order book: bids and asks as price levels with FIFO queues
        |  match -> emit events: Accepted, Trade(fill), Canceled, Rejected
        v
 [4] EVENT STREAM (output)     publishes every event with its sequence number
        |
        +--> Market data publisher   -> everyone sees the book (multicast)
        +--> Order-status / fills    -> back to the right gateway and trader
        +--> Drop copy / audit       -> regulators, risk, surveillance
        +--> Clearing handoff        -> settlement happens later, elsewhere

 [5] HOT STANDBY               replays the same journal, ends in the same state
 [6] SNAPSHOT + JOURNAL        recovery: load last snapshot, replay the tail
```

Each layer has one job and an analogy:

- **Gateways.** Job: take the messy outside world (many connections, malformed messages, fat-fingered orders) and hand clean, validated messages inward. Analogy: the airport check-in desks and security. Many desks, they can queue and grow, none of them decides who boards first.
- **Sequencer.** Job: decide a single total order for every message, forever. Analogy: the take-a-number ticket dispenser. Only one dispenser, so numbers never collide, and the number IS the truth about who was first.
- **Journal (write-ahead log).** Job: persist the ordered message before processing (Day 17). Analogy: the court stenographer who writes down every statement before the judge responds. If the judge faints, a new judge reads the transcript and rules the same way.
- **Matching engine.** Job: apply each message to the book, one at a time, in sequence order. Analogy: a single referee. Disputes are impossible because there is no second referee.
- **Order book data structure.** Job: find the best price in O(1) and the next order at that price in O(1). Section 5 draws it.
- **Output stream.** Job: tell everyone what happened, stamped with sequence numbers so consumers can detect gaps. Analogy: a radio broadcast where each sentence is numbered, so a listener who missed sentence 4,812 knows to ask for it.
- **Hot standby.** Job: a second copy that has applied exactly the same messages. Analogy: a shadow referee who writes the same scoreboard and can step in the moment the first collapses.

---

## 4. A real order, walked end to end

Let's use a made-up stock **ACME**. The book right now:

```
 ASKS (sellers)           BIDS (buyers)
 50.03   200              50.00   500   <- best bid
 50.02   300 (A: 100, B: 200)  49.99   150
 50.01   100  <- best ask
```

Order of arrival at the sequencer (the sequence numbers are what make this unambiguous):

- **#1001** Trader Ravi: SELL 100 ACME @ 50.01. Rests as best ask.
- **#1002** Trader Meera: BUY 250 ACME @ 50.02 (limit: willing to pay up to 50.02).

What the single matching thread does with #1002:

1. Best ask is 50.01 with 100 shares (Ravi). 50.01 <= 50.02, so it crosses. Trade 100 @ **50.01** (the resting order's price, not Meera's limit). Event: `Trade(seq 1002, buyer Meera, seller Ravi, 100 @ 50.01)`.
2. Meera has 150 left. Next ask level: 50.02 with A (100) then B (200), in FIFO order. A is first. Trade 100 @ 50.02 with A. Then 50 @ 50.02 with B (B has 150 left).
3. Meera is fully filled (250). Nothing rests. Events go out in order, all stamped from the same message #1002.

Notice what never happened: no lock, no thread handoff, no database call. It took a handful of pointer hops through memory, in the order of **hundreds of nanoseconds to a few microseconds** (inference, typical for tuned in-memory books; LMAX's own 6 million per second benchmark means under about 170 nanoseconds per order on average for the whole processor).

If the box dies right after #1002, the standby reads the journal up to #1002 and rebuilds the same book. Same Meera fills, same prices, same Ravi leftover (zero). Nobody is paid twice, because the engine is deterministic.

---

## 5. Under the hood: the data structures

**The order book per symbol.** Two sides, bids and asks. Each side is **price levels sorted by price**, and each level is a **FIFO queue of orders** (that queue IS the time priority).

- Price levels: a sorted structure keyed by price. Two common choices:
  - A **balanced tree or skip list** (O(log P) to find or insert a level, P = number of active price levels, usually tens or hundreds).
  - An **array indexed by price tick** (because prices move in fixed increments like 0.01, price 50.01 maps to an integer index). O(1) lookup, and most orders arrive near the best price so the cache stays hot. Memory cost is the width of the price range (inference: array-by-tick designs are widely used in low-latency engines; I did not verify a specific exchange's choice).
- Best price: keep a pointer to the best bid and best ask. Updating it is O(1) in the common case.
- Each level: a **doubly linked list** or ring buffer of orders in arrival order. Push to the back on a new order, pop from the front on a fill. Both O(1).
- **Cancel by order id** must be O(1) too, otherwise cancel-heavy traffic (the majority of exchange messages are cancels and modifies, a widely quoted pattern in market microstructure) is slow. So keep a **hash map orderId to node pointer** and unlink the node from its level's list in O(1).

Concrete: cancel of Ravi's #1001 = hash lookup, unlink node, if the level is now empty drop the level and move the best-ask pointer to the next one. A few pointer writes.

**Why a single thread, and why that is fast.**
- A lock costs tens to hundreds of nanoseconds even when uncontended, and microseconds when contended (a cache line bouncing between cores, or a thread parking). A single thread has none of that.
- Everything lives in L1/L2 cache. No allocation on the hot path (pre-allocated object pools, to avoid garbage-collection pauses, which are the classic latency spike in Java systems; LMAX and the Disruptor were designed around this).
- Fowler's article lists the lessons: keep state in memory, use event sourcing (the log of inputs IS the database), and treat the disk and network as separate threads that feed and drain queues around the one business thread. That matches the diagram above.

**The sequencer.** Just a counter. For each inbound message: `seq = next++`, append to the journal, hand to the matcher. Everything downstream is a pure function of the ordered stream. That gives you three free gifts:
1. **Fairness** (one global order, no clock arguments).
2. **Replay** (feed the same stream to a fresh process and get the same book).
3. **Gap detection** (every consumer knows `seq` must go 1, 2, 3, ... and can request a resend).

**The Disruptor, in plain words.** A pre-allocated **ring buffer** (a fixed circular array) with a producer cursor and consumer cursors. Producers claim a slot by incrementing a counter, write into it, publish. The consumer reads the next slot in order. No locks, no allocation, no queue nodes scattered across the heap, and the array is contiguous so the CPU prefetcher helps. Multiple readers (journaler, replicator, business logic) can each follow behind at their own cursor and do their job in parallel on the SAME stream. Source: Fowler's LMAX article, which I could not re-fetch from this sandbox today (host blocked), so the specifics above are stated from my reading of it earlier, the 6 million figure and hardware were confirmed through search.

**Determinism rules (the thing seniors check).** For replay to work, the matching step must not read the wall clock, call random, iterate a hash map in unspecified order, touch the network, or depend on thread timing. Time comes in as a field on the sequenced message (stamped by the sequencer). One violated rule and your standby diverges, and divergence in a trading system means a different set of fills.

**Failover without losing a trade.**
- Pattern A, **hot standby fed by the same stream**: standby applies the identical messages. On primary failure, standby already has the same state. Needs a way to avoid two primaries (Day 24 fencing tokens).
- Pattern B, **consensus-replicated log (Raft)**: the sequencer writes to a 3-node replicated log and only acknowledges once a majority has it (Day 11). A search today found an open-source project reporting end-to-end order matching in tens of microseconds on Aeron Cluster (Raft, 3 nodes) for durability, a self-published claim, not a statement about any real exchange. I could not verify which approach CME, Nasdaq, or Jane Street use internally.
- Recovery time = load last snapshot + replay the journal tail. Snapshot every few minutes bounds replay to seconds.

**Scale story at three tiers.**

- **1,000 orders/day (a toy or a campus trading game).** One Postgres table, one `SERIALIZABLE` transaction per match. It is correct and nobody notices the lock. Do NOT build a ring buffer.
- **100,000 orders/day to a few million.** Still one process. Move the book to memory, keep an append-only journal (a file or a table you only insert into), and replay on restart. That is event sourcing (Day 59). What breaks next: the process becomes a single point of failure, so add a hot standby reading the same log.
- **Hundreds of millions of messages per day (a real exchange, and the shape of a big crypto venue).** One symbol can still be handled by one thread, because the per-symbol rate is bounded. **Scale by symbol, not by thread count per symbol.** Partition symbols across many matching-engine instances (Day 10 and Day 45 logic: shard by symbol, and the shards never need to talk). Hot symbols get their own core. What breaks: a market-wide event sends every symbol hot at once, gateways are flooded, and you need admission control and rate limits at the edge (Day 13, Day 71) so the sequencer's input queue never grows unbounded. Cross-symbol orders (a spread or a basket) are the hard case: either they live in the same shard by design, or you need a coordinator, which brings back everything this design was trying to avoid.

---

## 6. The transferable mechanisms

1. **Single writer principle.** If many things must agree on one piece of state, let exactly ONE thread own it and send everyone else messages. You get serializability without locks. (Same idea as Redis being single-threaded for commands, and an actor model's mailbox.)
2. **Sequence number as the source of truth for order.** Do not trust timestamps from different machines. A counter at one point defines "who was first". Used in Kafka partition offsets (Day 42) and Postgres LSNs (Day 17).
3. **Event sourcing and deterministic replay.** Persist inputs, not just state. State becomes a cache you can rebuild. This also gives you perfect debugging: replay the exact production day on your laptop.
4. **Write-ahead, then act.** Journal before processing, so a crash can never leave an acknowledged-but-forgotten message.
5. **Shard by independent key, never by contention.** Symbols are independent, so each gets its own writer. Do not split ONE book across threads.
6. **Idempotency key on the edge.** A client that times out and retries must not create a second order. `clientOrderId` is deduped at the gateway/sequencer (Day 12).
7. **Separate the slow I/O from the hot thread.** Disk, network, and fan-out each run on their own thread following the same ring buffer. The business thread only ever touches memory.

---

## 7. Trade-offs (CAP made concrete, per data type)

| Data | Choice | Why |
|---|---|---|
| Order sequence and fills | **Consistency over availability.** If the primary and standby cannot agree, halt trading for that shard. | A wrong fill is a legal and financial incident. Real exchanges halt a symbol or market rather than risk divergence ("circuit breaker" in the human sense). |
| Market data (the public book feed) | **Availability and speed, eventual.** Consumers may see a slightly stale book, and recover gaps by sequence number. | A late tick is fine, a wrong fill is not. Multicast is used, so reliability is rebuilt by sequence numbers (inference from common practice). |
| Order history, reports | **Eventual.** Derived from the event stream. | Can be rebuilt from the journal at any time. |
| Clearing and settlement | **Strong, but asynchronous**, hours later. | Matching decides who traded with whom; moving money is a separate, ledger-style system (Day 82). |

**Cost vs latency.** The whole design buys microseconds with money: co-location (your servers in the exchange's data center so the cable is short, because light in fibre travels about 5 microseconds per kilometre, the speed-of-light wall again), kernel-bypass networking, pinned CPU cores, and **one big expensive machine per shard rather than a fleet of cheap ones**. Vertical scaling is the correct answer on the hot path. You scale OUT only where state is independent (gateways, market data fan-out, symbols).

---

## 8. The systems-thinking lens

**The feedback loop here is the unbounded input queue plus retries (a metastable failure).**

1. A burst arrives. The matcher is fast but the burst is faster, so the sequencer's input queue grows.
2. Queue wait time rises. Traders' clients hit their timeout and **cancel and resend** (or send more cancels to pull stale quotes). Those are NEW messages, so load goes UP exactly when the system is slow.
3. Market makers see delays, widen quotes or pull them, which causes price jumps, which causes more orders. Load rises again.
4. Even after the original burst ends, the backlog plus the retries keeps the queue full. The system is stuck in a bad stable state until someone intervenes.

**The senior fix breaks the loop, it does not add capacity:**
- **Admission control at the gateway**: a hard cap on messages per second per participant (token buckets, Day 8 and Day 71). Excess is rejected immediately with an explicit reason, not queued.
- **Bounded queue in front of the sequencer**: when full, reject, never grow. Fast failure beats slow success (Day 13).
- **Cancels are prioritized or have a dedicated lane**, since cancelling reduces risk and load.
- **Idempotent client order ids** so a retry is not a duplicate.
- **Market-wide and per-symbol halts** (circuit breakers) that deliberately stop matching for a short, announced pause to let the queue drain and humans reassess. The analog of a load-shedding switch.

Capacity helps a little, but the loop is a loop: retries create load that creates retries.

---

## 9. Map to Rare.lab (Supabase Postgres with RLS, Cloudflare R2 with content-addressed immutable scenes plus a manifest, an embeddable runtime with one shared WebGL context)

**What you already have that matches today's patterns:**
- **R2 content-addressed immutable scene JSON** is event-sourcing-shaped: immutable inputs, never edited in place. Your manifest is the tiny mutable pointer (like the "best bid"). This is the right split.
- **One shared WebGL context** is a single-writer. GL calls from many places would corrupt state, so you funnel them through one thread. That is exactly the "one referee" principle. Keep it, and keep all draws as a queue of commands rather than letting widgets call GL directly.
- **RLS in Postgres** is your gateway layer: reject bad input before it gets near the core.

**Where the next ceiling is:**
1. **Concurrent edits to one manifest.** If two collaborators or two tabs publish at once, last-write-wins will silently drop one. Apply the sequencer idea: a monotonic `version` per project, with a compare-and-swap update (`UPDATE ... SET manifest = $new, version = version + 1 WHERE id = $id AND version = $expected`). A failed CAS returns a conflict, never a lost write. Cost: one indexed row update. Do this before you add real-time collaboration (Day 3, Day 15, Day 73).
2. **Idempotency key on publish.** Key it by the content hash of the scene JSON. Double-clicking Publish then writes the same R2 object once (content addressing already helps) and one manifest bump.
3. **Command log for the editor.** Store editor operations as an ordered, replayable list (op #1, #2, ...) per project in addition to snapshots. You get undo/redo, crash recovery, and a way to reproduce a user's bug report by replaying their exact session. Determinism rule applies: no `Math.random()` or `Date.now()` inside the replayed reducer, pass them in as fields.
4. **Compile queue.** If many users hit "compile to shippable code" at once, use a bounded queue with rejection (Day 9 and Day 13), not an unbounded one, or a launch day will create the retry spiral from Section 8.

---

## 10. Reference summaries (what is inside each, in plain words)

Fetch note: the sandbox blocked martinfowler.com and janestreet.com for direct reading, so some summaries below come from search-result excerpts and my prior knowledge, and I mark which.

- **[The LMAX Architecture, Martin Fowler (2011)](https://www.martinfowler.com/articles/lmax.html).** Confirmed via search today: the Business Logic Processor handles about 6 million orders per second on one thread (benchmark on a 3 GHz dual-socket quad-core Nehalem Dell server, 32 GB RAM), runs fully in memory using event sourcing, and is surrounded by Disruptors, lock-free queues. Plain-language point: the team found that locks, queues and disk I/O cost more than the actual business logic, so they gave the logic its own thread and fed it from a ring buffer. From my memory of the article (not re-read today): input is journaled and replicated to a standby before processing, and state is rebuilt by replaying events.
- **[How to build an exchange, Jane Street tech blog](https://blog.janestreet.com/how-to-build-an-exchange/).** Confirmed via search: an announcement of a public tech talk by Brian Nigito, giving an overview of the architecture of a modern exchange, inspired by Jane Street's JX crossing engine. I did not read a transcript, so I do not summarize the content beyond that. If you want the real thing, search YouTube for Brian Nigito's "Building an Exchange" talk.
- **[Designing a matching engine that keeps price-time priority (techinterview.org)](https://www.techinterview.org/post/3233477258/matching-engine-price-time-priority-system-design/)** and **[Order-Matching Engine Internals](https://www.techinterview.org/post/3233474599/order-matching-engine-internals/?format=md)**. Interview-prep articles. From search excerpts: describe a single sequencer thread stamping a monotonic sequence number to fix total order, and journaling every event before acting so a crash can be recovered by replay. Secondary sources, useful for vocabulary, not authoritative on any real exchange.
- **[CME matching algorithms explained, Databento](https://databento.com/blog/cme-matching-algorithms-explained)** and **[Where is the CME matching engine located?](https://databento.com/blog/cme-colocation)**. From search excerpts: CME Globex is a central limit order book using price-time priority (CME also offers other algorithms, such as pro-rata, for some products), and its primary matching engine sits in Aurora, Illinois. That explains why trading firms put servers there (co-location).
- **[Aeron Exchange Accelerator (Adaptive)](https://weareadaptive.com/aeron-exchange-accelerator/).** A commercial product page built on Aeron (a high-performance messaging library with a Raft-based cluster mode). Shows that the sequencer plus replicated-log pattern is sold as a product. No latency numbers on the page.
- **Search for next:** the LMAX Disruptor technical paper (Thompson, Farley et al., 2011) and the Mechanical Sympathy blog (Martin Thompson) for the cache-line and false-sharing reasoning; Nasdaq's public ITCH/OUCH specs for the real message formats; Hello Interview's stock-exchange breakdown for the interview framing. I did not open these today.

**Inference, labeled:** the hot-row lock throughput for SQL, the per-symbol burst figure, the nanosecond-scale per-match cost, the choice of array-by-tick for price levels, and the symbol-sharding scale story are my back-of-envelope or class-of-problem estimates. No real exchange's internals beyond the cited public material were verified.

**Related ledger lessons:** Day 9 (queue), Day 10 (consistent hashing), Day 11 (Raft), Day 12 (idempotency), Day 13 (backpressure), Day 16 (hot keys), Day 17 (WAL), Day 24 (fencing tokens), Day 56 (real-time bidding), Day 59 (event sourcing), Day 71 (distributed rate limiting), Day 74 (LeetCode end to end), Day 76 (tail latency), Day 82 (double-entry ledger).
