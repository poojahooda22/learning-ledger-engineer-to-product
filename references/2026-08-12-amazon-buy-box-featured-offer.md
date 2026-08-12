# References: Amazon Buy Box / Featured Offer (2026-08-12)

Keeper links for the 2026-08-12 teardown of Amazon's Featured Offer (Buy Box).

## Primary / official
- Amazon Seller Central, "Featured Offer" (formerly Buy Box): eligibility, landed price, delivery, and performance factors. https://sell.amazon.com/blog/buy-box-featured-offer
- US Patent 8,630,923, "Virtual shelf with single-product choice and automatic multiple-vendor selection." https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/8630923

## Audits and measured data
- The Markup, "When Amazon Takes the Buy Box, It Doesn't Give It Up" (Oct 14, 2021). Audit of 3,492 popular products. Amazon won ~40% of featured offers; next seller ~0.5%; winner changed for fewer than 3 in 10 products over 12 weeks. https://themarkup.org/amazons-advantage/2021/10/14/when-amazon-takes-the-buy-box-it-doesnt-give-it-up
- Profitero, "Amazon.com Makes More Than 2.5 Million Price Changes Every Day" (Dec 2013). 2.5M/day, up 10x from 269,113/day a year earlier; Walmart 54,633 and Best Buy 52,956 changes in a month. https://www.profitero.com/blog/2013/12/profitero-reveals-that-amazon-com-makes-more-than-2-5-million-price-changes-every-day
- "Antitrust, Amazon, and Algorithmic Auditing" (arXiv 2403.18623). Academic audit of featured-offer selection signals. https://arxiv.org/pdf/2403.18623

## Recent changes (2023-2025)
- GoDataFeed, "Amazon sunset the Buy Box eligibility gate: feed architecture became a continuous ranking input." Boolean eligibility gate to continuous unified ranking. https://www.godatafeed.com/blog/amazon-buy-box-eligibility-changes
- Sentrykit, "Amazon Featured Offer Went Fulfillment-Neutral in 2025." The ~Nov 1, 2025 removal of FBA's built-in weight; delivery-speed emphasis; FBA still wins 3-5x more often. https://sentrykit.com/blog/amazon-featured-offer-fulfillment-neutral-2026/

## Practitioner explainers (factors, category weights, ~82% estimate)
- Feedvisor, "Amazon Buy Box: How to Win the Featured Offer." https://feedvisor.com/university/amazon-buy-box/

## Key numbers pulled into the teardown
- ~82% of Amazon sales flow through the Featured Offer (industry estimate, not Amazon-confirmed).
- Amazon's own retail arm won ~40% of featured offers in The Markup's sample; next seller ~0.5%.
- Winner rotation: ~7 in 10 products changed winner over 6 weeks (Northeastern, 2016); fewer than 3 in 10 over 12 weeks (The Markup).
- Landed price = item price + shipping + tax. Near-ties within ~5% of lowest landed price rotate for a share.
- Order Defect Rate must stay under 1% for eligibility. Professional account required; individual sellers never eligible.
- ~1.9M active third-party sellers; 60%+ use repricers adjusting every 5-15 minutes; ~2.5M+ price changes/day across the marketplace.
- 2023: Buy Box renamed Featured Offer. ~Nov 1, 2025: went fulfillment-neutral (FBA's structural advantage removed).

Note: The exact Featured Offer scoring is a trade secret. Data structures, sharding, and the rotation mechanism in the teardown are labeled inference, grounded in how this class of problem is solved.
