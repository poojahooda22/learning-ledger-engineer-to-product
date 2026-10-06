# References: Stripe 3D Secure (3DS2) authentication

Keeper links for the 2026-10-06 teardown of Stripe's 3D Secure cardholder authentication.

## Stripe primary

- Stripe, "Authenticate with 3D Secure" (authentication flow): https://docs.stripe.com/payments/3d-secure/authentication-flow
- Stripe Support, "Frictionless flow for charges created using 3D Secure 2 (3DS2)": https://support.stripe.com/questions/frictionless-flow-for-charges-created-using-3d-secure-2-3ds2
- Stripe, "3D Secure 2" guide: https://stripe.com/en-gb-us/guides/3d-secure-2

## Protocol (EMVCo 3DS2: 3DS Server, Directory Server, ACS; AReq/ARes/CReq/CRes)

- Gravitee, "3D Secure (3DS): Protocol, Flows, and Operational Integration": https://gravitee.io/corpus/gen-694/payment-processor/3d-secure.html
- Netcetera, "3-D Secure Directory Server" (directory routing role): https://www.netcetera.com/payments/Payment-networks/3-D-Secure-Directory-Server.html
- Netcetera 3DS Server documentation: https://3dss.netcetera.com/3dsserver/doc/current/
- Mastercard, "Top 10 Things to Know About 3DS" (PDF): https://www.mastercard.com/content/dam/public/mastercardcom/globalrisk/pdf/Top-10-Things-to-Know-About-3DS.pdf

## Device fingerprinting (the 3DS Method URL, hidden iframe)

- Sardine, "3D Secure Method URL": https://www.sardine.ai/blog/3d-secure-method-url

## 3DS1 vs 3DS2 data elements (~8-15 fields vs 100+)

- Checkout.com, "The sunsetting of 3DS1 and how businesses can prepare": https://www.checkout.com/blog/post/the-sunsetting-of-3ds1-and-how-businesses-can-prepare
- Convesio knowledge base, "EMV 3D Secure": https://convesio.com/knowledgebase/article/emv-3d-secure/

## PSD2 Strong Customer Authentication exemptions (low-value, TRA)

- PCI Proxy, "SCA exemptions under PSD2: a practical guide for payment teams": https://www.pci-proxy.com/blog-posts/sca-exemptions-under-psd2-a-practical-guide-for-payment-teams
- SEON, "TRA Exemptions in PSD2": https://seon.io/resources/tra-exemptions/

## India (RBI additional factor authentication)

- Business Standard, "RBI mandates stronger two-factor authentication in new guidelines": https://www.business-standard.com/finance/news/rbi-two-factor-authentication-digital-payments-guidelines-2026-125092501154_1.html
- RBI draft, "Framework on alternative authentication mechanisms for digital payment transactions": https://website.rbi.org.in/web/rbi/-/notifications/framework-on-alternative-authentication-mechanisms-for-digital-payment-transactions-draft

## Research

- "Investigation of 3-D Secure's Model for Fraud Detection" (arXiv 2009.12390): https://arxiv.org/pdf/2009.12390

## Notes

- Confirmed: the 3-domain architecture and message flow; frictionless (Y) vs challenge (C); 3DS2 carrying 100+ data elements vs ~8-15 in 3DS1; the 3DS Method hidden-iframe device fingerprinting; Stripe PaymentIntent lifecycle with a `requires_action` state; PSD2 low-value (30 EUR, reset after 5 / 100 EUR) and TRA (ceiling 500 EUR) exemptions; RBI additional-factor requirement in India.
- Inference (labeled in the report): the exact learned risk model each issuer's ACS runs; Stripe's internal 3DS Server sharding; the precise per-payment exemption cost model.
