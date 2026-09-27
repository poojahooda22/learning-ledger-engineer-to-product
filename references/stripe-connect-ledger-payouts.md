# References: Stripe Connect and the money-movement ledger (2026-09-27 teardown)

Keeper links for the Connect + double-entry ledger + payouts teardown. During this run the
network egress proxy blocked direct fetches of stripe.dev, docs.stripe.com, and several other
domains, so specifics were pulled from search-result summaries of those pages. Links kept for
future re-fetch when access allows.

## Primary (Stripe)

- Ledger: Stripe's system for tracking and validating money movement (Stripe engineering blog)
  https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement
  The core source. Immutable append-only log of events; models data-producing systems as state
  machines; applies double-entry principles so credits and debits balance; Data Quality platform
  measuring clearing, timeliness, completeness; goal of "99.9999% explainability of money movement";
  built across Stripe's Global Payments and Treasury Network (GPTN).

- Understand how charges work in a Connect integration
  https://docs.stripe.com/connect/charges
  Direct charges vs destination charges vs separate charges and transfers; who is merchant of record;
  which balance the money touches first.

- Create destination charges
  https://docs.stripe.com/connect/destination-charges
  transfer_data[destination], application_fee_amount vs transfer_data[amount], on_behalf_of for
  cross-region settlement, presentment-to-settlement currency conversion.

- Accept a payment using separate charges and transfers
  https://docs.stripe.com/connect/marketplace/tasks/accept-payment/separate-charges-and-transfers

- Funds segregation for separate charges and transfers
  https://docs.stripe.com/connect/funds-segregation
  Why the platform never holds the seller's funds (money-transmitter avoidance).

- Money movement timelines / payout schedules FAQ (rolling T+2 for cards, pending vs available)
  https://docs.stripe.com/treasury/connect/money-movement/timelines
  https://support.stripe.com/questions/payout-schedules-faq

- Stripe 2024 update: ~$1.4T total payment volume, ~1.3% of global GDP
  https://stripe.com/newsroom/news/stripe-2024-update

## The class-of-problem sources (double-entry at scale, hot-account contention)

- TigerBeetle: Debit/Credit: The Schema for OLTP
  https://docs.tigerbeetle.com/concepts/debit-credit/

- TigerBeetle ARCHITECTURE.md (hot accounts, Pareto distribution, contention numbers)
  https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/ARCHITECTURE.md
  Naive locking pins to ~76 TPS on a hot account; relational + stored procedures ~7k TPS;
  TigerBeetle ~450k TPS by batching transfers into one consensus write.

- Modern Treasury: Enforcing Immutability in your Double-Entry Ledger
  https://www.moderntreasury.com/journal/enforcing-immutability-in-your-double-entry-ledger

## Secondary deep-dives (Stripe Ledger)

- Density Labs: Building Trust: How Stripe Ensures Financial Accuracy with Ledger
  https://densitylabs.io/blog/building-trust-how-stripe-ensures-financial-accuracy-with-ledger/

- Fintech Wrapup: Deep Dive on Stripe's Ledger
  https://www.fintechwrapup.com/p/deep-dive-ledger-stripes-system-for
</content>
