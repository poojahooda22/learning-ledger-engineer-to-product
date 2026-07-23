# Stripe: Balance, funds availability, and payouts (the money-movement ledger underneath)

Date: 2026-07-23
Product: Stripe
Feature: The merchant Balance page, funds availability (pending to available), and payouts to the bank, and the immutable double-entry Ledger that keeps all of it correct.

A note on what this teardown is. Most of this ledger has been about search, ranking, and recommendation: match a query, rank a shortlist, serve a cached answer. This one is a different animal. There is no ranking here. The whole job is to never be wrong about money, not even by one cent, across five billion events a day. The interesting data structure is not an inverted index or a nearest-neighbor graph. It is an append-only log and a rule that every transaction must sum to zero. That is the spine of every real payments company, and it is worth one deep look.

---

## 1. The user, and what they are doing when they hit this

Meet Meera. She runs a small ceramics studio called Clay and Co out of a rented unit in Bengaluru, and she also ships mugs to customers in the US through a Shopify store wired to Stripe. It is Tuesday morning. She has three orders to pack, a kiln that needs a repair, and a part-time helper she pays every Friday.

Before she touches any of that, she opens the Stripe Dashboard on her phone and looks at one screen: the Balance. Two numbers. "Available soon" and "Available to pay out." She is not reading engineering blogs. She is asking one very human question: how much of my money is actually mine right now, and when does it hit my bank so I can pay Ravi on Friday.

That is the moment this feature lives in. A person with bills, standing in a workshop, trusting a number on a screen to be true.

---

## 2. The real problem, described like a friend would

Here is the honest version of the pain, no PM deck.

When a customer taps "Pay" for a $40 mug, the money does not appear in Meera's bank account that second. It cannot. Card money moves through a slow chain: the customer's bank, the card network (Visa, Mastercard), the acquiring bank, and finally Stripe. That chain takes days to actually settle. But Meera needs to know, right now, that the sale worked and that roughly $38 of it is coming to her.

So Stripe is sitting in the middle of a lie that has to become true. It tells Meera "you got paid" the instant the card is authorized, days before the cash physically arrives, and then it has to make sure that promise is kept for every one of the millions of sellers on the platform, minus the exact right fee, in the right currency, landing in the right bank on the right day.

Now multiply the danger. A refund. A chargeback where the customer disputes the mug. A partial refund. A Stripe processing fee. A currency conversion from USD to INR. A payout that fails because Meera fat-fingered her account number. Every one of these moves money, and every one of them is a chance to be off by a cent. Be off by a cent a million times a day and you have lost real money and, worse, you have lost the one thing a payments company sells: trust that the number is true.

The problem is not "show a balance." The problem is "be provably, auditably, regulator-satisfyingly correct about money movement forever, at a scale where a human can never check the math."

---

## 3. The feature in one sentence

Stripe shows every seller a live, trustworthy balance (money pending, money available, money paid out) that is derived from an immutable double-entry ledger recording every single movement of money so the totals can always be re-proven and never silently corrupted.

---

## 4. Jobs to be done

What is Meera really hiring this feature to do.

- "Tell me the sale actually worked, before the cash arrives." Confidence at authorization time.
- "Tell me how much is truly mine and how much is still in the pipe." The pending vs available split.
- "Get my money to my bank on a schedule I can plan payroll around." Predictable payouts.
- "Never cheat me, and never accidentally pay me too much either." Correctness in both directions.
- "If a customer disputes or I refund, adjust my balance cleanly and show me why." Traceable adjustments.
- "When my accountant or a regulator asks, prove every rupee." Auditability.

Notice that only the first three are visible. The last three are the ones that make the whole thing hard, and they are why the machinery underneath is a ledger and not a balance column.

---

## 5. How it works for the user, the visible experience

Meera's $40 mug sale, as she sees it:

1. The customer pays. Within seconds the Dashboard shows a new payment of $40.00, and her balance shows something like "$38.54 pending." Stripe's fee on that US card is 2.9% plus $0.30, which is $1.46, so her net is $38.54. She did not do that math. The screen did.
2. A couple of days later that $38.54 quietly moves from the "pending" column to the "available" column. Same money, new status.
3. On her payout schedule (she is on daily automatic payouts), Stripe gathers up whatever is available and sends it to her bank. A day or two later her bank shows a single deposit, say a lump of $211.87 that is the sum of several days of sales minus fees, with a matching entry in her Stripe payouts list.
4. If she ever refunds a customer, she sees a negative balance transaction appear, and her available balance drops by the refunded amount plus the returned handling. Every line is there.

The magic she feels is boring in the best way: the number is always right, and money shows up when Stripe said it would. She never thinks about the card networks, the settlement lag, or the accounting. That invisibility is the product.

---

## 6. The actual flow, step by step

Tap by tap, and then event by event underneath, for the single $40 mug.

What Meera and her customer do:
1. Customer clicks Pay on the Shopify checkout. Card details go to Stripe, never to Meera's server.
2. Stripe authorizes the card with the network. Authorization succeeds in about a second. The order is confirmed.
3. Meera's Dashboard shows a new charge, $40.00, and pending balance rises by $38.54.

What the money actually does over the next days:
4. Stripe captures the payment. The card network and banks begin the slow settlement dance. Cash is in transit.
5. After the settlement window for her country and card type (in the US the standard is roughly T plus 2 business days, where T is the transaction time), the funds are treated as settled. The balance transaction for that charge flips from status "pending" to status "available."
6. On her payout schedule, a payout job sweeps all available balance transactions that have not yet been paid out, sums them, and creates one payout. That payout is itself a balance transaction, a negative one, that drains the available balance.
7. Stripe instructs its bank to send that amount to Meera's bank account. One to two days later it lands.

Every arrow in that flow, the charge, the fee, the settle, the payout, the eventual bank credit, is written down as an immutable record. That written-down record is the ledger, and it is where the real engineering lives.

---

## 7. Under the hood, like the engineer

This is the heart of it. I will separate what Stripe has publicly described (fact) from the well-established way this class of system is built (inference), and label each.

### 7a. The one idea: derive the balance, never store it

The naive design, the one almost every first-time engineer reaches for, is a table of accounts with a `balance` column, and you `UPDATE balances SET balance = balance - 146 WHERE id = 'meera'` on every fee. This is a trap. The moment you store a balance and mutate it in place, you have thrown away history, and a single bug, race condition, or partial failure corrupts the number with no way to know it happened or to reconstruct the truth (general fintech-ledger practice, well documented by Modern Treasury and Formance).

The correct design flips it. You never store a balance as a mutable number. You store an append-only log of entries, and the balance is a derived quantity: the sum of the entries for an account. Corrections are not edits, they are new compensating entries. The log is never updated and never deleted. If you made a mistake, you add a reversing entry, and the evidence of the mistake stays in the record forever (fact, this is the defining property of a ledger; Stripe describes Ledger as "an immutable and auditable log" that is the system of record for all of Stripe's financial data).

Concrete example. Meera's $40 charge is not "balance += 3854." It is a set of paired records:

```
Transaction T1 (the $40 charge), all amounts in cents:
  debit   customer_funds_in_transit   4000
  credit  meera_pending_balance       4000
Transaction T2 (the Stripe fee):
  debit   meera_pending_balance        146
  credit  stripe_revenue               146
```

Two things to see. First, each transaction's entries sum to zero (4000 out equals 4000 in; 146 out equals 146 in). That is double-entry bookkeeping: money is never created or destroyed, it only moves between accounts, so every legal transaction must balance to zero (fact, Stripe: Ledger "applied traditional accounting principles" so that "credits and debits balance out"). Second, Meera's pending balance is now `4000 - 146 = 3854`, and nobody wrote `3854` anywhere. It is the sum of her entries. Ask for her balance and you add up her postings.

### 7b. The data structures, and why each one

- An append-only log (fact). Not a mutable table. Think of it as an ever-growing array of immutable entry records, the same log-structured idea as an LSM tree's write path (see this ledger's Lesson 21). Writes only ever append. This is what makes the whole thing auditable and safe under concurrency: you can never corrupt a row you are not allowed to touch, because you never touch old rows.

- Accounts and entries (fact). The vocabulary is Accounts, and Entries (debits and credits) grouped into balanced Transactions. Meera's pending balance, Meera's available balance, Stripe's revenue, the customer-funds-in-transit pool, each is an account. An entry is a signed amount posted to one account. This is a classic general-ledger shape.

- A hash map from account id to a cached balance snapshot (inference, standard practice). Summing an account's entire history on every read gets expensive fast, so real systems keep a checkpoint: a stored snapshot balance as of entry number N, plus you only sum the tail of entries after N. The log stays the source of truth; the snapshot is a cache you can always rebuild by replaying the log. Modern Treasury describes exactly this kind of balance caching to get "read-after-write consistency and super fast batched writes on hot accounts."

- State machines for producer systems (fact). Stripe describes Ledger as modeling its internal data-producing systems as state machines and encoding their behavior as "logical fund flows," the movement of balances between accounts. So a charge system, a payout system, a disputes system each become a state machine whose transitions emit balanced entries into the log. This is the adapter layer that turns "a card got authorized" into "post these two entries."

- Idempotency keys on ingestion (inference, and consistent with Stripe's published idempotency work, see the 2026-06-20 teardown). Five billion events a day means retries, duplicates, and replays are constant. Each incoming money-movement event carries a unique key so that ingesting it twice posts the entries once. Without this, one network retry double-counts a payout. This is the same forward-only, exactly-once discipline from the idempotency-keys teardown, now applied to the ledger's front door.

### 7c. Matching and ranking? No. Ingest and verify.

The two-half pattern that runs through most of this repo (match then rank) does not apply here. The two halves of a ledger are different: ingest (accept the event, post balanced entries, exactly once) and verify (continuously re-prove that the books balance and that the ledger agrees with the outside world).

Verification is the part outsiders never see and the part Stripe is proudest of. On top of Ledger they built a Data Quality (DQ) Platform that automates detection of money-movement issues and provides response tooling (fact). Two kinds of checks:

- Internal consistency: does every transaction still sum to zero, and does every account's snapshot equal the re-derived sum of its entries. Because the log is immutable, you can recompute any balance from scratch and compare. A mismatch is a bug or a corruption, and it is caught by math, not by a human noticing.
- External reconciliation: does Stripe's ledger agree with what the banks and card networks actually did. Money leaves the card network on their timeline, not Stripe's, so the ledger's "we should receive $38.54 for this charge" must be matched against the bank's settlement file. Stripe reports that 99.99% of dollar volume is ingested and verified within four days (fact). That four-day window is the settlement reality of card money, and the DQ platform is what closes the loop.

The deep point: correctness here is not asserted once at write time and forgotten. It is an ongoing property that the system re-proves continuously, because the only way to trust a number you cannot personally check is to have a machine that re-derives it and screams when it drifts.

### 7d. Where the sort happens, and where the money math happens

Same discipline as the rest of this repo: the expensive, authoritative work happens server-side, never on Meera's phone. Her phone does not compute her balance. It asks Stripe's API, and Stripe returns a Balance object with the buckets already computed: `available`, `pending`, and for platforms `connect_reserved` and `instant_available` (fact, these are the real fields on the Stripe balance object). The phone renders numbers it was handed. The fee arithmetic, the pending-to-available transition, the payout batching, all of it is decided on the server against the ledger.

### 7e. The scale story at three tiers

Tier one, about 1,000 events. A single Postgres table of entries. Balance is `SELECT SUM(amount) FROM entries WHERE account_id = 'meera'`. Double-entry enforced by a check that each transaction's entries sum to zero. This is genuinely all you need for a small business, and it is correct. Nothing breaks. This is the tier where founders wrongly conclude a `balance` column would have been fine; it would have, right up until it wasn't.

Tier one hundred thousand events. Two things start to hurt. First, summing an account's full history on every read is now slow, so you introduce checkpointed balance snapshots and only sum the tail since the last snapshot (inference, standard). Second, and more painful, the hot account problem appears. Some accounts are touched by almost every transaction: the Stripe revenue account that every fee credits, or a big marketplace platform's central funding account. Under load, concurrent writes to that one row serialize behind a lock and become the bottleneck (fact, this is the well-known hot-account contention problem; Modern Treasury calls out "truly hot accounts such as a shared SYSTEM_FUNDING row"). This is the exact same celebrity/hot-key problem this repo hit in Zepto's last-carton counter and Razorpay's per-terminal counter (Lesson 16).

Tier ten million plus, up to Stripe's five billion events a day. Now you need every trick at once (mix of Stripe fact and general fact):
- Shard the ledger by account so unrelated accounts never contend. Meera's studio and a merchant in Berlin live on different shards and never block each other.
- Split the hot account. The single revenue or funding row becomes N sub-accounts (sub-account sharding), and you sum across them for the true balance. This is splitting one hot counter into many, the identical fix used for contested inventory in the Zepto teardown.
- Async queueing for the hottest rows, so writes to a contended account are serialized through a queue instead of fighting over a lock (fact, Modern Treasury lists async queueing and sub-account sharding as the two fixes beyond pessimistic locks).
- Cells-based architecture, where an entire ledger deployment is one cell and you add cells rather than growing one giant system, which also bounds the blast radius of any single failure (fact, Modern Treasury; and see this repo's Lesson 34 on cell-based architecture).
- The DQ platform running continuously to catch any drift, with that four-day, 99.99%-of-dollars verification window as the backstop.

The through-line: the append-only log is what makes all of this safe. You can shard it, cache it, replay it, and re-derive any balance, precisely because you never mutate old records. The immutability is not a compliance nicety bolted on top. It is the property that lets the system scale without going wrong.

---

## 8. The retention and habit mechanic

This feature does not build a dopamine loop. It builds dependence, which is stronger.

The habit is small and daily: Meera opens the Dashboard and checks her balance, the way you check a bank app. Every time the number is right and the payout lands exactly when promised, a little more trust is deposited. That trust compounds into something Stripe never has to advertise: switching cost. Once Meera runs her Friday payroll off the rhythm of Stripe payouts, once her cash flow assumes "available money hits my bank in T plus 2," leaving Stripe means re-plumbing the artery her business lives on. That is scary in a way a competitor's slightly lower fee cannot overcome. The metric this moves is retention, and it is defensive: reliability as a moat, the same shape seen in the Stripe Radar and WhatsApp encryption teardowns.

There is also a direct revenue lever bolted onto the exact pain point this feature creates. The pending-to-available lag is a real wait, and waiting is a thing people will pay to skip. So Stripe sells Instant Payouts: for a fee (around 1.5% of the amount, with a small minimum), the available balance can be pushed to a debit card in minutes instead of waiting for the schedule (fact, Instant Payouts is a real paid Stripe feature). The ledger makes this safe: because the system knows precisely which funds are truly available and can post the payout as a balanced, idempotent transaction, it can hand money out early without risking a double payout. So the same correctness machine that builds trust also directly monetizes impatience. The mechanic moves both retention and revenue.

Real observed example: a seller doing weekend market sales who needs cash to restock Monday morning will happily pay 1.5% to have Saturday's sales in hand Sunday night rather than waiting for Tuesday's scheduled payout. The lag is the product surface; instant is the upsell.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus an embeddable runtime. The lesson from Stripe's ledger is about how you store the state of a project, and it is directly a scalability and performance lesson.

Do not store the compiled state of the graph as mutable values you overwrite in place. Store an append-only log of edit operations (add node, connect, change a parameter, delete), and derive the current graph and the compiled shader by folding that log. Keep periodic checkpoints (snapshots of the derived state as of operation N) so that recompute cost tracks the tail of new operations, not the full history of the project. This is exactly the ledger's derive-do-not-store discipline, and it buys Rare.lab four things at once:

1. Undo and redo become free and exact. Fold the log to operation N and you are precisely at that past state, with no special-cased inverse operations to maintain.
2. Deterministic replay and audit. When a user asks "why is this pixel this color" or "why did the compile change," you can replay the log and show the exact operation that caused it, the same way Stripe can prove any balance from its entries.
3. Multiplayer merge for free-ish. Operations are the natural substrate for collaborative editing and CRDT merging, connecting straight to the Figma and Notion teardowns in this repo. Two artists editing the same graph are two producers appending to one log.
4. Performance that does not degrade with project age. Because you checkpoint and only fold the recent tail, a two-year-old project with a million historical edits recompiles as fast as a fresh one. Cost tracks recent change, not total size. That is the same snapshot-plus-tail trick that keeps a hot ledger account cheap to read no matter how many transactions it has seen.

And steal the double-entry invariant idea for GPU resources. Enforce a balance rule at the boundary of the runtime: every buffer, texture, or pass allocated for a frame must be accounted for and freed, and the "in" must equal the "out." Check it at the API layer where operations are created, not deep in application code, so a whole class of resource-leak and double-free bugs becomes impossible by construction rather than caught later by a profiler. The strongest correctness guarantees come from an interface that refuses to let you create an unbalanced state in the first place, which is precisely why Stripe puts the balance check in the ledger and not in the app.

In one line: make the log the source of truth, derive everything else, checkpoint so the hot path stays cheap, and enforce your invariants at the write boundary so corruption cannot happen instead of hoping to notice it.

---

## Sources

- Stripe engineering blog, "Ledger: Stripe's system for tracking and validating money movement" (primary source, describes the immutable auditable log, double-entry validation, state-machine modeling of producer systems, logical fund flows, five billion events per day, 99.99% of dollar volume verified within four days, and the Data Quality platform): https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement
- Sam Boboev / Fintech Wrap Up, "Deep Dive: Ledger, Stripe's system for tracking and validating money movement": https://www.fintechwrapup.com/p/deep-dive-ledger-stripes-system-for
- Density Labs, "Building Trust: How Stripe Ensures Financial Accuracy with Ledger": https://densitylabs.io/blog/building-trust-how-stripe-ensures-financial-accuracy-with-ledger/
- Stripe API Reference, the Balance object (fields: available, pending, connect_reserved, instant_available): https://docs.stripe.com/api/balance
- Stripe API Reference, the Balance Transaction object (types including charge, refund, payout, stripe_fee, adjustment; amount, fee, net; status available vs pending): https://docs.stripe.com/api/balance_transactions
- Stripe Docs, Payouts FAQ and payout schedules / settlement timing (T plus X): https://support.stripe.com/questions/payouts-faq
- Stripe Docs, Instant Payouts: https://docs.stripe.com/payouts/instant-payouts-with-advance-funding
- Modern Treasury Journal, "How to Scale a Ledger" series (immutability and double-entry, hot-account contention, balance caching, sub-account sharding, cells-based architecture): https://www.moderntreasury.com/resources/how-to-scale-a-ledger
- Formance blog, ledger integrity and double-entry accounting for engineers (append-only, balances derived not stored, compensating entries): https://www.formance.com/blog/engineering/double-entry-accounting-for-engineers-building-financial-products
