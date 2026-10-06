# Day 82: How does a payments company keep every rupee accounted for when millions of transfers hit the same few accounts? Double-entry ledgers, immutability, and the hot account problem

**Date:** 2026-10-06
**Difficulty:** Advanced (correctness invariants as a data model, lock contention as a scaling wall, sync vs async balances)
**Topic:** A money ledger as a distributed system. Why money is stored as immutable entries (not a balance column), why one popular account becomes a serial bottleneck, and the three ways real systems break that wall.
**Stack relevance:** Rare.lab will meter usage (renders, credits, seats). A credits balance is a tiny ledger. Section 7 maps it onto Supabase Postgres.

---

## 0. The framework (same six steps as Day 74)

1. Functional: move money between accounts, show a balance, show history, support holds (authorize now, capture later), refunds, and audits.
2. Non-functional: never lose or invent a cent, never let a balance go below its limit, tolerate retries, keep history forever, serve high write rates.
3. Entities: Account, Transfer (or Transaction), Entry (one leg of a transfer), Balance.
4. API: `POST /transfers {id, debit_account, credit_account, amount, currency}` and `GET /accounts/{id}/balance`.
5. Naive design: one `accounts` table with a `balance` column. `UPDATE accounts SET balance = balance - 500 WHERE id = A`, then `UPDATE ... balance = balance + 500 WHERE id = B`.
6. Deep dives: double-entry as a built-in checksum, immutability, idempotency, the hot account wall, sync vs async balance, purpose-built ledger databases.

---

## 1. The company and the breaking number

**Stripe, Square, Uber, Razorpay, any marketplace: the platform's own fee account is a party to EVERY payment. One account, every transaction. Call it 1,000 payments per second, so 1,000 updates per second to ONE row.**

- Ledger vendors name this the **hot account problem**. Modern Treasury calls it "an issue inherent to double-entry accounting at scale" (per their ledger docs, via search excerpt).
- TigerBeetle's docs say the same thing: "a small number of hot accounts are often involved in a large proportion of the transactions, so the shards responsible for those accounts become bottlenecks" (via search excerpt).
- Measured, not guessed: a SoftwareMill benchmark (search excerpt) found that under heavy skew (Zipfian 2.0, meaning a few accounts get most traffic) PostgreSQL fell to about **753 TPS** while TigerBeetle held about **4,461 TPS** in the same test. Same hardware, same workload, 6x gap, caused only by contention on hot rows. Treat as one vendor-adjacent benchmark, not a law.
- Scale reference: Uber's LedgerStore holds **more than a trillion entries, a few petabytes** (Uber engineering blog, April 2024, via search excerpt). History never shrinks, so storage also grows forever.

Analogy: a shop with 50 tills but one cash drawer that every till must open to log each sale. Fifty cashiers, one drawer, and they queue.

---

## 2. Why the naive design dies

**Naive: a `balance` column, updated in place.**
- **No history, no proof.** After `balance = 4,500`, you cannot say how it got there. An auditor asks "why is this 4,500?" and you have nothing. A bug that skips an update silently destroys money and nothing flags it.
- **Lost updates under retry.** The app sends the debit, the network drops the reply, the client retries. Without a unique transfer id the debit runs twice. Real example: a customer taps "Pay 500" on a flaky 4G connection and is charged 1,000.
- **Half-done transfers.** Debit A succeeds, the server crashes, credit B never happens. 500 rupees vanish from the world. A transaction fixes this, but only on one database node.
- **The hot row wall.** Every update takes a row lock. 1,000 transfers per second to the platform fee account means 1,000 transactions queue on one row, each waiting for the previous commit. Throughput is capped at about 1 divided by commit time. If a commit takes 5 ms, that is about **200 updates per second on that row, no matter how many servers you add**.
- **Sharding does not rescue it.** Put accounts on different shards and the transfer now spans two shards (the Day 20 distributed transaction problem), and the hot account still lives on exactly one shard.

---

## 3. The architecture, top to bottom

```
Clients (app, checkout, partner API)
        |
        v
Edge / API gateway ......... auth, rate limit (Day 8), attach an idempotency key
        |
        v
Stateless app tier ......... validate, build the transfer, never touches balances directly
        |
        +--> Idempotency check (unique transfer id) ... "have I seen this id?"
        |
        v
Ledger core (the only writer) .. appends immutable entries, enforces invariants
        |            \
        |             +--> Balance cache / projection (running totals per account)
        v
Append-only entry log ...... durable, replicated, never updated or deleted
        |
        v
Async pipeline ............. reconciliation, reports, webhooks, warehouse export
```

- **Edge:** attach the idempotency key. Job: make retries safe. Analogy: a numbered receipt book, so a duplicate receipt is obviously a duplicate.
- **Stateless app tier:** builds the transfer, holds no money state. Job: scale out freely. Analogy: tellers who fill in forms but cannot touch the vault.
- **Ledger core:** the only code allowed to write entries. Job: enforce "debits equal credits" and "no overdraft" atomically. Analogy: the vault clerk who checks the form adds up before opening the door.
- **Append-only entry log:** the source of truth. Job: history that can only grow. Analogy: a bank passbook written in ink.
- **Balance projection:** a derived running total. Job: answer "what is my balance" in O(1) instead of summing a million entries. Analogy: the total written at the bottom of the passbook page.
- **Async pipeline:** everything that does not need to be inside the money-moving path. Job: keep the hot path tiny.

**Worked example, one real shape.** A buyer pays Rs 1,000 for a seller on a marketplace with a 2% fee (Rs 20):

| Transfer leg | Debit account | Credit account | Amount |
|---|---|---|---|
| Buyer pays | Buyer funds (customer card clearing) | Platform holding | 1,000 |
| Fee | Platform holding | Platform fee revenue | 20 |
| Seller share | Platform holding | Seller payable | 980 |

Sum of debits = sum of credits = 2,000 in the log, and "Platform holding" nets to zero once all three are posted. Platform fee revenue is the hot account: it appears in every single marketplace sale.

---

## 4. The transferable mechanisms

1. **Double-entry as a built-in checksum.** Every transfer writes at least two entries, one debit and one credit, equal in amount. Invariant: across the whole system, debits minus credits is exactly zero. If it is ever not zero, a bug exists and you know within one query. Single-entry bookkeeping has no such alarm. Teach it as a database constraint, not an accounting ritual. Modern Treasury says it never violates three principles: double-entry, auditability, immutability (excerpt).

2. **Immutability, append-only.** Never update an entry. A mistake is fixed by writing a new reversing entry. Balance is a pure function of the entries, so any past balance can be rebuilt. Real example: a Rs 500 charge refunded is two facts in the log (charge, refund), not one deleted row. Enforce it in the database, for instance with a Postgres trigger that rejects UPDATE and DELETE on the entries table, and restrict roles so app code cannot bypass it. Square's "Books" is described as an "immutable double-entry accounting database service" built on Google Cloud Spanner (Square blog, 2019, via excerpt).

3. **Idempotency key as the transfer id.** Make the client-chosen transfer id the primary key. A retry hits the unique constraint and returns the original result. TigerBeetle makes transfer ids unique and client-generated for this reason (their docs, via excerpt). Ties to Day 12.

4. **Two-phase (pending) transfers.** A card authorization reserves money without moving it: write a pending transfer, later post it (capture) or void it (release), usually with a timeout. Real example: a hotel authorizes Rs 10,000 at check-in, captures Rs 8,400 at checkout, releases the rest. This is "reservation plus TTL" from the sale-day lesson applied to money.

5. **Balance as a cache of the log, with a lock only where you need one.** Store a running balance per account for O(1) reads (Modern Treasury describes a total balance cache and an effective-time balance cache, per excerpt). Row-lock the balance only when a constraint such as "no overdraft" needs a serial check. If an account may go negative (a platform fee account), you do not need the check, so you do not need the lock.

6. **Sync vs async entries.** Modern Treasury splits writes: synchronous entries lock the account and are rate limited because they "must be processed serially"; asynchronous entries are applied to the balance later, and for hot accounts "all Ledger Entries will be applied ... within 60 seconds" (docs, via excerpt). The ledger log is always exact. Only the cached balance lags. That one distinction is the hot-account escape hatch.

7. **Batch and single-writer.** TigerBeetle takes up to **8,190 transfers per request** and "funnels all writes through a single core on the primary" node (search excerpts), with replication using Viewstamped Replication (a Raft-family consensus, Day 11). No row locks: one thread applies a whole batch in memory, so a hot account costs nothing extra. Its docs target around **1 million transfers per second** on commodity hardware (design goal; independent numbers vary, see section 5).

---

## 5. The trade-offs accepted

- **Consistency vs availability, per data type.**
  - Entry log: strongly consistent, always. A ledger that is "eventually right" is wrong. Choose CP. During a partition, refuse writes rather than risk a double spend.
  - Balances shown to a user: can lag a few seconds if you accept the risk of a brief stale view. Fine for a dashboard, not fine as the input to an overdraft check.
  - Reports, analytics, exports: eventually consistent, built from the log.
- **Cost vs latency.** A purpose-built ledger (TigerBeetle) buys 6x or more under skew but is a new database to operate (their docs recommend six replicas across three cloud providers for production, via excerpt). Postgres with a balance cache costs nothing new and fits a few hundred to low thousands of TPS per hot account. Pick by your hot-account rate, not by hype.
- **Flexibility vs speed.** TigerBeetle has two fixed tables (accounts, transfers) and no custom columns (excerpt). That rigidity is exactly why it is fast. Postgres lets you add metadata but pays for it in locks and bloat.
- **Storage forever.** Immutable means the log only grows. Uber moved from DynamoDB hot storage (about 12 weeks of data) plus an S3-like blob store to its own LedgerStore, saving about **$6 million a year** (InfoQ and Uber blog via excerpt). Lesson: hot recent data on fast storage, cold history on cheap storage, one logical ledger.

Honest caution on numbers: published TPS figures for ledger databases come from vendors or vendor-adjacent benchmarks and use different hardware. Compare shapes (contention helps or hurts), not headline digits.

---

## 6. The systems-thinking lens

**The loop: lock convoy on a hot row, amplified by retries.**

```
hot row lock held -> next requests wait -> latency climbs past client timeout
      ^                                                  |
      |                                                  v
more queued transactions  <---  client retries the timed out transfer
```

- Each timed-out client retries. Retries add more waiters to the same row. Waiters hold connections, which starves the pool (Day 78). Latency rises, more timeouts, more retries. It is the retry death spiral (Day 13) with a row lock as the choke point.
- Adding database CPU does nothing, because the work is serial by design: only one transaction can hold the lock.
- The senior fixes break the loop, they do not add capacity:
  1. **Remove the lock from the hot path:** asynchronous entries and a lagging balance for accounts with no overdraft rule.
  2. **Idempotency keys:** a retry of an already committed transfer is a cheap lookup, not a second lock wait.
  3. **Shard the hot account itself:** instead of one fee account, keep N sub-accounts (`fee_0` to `fee_15`), write to a random one, and sum them for reporting. Same trick as sharded counters. You trade a simple balance read for 16 reads and 16x less contention.
  4. **Batch:** apply 8,000 transfers in one commit so the lock is taken once per batch, not once per transfer (the TigerBeetle move).
  5. **Backpressure:** when queue depth passes a limit, reject early (Day 13) so waiting never exceeds client timeout.

---

## 7. Map to Rare.lab's stack (Supabase Postgres with RLS, R2 immutable scene JSON plus manifest, one shared WebGL context)

If Rare.lab sells credits or metered renders, you need a tiny ledger.

| Ledger pattern | Rare.lab touchpoint | Status | Action |
|---|---|---|---|
| Immutable, content-addressed objects | R2 scene JSON plus manifest | **Already doing this** | Same philosophy: never edit in place, append a new version, flip the pointer. A credits ledger is the same idea applied to numbers. |
| Append-only entries | Credits or usage in Supabase Postgres | Do it now if not yet | `credit_entries(id uuid primary key, account_id, delta int, reason, created_at)`. No `UPDATE` or `DELETE` grants. Balance = `sum(delta)`. A user's balance column, if it exists, becomes a cache. |
| Idempotency key | Render job and Stripe or Razorpay webhooks | Must have | Use the webhook event id or job id as the entry primary key (`insert ... on conflict do nothing`). Webhooks are delivered at least once, so duplicates are certain. |
| Pending (hold) then capture | Reserve credits when a render starts, settle on finish | Good fit | Write a `hold` entry at start, a `capture` or `release` entry at end, with a timeout job. Prevents users from starting 10 renders on 1 credit. |
| RLS | Per-user ledger reads | **Already doing this** | Users may `select` their own entries only. Writes only via a `security definer` function, never directly from the client. |
| Hot account | A shared "platform revenue" or "free tier pool" account | Future ceiling | Do not row-lock a single balance row for every render. Write entries only and compute the total in a periodic job. |

**Where the next ceiling is (inference, not measured):** two or three hundred committed writes per second on one hot Postgres row is a realistic soft limit, and you hit it only if every render updates one shared counter. Before that, `sum(delta)` over a user's entries slows past about a million rows per user (unlikely). Fixes in order: partial index plus a periodic balance snapshot entry, then sharded sub-accounts, then (only at serious scale) a dedicated ledger engine.

**One-line lesson for Rare.lab:** store credits as an append-only log with a unique id per event and treat the balance as a rebuildable cache, because that one choice gives you auditability, safe retries and an escape from hot-row locks.

---

## 8. Sources and what is actually in them

**Honest note on access:** moderntreasury.com, developer.squareup.com, ximedes.com and docs.tigerbeetle.com were blocked by the network proxy in this session. I could not open them. Every fact below comes from search result excerpts and is marked "via excerpt". Verify numbers before quoting them.

- [Modern Treasury: Behind the scenes, how we built Ledgers for high throughput](https://www.moderntreasury.com/journal/behind-the-scenes-how-we-built-ledgers-for-high-throughput). Their engineers explain the hot account problem and their answer: cached balances plus Postgres row locking when you need serial checks. Source of the "inherent to double-entry accounting at scale" line.
- [Modern Treasury docs: Design a Ledger for Concurrency](https://docs.moderntreasury.com/docs/handling-concurrency) and [Synchronous Ledger Entry Rate Limiting](https://docs.moderntreasury.com/docs/synchronous-ledger-entry-rate-limiting). Practical rules: lock versions and balance conditions for safety, skip them on hot accounts, async entries land within 60 seconds.
- [Modern Treasury: Enforcing immutability in your double-entry ledger](https://moderntreasury.com/journal/enforcing-immutability-in-your-double-entry-ledger). How to make posted transactions unchangeable and why. Not opened.
- [Modern Treasury: How to scale a ledger (series, parts II, IV, VI)](https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-ii). A multi-part series on the same problem. Not opened.
- [Square Engineering: Books, an immutable double-entry accounting database service](https://developer.squareup.com/blog/archive/16). A large payments company built an immutable ledger on Cloud Spanner. Not opened.
- [TigerBeetle: Performance concepts](https://docs.tigerbeetle.com/concepts/performance/) and [OLTP concepts](https://docs.tigerbeetle.com/concepts/oltp/). Why a ledger is a different workload from normal OLTP: batching, single-core apply, no row locks.
- [TigerBeetle vs PostgreSQL benchmark (SoftwareMill)](https://softwaremill.com/tigetbeetle-vs-postgresql-performance-benchmark-harness-cloud-tests). Source of the 753 vs 4,461 TPS under contention figure. Third-party, but read the setup before trusting it.
- [Jepsen analysis of TigerBeetle 0.16.11](https://jepsen.io/analyses/tigerbeetle-0.16.11). Independent correctness testing of the database. Not opened. Highest-authority source for whether the safety claims hold.
- [Ximedes: Solving the hot key problem](https://ximedes.com/blog/solving-the-hot-key-problem). A payments processor's write-up of hot accounts. Not opened.
- [Uber: How LedgerStore supports trillions of indexes](https://www.uber.com/en-IE/blog/how-ledgerstore-supports-trillions-of-indexes) and [Migrating a trillion entries from DynamoDB to LedgerStore](https://eng.uber.com/en-FR/blog/migrating-from-dynamodb-to-ledgerstore/). Immutable ledger store at trillion-entry scale, and the $6M a year saving. Via excerpt.
- [InfoQ coverage of the Uber migration](https://www.infoq.com/news/2024/05/uber-dynamodb-ledgerstore) and [ByteByteGo summary](https://blog.bytebytego.com/p/trillions-of-indexes-how-ubers-ledgerstore). Readable retellings.
- [Hello Interview: Design a ledger](https://www.hellointerview.com/community/questions/internal-ledger-system/cm8lxn4te00033x68eynjv94p). Interview-style walkthrough of the same design.

**Inference, labeled:** the 5 ms commit and 200 updates per second arithmetic, the 1,000 payments per second scenario, the marketplace fee example, sub-account sharding of the fee account, and all of section 7.

**About the video in your prompt:** the "Design LeetCode" mock interview is fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md), so I did not repeat it. Use its recipe here: requirements, entities, API, naive design, find the breaking number, fix one layer at a time, name the trade-off. Today's lesson is that recipe applied to money.

**Related ledger lessons:** Day 6 (Stripe correctness), Day 12 (idempotency), Day 13 (backpressure), Day 16 (hot key), Day 20 (distributed transactions), Day 59 (event sourcing), Day 78 (connection pooling).
