# References: Zomato Food Preparation Time (KPT/FPT) prediction

Saved 2026-06-29.

## Primary (Zomato engineering)
- The Deep Tech Behind Estimating Food Preparation Time: https://www.zomato.com/blog/food-preparation-time/
  - Medium mirror: https://medium.com/zomato-technology/the-deep-tech-behind-estimating-food-preparation-time-e5068807acb0
  - Architecture: running orders (max 5) + last 5 completed orders -> stacked LSTM -> concat with present-order features + Restaurant Embedding Vector -> 2-layer dense -> regress on FPT.
  - Stated limitations: rider-influenced FOR marking, no kitchen-wide rush signal, manual-marking bias.
- Predicting your order's Food Preparation Time: https://blog.zomato.com/predicting-fpt-optimally
  - Modified quantile loss with penalty factors m and n; asymmetric penalty (q<0.5 predicts lower, q>0.5 higher, q=0.5 == MAE).
  - FOR ("Food Order Ready") button in Restaurant Partner app; ~9% improvement in within-5-minutes accuracy.
- The accurate ETA to customer satisfaction (Part One/Two): https://blog.zomato.com/the-accurate-eta-to-customer-satisfaction-part-one , https://blog.zomato.com/the-accurate-eta-to-customer-satisfaction-part-two
  - Total delivery time = KPT + rider-to-restaurant + restaurant-to-customer + dynamic buffer.
  - Switched travel-time ETA from map-graph model to tree-based LightGBM.
  - Equation-based ultra-fast ETA for the assignment inner loop, +4% assignment-prediction compliance.
  - Metric framing: punctuality + tolerance interval.
- The elements of scalable ML: https://blog.zomato.com/elements-of-scalable-machine-learning
  - Real-time features via Kafka + Flink -> Redis; batch features via Spark -> key-value store.

## Scale numbers
- ~2M orders in a single day on 20 June 2024 (~1,400/min): https://www.business-standard.com/companies/news/what-is-zomato-s-daily-order-count-company-s-new-feature-spills-the-beans-124062200356_1.html
- 647M orders FY2023 (+21% YoY): https://www.statista.com/statistics/1110238/zomato-number-of-orders/
- ~1.3M orders/day from ~150k restaurants (Zomato "How India is coming of age"): https://blog.zomato.com/real-india

## Cross-references in this ledger
- Restaurant embedding = same dense-id trick as YouTube (2026-06-22) and Amazon (2026-06-23).
- Offline-think / online-lookup spine = Discover Weekly (2026-06-13), YouTube (2026-06-22), Razorpay routing (2026-06-26).
- Distinct from Swiggy live tracking (2026-06-24): that was the moving rider + ETA legs; this is the kitchen-time prediction that feeds dispatch timing.
