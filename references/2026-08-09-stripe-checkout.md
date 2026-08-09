# References: Stripe Checkout (2026-08-09)

Keeper links for the Stripe Checkout teardown (hosted pay page, payment-method
ranking, PaymentIntent state machine, 3D Secure).

## PaymentIntent state machine and confirmation
- The Payment Intents API (states and lifecycle): https://docs.stripe.com/payments/payment-intents
- Finalize payments on the server (handleNextAction, requires_action handling): https://docs.stripe.com/payments/finalize-payments-on-the-server
- stripe.handleNextAction (Stripe.js reference): https://docs.stripe.com/js/payment_intents/handle_next_action
- stripe.confirmPayment (Stripe.js, redirect: if_required): https://docs.stripe.com/js/payment_intents/confirm_payment

## 3D Secure and SCA
- Authenticate with 3D Secure (frictionless vs challenge, Directory Server / ACS): https://docs.stripe.com/payments/3d-secure/authentication-flow
- Strong Customer Authentication readiness (PSD2, 14 Sept 2019): https://docs.stripe.com/strong-customer-authentication
- Designing card payment flows for SCA (guide): https://stripe.com/guides/sca-payment-flows
- Surprising findings from our analysis of 3DS transactions in the US (blog): https://stripe.com/au/blog/surprising-findings-from-our-analysis-of-3ds-transactions-in-the-us

## Dynamic / adaptive payment methods (the ranker)
- Dynamic payment methods (docs): https://docs.stripe.com/payments/payment-methods/dynamic-payment-methods
- How Stripe is using AI to create personalized checkout experiences (blog): https://stripe.com/blog/stripe-ai-personalized-checkout-experiences
- Optimizing payments at scale: AI across the payment lifecycle (guide): https://stripe.com/guides/optimizing-payments-at-scale

## Scale figures
- Stripe Newsroom, BFCM 2023 ($18.6B, 300M+ transactions): https://stripe.com/newsroom/news/bfcm2023
- StarTree, Stripe's BFCM journey with Apache Pinot (real-time dashboard, 27,395 req/s peak, 99.999% uptime): https://startree.ai/user-stories/stripe-journey-to-18-b-of-transactions-with-apache-pinot/

## Key numbers to remember
- 11.9% average revenue lift from the Optimized Checkout Suite (dynamic ordering + saved credentials + Adaptive Pricing).
- Up to 15% conversion drop from showing one geographically irrelevant payment method.
- 17.8% cross-border revenue lift from Adaptive Pricing.
- PaymentIntent states: requires_payment_method, requires_confirmation, requires_action, processing, requires_capture, succeeded, canceled.
- SCA / PSD2 in force since 14 September 2019.
- 2023: $1T total payment volume; BFCM 2023 peak 27,395 API requests/sec at 99.999% uptime.
