# References: Zepto in-store picking and slotting (2026-09-09)

Keeper sources for the dark-store order-picking / slotting / batching teardown.

## Foundational academic (the algorithms)

- Ratliff, H.D. and Rosenthal, A.S. (1983). "Order-Picking in a Rectangular Warehouse: A Solvable Case of the Traveling Salesman Problem." Operations Research 31(3):507-521.
  https://pubsonline.informs.org/doi/abs/10.1287/opre.31.3.507
  The pick path is a TSP; a rectangular warehouse with cross-aisles is a solvable special case. Dynamic programming, sparse graph of aisles, runtime LINEAR in the number of aisles (not exponential in items). First exact approach to picker routing. Extended by Roodbergen and de Koster (2001) to more cross-aisles.

- de Koster, R., Le-Duc, T., Roodbergen, K.J. (2007). "Design and control of warehouse order picking: A literature review." European Journal of Operational Research 182(2):481-501.
  PDF: https://repub.eur.nl/pub/7322/ERS%202006%20005%20LIS.pdf
  Semantic Scholar: https://www.semanticscholar.org/paper/Design-and-control-of-warehouse-order-picking:-A-Koster-Le-Duc/d482423c1f77da795531fed05046d13c9669704b
  Key stats: travel is ~50%+ of total pick time; order picking is up to ~55% of warehouse operating cost. Covers storage assignment (velocity/ABC, correlated/affinity), routing heuristics (S-shape, return, midpoint, largest gap, combined), order batching, zoning.

- "Exact algorithms for the order picking problem," arXiv: https://arxiv.org/pdf/1703.00699
- "Improved formulations of the joint order batching and picker routing problem," arXiv: https://arxiv.org/pdf/2207.05305
  The joint batching+routing problem (entangled, hard at scale).

## Batch picking (the travel-amortization trick)

- NetSuite, "What Is Batch Picking?": https://www.netsuite.com/portal/resource/articles/inventory-management/batch-picking.shtml
  Batch picking cuts travel ~40 to 60% vs single-order picking by sharing the fixed walk cost across many orders.

## Zepto / quick-commerce operations (facts, some inference-labeled in report)

- MetricsCart, dark store operations: https://metricscart.com/insights/dark-store-operations/
  Optimized picking path, digital pick lists, shelf maps, barcode validation, high-velocity SKUs near packing, co-ordered items slotted adjacent, min-max replenishment.
- 42Signals, Zepto model: https://www.42signals.com/blog/zepto-business-model-explained/
  Aadit Palicha: within 76 seconds of an order, it is packed and ready for pickup. Data-driven layout.
- AppsRhino: https://www.appsrhino.com/blogs/zepto-10-minute-grocery-delivery
  Stores within 2 to 3 km; picking often under 2 minutes; ~60 seconds average.
- Prozo, quick-commerce fulfilment: https://www.prozo.com/warehousing/quick-commerce/
  Shelf position mapped to picking frequency; micro-picking; mechanized metro stores 6,000 to 8,000 sq ft, 25,000 SKUs; algorithmic batch picking within 120 seconds; if a store stocks out of a SKU 3x/week the product drops in search ranking for that pin code; target availability 95 to 98%.
- OneVisionMedia, Zepto case study: https://onevisionmedia.in/zepto-business-model-case-study-dark-store-unit-economics/
  Store size 2,000 to 4,400 sq ft; curated ~2,500 to 3,000 SKUs per store.

## Cross-links in this ledger

- 2026-06-17 Zepto dark-store inventory and order routing (which store, atomic stock decrement, which rider). This teardown is the missing middle: what happens INSIDE the store.
- 2026-08-23 Zepto "10-minute promise" (ETA + dispatch to rider).
- 2026-07-12 Amazon item-to-item collaborative filtering (co-occurrence / market-basket = the same idea reused for affinity slotting).
- Offline-think/online-lookup pattern: Discover Weekly, YouTube, Amazon search, Google index build.
