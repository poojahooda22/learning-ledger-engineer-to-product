# References: Zepto grocery search (2026-10-18)

Zepto does not publish its search internals, so the confirmed engineering here is
Instacart's (the best-documented grocery search), used as the class-of-problem
reference. Zepto-specific numbers and ranking behavior come from secondhand
industry/marketing analyses and IPO (DRHP) reporting, labeled as such.

## Instacart engineering (CONFIRMED class-of-problem)

- An Embedding-Based Grocery Search Model at Instacart (SIGIR eCom 2022).
  Two-tower transformer, per-retailer FAISS indexes that handle store-level
  inventory, product embeddings precomputed daily, hybrid merge with keyword
  retrieval, +4.1% cart-adds per search and +1.5% GMV online, +10% offline
  Recall@20, cart-adds are a noisy relevance label.
  - https://arxiv.org/abs/2209.05555
  - https://sigir-ecom.github.io/ecom22Papers/paper_8392.pdf
- How Instacart Uses Embeddings to Improve Search Relevance (ranking blends
  relevance, popularity, personal preference).
  - https://www.instacart.com/company/tech-innovation/how-instacart-uses-embeddings-to-improve-search-relevance
- Optimizing search relevance at Instacart using hybrid retrieval (keyword +
  embedding merged, similarity as a ranker feature).
  - https://tech.instacart.com/optimizing-search-relevance-at-instacart-using-hybrid-retrieval-88cb579b959c
- How Instacart Uses ML-Driven Autocomplete (candidates from past searches,
  doc2query catalog generator for cold-start retailers, multi-objective model
  predicting add-to-cart, semantic dedup of "fresh bananas" vs "bananas fresh").
  - https://company.instacart.com/tech-innovation/how-instacart-uses-machine-learning-driven-autocomplete-to-help-people-fill-their-carts
- Building The Intent Engine: Query Understanding with LLMs (RAG, taxonomy
  validation, query-category classification powers recall and ranking).
  - https://tech.instacart.com/building-the-intent-engine-how-instacart-is-revamping-query-understanding-with-llms-3ac8051ae7ac
- How Instacart Uses ML to Suggest Replacements for Out-of-Stock Products
  (heuristic retrieval on past replacement pairs + semantic similarity, then ML
  rank).
  - https://www.predictiveanalyticsworld.com/machinelearningtimes/how-instacart-uses-machine-learning-to-suggest-replacements-for-out-of-stock-products/14118/
- Two-tower models in industry (summary: per-retailer FAISS, daily embeddings).
  - https://machinelearningatscale.substack.com/p/recsys-part-2-two-tower-models-in

## Zepto / quick commerce (facts and secondhand analyses, labeled in text)

- Business Today, how dark stores power quick commerce. Staff pack an order in
  about 2 minutes, the rider then has roughly 8 to 10 minutes.
  - https://www.businesstoday.in/magazine/deep-dive/story/how-dark-stores-are-powering-growth-of-quick-commerce-players-374569-2023-03-23
- Quick-commerce search ranking (availability-first, out-of-stock disappears,
  sales velocity dominant; secondhand marketing analysis).
  - https://zane.marketing/quick-commerce-search-ranking/
- Zepto IPO / DRHP reporting: roughly 1,139 dark stores across 66 cities,
  weighted-average near 49,602 SKUs per dark-store geography (per-city catalog,
  not per-store). Per-store curated assortment reported at ~2,500 to 3,000 SKUs
  by industry blogs. Verify both against the filing.
  - https://hdfcsky.com/ipo/zepto-limited-ipo

## Key facts used in the teardown

- CONFIRMED (Instacart): per-store/per-retailer scoped FAISS index, so the
  retrieval index already contains only that store's stock. This is the exact
  mechanism behind Zepto's hyperlocal, in-stock-only results (inference that
  Zepto does the same).
- CONFIRMED (Instacart): embeddings precomputed daily; offline-think,
  online-lookup.
- SECONDHAND: Zepto/Blinkit rank availability-first (OOS disappears) then
  velocity; fill-rate thresholds around 90% are cited but conflict across
  sources and are unverified.
