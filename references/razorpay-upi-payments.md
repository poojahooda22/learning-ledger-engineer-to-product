# References: Razorpay UPI payments (intent vs collect, async switch, deemed success, reconciliation)

Saved 2026-08-01 for the teardown at `product-teardowns/2026-08-01-razorpay-upi-payments.md`.

## Scale numbers (NPCI, fact)
- July 2026: 23.66 billion UPI transactions in one month, worth ₹29.88 trillion, ~737 million/day.
  - Business Standard: https://www.business-standard.com/finance/news/upi-transactions-hit-record-23-66-billion-in-july-value-rs-29-88-trillion-126080100548_1.html
- FY25-26 annual: 24,162 crore transactions, +30% YoY; May 2026 record 23.2B / ₹29.9T.
  - ANI: https://www.aninews.in/news/business/upi-hits-new-high-in-may-2026-with-232-billion-transactions-worth-rs-299-trillion-npci-data-shows20260602155337/
- Engineered for ~100,000 API requests/sec, ~600M users daily; <2s end-to-end target; stateless switch, sharding+replication, Avro/Protobuf.
  - https://medium.com/@ankit.gochhayat90/the-data-behind-upi-understanding-the-architecture-of-real-time-financial-pipelines-e8d5c6f89b16
  - https://medium.com/@himanshusingour7/npci-upi-handles-around-600-million-users-daily-and-up-to-100-000-api-requests-per-second-3f09933ac139

## Intent vs collect (Razorpay, fact)
- Intent flow ~92-95% success, +10-15% over collect; Android intent / iOS deep link, pre-filled, no VPA to type.
  - https://razorpay.com/blog/upi-intent-vs-collect-success-rates/
  - https://razorpay.com/blog/upi-collect-vs-intent-flow-business-guide/
- Deep link shape: `upi://pay?pa=<payeeVPA>&pn=<name>&am=<amount>&tr=<txnref>&cu=INR`.

## Deemed success, TCC, URCS (fact)
- Beneficiary-bank timeout -> NPCI treats as deemed approved, settles as approved. TCC-102 = credited online but no online response; TCC-103 = not credited online, credit post-recon.
  - RMA Bhutan UPI Global QR 2024 guidelines (PDF): https://www.rma.org.bt/media/Laws_By_Laws/Guidelines%20for%20UPI%20Global%20QR%202024.pdf
- RazorpayX on deemed success / payout states / NPCI recon: https://razorpay.com/blog/business-banking/payout-processing-imps-upi-transactions-deemed-success-npci
- URCS auto accept/reject chargebacks on TCC/RET in next settlement cycle, effective Feb 15 2025:
  - https://www.business-standard.com/finance/personal-finance/new-upi-rule-on-automatic-acceptance-rejection-of-chargebacks-from-feb-15-125021200820_1.html

## API primitives (NPCI, fact)
- ReqPay/RespPay (money), ReqAuthDetails/RespAuthDetails (VPA resolution), the mapper (`upinumber@mapper.npci`), 12-digit RRN, duplicate-RRN rejection.
  - NPCI API Descriptions (PDF): https://s3-ap-southeast-1.amazonaws.com/he-public-data/NPCI%20API%20Descriptionsb9bceb7.pdf
  - NPCI UPI Error and Response Codes v2.9 (PDF): https://dth95m2xtyv8v.cloudfront.net/tesseract/assets/upi-tpap-sdk/UPI_Error_and_Response_Codes_2_9-HHLrJ.pdf
  - UPI Number mapper API (PayU docs): https://docs.payu.in/reference/upi-number-mapper-api

## Gateway behaviour (documented, applied to Razorpay as grounded inference)
- Collect expiry default 3 minutes; webhook callbacks can lag up to ~15 min; get-transaction-status API for pending->success/failure flips; idempotency via unique merchant_txn_id, reject duplicates.
  - https://webhooks.isac.happay.in/upi-webhooks/upi-payments-webhooks

## Fact vs inference boundary
- Fact: everything NPCI-level (ReqPay, mapper, RRN, deemed success, TCC codes, URCS, scale numbers) and Razorpay's public blog claims (intent vs collect success rates).
- Inference (grounded): Razorpay's own internal payment state machine, reconciliation poller, and queue design. Modeled from documented gateway behaviour (status API, webhook lag, idempotency), not Razorpay's private code.
