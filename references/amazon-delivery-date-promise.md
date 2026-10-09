# References: Amazon delivery date promise (the "Arriving Thursday" line)

Saved 2026-10-09 for the teardown at `product-teardowns/2026-10-09-amazon-delivery-date-promise.md`.

## Primary and official (CONFIRMED)

- Amazon Science, "How Amazon reworked its fulfillment network to meet customer demand". Regionalization into 8 interconnected, largely self-sufficient US regions.
  https://www.amazon.science/news-and-features/how-amazon-reworked-its-fulfillment-network-to-meet-customer-demand

- INFORMS, "Regionalize and Scale: Amazon's Fulfillment Network Design for Faster and Cheaper Delivery". Full deployment around March 2023; in-region fulfillment 62% to 76%; about 15% reduction in site-to-customer distance; first cost-to-serve-per-unit drop since 2018.
  https://pubsonline.informs.org/doi/abs/10.1287/inte.2025.0295

- Amazon Science, "SPEEDY: Framework for sharpening promise time estimates in sub-same-day delivery". Dynamic, faster delivery time slots; sharpening the promise-time estimate.
  https://www.amazon.science/publications/speedy-framework-for-sharpening-promise-time-estimates-in-sub-same-day-delivery

- US Patent filing 11,334,845, "System and method for generating notification of an order delivery". Estimated delivery date from fulfillment location plus postal-code-to-postal-code logistics lead time; ML methods listed: quantile regression (QR), quantile regression forest (QRF), gradient boosting trees, neural networks, XGBoost, or ensembles.
  https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/11334845

- About Amazon (Canada), "Amazon is delivering at its fastest speeds ever for Prime members globally". 2024: more than 5 billion Prime items delivered same or next day globally; speeds up ~30% year over year; attributed in part to same-day expansion, regionalization, and machine learning for demand prediction and inventory placement.
  https://aboutamazon.ca/news/retail/amazon-is-delivering-at-its-fastest-speeds-ever-for-prime-members-globally

- EcommerceBytes, "Amazon Says Regionalization of Fulfillment Centers Is Working". Q2 2023: more than half of Prime orders across the top 60 US metros arrived same or next day.
  https://www.ecommercebytes.com/2023/07/31/amazon-says-regionalization-of-fulfillment-centers-is-working/

- Doug Herrington (Amazon Worldwide Stores CEO) on faster delivery driving conversion, measured on product detail pages (coverage).
  https://homepagenews.com/retail-articles/amazon-says-faster-delivery-drives-marketplace-success-prime-member-satisfaction/

## Academic and secondary (useful, weaker authority)

- MIT DSpace thesis on Amazon promise policy and two-day cutoff optimization. Prime Day 2018 pilot recommended an 18:00 Pacific cutoff; error rates 8.9% hourly and 7.3% daily; promise policy sets daily order deadlines (cutoffs).
  https://dspace.mit.edu/handle/1721.1/122587

- MIT DSpace thesis applying quantile regression forests to Amazon linehaul transit times. QRF lets you specify the on-time probability p you want the model to predict.
  https://dspace.mit.edu/handle/1721.1/90751

- arXiv 2105.00315, "Online Fashion Commerce: Modelling Customer Promise Date" (Myntra). The asymmetric tradeoff: a promise later than actual can lose the order, an overly early promise harms experience; proposes asymmetric loss functions.
  https://arxiv.org/pdf/2105.00315

- Veeqo (an Amazon company), "Delivery Day Prediction on Amazon". Secondary account: ML trained on billions of historical shipments; zip-to-zip lanes and carrier performance; the predicted date becomes the Promised Delivery Date used for On-Time Delivery Rate; example lane NJ 07097 to Baltimore 21201 around 1.8 days on ground.
  https://www.veeqo.com/blog/delivery-day-prediction-on-amazon-what-it-is-how-it-works-and-why-it-matters

## What stays inference (NOT confirmed Amazon internals)

- The exact production model and the specific quantile p used for a given listing.
- The precise cache keys, TTLs, inventory-map layout, and lane-graph representation (described in the report as the well-grounded "how this class of problem is solved" version).
- That Amazon trains with an asymmetric loss specifically (confirmed for Myntra and JD.com; strong but labeled inference for Amazon).
