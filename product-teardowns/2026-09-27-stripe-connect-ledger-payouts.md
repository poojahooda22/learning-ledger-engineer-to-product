# Stripe Connect: splitting one payment across a marketplace, and the double-entry ledger that keeps every rupee honest

Date: 2026-09-27
Product: Stripe
Feature: Connect (the marketplace money split), the immutable double-entry ledger that tracks the movement, and the payout that lands the money in a seller's bank

Note on scope: earlier Stripe teardowns covered idempotency keys (2026-07-31), Radar fraud scoring (2026-08-09), webhooks (2026-08-31), Checkout (2026-09-01... the hosted pay page), and rate limiting and load shedding (2026-08-30). This one is a different animal. Those were about accepting one payment from one buyer to one merchant. This is about what happens when the money that comes in the front door has to be split between many parties, tracked so that not a single paisa goes missing, and then paid out to a seller's bank on the right day for the right amount. The heart of it is not the payment. It is the ledger.

---

## 1. The user

Meet two people, because this feature only exists because of the gap between them.

The first is Meera. She bakes custom cakes out of her kitchen in Pune and sells them on a marketplace app called CakeBazaar. She does not have a payment gateway. She does not have a company bank integration. She has a phone, an oven, and a page on CakeBazaar with photos of her red velvet. On Saturday morning a customer orders a two kilogram cake for 2,000 rupees. Meera bakes it, hands it over, and now the only thing she cares about is one sentence on her CakeBazaar dashboard: "You will be paid 1,600 rupees on Tuesday." She wants that number to be exactly right, and she wants it to actually arrive on Tuesday.

The second is Arjun. He runs CakeBazaar. He has 4,000 bakers like Meera. Every order that comes in is one payment from a customer, but that one payment belongs to three parties at once: the baker who made the cake, CakeBazaar which takes a 20% commission, and Stripe which takes its processing cut. Arjun never wants to touch that money with his own hands or hold it in his own company account, because the moment he does, he becomes a regulated money transmitter and his life becomes lawyers. He wants the split to happen automatically, correctly, and provably.

They hit this feature at the same moment: the instant a customer taps Pay on a 2,000 rupee cake, that single payment has to fan out to the right people in the right amounts, and both Meera and Arjun have to be able to trust the numbers without ever checking Stripe's math by hand.

---

## 2. The real problem

Here is the pain, described the way Meera and Arjun would actually say it.

Arjun first. "A customer pays me 2,000 rupees for Meera's cake. But it is not my 2,000 rupees. 1,600 is Meera's, and I am just standing in the middle. If that money lands in CakeBazaar's normal bank account, mixed with my own revenue, then legally I am holding other people's money, which in India and the US and the EU makes me a money transmitter and I need licenses I cannot get. I also have 4,000 bakers, each on a different commission, some with refunds pending, some with a chargeback from last week I still have to claw back. If I try to track who is owed what in a spreadsheet, I will get it wrong, and getting it wrong means either I underpay a baker who then leaves, or I overpay and eat the loss. At scale, small errors are not small. A rounding mistake on a paisa, multiplied by a million orders, is a real hole in my books that an auditor will find."

Meera next. "I do not care about any of that. I made a cake. I want to know two things. How much do I get, and when does it hit my bank. If the app says 1,600 on Tuesday and then 1,540 shows up on Thursday, I will stop trusting the app, and I will go sell my cakes somewhere else or just take cash."

Underneath both of them is the deepest problem in all of payments: money must be conserved. Every rupee that comes in has to be accounted for somewhere. It cannot be created and it cannot vanish. A ranking feature can show a slightly wrong result and nobody dies. A ledger that is wrong by one rupee is not slightly wrong. It is broken, because now the system does not know where money is, and in finance not knowing where money is means either you have lost it or you have committed fraud. There is no "close enough."

---

## 3. The feature in one sentence

Stripe Connect lets a marketplace take one payment from a buyer, split it automatically between the platform's fee and one or more sellers without the platform ever holding the money, record every leg of that movement in an immutable double-entry ledger that must always balance to zero, and then pay each seller's share into their own bank account on a schedule.

---

## 4. Jobs to be done

What is Meera really hiring this for? "Get me paid the exact right amount, on the day you promised, into my bank, without me having to do any accounting or trust anyone's spreadsheet."

What is Arjun really hiring this for? Three jobs, stacked.

- "Split every payment correctly, automatically, so I never manually calculate who gets what."
- "Keep me out of the money. Never let other people's funds sit in my company account, so I do not become a regulated money transmitter."
- "Give me books that provably balance, so that when an auditor or a tax authority or my own finance team asks 'where is every rupee,' I have an answer for every single one, not an estimate."

And there is a fourth job that is invisible until it fails: "When something goes wrong (a refund, a chargeback, a failed bank transfer, a server crashing mid-transaction), do not lose money and do not double-move money. Recover to a state that still balances."

---

## 5. How it works for the user

For the buyer, it is invisible. The customer on CakeBazaar sees a normal pay page, enters a card, pays 2,000 rupees, done. They have no idea the money is about to be split three ways. That invisibility is the point.

For Meera the baker, it is a dashboard and a bank deposit. She sees "Order #4471: 2,000 rupees. CakeBazaar commission: 400. Processing fee: about 40. Your earnings: 1,560. Status: Pending, available Tuesday." On Tuesday, 1,560 rupees lands in her bank account with a reference she can match. She onboarded once, at the start, by entering her PAN, bank account, and a few details in a Stripe-hosted form that CakeBazaar embedded. She never saw Stripe's name much; to her it felt like CakeBazaar paid her.

For Arjun the platform, it is an API call and a balance. When the order is paid, his server makes one call that says, in effect, "charge 2,000, send 1,600 to Meera's connected account, keep 400 as my fee." He watches a dashboard showing his platform balance and every connected account's balance. He never moves the 1,600 by hand.

The three "charge types" Connect offers are really three answers to one question: whose balance does the money touch first?

- **Direct charge**: the money is charged directly on Meera's connected account, and CakeBazaar pulls its 400 fee out as an `application_fee_amount`. Meera is the merchant of record. Good when the seller is basically running their own store (think Shopify merchants).
- **Destination charge**: the money is charged on CakeBazaar's account, and in the same API call Stripe transfers 1,600 to Meera via `transfer_data[destination]`. CakeBazaar is the merchant of record and controls the experience. This is the common marketplace pattern.
- **Separate charges and transfers**: CakeBazaar charges the full 2,000 to its own account now, and later, maybe after the cake is delivered, issues a separate transfer of 1,600 to Meera. Good when you do not know the split at payment time, or when one payment funds several sellers (a customer buys a cake from Meera and cupcakes from another baker in one basket).

---

## 6. The actual flow, step by step

Walk the real 2,000 rupee cake order, tap by tap, for a destination charge (the typical marketplace flow).

1. Customer taps Pay on CakeBazaar. CakeBazaar's server creates a PaymentIntent for 2,000 rupees with `transfer_data[destination] = acct_Meera` and `application_fee_amount = 400`.
2. The card is charged. 2,000 rupees is captured on CakeBazaar's Stripe account.
3. In the same operation, Stripe creates a Transfer object moving 1,600 to Meera's connected account. The 400 application fee stays with CakeBazaar. The Stripe processing fee (roughly 2% plus a fixed bit, so about 40 rupees here) is deducted from CakeBazaar's side.
4. Two balances update. Meera's connected-account balance goes up by 1,600, sitting as **pending**. CakeBazaar's platform balance reflects its 400 fee minus Stripe's cut.
5. The money "settles." For card payments Stripe uses a rolling schedule where funds become **available** on the connected-account balance about two business days after the charge (T+2). So Saturday's cake becomes available around Tuesday. This is why the dashboard says Tuesday.
6. On Meera's payout day, Stripe runs a **payout**: it takes her available balance (1,600, assuming no other orders) and initiates a bank transfer to her account via the local rail (in India, an IMPS or NEFT style bank credit; in the US, an ACH credit). A Payout object is created, moves from `pending` to `in_transit` to `paid`.
7. 1,600 rupees lands in Meera's bank. She matches it to Order #4471 by the reference.

Now the parts nobody sees. If the customer requests a refund on Monday, Stripe has to reverse the right legs: pull back part of Meera's 1,600, return CakeBazaar's fee (or not, per Arjun's policy), and refund the customer, all while the ledger still balances. If Meera's bank rejects the payout (wrong account number), the Payout flips to `failed`, the 1,600 returns to her Stripe balance, and it retries. If a customer files a chargeback three weeks later, after Meera already got paid, the money has to be clawed back from a future payout or from CakeBazaar. Every one of these is a new set of ledger entries, never an edit to the old ones.

The critical design fact: **the split is not the platform receiving money and then sending it out.** In a well-built Connect flow the platform's own bank account never holds Meera's 1,600. Stripe segregates the funds. That is what keeps Arjun from being a money transmitter, and it is enforced not by policy but by where the money physically sits (in Stripe's regulated flow of funds, tracked per connected account).

---

## 7. Under the hood, like the engineer

This is the heart. The visible feature is "split a payment." The engineering feature is "track money movement so that it is always conserved, always explainable, and always recoverable after a crash." That machine is a **double-entry ledger**, and it is worth understanding exactly why that shape and not any other.

### 7a. Why double-entry, and what the data structure actually is

The naive way to track balances is a single number per account. Meera has a `balance` column; when she earns 1,600, you run `UPDATE accounts SET balance = balance + 1600`. This is how a beginner builds a wallet. It is also how you lose money, because that single number has no memory of why it changed, and a mutable number can be corrupted by any bug, any race, any partial failure, with no way to detect it afterward.

Double-entry bookkeeping, a 500-year-old accounting idea, fixes this by refusing to ever store a balance as the source of truth. Instead every movement of money is recorded as a **transaction** made of at least two **entries**: one debit and one credit, and within a transaction the debits and credits must sum to zero. Money is never created or destroyed; it only moves from one account to another.

For the cake order, the single event "customer pays 2,000" is not one row. It is a balanced transaction that might look like:

- Debit: Customer's card funding source, 2,000
- Credit: CakeBazaar platform holding account, 2,000

Then the split "pay Meera, keep the fee" is another balanced transaction:

- Debit: CakeBazaar platform holding account, 1,600
- Credit: Meera's connected-account balance, 1,600

Notice the shape. Every transaction sums to zero. Add up every entry in the entire system and you get zero, forever. That is the invariant. Meera's balance is never stored as a number you trust; it is **derived** by summing all entries against her account. If that derived number and any cached copy ever disagree, you have found a bug before it becomes a lost rupee.

The data structure, then, is an **append-only log of immutable entries**, plus accounts that entries reference. Stripe's own Ledger, described in their engineering writing, is exactly this: "an immutable log of events" where "new information is simply added" to create "a chronological history of changes," and it "applies traditional accounting principles to validate money movement, ensuring that credits and debits balance out." Append-only means you never UPDATE and never DELETE. A refund is not an edit of the original charge; it is a new, later transaction that moves money back. The history is the truth. This is the same append-only, forward-only instinct as the Stripe idempotency and webhooks teardowns, but here it is load-bearing for correctness, not just retries.

Why immutable? Because an auditor's question is not "what is Meera's balance now" but "prove how it got there." An append-only log answers that by construction. And because immutability is what makes the system safe under concurrency and crashes: you can replay the log, you can never half-edit a row, and two processes appending different transactions never corrupt each other's data the way two `UPDATE balance` statements race.

Stripe also models each money-producing system as a **state machine** and lets the ledger record state transitions. A Payout is a tiny state machine: `pending -> in_transit -> paid`, or `pending -> in_transit -> failed`. A charge, a transfer, a refund each have their own states. Modeling them as explicit state machines means every legal transition is known, illegal transitions are rejected, and the current state is always the result of a known sequence of recorded events, not a mystery field someone wrote to.

### 7b. Matching versus ranking has no equal here, but there is a two-half split

Search features split into matching then ranking. A ledger has a different but equally important two-half split: **recording** versus **validating**.

Recording is the hot path: when money moves, append a balanced transaction, fast and correct. Validating is the second half, and Stripe built a whole **Data Quality platform** on top of the ledger to do it continuously. It watches metrics they name directly: **clearing** (does every expected movement actually reconcile against reality), **timeliness** (did it happen on time), and **completeness** (is anything missing). Their stated goal is striking: **"99.9999% explainability of money movement."** That is six nines of "we can explain where this money is." The recording half makes the entries; the validating half proves, continuously, that the entries match the real world.

The real-world part matters. The ledger's internal entries have to agree with what actually happened at the banks. That agreement is **reconciliation**: every day, pull the bank's own statement of what settled, and match it line by line against the ledger's entries. If Stripe's ledger says 1,600 went to Meera's bank but the bank rail reports 1,599 cleared, that one rupee gap is a reconciliation break, and it gets investigated, not ignored. Reconciliation is the outer loop that keeps an internally-consistent ledger honest against the messy outside world of banks, card networks, and currency conversion.

### 7c. Idempotency, because a debit cannot be un-rung

A bank debit is a foreign side effect you cannot roll back cleanly. If Stripe's server sends "pay Meera 1,600" to the bank and the network drops the response, did it happen? Retrying blindly might pay her twice. So every money-moving operation carries an **idempotency key**, and the ledger uses stable natural keys (this charge, this transfer, this attempt number) so that a retry of the same logical operation appends the same transaction once, not twice. This is the exact discipline from the idempotency teardown, but here the cost of getting it wrong is a duplicate payout, which is real money out the door.

### 7d. The scale story: 1,000, then 100,000, then 10 million-plus

What grows here is not a catalog to search. It is the **rate of money movements** and, critically, the **contention on the busiest accounts.** Follow it up the tiers.

**Tier one, about 1,000 transactions.** CakeBazaar's first month: a few thousand orders. A single Postgres database with an `entries` table and a `transactions` table is not just fine, it is correct and simple. Balances are computed by `SELECT SUM(amount) FROM entries WHERE account_id = ?`. You wrap each money movement in a database transaction so the two entries commit together or not at all (atomicity is exactly what a DB transaction gives you). Nothing breaks. Building anything fancier here would be a mistake; you would be adding sharding and queues to serve traffic a laptop handles. Ship the simple ledger and go bake.

**Tier two, about 100,000 transactions a day.** CakeBazaar is now in ten cities. Two things start to hurt.

First, computing a balance by summing every entry gets slow once an account has a hundred thousand entries. The fix is not to abandon double-entry; it is to keep periodic **balance snapshots** (materialized running balances) so you sum only the entries since the last snapshot, while the append-only log stays the source of truth you can always fall back to and replay. Read replicas absorb the dashboard reads (Meera refreshing her earnings) so they do not compete with writes.

Second, and this is the deep one, **contention on hot accounts.** Here is the trap that surprises everyone. In double-entry, every one of Meera's earnings also touches a shared account: CakeBazaar's platform holding account, or an internal "cash" account. Every single transaction across all 4,000 bakers debits or credits that one shared account. If you protect correctness by locking the accounts in each transaction (which you must, to keep balances consistent), then every transaction in the whole system is fighting for a lock on that one shared row.

The numbers on this are brutal and public. Analyses of accounting workloads (the TigerBeetle project documents this clearly) show that because accounts are **Pareto-distributed**, a few accounts carry the bulk of the transfers, and locking that hot account plus crossing the network to the app to compute the update can pin a naive system to as low as **76 transactions per second.** A relational database with stored procedures (moving the computation next to the data to cut the network round trips) gets to roughly **7,000 per second.** A purpose-built accounting database like TigerBeetle, which does nothing but debits, credits, and balances and batches thousands of transfers into one consensus write, reaches about **450,000 per second** in benchmarks. The lesson is not "use TigerBeetle." The lesson is: **the bottleneck in a ledger is not total volume, it is contention on the few hottest accounts,** and you survive the next tier by attacking that specific contention (batch many transfers into one write, move the balance math next to the data, avoid the app-server round trip per transfer).

**Tier three, 10 million-plus movements, Stripe's actual world.** Stripe processed about **1.4 trillion dollars** in total payment volume in 2024, roughly 1.3% of global GDP. Connect sits under a huge share of the internet's marketplaces. At this scale several walls appear at once.

- **Sharding by account.** The ledger is partitioned so that different accounts live on different shards, and a transaction's entries route to where those accounts live. This is the same self-routing shard-by-tenant trick the ledger keeps rediscovering: Notion shards by workspace, Stripe shards by account. It works because most transactions are local to a small set of accounts and do not need global coordination.
- **The hot shared account still has to be tamed.** You cannot shard away a single global cash account that every transaction touches. The answer is to **batch** many small transfers into fewer, larger ledger writes, and to represent hot internal accounts in ways that avoid a single serialized lock per transfer. Batching is the same move as picking many warehouse orders in one walk, or instancing many draws that share a pipeline: amortize a fixed serialization cost across many payloads.
- **Reconciliation becomes a massive parallel batch job**, not a nightly script. Matching billions of ledger entries against dozens of banks' and card networks' settlement files is itself a big-data pipeline, run offline, sharded, with the Data Quality platform flagging any break against the clearing, timeliness, and completeness metrics.
- **The freedom that makes it all possible: the debit is not a live user request.** When Stripe pays Meera on Tuesday, Meera is not sitting there waiting on a spinner. The payout is a scheduled, asynchronous job. Latency is nearly irrelevant; only correctness and throughput matter. That freedom is what lets Stripe batch, retry, jitter across the day, and spread load however the rails allow. It is the same "the user is asleep" luxury the Razorpay subscriptions teardown found: the most expensive money movement is the one nobody is watching in real time, so you are free to schedule it for correctness instead of speed.

The through-line the ledger keeps finding, again: push the expensive, whole-corpus, contended work off the hot path and spread it over time. Recording a balanced transaction is cheap and immediate. Validating, reconciling, snapshotting, paying out: offline, batched, sharded.

### 7e. What is fact and what is inference

Fact, from Stripe's own writing and docs: the ledger is an immutable append-only log; it applies double-entry principles so credits and debits balance; systems are modeled as state machines; there is a Data Quality platform measuring clearing, timeliness, and completeness aiming at 99.9999% explainability; Connect offers direct, destination, and separate charges with `application_fee_amount`, `transfer_data[destination]`, and `on_behalf_of`; card payouts settle on a rolling T+2 basis; 2024 volume was about 1.4 trillion dollars.

Inference, clearly labeled: the exact sharding key, the exact snapshot cadence, and the precise mechanism Stripe uses to defeat hot-account contention are not fully public. The 76 / 7,000 / 450,000 numbers are from the TigerBeetle project's analysis of the accounting-workload class, not a Stripe benchmark; they are cited to show the shape of the contention problem any large ledger faces, which is the well-grounded "this is how this class of problem is solved" version.

---

## 8. The retention and habit mechanic

This is not a feature you open every day like a Stories tray. Its retention works at a deeper level: it is a **switching cost built out of trust and money.**

For Meera, the loop is simple and powerful: she gets paid the exact promised amount on the exact promised day, over and over, and that reliability becomes a reflex. She stops checking. She stops worrying. She just bakes and gets paid. The day the number is wrong or the money is late is the day the reflex cracks, so the entire invisible ledger machine exists to make sure that day never comes. This is the same invisible-papercut retention as Spotify loudness normalization or Netflix's picture quality: the feature you only notice when it fails, and its job is to never fail. The metric it moves for the marketplace is **seller retention** (Meera does not leave CakeBazaar) and therefore marketplace liquidity.

For Arjun the platform, the retention is even stickier, and it is Stripe's real business. Once CakeBazaar has 4,000 sellers onboarded onto Stripe Connect, with all their bank details, tax IDs, payout histories, and reconciliation flowing through Stripe, moving to another provider means re-onboarding 4,000 sellers and re-verifying every identity. That is measured in months and lost sellers. The ledger's provable correctness is what lets Arjun sleep, and the onboarding graph is what locks him in. The metric here is **platform retention and net revenue retention:** as CakeBazaar grows, Stripe's take grows with it automatically, and the cost of leaving grows too.

The real observed example of the stakes: when India's regulator forced changes to how recurring and stored-money flows work (the same regulatory wall the Razorpay subscriptions teardown described), and more broadly whenever a platform accidentally holds other people's funds, the penalty is not a bug ticket, it is a licensing and legal crisis. Connect's fund segregation is a retention feature precisely because it keeps the platform out of that crisis. Arjun stays because leaving means rebuilding the one thing that keeps him legal.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus an embeddable runtime. The obvious reaction is "we do not move money, this is irrelevant." That is wrong. The ledger's core idea is not about money. It is about **state you must never corrupt, must be able to explain, and must be able to recover after a crash.** A shader graph's edit history and a running effect's GPU state are exactly that kind of state.

Here is the concrete, actionable lesson, biased toward scalability and performance.

**Make the graph's edit history an immutable, append-only, double-entry-style event log, not a mutable in-place document.** Most node editors mutate the graph object directly: the user drags a slider, you overwrite `node.frequency = 0.7`. That is the single-mutable-balance mistake. When something goes wrong (an undo that half-applies, a collaborative edit that races, a crash mid-edit, a compile that produces a broken artifact), you cannot explain how the graph got into its state, and you cannot cleanly recover. Instead, record every edit as an immutable event appended to a log: `SetParam(node=noise1, key=frequency, from=0.5, to=0.7, at=t)`. The current graph is the **derived** result of replaying the log, exactly as Meera's balance is derived from her entries. This buys you four things at once, and each maps to a ledger property:

- **Undo/redo and time travel for free**, because history is the truth and you can replay to any point (the auditor's "prove how it got here").
- **Deterministic, crash-safe recovery.** After a crash, replay the log to the last consistent point. You never find a half-written graph, the same way an append-only ledger never has a half-edited row.
- **Correct collaboration.** Two users editing the same graph append events; you merge event streams instead of racing on a mutable object, the same reason two ledger transactions never corrupt each other.
- **A validation invariant, borrowed from double-entry.** Give the graph a conservation law it must always satisfy and check it continuously, the way the ledger checks that every transaction sums to zero. For a shader graph the natural invariant is a **compute-budget ledger**: every node "spends" an estimated GPU cost, the frame has a fixed budget, and the sum of allocated node costs must never exceed the frame budget. Record each allocation as a balanced entry (this pass debits the frame's budget, the budget account credits it), and run a background check that the books balance. When they do not, you have found an effect that will blow the frame budget **before** it ships to a customer's page, not after their phone drops to 12fps. That is Stripe's Data Quality platform (clearing, completeness, six-nines explainability) reborn as "explain every GPU millisecond this frame spends."

And copy the scale discipline exactly. The recording path (append an edit event) must be cheap and immediate, so the editor feels instant. The expensive work (full recompile, budget reconciliation across the whole graph, snapshotting the derived state so you do not replay from the beginning of time) goes **offline and batched**, off the hot path, the same way payouts and reconciliation run asynchronously while the user is not watching. Keep periodic snapshots of the derived graph so replay is bounded, exactly as the ledger snapshots balances so it does not re-sum a million entries. And watch for your own hot-account contention: if every edit event touches one global "graph version" counter or one shared root node, that single point of coordination becomes your 76-transactions-per-second bottleneck, so batch edits and avoid a global lock per keystroke.

One line: money teaches the discipline that GPU state needs. Never store trusted state as a mutable number; store an immutable append-only log of balanced events, derive the current state from it, snapshot for speed, reconcile against a conservation law offline, and keep the hot path a cheap append. Do that for the shader graph's edit history and its per-frame compute budget, and Rare.lab gets undo, crash recovery, correct collaboration, and a pre-ship guarantee that no effect blows the frame budget, all from one idea that Stripe uses to keep 1.4 trillion dollars honest to six nines.

---

## Sources

- Stripe engineering: "Ledger: Stripe's system for tracking and validating money movement." https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement (immutable append-only log, double-entry validation, state machines, Data Quality platform metrics of clearing/timeliness/completeness, 99.9999% explainability).
- Density Labs deep dive on Stripe Ledger. https://densitylabs.io/blog/building-trust-how-stripe-ensures-financial-accuracy-with-ledger/
- Fintech Wrapup deep dive on Stripe Ledger. https://www.fintechwrapup.com/p/deep-dive-ledger-stripes-system-for
- Stripe docs: "Understand how charges work in a Connect integration" (direct, destination, separate charges and transfers). https://docs.stripe.com/connect/charges
- Stripe docs: "Create destination charges" (application_fee_amount, transfer_data[destination], on_behalf_of, cross-border). https://docs.stripe.com/connect/destination-charges
- Stripe docs: "Accept a payment using separate charges and transfers." https://docs.stripe.com/connect/marketplace/tasks/accept-payment/separate-charges-and-transfers
- Stripe docs: "Funds segregation for separate charges and transfers." https://docs.stripe.com/connect/funds-segregation
- Stripe docs: "Money movement timelines" and payout schedules (rolling T+2). https://docs.stripe.com/treasury/connect/money-movement/timelines and https://support.stripe.com/questions/payout-schedules-faq
- Stripe newsroom: 2024 total payment volume ~$1.4 trillion, ~1.3% of global GDP. https://stripe.com/newsroom/news/stripe-2024-update
- TigerBeetle: "Debit/Credit: The Schema for OLTP" and ARCHITECTURE.md (hot-account contention, Pareto distribution, ~76 TPS naive, ~7k relational stored procedures, ~450k TigerBeetle). https://docs.tigerbeetle.com/concepts/debit-credit/ and https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/ARCHITECTURE.md
- Modern Treasury: "Enforcing Immutability in your Double-Entry Ledger." https://www.moderntreasury.com/journal/enforcing-immutability-in-your-double-entry-ledger
</content>
</invoke>
