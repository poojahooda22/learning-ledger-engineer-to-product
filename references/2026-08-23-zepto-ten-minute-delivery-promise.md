# References: Zepto 10-minute delivery promise (ETA + dispatch)

Saved 2026-08-23 for the Zepto delivery-time-promise teardown.

## Zepto-specific (public reporting; exact internals not published)
- Analytics Vidhya, "The Data Science Behind Zepto's 10-Minute Delivery Success" (2025). ETA as regression (distance, traffic, history, rider speed; random forest / gradient boosting), demand forecasting with ARIMA / Prophet / LSTM, dispatch solving rider allocation (proximity, scooter battery) plus traffic modeling with real-time serviceable-radius shrink. https://www.analyticsvidhya.com/blog/2025/10/zepto-data-science/
- TechResearchOnline, "How Zepto Delivers Groceries in 10 Minutes." Dark store size 2,000-4,400 sq ft, sited within 1.5-2 km of high-demand neighbourhoods, closed-access micro-warehouses built only for online fulfilment. https://techresearchonline.com/blog/zepto-10-minute-delivery-business-model/
- BigStory, "Zepto's Dark Store: How Kaivalya Vohra Built India's Time Engine." Founder framing of the dark store as a time engine. https://www.bigstorynetwork.com/content/zepto-business-model-kaivalya-vohra-playbook

## DoorDash engineering (primary sources for the dispatch + ETA class of problem)
- "Using ML and Optimization to Solve DoorDash's Dispatch Problem." DeepRed engine; ML estimates + optimization layer; one-delivery-per-route framed as bipartite matching solved with the Hungarian algorithm. https://careersatdoordash.com/blog/using-ml-and-optimization-to-solve-doordashs-dispatch-problem/
- "Next-Generation Optimization for Dasher Dispatch at DoorDash." Batching multiple deliveries, moving beyond bipartite matching to routing. https://careersatdoordash.com/blog/next-generation-optimization-for-dasher-dispatch-at-doordash/
- "Scaling a routing algorithm using multithreading and ruin-and-recreate." How the NP-hard routing/batching is solved fast under a time budget. https://careersatdoordash.com/blog/scaling-a-routing-algorithm-using-multithreading-and-ruin-and-recreate/
- "Precision in Motion: Deep learning for smarter ETA predictions." Staged decomposition of delivery time; multi-task mixture-of-experts with probabilistic (distribution) forecasts. https://careersatdoordash.com/blog/deep-learning-for-smarter-eta-predictions/

## Background
- Ilya Zinkovich, "Evolution of Food Delivery Dispatching." Greedy to bipartite matching to batching/VRP progression. https://ilyazinkovich.github.io/2020/06/16/delivery-dispatching-evolution.html
- Uber Engineering, "H3: Uber's Hexagonal Hierarchical Spatial Index." Grounding for the geospatial-index inference; equidistant hex neighbours. https://www.uber.com/blog/h3/

## Key takeaways used
- The promise is a prediction (regression), not a countdown; predicted before payment on minimal basket info.
- Predict a distribution and promise a high percentile (beat it ~90% of the time), not the mean, or you are late half the time.
- Staged sum: assign + pick/pack + travel + buffer, each stage a different signal.
- The predicted promise is a HARD CONSTRAINT on dispatch: a clubbed bundle is legal only if it keeps every order inside its promise.
- Dispatch = min-cost bipartite matching (Hungarian, O(n^3)); clubbing makes it a small NP-hard VRP solved by ruin-and-recreate under a tight clock.
- Scale survival: demand forecasting + pre-positioning for the evening wall, geo-sharding for dispatch load, per-store cached promise with short TTL for read load, radius-shrink/unavailable for correlated rain shocks.
- Offline-think / online-lookup: the home-screen number is mostly a cache read.
