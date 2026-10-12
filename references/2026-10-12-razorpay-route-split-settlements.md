# References: Razorpay Route (split payments and settlements)

Saved 2026-10-12 for the Razorpay Route teardown.

## Primary (Razorpay official)

- Route product page: https://razorpay.com/route/
- About Route (docs): https://razorpay.com/docs/route/
- Route docs (payments): https://razorpay.com/docs/payments/route/
- Linked Accounts: https://razorpay.com/docs/payments/route/linked-account/
- Transfers API, create from a payment (`POST /v1/payments/:id/transfers`; transfers array fields: account, amount, currency, notes, linked_account_notes; sum of transfers + fees cannot exceed captured amount): https://razorpay.com/docs/api/payments/route/transfers
- Direct transfers API (`POST /v1/transfers`, from your balance to a linked account, on-demand feature, INR only): https://razorpay.com/docs/api/payments/route/transfers/direct-transfers/
- Modify settlement hold (`PATCH /v1/transfers/:id` with on_hold: true and on_hold_until Unix timestamp): https://razorpay.com/docs/api/payments/route/modify-settlement-hold.md
- Transfers entity (settlement_status: pending | on_hold | settled; status reversed / partially_reversed; amount_reversed field): https://razorpay.com/docs/us/api/payments/route/transfers-entity
- Route blog with the HomeClap home-services split example (500 plumbing + 1,000 cleaning): https://razorpay.com/blog/effortlessly-split-and-transfer-your-payments-with-razorpay-route/
- Settlements docs (default domestic T+2 working days, no weekend settlements): https://razorpay.com/docs/payments/settlements.md
- Instant Settlements blog (T+0, 24x7, small fee, daily withdrawable cap; Same-Day Settlements option): https://razorpay.com/blog/instant-settlement-payment-gateway/

## Scale numbers

- Razorpay annualized transaction volume: 180 billion USD plus (Razorpay careers/newsroom).
- UPI Switch (with Airtel Payments Bank, Apr 2024): built for up to 10,000 TPS, 99.99% uptime target.
- Optimizer (AI payment routing, 2024): built for 5,000+ online TPS; India averages ~300 online payment TPS.

## Grounded inference (labeled as such in the report)

- Internal ledger is append-only double-entry; reversals are compensating entries, not edits. Pattern source: Modern Treasury "How to Scale a Ledger": https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-i
- Scaling moves (shard ledger by merchant/account id, sub-shard hot linked-account counters, batch settlement off the hot path, per-bank token buckets/backpressure, nightly reconcile by replaying the immutable log) are inferred from the patterns documented in the Stripe ledger, UPI URCS, and Razorpay subscriptions teardowns in this repo, not from a published Razorpay architecture post.
- Payout rails (NEFT batches, IMPS/UPI for instant) are general India-payments knowledge applied to Route; Razorpay publicly confirms T+2 default plus Instant and Same-Day add-ons, not the rail-by-rail routing.

## Regulatory

- Route funds sit in Razorpay escrow (how a marketplace avoids being a payment aggregator itself). Direct Settlement is an add-on where funds skip escrow.
- RBI Payment Aggregator guidelines (Sept 2025) set turnover and payer-to-payee transparency criteria for certain Route modes.
