# Day 29 — How do two transactions that never touch the same row still corrupt an invariant that spans both?

**Date:** 2026-07-11
**Difficulty:** Expert
**Topic:** Write skew and Serializable Snapshot Isolation (SSI): the class of correctness bug that survives every write-write conflict check ever built, via Berenson, Bernstein, Gray, Melton, O'Neil and O'Neil's 1995 paper that first named it, Cahill, Röhm and Fekete's 2008 SSI algorithm that closes it without falling back to full locking, PostgreSQL 9.1's real production implementation of that algorithm (Ports and Grittner, 2012), and CockroachDB's on-call-doctors example of the identical bug surfacing in a modern distributed database
**Stack relevance:** Supabase Postgres ships with a default isolation level (READ COMMITTED) that permits write skew by design, and Row Level Security does nothing to stop it, RLS filters *which rows a query is allowed to see*, it says nothing about *what two concurrent queries are allowed to jointly decide*. Any Rare.lab invariant that spans more than one row (a per-team compile-quota, a "one active publish per project" rule, a seat limit on concurrent editors) is exactly this bug, sitting dormant until two requests overlap

---

## 1. The company and the breaking number

**PostgreSQL and MySQL, the two most widely deployed open-source relational databases on earth, both ship with a default transaction isolation level that permits this bug, out of the box, with zero configuration.** PostgreSQL's default is `READ COMMITTED`, the weakest of the four ANSI SQL isolation levels. MySQL's InnoDB engine defaults to `REPEATABLE READ`, which is stronger but is InnoDB's own approximation of snapshot isolation, not full serializability. Supabase, built directly on PostgreSQL, inherits `READ COMMITTED` as its default for every project created, unless a developer explicitly opts into something stronger.

Here is the breaking number, and unlike most numbers in this series it is not a rate or a latency, it is a **probability: 100%.** Every write-write conflict detector ever built, from a plain `UPDATE ... WHERE id = X`'s row lock to PostgreSQL's first-committer-wins snapshot isolation check, catches exactly one shape of bug: two transactions racing to write **the same row**. The moment two transactions read overlapping data and then write to **different rows**, none of those detectors fire, not occasionally, never. If the invariant those two rows jointly enforce can be violated by writing to each independently, and both transactions' reads happen to land in the same overlap window, the invariant breaks **every single time that overlap occurs**, deterministically, with no conflict ever logged anywhere.

CockroachDB's own documentation walks the canonical version of this bug with a hospital on-call schedule. A table of doctors has a rule enforced only in application logic: at least one doctor must be on call at all times. Two doctors, both currently on call, each decide independently to request leave at nearly the same moment. Doctor A's transaction reads the table, sees doctor B is also on call, concludes "it's safe for me to go off call, someone else is covering," and commits. Doctor B's transaction, running concurrently against its own snapshot taken before A's commit, reads the same table, sees doctor A is on call, reaches the identical conclusion, and commits. Neither transaction wrote to the other's row. Neither transaction's write-write conflict check had anything to catch. Both commits succeed. The hospital now has zero doctors on call, and the one invariant the whole scheme existed to protect is gone, without a single database error raised at any point in the sequence.

The bank-account version is even older and just as sharp, drawn from Berenson, Bernstein, Gray, Melton, O'Neil and O'Neil's 1995 SIGMOD paper "A Critique of ANSI SQL Isolation Levels," the paper that gave this anomaly its name. One customer, two accounts, V1 and V2, each holding $100. The bank's rule: either account may run a deficit individually, but V1 + V2 must never go negative. The customer fires two withdrawal requests at once, $200 from V1 and $200 from V2, arriving on two concurrent transactions. Transaction 1 reads V1 = $100 and V2 = $100, checks $100 + $100 - $200 = $0, which is not negative, so it approves and debits V1 by $200. Transaction 2, on its own snapshot, performs the identical check against the same starting balances, approves, and debits V2 by $200. Both commit. The account now holds $100 - $200 = -$100 on V1, plus $100 - $200 = -$100 on V2, a combined -$200, a constraint violation neither transaction could see coming and neither database engine's row-level conflict detector was ever designed to catch.

Hold onto that number: **zero row-level conflicts detected, one hundred percent of invariant-spanning overlaps corrupted.** Every mechanism in this lesson exists to answer one question: given that a write-write check only ever compares two transactions' writes against each other, how do you also catch the case where transaction A's *write* invalidates the *assumption transaction B's read relied on*, when A and B never touched a single row in common?

---

## 2. Why the naive (demo) design dies

The demo version of "enforce an invariant across rows" looks completely reasonable, and it is exactly what most application code does without a second thought:

```
BEGIN;
SELECT count(*) FROM doctors WHERE on_call = true;
-- app checks: is count > 1? if yes, proceed
UPDATE doctors SET on_call = false WHERE id = my_doctor_id;
COMMIT;
```

or the inventory-flavored version:

```
BEGIN;
SELECT sum(allocated) FROM seats WHERE event_id = X;
-- app checks: is sum + requested <= capacity? if yes, proceed
INSERT INTO seats (event_id, allocated) VALUES (X, requested);
COMMIT;
```

This fails in three concrete, distinct ways once two of these run concurrently against a real production isolation level.

**a. Under `READ COMMITTED` (PostgreSQL's actual default), the check is not even reading a consistent snapshot, each statement inside the transaction sees whatever committed data exists at the moment *that statement* runs.** This is weaker than the on-call example even needs: a second query inside the same transaction can see a completely different world than the first query saw, because `READ COMMITTED` re-reads committed state per-statement, not per-transaction. Any invariant that depends on "the state I read a moment ago is still true right now" is unprotected by construction at this level, which is precisely why it is the cheapest and fastest of the four ANSI levels, and precisely why it is the wrong default for anything enforcing a cross-row rule.

**b. Under `REPEATABLE READ` (PostgreSQL's actual implementation of snapshot isolation, and InnoDB's real default), the transaction does get one consistent snapshot for its whole duration, which fixes 2a but not the core bug.** Snapshot isolation guarantees each transaction sees a single, unchanging point-in-time view and guarantees that two transactions writing the **same row** will have one of them rejected (PostgreSQL raises `ERROR: could not serialize access due to concurrent update`). But the on-call doctors write to two *different* rows, and the seat-booking example writes two *different* rows into the same table. Snapshot isolation's write-write check has nothing to compare, because there is no row in common, so it lets both commits through cleanly, and the underlying constraint breaks anyway. Berenson et al.'s paper is explicit that this exact gap, which they name **write skew**, is a known, structural blind spot of snapshot isolation, not a bug in any particular vendor's implementation of it.

**c. A `CHECK` constraint cannot express the invariant that actually needs enforcing, because standard SQL check constraints evaluate against a single row, not an aggregate across many rows.** The tempting fix, "just add a database constraint so the engine enforces this instead of application code," runs straight into the fact that "at least one doctor on call" and "sum of allocated seats <= capacity" are not properties of any single `doctors` or `seats` row, they are properties of the whole table, evaluated as of a moment in time that two concurrent transactions do not agree on. A trigger that runs the same `SELECT count(*)` check inside each transaction inherits the identical blind spot as the application-level version, because the trigger is still executing against that transaction's own isolated snapshot, unaware of the other transaction's concurrent, not-yet-committed intent.

The naive design's mistake is treating "no database error was raised" as equivalent to "the operation was safe," when a write-write conflict check was only ever built to catch one narrow shape of collision, and a cross-row invariant is a completely different shape.

---

## 3. The architecture

Top to bottom, the concurrency-control pipeline every serializable-capable engine (PostgreSQL 9.1+, CockroachDB, FoundationDB) converges on, once the naive design in section 2 is fixed properly:

```
[Client / app tier: BEGIN ... check invariant ... write ... COMMIT]
   the transaction boundary is the unit of correctness; everything
   below exists to make "this block ran as if it were alone" true
   analogy: a person stepping into a photo booth and pulling the
   curtain, believing nothing outside can see or interfere until
   they step back out
   |
   v
[Connection pool -> DB primary: snapshot assignment on BEGIN]
   every transaction gets a private, consistent read snapshot the
   instant it starts; reads against that snapshot never block and
   are never blocked, this is MVCC (multi-version concurrency
   control), the same mechanism every modern engine uses for reads
   analogy: everyone in the building gets handed their own dated
   copy of today's newspaper the moment they walk in; nobody has
   to wait for anyone else to finish reading theirs
   |
   v
[Execution against the snapshot: reads recorded, writes buffered]
   PostgreSQL's SSI additionally records every row/page/relation a
   transaction READS as a SIREAD lock, a lock that blocks nothing,
   it exists purely as a marker: "this transaction's conclusions
   depended on this data being what it was"
   analogy: a detective jotting down every witness statement they
   relied on to reach a conclusion, not to silence the witnesses,
   just so the notes exist if the conclusion later needs checking
   |
   v
[Commit-time conflict detection, two distinct checks]
   (i) ww-conflict: did another concurrent transaction already
       commit a write to a row THIS transaction also wrote? first
       committer wins, the loser is rejected outright (this is
       plain snapshot isolation, section 2b's protection, it stops
       here in every non-serializable engine)
   (ii) rw-antidependency: did another concurrent transaction WRITE
       something THIS transaction READ (or vice versa)? this edge
       is recorded but does NOT by itself cause an abort
   |
   v
[Dangerous structure check: Tin --rw--> Tpivot --rw--> Tout]
   SSI aborts only when it finds TWO adjacent rw-conflict edges
   through a single transaction (the "pivot"), because Cahill,
   Röhm and Fekete proved that exact shape is the only pattern
   capable of producing a non-serializable outcome; a single
   rw-edge alone is provably always safe to allow
   analogy: one rumor passed between two people is harmless; it's
   the THIRD person who hears it from both sides and acts on the
   contradiction that causes the problem, catch that person, not
   every rumor
   |
   v
[Commit or abort-with-retry (SQLSTATE 40001, "serialization
 failure")]
   the app tier is expected to catch this specific error code and
   retry the whole transaction from BEGIN, exactly the same
   contract as a queue redelivering a message (Day 9)
   |
   v
[Read replicas: serve stale-but-internally-consistent reads]
   rw-conflict tracking is a primary-side, single-writer-domain
   mechanism; a replica cannot participate in an abort decision
   about a transaction it never saw begin, so serializable
   guarantees apply to the primary's write path, not to whatever a
   replica happens to be showing a reader a few hundred
   milliseconds behind
```

The one box that has no equivalent in ordinary snapshot isolation is the dangerous-structure check. Everything above it (MVCC snapshots, SIREAD/read-tracking, ww-conflict detection) exists in plain snapshot-isolation engines too. It is specifically the **rw-antidependency graph plus the two-edges-through-one-pivot rule** that turns "does not corrupt data on a same-row collision" into "does not corrupt data, period," and it is also specifically what section 1's hospital and bank examples were missing.

---

## 4. The transferable mechanisms

**a. MVCC: give every reader its own private, consistent snapshot so reads never block writes and writes never block reads.** This is the foundation everything else sits on, and it is why serializable isolation in a modern engine is nowhere near as expensive as old-style two-phase locking (2PL), where a reader physically blocks a writer until the reader's transaction ends. PostgreSQL, CockroachDB, and effectively every serious OLTP engine built in the last twenty-five years use some form of MVCC specifically to keep the common case (no real conflict) fast, reserving the expensive check for commit time.

**b. First-committer-wins write-write conflict detection: the cheap, table-stakes protection that catches same-row races but nothing else.** This is what "snapshot isolation" alone buys you, and it is a genuinely different mechanism from lesson 6's idempotency keys: an idempotency key stops the *same logical request*, resubmitted, from double-applying; a ww-conflict check stops two *different, concurrent* transactions from silently overwriting each other's write to the *same row*. Both matter, and neither substitutes for the other.

**c. rw-antidependency tracking (SIREAD locks): record what was read, without blocking anything, purely to make a later correctness check possible.** This is the mechanism that costs almost nothing at read time (no blocking, just bookkeeping) and pays off only at commit time, which is exactly why Cahill, Röhm and Fekete's original paper reports SSI's throughput staying close to plain snapshot isolation's under low contention: the expensive part, walking the conflict graph, only has real work to do when transactions are actually overlapping.

**d. Dangerous-structure (pivot) detection: abort on a proven risk pattern, not on every conflict.** The insight that makes SSI practical rather than a full-lockdown scheme is narrow and precise: a *single* rw-conflict edge between two transactions can never by itself produce a non-serializable history, it takes two adjacent rw-edges passing through one common transaction (the pivot: `Tin -rw-> Tpivot -rw-> Tout`) before a real risk exists. PostgreSQL's own implementation notes (its `README-SSI` source file) go further, deferring even that abort decision until `Tout` actually commits, and applying a read-only optimization that skips the check entirely when `Tin` is a read-only transaction that started after `Tout` already committed, both of which exist purely to shrink the false-positive abort rate down to only the transactions provably at risk.

**e. Optimistic concurrency plus abort-and-retry as a correctness primitive, not a failure mode.** SSI never blocks a transaction to protect it in advance, it lets everything run, and only when a transaction reaches commit and a dangerous structure is confirmed does it reject that one transaction with a specific, catchable error. This pushes the cost of contention onto the rare transaction that actually collides, rather than making every transaction pay an upfront locking cost, the same underlying trade Day 12's idempotency and Day 13's backpressure lessons make: detect and reject cheaply and specifically, rather than prevent expensively and generally.

**f. Explicit predicate locking (`SELECT ... FOR UPDATE`, or a hand-rolled advisory lock) as the pessimistic fallback when abort storms are unacceptable.** Not every workload tolerates retries well, a long-running report generation job that gets aborted at minute nine of a ten-minute transaction is a worse outcome than simply blocking it behind a lock for those same ten minutes. `SELECT ... FOR UPDATE` explicitly takes a row lock at read time, trading MVCC's non-blocking optimism for predictable, if slower, correctness, the same choice Day 24's fencing-token lesson frames as pessimistic-versus-optimistic concurrency in the distributed-lock setting.

---

## 5. The trade-offs

**This is CAP made concrete, per data type, at the isolation-level knob rather than at the replication topology.** PostgreSQL ships `READ COMMITTED` as the default specifically because most application code, most of the time, does not need cross-row invariant protection, and paying serializable isolation's abort-and-retry tax on every transaction, everywhere, would be the wrong default for throughput-sensitive workloads that never touch a multi-row constraint. Cahill, Röhm and Fekete's own benchmark results back this up directly: at low contention, SSI's throughput sits close to plain snapshot isolation's, but as contention rises, so does the abort rate, because more overlapping transactions means more dangerous structures actually forming, not more false positives. The system's real behavior under load is a curve, not a fixed cost, and that curve is the trade-off a team is actually signing up for by turning `SERIALIZABLE` on.

**Pessimistic locking (`SELECT ... FOR UPDATE`) trades that variable abort-retry cost for a predictable, but strictly worse, latency floor: every transaction touching a locked row waits, even the ones that would never have actually conflicted.** SSI only punishes the transactions that provably collide; row locking punishes every transaction that merely *might* have collided, which is a strictly more pessimistic (and under low real contention, strictly more wasteful) trade. The right choice depends on whether the workload's true collision rate is low (favor optimistic SSI) or high and predictable (favor explicit locking, where the wait is bounded and known rather than an unbounded retry loop).

**Consistency-per-data-type is the practical resolution, not a single database-wide setting.** A financial ledger, a seat-capacity counter, or an on-call schedule needs the invariant-spanning guarantee `SERIALIZABLE` provides, and should pay its cost explicitly and locally, wrapping only that specific transaction. A page-view counter, a presence "last seen" timestamp, or a non-critical analytics rollup needs none of it, and forcing it through `SERIALIZABLE` would only add abort-retry latency for zero correctness benefit. Real production PostgreSQL usage reflects exactly this: `READ COMMITTED` for the bulk of a schema, `SERIALIZABLE` opted into per-transaction, only where an aggregate or cross-row rule is actually being enforced.

---

## 6. The systems-thinking lens

**The feedback loop: rising contention on a shared invariant increases the abort rate, which increases retries, which increases contention further, the same retry-amplification shape Day 9 and Day 13 name for queues and backpressure, now happening inside a single database's commit path instead of across a network.** Trace it through: a seat-booking table under normal load has few concurrent writers, SSI's abort rate stays near zero, the system feels indistinguishable from plain snapshot isolation → a flash sale drives many concurrent transactions at the same capacity row's *neighborhood* (not the same row, but rows whose combined sum enforces the same constraint) → more pairs of transactions now overlap in time, so more dangerous structures actually form, so the abort rate climbs → every aborted transaction's naive response, "just retry immediately," puts an *identical* transaction back into an *even more* contended window a moment later, which raises the collision probability for the retry above what it was for the original attempt. Left unchecked, this is the same metastable pattern as a retry storm anywhere else in this ledger: the system's own correctness mechanism becomes the thing amplifying the load that triggers it.

**The senior fix is not "add a bigger machine" or "raise the retry limit," it is to shrink what has to be serializable in the first place, exactly the same move Day 13's backpressure lesson makes at the network layer.** Three concrete versions of that move, all cheaper than living with the retry loop: first, add jittered exponential backoff before a retry, so a burst of simultaneous aborts does not immediately re-collide (identical mechanism to Day 24's lock-service backoff); second, shrink the transaction's actual footprint and hold time, touching only the rows the invariant strictly needs rather than a broad `SELECT *` that inflates the read set SSI has to track; third, and the real structural fix, redesign the invariant so it no longer spans multiple rows at all. "Sum of allocated seats across N rows must not exceed capacity" is a cross-row invariant that needs serializability. "A single `remaining` counter, decremented atomically with `UPDATE seats SET remaining = remaining - 1 WHERE remaining > 0`" is a single-row, single-statement, atomic read-modify-write that is safe under plain `READ COMMITTED`, no serializable isolation required, because there is no longer a second row whose independent write can silently invalidate the first row's read. Turning a cross-row invariant into a single-row atomic operation does not tune the retry loop, it deletes the condition that created it.

---

## Map to Rare.lab's stack

**Where this bites concretely:** picture a per-team compile-quota rule, "total concurrently-compiling shader jobs across all of a team's active sessions must not exceed the team's plan limit," checked the naive way: `SELECT count(*) FROM compile_jobs WHERE team_id = X AND status = 'running'`, compare against the plan's limit, and if under, `INSERT` a new running job row. Supabase's default `READ COMMITTED` isolation, inherited straight from PostgreSQL, does not protect this at all, and RLS is irrelevant here since RLS only decides *which rows a given user's query is allowed to see*, not *whether two concurrent queries are allowed to jointly push a shared count over its limit*. Two team members each starting a compile within the same tens-of-milliseconds window will each see "2 of 3 slots used, I'm fine," both insert, and the team ends up running 4 concurrent compiles against a plan limit of 3, exactly the on-call-doctors bug wearing a billing hat.

**The concrete fix, in the same shape as section 6's structural move:** for any invariant that genuinely needs to reason across multiple rows (a per-team seat count spread across session rows, a "only one active publish per project" rule spanning a `publishes` table), wrap that specific transaction in `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE` explicitly, since Postgres and Supabase both support it, it simply is not the default, and handle SQLSTATE `40001` in the application layer with a retry, the same catchable-and-retryable contract section 3 describes. But for the far more common case, a simple counter or capacity limit, skip serializable isolation entirely and collapse the invariant into a single-row atomic statement: a `team_quota` row with a `used` column, updated via `UPDATE team_quota SET used = used + 1 WHERE team_id = X AND used < limit RETURNING *`, checking the affected row count in the application to know whether the slot was actually granted. That single atomic statement is safe under Supabase's default `READ COMMITTED`, costs nothing extra, and never enters an abort-retry loop at all, which is the same lesson as this ledger's Day 16 hot-key discussion: the cheapest fix for a shared, contended invariant is usually to make it a single atomic thing to check, not to make the transaction wrapped around it smarter.

---

## References and summaries

**Berenson, Bernstein, Gray, Melton, O'Neil, O'Neil: "A Critique of ANSI SQL Isolation Levels"** (Microsoft Research technical report MSR-TR-95-51 / ACM SIGMOD 1995)
https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf
The original paper that named and formally defined write skew, using this lesson's exact bank-account example (two balances V1 and V2, constraint V1 + V2 >= 0, two concurrent $200 withdrawals). It is the paper every later isolation-level discussion, including Kleppmann's, ultimately cites as the source of the anomaly's definition.

**Cahill, Röhm, Fekete: "Serializable Isolation for Snapshot Databases"** (ACM SIGMOD 2008, later ACM TODS 2009)
https://www.cs.usyd.edu.au/~fekete/serialisation/
The algorithm behind this lesson's section 3 and 4: rw-antidependency tracking, the dangerous-structure (pivot) detection rule, and the benchmark result that SSI's throughput stays close to plain snapshot isolation's under low contention while the abort rate rises with contention, the primary source for why SSI is practical rather than a full-lockdown scheme.

**Ports, Grittner: "Serializable Snapshot Isolation in PostgreSQL"** (VLDB 2012)
https://arxiv.org/abs/1208.4179
The paper describing the real, shipped implementation of SSI added to PostgreSQL in the 9.1 release (2011) by Kevin Grittner and Dan Ports, adapting Cahill et al.'s algorithm to PostgreSQL's actual MVCC engine, including the SIREAD lock mechanism this lesson's section 3 diagram is built on.

**PostgreSQL source: `src/backend/storage/lmgr/README-SSI`**
https://github.com/postgres/postgres/blob/master/src/backend/storage/lmgr/README-SSI
The primary engineering documentation for the dangerous-structure rule (`Tin -rw-> Tpivot -rw-> Tout`), the deferred-rollback optimization (no transaction is canceled until a conflicting transaction actually commits), and the read-only-transaction optimization this lesson's section 4d cites directly from PostgreSQL's own source tree.

**CockroachDB documentation: "Demo: Serializable Transactions"** and blog: **"What write skew looks like"**
https://www.cockroachlabs.com/docs/stable/demo-serializable · https://www.cockroachlabs.com/blog/what-write-skew-looks-like/
The on-call-doctors scenario this lesson's section 1 opens with: a hospital scheduling table, a minimum-one-doctor-on-call rule enforced only in application logic, and a live, runnable demonstration of the identical anomaly occurring in a modern distributed SQL database that, unlike PostgreSQL, defaults to `SERIALIZABLE` isolation for every transaction.

**Martin Kleppmann: "Designing Data-Intensive Applications," Chapter 7, Transactions**
https://dataintensive.net/
The widely read synthesis connecting write skew to the broader family of weak-isolation anomalies (dirty reads, lost updates, phantoms), and the source of the plain-language framing, "it's very hard to reason about the meaning of a query if the data on which it operates is changing at the same time the query is executing," used throughout this lesson's section 2.
