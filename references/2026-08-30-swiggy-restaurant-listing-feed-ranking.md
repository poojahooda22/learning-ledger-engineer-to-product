# References: Swiggy restaurant listing and home-feed ranking (2026-08-30)

Keeper links for the Swiggy feed-ranking + serviceability teardown.

## Primary sources (Swiggy Bytes engineering blog)

- Jairaj Sathyanarayana, "Evolution of and experiments with feed ranking at Swiggy."
  GBDT on Spark MLlib chosen for non-linearity + production fit; embeddings for restaurants,
  customers, food-items, delivery partners; city and time-slot embeddings fed to the wide
  memorization layer; sparse interaction features into the wide input; recency + similarity into
  a TensorFlow Lattice head for monotonic domain rules.
  https://bytes.swiggy.com/evolution-of-and-experiments-with-feed-ranking-at-swiggy-17204769e79f

- Ashay Tamhane, Jagrati Agrawal, Rutvik Vijjali, Akash Deep, "Learning To Rank Restaurants."
  Pointwise -> pairwise/listwise LTR; pairwise loss works best; multi-objective relevance +
  delivery experience via linear scalarization; NDCG comparison where Wide-and-Deep + Lattice
  beats GBT and the pairwise-DNN; features = dish/restaurant relevance, restaurant features,
  popularity, distance, predicted delivery time; future = segment-specific models + real-time
  in-session intent.
  https://bytes.swiggy.com/learning-to-rank-restaurants-c6a69ba4b330

- Somsubhra Bairi, "Designing the Serviceability Platform at Swiggy for High Scale, Part 1."
  Geo-filtering via Point-in-Polygon across thousands of clusters in hundreds of cities; GeoHash
  structure for efficient spatial queries; route distance; predicted delivery time from fleet +
  store + route + traffic; stress / graceful-degradation Finite State Machine (nodes = stress
  levels, edges = transitions); surge decision.
  https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-1-751a631f0379

- Somsubhra Bairi, "What Serviceability means at Swiggy?"
  https://bytes.swiggy.com/what-serviceability-means-at-swiggy-c94c1aad352a

- Ramkishore Saravanan, "Real-time ML Ranking for Autocomplete: Deploying Learning-to-Rank inside
  OpenSearch (Part 1)." Two-phase retrieve-then-LTR-rerank-top-k to bound latency.
  https://bytes.swiggy.com/real-time-ml-ranking-in-autocomplete-part-1-3cdbbd44f85a

## Supporting

- "Personalized Restaurant Ranking with a Two-Tower Embedding Variant," Towards Data Science.
  Two-tower (user tower + item tower) as a low-latency final ranking layer with ANN candidate
  generation upstream, for limited-compute settings.
  https://towardsdatascience.com/personalized-restaurant-ranking-with-a-two-tower-embedding-variant/

- Ben Feifke, "Geospatial Indexing Explained: A Comparison of Geohash, S2, and H3."
  Why prefix-based cells turn a polygon scan into a keyed lookup.
  https://benfeifke.com/posts/geospatial-indexing-explained/

## Scale numbers (Swiggy financials / press)

- Swiggy Q2 FY25 results press release: 17.1M average monthly transacting users (+19% YoY),
  230M orders in the quarter (up from 192M), 233,600 average monthly transacting restaurants
  (up from 189,800).
  https://www.swiggy.com/corporate/wp-content/uploads/2024/12/Swiggy_Press-release_Q2FY25-results.pdf

- Medianama coverage of the same quarter (17.1M MTU).
  https://www.medianama.com/2024/12/223-swiggy-earnings-q2fy25-active-users-increase-to-17-million-yoy/

- Swiggy Annual Report FY 2023-24 (2 lakh+ restaurants, 680+ cities; ~118M users and ~3.5B
  orders over ten years; 1.96B delivery km in 2024).
  https://www.swiggy.com/corporate/wp-content/uploads/2024/10/Annual-Report-FY-2023-24-1.pdf

## Note on access

bytes.swiggy.com, towardsdatascience.com, and web.archive.org were blocked by the network egress
proxy during this run, so the Swiggy Bytes details above were sourced via search-engine result
summaries of those primary posts rather than direct fetches. Confirmed facts are attributed to
those posts; serving-pipeline specifics (two-stage candidate-then-rerank, ANN first stage,
per-location caching, exact latency/shard boundaries) are labeled inference in the report.
</content>
