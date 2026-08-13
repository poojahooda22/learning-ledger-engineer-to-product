# References: Razorpay Magic Checkout + RTO risk engine (2026-08-13)

Saved keepers for the Magic Checkout teardown. Note: razorpay.com was blocked by the
network egress proxy during this run, so primary Razorpay pages were captured via search
result summaries rather than full-page fetches. URLs kept for later direct reading.

## Razorpay primary (product + engineering)
- One-Click Checkout 2.0 / Magic Checkout: https://razorpay.com/blog/razorpay-one-click-checkout-2-0-magic/
- Magic Checkout's New Single-Page Checkout (conversion + RTO numbers): https://razorpay.com/blog/magic-checkouts-new-single-page-checkout/
- MagicX for Shopify Plus (up to 50% RTO reduction, proprietary AI/ML model): https://razorpay.com/blog/razorpay-magicx-for-shopify-plus/
- RTO Overview (docs): https://razorpay.com/docs/payments/magic-checkout/rto-analytics/overview/
- RTO Insights (docs): https://razorpay.com/docs/payments/magic-checkout/rto-analytics/rto-insights/
- Introducing RTO Analytics Dashboard: https://razorpay.com/blog/introducing-rto-analytics-dashboard/
- Footwear brand case study: https://razorpay.com/blog/footwear-brand-improves-its-checkout-experience/

## Thirdwatch (the acquired RTO/fraud engine, Mitra)
- Thirdwatch has Merged with Magic Checkout: https://razorpay.com/blog/thirdwatch-has-merged-with-magic-checkout/
- Razorpay Announces First Acquisition - Thirdwatch (Mitra, 200+ params, real-time trust score, device fingerprinting, location profiles, IP verification): https://razorpay.com/blog/thirdwatch-acquisition-rto-fraud-ecommerce/
- Razorpay Tech, Using ML to Detect Fraud (Thirdwatch): https://razorpay.com/blog/detect-fraud-using-ml-ai-thirdwatch/
- The Paypers, Razorpay buys Thirdwatch (Aug 2019): https://thepaypers.com/mergers-aquisitions-and-investments/news/razorpay-buys-real-time-fraud-prevention-platform-thirdwatch

## Industry context on RTO in Indian D2C (fact-check the ranges)
- HillTeck, Reduce RTO for Indian D2C 2026 (COD 30-40% vs prepaid 5-10% RTO): https://www.hillteck.com/blog/reduce-rto-ecommerce-india.html
- bePragma, Pincode clusters with high RTO risk: https://www.bepragma.ai/blogs/using-data-to-identify-pincode-clusters-with-high-rto-risk
- Egrow, Complete Guide to Reducing RTO in COD E-commerce 2026: https://www.egrow.com/en/blog/the-complete-guide-to-reducing-return-to-origin-rto-in-cod-e-commerce-2026

## Key facts captured
- Network: 8M+ merchants, 100M+ registered shoppers; address autofill improves with adoption (network effect).
- Checkout time: ~9 seconds via prefill. Conversion lift: 20-40 bps (single-page), 12-17% improvement; RTO reduction 21-26% (single-page brands), up to 50% by dynamically disabling COD for high-risk shoppers.
- Thirdwatch: acquired Aug 2019, founded 2016 Gurugram (Adarsh Jain, Shashank Agarwal); Mitra engine, 200+ parameters/transaction, real-time trust score, network learning effect.
- Fraud + RTO ~4-5% of Indian e-commerce orders, $5B+ losses (Razorpay/Thirdwatch estimate); Razorpay CEO estimated 30-40% fraud reduction from the acquisition.
- RTO actions: block COD, nudge prepaid, partial COD, differential COD fees, per merchant-configured thresholds. RTO Protection reimburses verified RTO losses on model-cleared orders.
- Class-of-problem inference: gradient-boosted trees (XGBoost/LightGBM) over pincode RTO history, customer order history, COD value vs category average, device/session, address quality; delayed + imbalanced labels; threshold is the product decision.
</content>
