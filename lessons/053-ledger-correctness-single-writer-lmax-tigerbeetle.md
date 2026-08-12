# Day 53 — How does a ledger process a million transfers a second without ever locking the account everyone is fighting over?

**Date:** 2026-08-12
**Difficulty:** Expert
**Topic:** Ledger correctness under extreme write concurrency: the single-writer, lock-free "mechanical sympathy" pattern LMAX Exchange built for a trading engine in 2011 (the Disruptor, via Martin Fowler's LMAX Architecture writeup and Trisha Gee's contemporaneous benchmark post), and how TigerBeetle, a purpose-built distributed database for double-entry accounting, generalized that exact pattern to solve the "hot account" problem that caps an ordinary row-locking database at roughly 100 transfers a second against any single account, no matter how big the machine underneath it is.
**Stack relevance:** Rare.lab does not run a trading exchange, but the moment it has to meter compute, credits, or usage per tenant (charge a workspace for every shader compile or render second, never double-charge, never silently drop a debit), it owns a ledger with the identical correctness shape as a bank's: two-legged writes that must both happen or neither happens, against a small number of "hot" rows (a popular team's shared credit balance) that every concurrent request wants to touch at once. The content-addressed, append-only pattern R2 already uses for immutable scene JSON plus a manifest is the same idea as an event-sourced ledger, one write per historical fact, never an in-place edit, and this lesson is the argument for carrying that same append-only discipline into anything Rare.lab ever counts or bills.

---

## 1. The company and the breaking number

**LMAX Exchange**, in Martin Fowler's 2011 article *"The LMAX Architecture"* and in Trisha Gee's companion post *"Dissecting the Disruptor: Why it's so fast (part one), Locks Are Bad,"* both written by LMAX engineers documenting the reasoning behind LMAX's Disruptor pattern. The number that motivated the entire design is a small, brutal microbenchmark: incrementing a 64-bit counter **500 million times**. A single thread doing this with no locking at all takes **300 milliseconds**. Add a lock around the increment, still on a single thread with zero contention, nothing else is even fighting for that lock, and the same 500 million increments take **10,000 milliseconds**, roughly 33 times slower just for the lock's own bookkeeping. Now split the same 500 million increments across **two threads** sharing one lock, the setup that intuition says should be faster because there are twice as many CPUs doing the work: it takes **224,000 milliseconds**, nearly 750 times slower than the single unlocked thread, using twice the hardware to do it. Adding a second worker made the system almost three orders of magnitude slower, not faster. That inversion, more compute capacity producing catastrophically worse throughput, is the founding breaking number behind LMAX's entire architecture.

**TigerBeetle**, a database purpose-built for double-entry financial accounting (founded by Joran Dirk Greef, whose earlier work profiling Mojaloop, a real-time payment switch used by central banks, is publicly credited as the direct trigger for building it). TigerBeetle's own published reasoning states the modern, generalized version of the same problem plainly: general-purpose databases were never designed to increment the same integer rapidly, the "hot key" or "hot account" problem, and as a direct consequence of Amdahl's Law they typically **cap out around 100 transactions a second against any single account row**, regardless of how many CPU cores or how much RAM the machine has, because every transfer touching that row has to wait for a lock held by the transfer in front of it. The real-world case TigerBeetle names is a central bank switch: a cluster with only a **half-dozen to a few hundred account rows**, one per partner bank, processing **hundreds of millions of transactions a day**. Almost every one of those transactions touches one of that tiny handful of rows. A naive ledger built on ordinary row-locking tops out around 100 transfers a second on exactly the row that matters most; a central bank needs that same row to absorb a rate many multiples higher, sustained, all day.

## 2. Why the naive (demo) design dies

**Version one, the one anyone builds first: an `accounts` table and a `transfers` table in Postgres or MySQL, one stateless API server per request, wrapped in `BEGIN; UPDATE accounts SET balance = balance - x WHERE id = A; UPDATE accounts SET balance = balance + x WHERE id = B; COMMIT;`.** It is obviously correct on paper, the database's own transaction isolation is supposed to handle concurrency for you. It dies on three separate axes, and the first two are LMAX's benchmark playing out inside the database engine itself, not a hypothetical.

**Lock overhead is a fixed tax, paid even before any two transfers actually collide.** LMAX's 300ms-versus-10,000ms result shows that acquiring and releasing a lock costs real, non-trivial time on its own, roughly 33x here, purely from the lock's bookkeeping and the memory-barrier and cache-coherency traffic it forces between CPU cores. A row-locking database pays a version of this tax on every single write, whether or not another transaction is currently waiting on that row.

**Contention on a hot row turns that fixed tax into an unbounded one.** The moment two transfers actually want the same account at the same instant, the 224,000ms result is the shape of what happens: the second transaction blocks behind the first, the third blocks behind the second, and so on, a lock convoy. For an ordinary customer account that gets touched once an hour, this never shows up. For the handful of rows every real ledger has, the platform's own operating account, a popular merchant's balance, a central bank's per-partner clearing account, it shows up constantly, and it caps throughput on exactly the row where the business needs the most capacity, not the least.

**Guaranteeing "both legs commit or neither does" under contention multiplies retries, and retries add more contending transactions to the same hot row.** Enforcing the double-entry invariant (every debit has a matching credit, no balance ever goes negative) either holds locks across both UPDATEs for the whole transaction, widening the contention window, or uses optimistic concurrency and retries on conflict, serializable isolation with retry-on-serialization-failure. Under load, retries do not relieve pressure on a hot row, they add a second attempt from every failed transfer on top of the fresh ones already queued, which is precisely the feedback loop this lesson returns to in section 6.

**Sharing the write path with the read path degrades both.** A ledger's write-heavy `transfers` table is also where balance checks, statements, and reporting queries want to read from. MVCC databases keep old row versions around for in-flight reads, so a write-heavy hot table under read pressure accumulates bloat that has to be vacuumed, and vacuuming a table under constant contention competes for the very I/O and locks the live traffic needs. The naive design has no separation between "the one true source of the current balance" and "everyone who wants to look at it," so growth in one starves the other.

## 3. The architecture

```
[Clients: bank apps, payment processors, an internal billing job,
 submitting a transfer request tagged with a client-generated
 idempotency key]
   analogy: a customer filling out a deposit slip and writing their
   own reference number on it before handing it to anyone
   |
   v
[Edge / API gateway: validates request shape, checks the idempotency
 key against recently-seen keys, rejects malformed or duplicate
 submissions before they cost the system anything further]
   analogy: the numbered-ticket machine at a government office,
   resubmitting the same ticket twice never gets you served twice
   |
   v
[Stateless app tier: does NOT touch balances directly; batches many
 individual transfer requests together and routes each batch to the
 one process that owns the accounts involved]
   analogy: a runner who collects a stack of order slips and carries
   the whole stack to the one clerk allowed to write in the ledger,
   instead of every customer pushing to the counter individually
   |
   v
[Single-writer deterministic state machine, one per shard: the ONLY
 thread allowed to mutate account balances for its slice of accounts;
 applies each transfer in strict arrival order, checks both legs'
 invariants in-process, in memory, no row locks exist because there
 is structurally only ever one writer]
   analogy: the one bookkeeper with the one physical ledger book;
   nobody argues about who wrote a line first because only one hand
   ever holds the pen
   |
   v
[Append-only log (LSM-tree storage): every accepted transfer is
 written as a new, immutable entry, balances are a fold over the
 log, never an in-place edit]
   analogy: a carbon-copy receipt book, you never erase a line, a
   mistake gets corrected with a new offsetting entry, not a rewrite
   |
   v
[Consensus replication (quorum write to N replicas, e.g. Viewstamped
 Replication): a transfer is not acknowledged as durable until a
 MAJORITY of independent replicas have it in their own write-ahead
 log]
   analogy: a contract isn't binding until enough independent
   witnesses have signed it, so no single lost signature undoes the
   deal
   |
   v
[Read replicas / async projections: balance lookups, statements,
 dashboards, reconciliation reports read from continuously-updated
 copies, never from the single writer's own hot path]
   analogy: photocopies of the ledger pinned on the wall, updated a
   moment after the real book, so checking your balance never makes
   you stand in the line behind an active transfer
   |
   v
[Sharding across independent single-writer clusters, split by
 account range, currency, or tenant, so no one thread has to
 serialize the whole world's transfers]
   analogy: opening more teller windows, each with its OWN ledger
   book covering its own slice of customers, instead of forcing one
   teller to serve everyone alive
```

Two structural choices carry the real weight here.

**The single writer is not a bottleneck you tolerate, it is the mechanism that removes contention entirely.** A row-locking database has many writers coordinating through locks, and coordination cost grows with contention, exactly LMAX's 224,000ms result. A single-writer design has exactly one writer per shard, so there is nothing to coordinate: every transfer's order is already decided by the order it arrives in that one process's queue. This is why TigerBeetle's design and LMAX's Business Logic Processor both look, from a distance, like a step backward, "only one thread does the real work?", and are in fact the opposite: removing the possibility of a lock removes the possibility of a lock convoy.

**Durability and throughput are decoupled by batching the boundary between them, not by weakening either.** TigerBeetle's public interface batches up to **8,190 transfers into a single request**, and one durable write-ahead-log "prepare" commits the entire batch to the quorum at once. The single writer still processes transfers one at a time internally, in order, but the expensive part, the disk fsync and the network round trip to a majority of replicas, is paid once per batch instead of once per transfer. This is the direct descendant of the Disruptor's ring buffer: LMAX's business logic never blocks the writer to wait on I/O either, I/O happens on the outside, in bulk, off the critical path of deciding "what happened, in what order."

## 4. The transferable mechanisms

**The single-writer principle.** Give exactly one thread ownership of a piece of mutable state, and every operation on that state becomes lock-free by construction, because there is no second writer to coordinate with. This generalizes past ledgers to anything with a hot, frequently-mutated piece of shared state: a matching engine's order book, a game server's authoritative world state, a rate limiter's shared counter. The corollary is equally important: the moment more throughput is needed than one core can give, the fix is more single writers on different slices of the data (sharding), never more writers on the same slice.

**Double-entry atomic guarded write, checked in-process instead of coordinated across services.** A transfer only commits if both legs' invariants hold, sender has sufficient balance, both accounts exist, checked by one thread with both rows already in its own memory, not by a distributed transaction spanning two databases. Collapsing "the invariant check" and "the only writer who can violate it" into the same single-threaded process is what makes the check free of races without needing locks or two-phase commit.

**Append-only log / event sourcing as the actual source of truth.** Current balance is never stored and mutated in place, it is derived by folding over an immutable sequence of past transfers. This buys two things at once: durability is cheap because appending to a log is a sequential write, the fastest thing a disk does, and correctness is auditable because the entire history can be replayed to prove how any balance got to where it is. This is the same shape as event sourcing in application architecture generally, and it is why TigerBeetle stores data as a forest of LSM-trees, structures optimized for exactly this write-heavy, append-mostly pattern.

**Batching to amortize the one truly expensive operation.** Grouping thousands of logically-independent writes into one durable commit (one fsync, one consensus round) trades a small, bounded amount of added latency per individual write for a large multiplier in aggregate throughput. The general rule: find the operation on your critical path with a large fixed cost, and pay that fixed cost once per batch instead of once per item.

**Quorum-based replication for durability without a single point of failure.** A write only counts as done once a majority of independent replicas have it durably logged, so losing any minority of machines, including the current leader, loses zero committed transfers. TigerBeetle's own recommended production topology is six replicas, tolerant of up to two failures with the primary intact, or a new leader election if the primary itself is among the failures.

**Sharding as the escape hatch from Amdahl's law, once a lock-free single writer still saturates.** A single writer removes coordination cost, but its ceiling is still bounded by one core's clock speed. The next lever is not a bigger machine, it is more independent single writers, each owning a slice chosen so the overwhelming majority of transactions need only one shard, exactly the geospatial and account-hash sharding strategies covered in earlier lessons on consistent hashing and cell-based architecture.

## 5. The trade-offs

**Balances and transfers: strict serializability, never eventually consistent, and the system pays for it in latency.** TigerBeetle deliberately supports only the strictest isolation level it can offer, because a double-spent cent is a real, irreversible loss of money, not a stale UI label a refresh will fix. That choice is paid for with a quorum round trip on every write, an extra hop of latency that a weaker-consistency system would not incur. This is the consistency-versus-availability axis made concrete and resolved in consistency's favor, specifically because the data type is money.

**Statements, dashboards, and reporting reads: eventual consistency is not just tolerable, it is the right choice.** A balance shown 200 milliseconds stale on a reporting dashboard costs nothing, while forcing every dashboard read to queue behind the single writer would cost real transfer throughput for no correctness benefit. Splitting the read path onto asynchronously-updated replicas is choosing availability and low latency for the data type where a slightly stale answer is harmless.

**Cost versus latency in replication.** Six replicas and a quorum write cost roughly six times the machines and one extra network round trip compared to an unreplicated single database, bought specifically so that no committed transfer can vanish if any one or two machines die. That is a real, ongoing infrastructure cost, accepted because the alternative, a ledger that can silently lose a committed transaction, is not an acceptable trade at any price for this data type.

**Giving up the usual "just add more app servers" scaling trick, on purpose, for this one layer.** Ordinary stateless services scale writes by adding more identical workers behind a load balancer. A single-writer ledger shard cannot do that, by design, for the exact reason it has no lock contention: there genuinely is only one writer. The system still scales, but the axis it scales on is shards, not workers within a shard, which is a deliberate, load-bearing architectural constraint rather than an oversight to be optimized away later.

## 6. The systems-thinking lens

The feedback loop worth naming here is a **lock convoy that becomes a retry storm**, a specific flavor of metastable failure, not a thundering herd.

Picture the naive row-locking design under a burst of legitimate load against one hot account. The first few transfers queue normally behind the row's lock. As the queue grows, per-transfer latency grows with it, and clients or the app tier, seeing timeouts, start retrying. Each retry is a brand-new transaction that also wants the same hot row, so it joins the back of the same queue that is already the problem. The queue is now growing from two sources at once: fresh legitimate traffic, and retries generated by the queue's own backlog. Even if the original burst of legitimate traffic subsides completely, the retry-generated load can keep the queue long, keep latency high, keep timeouts firing, keep generating more retries. The system does not recover on its own once the trigger has passed, because the recovery mechanism, retrying, is now the thing sustaining the failure. That is the textbook shape of a metastable failure: a normal operating condition (retries help transient errors) becomes, past a threshold, the mechanism preventing recovery.

Buying a bigger database server does not touch this loop, the queue is forming around one row's lock, not around raw CPU or memory capacity. The senior fix breaks the correlation between "the queue is long" and "more attempts get added to it":

- **Remove the lock the convoy forms around.** This is what the single-writer design does structurally: there is no lock to convoy behind, only a deterministic queue with a known, bounded processing order, so "waiting" never compounds into "waiting behind other things that are also waiting."
- **Batch and backpressure at the edge, instead of retrying at the edge.** Grouping requests into batches before they reach the single writer, and explicitly rejecting or shedding excess load with a clear signal rather than letting callers retry blindly, keeps the queue's growth bounded by the writer's actual throughput instead of by how aggressively clients retry.
- **Idempotency keys make retries safe without making them cheap.** A retried transfer with the same idempotency key is recognized and answered from the already-committed result instead of being reprocessed, which stops accidental duplicate processing, but it does not by itself stop a retry storm from adding load; it has to be paired with backpressure that discourages retrying in the first place.

The general principle again: when the mechanism meant to recover from a problem, a retry, can itself become the load that prevents recovery, the fix is to remove or bound the contention point the retries are piling onto, not to add capacity to the system around it. A faster database does not stop a lock convoy from convoying harder.

---

## Sources

- [The LMAX Architecture, Martin Fowler](https://martinfowler.com/articles/lmax.html)
- [Dissecting the Disruptor: Why it's so fast (part one), Locks Are Bad, Trisha Gee](https://trishagee.com/2011/07/16/dissecting_the_disruptor_why_its_so_fast_part_one__locks_are_bad/)
- [LMAX Disruptor: High performance alternative to bounded queues for exchanging data between concurrent threads (technical paper)](https://lmax-exchange.github.io/disruptor/disruptor.html)
- [Why General Databases Cap Out at 100 TPS: TigerBeetle CEO Joran Dirk Greef on Solving the "Hot Key" Problem, Ximedes](https://ximedes.com/blog/solving-the-hot-key-problem)
- [Rediscovering Transaction Processing From History and First Principles, TigerBeetle blog](https://tigerbeetle.com/blog/2024-07-23-rediscovering-transaction-processing-from-history-and-first-principles/)
- [TigerBeetle Docs: Performance](https://docs.tigerbeetle.com/concepts/performance/)
- [TigerBeetle Docs: Safety](https://docs.tigerbeetle.com/concepts/safety/)
- [TigerBeetle Docs: Cluster Recommendations](https://docs.tigerbeetle.com/operating/cluster/)
- [Jepsen: TigerBeetle 0.16.11 (independent consistency analysis)](https://jepsen.io/analyses/tigerbeetle-0.16.11)
- [TigerBeetle Architecture, GitHub docs/ARCHITECTURE.md](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/ARCHITECTURE.md)

---

*Inference vs. fact, stated plainly: the 500-million-increment benchmark (300ms unlocked, 10,000ms locked-uncontended, 224,000ms locked-and-contended across two threads) is drawn directly from Trisha Gee's own published LMAX benchmark writeup, corroborated by summaries of Martin Fowler's companion article, whose full text could not be directly re-fetched while researching this lesson, so its figures here rest on Gee's account and secondary summaries rather than a freshly re-read primary transcript. TigerBeetle's "roughly 100 TPS per hot account" figure, the "half-dozen to a few hundred accounts, hundreds of millions of transactions a day" central-bank example, and the 8,190-transfer batch limit are drawn from TigerBeetle's own published blog, documentation, and CEO interview content. The six-replica cluster recommendation and the strict-serializability claim are drawn from TigerBeetle's own docs and are independently exercised, not merely claimed, by the third-party Jepsen analysis cited above. The characterization of the failure mode as a lock-convoy-driven metastable failure, and the specific claim that batching plus backpressure rather than idempotency alone is what breaks it, is this lesson's own reasoned synthesis of how the documented single-writer and batching mechanisms would behave under a retry storm, not a quoted incident report from either company.*
