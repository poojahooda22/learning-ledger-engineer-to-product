# References: Amazon Sponsored Products ad auction

Keeper links for the 2026-09-12 teardown. Grouped by what they prove.

## Auction mechanics (GSP, ad rank = bid x relevance, second price)
- How Amazon's ad auction actually works (not highest bid wins): https://rel.ai/blog/how-amazons-ad-auction-works
- Cornell INFO 2040, complexities of advertising bidding on Amazon: https://blogs.cornell.edu/info2040/2022/09/17/the-complexities-of-advertising-bidding-on-amazon/
- How the Amazon PPC auction works: https://www.aihello.com/resources/blog/how-does-the-amazon-ppc-auction-work/

## FTC lawsuit numbers (the strong, confirmed stats)
- Karooya, FTC v. Amazon sponsored-ads auction-pricing (92% not highest bid; winner ~128th bid by amount): https://www.karooya.com/blog/ftc-v-amazon-what-the-sponsored-ads-auction-pricing-lawsuit-means-for-advertisers/
- Amazon's own response to the FTC sponsored-ads lawsuit: https://www.aboutamazon.com/company-news/amazon-ftc-sponsored-ads-lawsuit-response

## Retrieval and ranking pipeline (matching vs ranking, multi-stage funnel)
- Amazon Science, from structured search to learning to rank and retrieve: https://www.amazon.science/blog/from-structured-search-to-learning-to-rank-and-retrieve
- Amazon, extreme multi-label learning for semantic matching in product search: https://arxiv.org/pdf/2106.12657
- SIGIR eCom 2025, model-based performance filtering for ad quality: https://sigir-ecom.github.io/eCom25Papers/paper_5.pdf
- PCDF, parallel distributed framework for sponsored search serving: https://arxiv.org/pdf/2206.12893
- Alibaba, constrained optimization of auction mechanisms in sponsored search: https://arxiv.org/pdf/1807.11790

## Two-tower and ANN retrieval (standard pattern, labeled inference in the report)
- Taobao, embedding-based product retrieval in search: https://arxiv.org/pdf/2106.09297
- Google Cloud, scaling deep retrieval with a two-tower architecture: https://cloud.google.com/blog/products/ai-machine-learning/scaling-deep-retrieval-tensorflow-two-towers-architecture

## Dynamic bidding and budget rules
- Amazon Ads, guide to dynamic bidding up and down: https://advertising.amazon.com/library/guides/dynamic-bidding-sponsored-products
- Sellermetrics, Amazon's 2025 budget rules update (up to 25% daily overshoot, dynamic allocation): https://sellermetrics.app/amazon-2025-budget-rules-update/
- Feedvisor, default, suggested and dynamic bids: https://feedvisor.com/university/amazon-sponsored-products-default-suggested-and-maximum-bids/

## Scale (catalog size, revenue, request volume)
- Redstag, how many products Amazon carries (~600M active listings): https://redstagfulfillment.com/how-many-products-does-amazon-carry/
- Slashdot, Amazon purges billions of listings ("Bend the Curve", ~24B ASINs): https://slashdot.org/story/25/05/30/1954240/amazon-purges-billions-of-product-listings-in-cost-cutting-drive
- Marketing Dive, Amazon annual ad revenue passes 68 billion: https://www.marketingdive.com/news/amazon-annual-ad-revenue-passes-68b-boosted-by-full-funnel-strategy/811569/
- eMarketer, Amazon retail media ad revenue past 60 billion in 2025: https://www.emarketer.com/content/amazon-retail-media-ad-revenues-will-pass-60-billion-2025
- Amazon Jobs, ML Engineer II, Sponsored Products Search Sourcing (billions of requests per day, millisecond latency): https://amazon.jobs/en/jobs/3061807/machine-learning-engineer-ii-sponsored-products-search-sourcing-amazon-advertising
