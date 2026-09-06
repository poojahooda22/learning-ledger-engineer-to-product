# References: Razorpay Subscriptions and UPI AutoPay (2026-09-06)

Keeper links for the recurring-payments / e-mandate teardown.

## The rail (confirmed, primary)

- NPCI, UPI AutoPay (launched at the Global Fintech Festival, 22 July 2020):
  https://www.npci.org.in/what-we-do/upi/upi-autopay
- Razorpay Docs, Subscriptions overview:
  https://razorpay.com/docs/payments/subscriptions/
- Razorpay Docs, Subscriptions States (the confirmed state machine: created,
  authenticated, active, pending, halted, cancelled, completed, expired):
  https://razorpay.com/docs/payments/subscriptions/states/
- Razorpay Docs, UPI AutoPay S2S recurring integration (customer + order +
  authorization payment -> token_id, then charge by token):
  https://razorpay.com/docs/payments/payment-gateway/s2s-integration/recurring-payments/upi/
- Razorpay Blog, "Introducing UPI AutoPay on Razorpay Subscriptions":
  https://razorpay.com/blog/what-is-upi-autopay-recurring-payments-razorpay-subscriptions/
- Razorpay Blog, "UPI AutoPay with Intelligent Revenue-Protect" (retry / churn
  recovery at registration, debit, and churn):
  https://razorpay.com/blog/upi-autopay-with-intelligent-revenue-protect/
- Razorpay Blog, "What is a UPI Mandate":
  https://razorpay.com/blog/what-is-upi-mandate/
- PayU Docs, Pre-Debit Notification API and Recurring Payment Transaction API
  (the mandatory sequence: check mandate status, send pre-debit notification,
  then execute the recurring transaction; execution sequence numbers):
  https://docs.payu.in/reference/pre_debit_notification_api

## The regulation (confirmed, primary/secondary)

- NovoJuris, "RBI Guidelines on E-mandates for recurring transactions"
  (framework summary, AFA at registration, 24h pre-debit notice):
  https://www.novojuris.com/thought-leadership/rbi-guidelines-on-e-mandates-for-recurring-transaction.html
- Business Standard, "RBI enhances limit for e-mandates on credit/debit cards to
  Rs 15,000" (2022-06-08):
  https://www.business-standard.com/article/finance/new-e-mandate-guidelines-rbi-enhances-limit-for-e-mandates-on-credit-debit-cards-to-rs-15-000-122060800417_1.html
- Business Standard, "RBI raises limit of e-mandates for recurring online
  transactions to Rs 1 lakh" (2023-12-08):
  https://www.business-standard.com/economy/interviews/rbi-raises-limit-of-e-mandates-for-recurring-online-transactions-to-1-lakh-123120801110_1.html

## The disruption (context, secondary)

- TechCrunch, "India's stringent recurring payments rule goes into effect"
  (2021-09-30): https://techcrunch.com/2021/09/30/india-recurring-payments-rbi/
- Medianama, "Netflix starts accepting UPI AutoPay as RBI auto-debit regulations
  for cards loom" (2021-09):
  https://www.medianama.com/2021/09/223-netflix-upi-autopay-rbi-regulations/
- Medianama, "Google halts recurring payment options on Play Store" (2021-05):
  https://www.medianama.com/2021/05/223-google-recurring-payments-play-store/

## The engineering primitives (for the scheduler layer, standard references)

- PostgreSQL docs, SELECT ... FOR UPDATE SKIP LOCKED (the atomic batch-claim
  primitive that lets a worker pool pull due rows without contending):
  https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
- Redis docs, Sorted Sets (ZADD/ZRANGEBYSCORE as an externalized min-heap /
  delay queue keyed by fire timestamp):
  https://redis.io/docs/latest/develop/data-types/sorted-sets/

## Notes on confidence

- Confirmed: the UPI AutoPay rail mechanics, the RBI caps and pre-debit rule, the
  Razorpay subscription state names and transitions, the Oct 2021 disruption.
- Inference (clearly labeled in the report): Razorpay's internal scheduler
  storage, sharding, time-bucketing, and per-bank rate limiting. These are the
  standard way this class of problem is solved at scale, not published Razorpay
  internals.
