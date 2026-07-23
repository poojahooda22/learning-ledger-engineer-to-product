# References: Stripe Ledger, Balance, and payouts (money-movement)

Saved 2026-07-23 for the Stripe Balance/payouts/ledger teardown.

## Primary (Stripe)
- Ledger: Stripe's system for tracking and validating money movement (Stripe engineering blog). The core primary source: immutable auditable log as the system of record, double-entry validation (credits and debits balance out), producer systems modeled as state machines emitting "logical fund flows" between accounts, 5 billion events/day, 99.99% of dollar volume ingested and verified within four days, and the Data Quality (DQ) platform for automated discrepancy detection.
  https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement
- Balance object API reference (fields: available, pending, connect_reserved, instant_available).
  https://docs.stripe.com/api/balance
- Balance Transaction object API reference (types: charge, refund, payout, stripe_fee, adjustment, reserve_*, etc.; fields amount / fee / net; status available vs pending).
  https://docs.stripe.com/api/balance_transactions
- Payouts FAQ and payout schedule vs settlement timing (T plus X; daily/weekly/monthly/manual schedules).
  https://support.stripe.com/questions/payouts-faq
- Instant Payouts (paid feature, ~1.5% fee to skip the pending-to-available wait).
  https://docs.stripe.com/payouts/instant-payouts-with-advance-funding

## Deep dives (secondary, for framing)
- Sam Boboev / Fintech Wrap Up deep dive on Stripe Ledger.
  https://www.fintechwrapup.com/p/deep-dive-ledger-stripes-system-for
- Density Labs, "Building Trust: How Stripe Ensures Financial Accuracy with Ledger."
  https://densitylabs.io/blog/building-trust-how-stripe-ensures-financial-accuracy-with-ledger/

## General ledger engineering (grounded inference for scale + immutability sections)
- Modern Treasury Journal, "How to Scale a Ledger" series: immutability and double-entry as scalability guarantees, hot-account contention (shared SYSTEM_FUNDING row), balance caching for read-after-write + batched writes, sub-account sharding, async queueing, cells-based architecture, 5,000+ QPS ledgers.
  https://www.moderntreasury.com/resources/how-to-scale-a-ledger
- Formance blog, double-entry accounting for engineers and ledger integrity: balances derived (sum of postings) not stored, append-only log, corrections as compensating entries, enforce balanced writes at the API boundary.
  https://www.formance.com/blog/engineering/double-entry-accounting-for-engineers-building-financial-products

## Key facts to reuse
- Balance is derived (sum of an account's entries), never a mutable stored column. Corrections are new compensating entries, never edits or deletes.
- Double-entry invariant: every transaction's entries sum to zero. Money only moves between accounts.
- Immutability is what enables safe sharding, caching, replay, and continuous re-derivation of any balance.
- Hot-account contention is the same celebrity/hot-key problem as Zepto's last-carton counter and Razorpay's per-terminal counter (repo Lesson 16); fix by sub-account sharding + async queueing.
- Idempotency keys on ingestion give exactly-once posting (ties to the 2026-06-20 Stripe idempotency teardown).
- US card settlement standard is roughly T plus 2 business days; payout schedule (when payouts fire) is separate from settlement timing (when funds become available).
