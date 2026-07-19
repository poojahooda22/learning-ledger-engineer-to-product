# References: Swiggy search and ranking

Saved for the 2026-07-19 teardown (Swiggy search box: serviceability, autocomplete,
retrieval, ranking).

## Primary (Swiggy engineering blog: bytes.swiggy.com)

- Using Deep Learning for Ranking in Dish Search (Ramkishore Saravanan)
  https://bytes.swiggy.com/using-deep-learning-for-ranking-in-dish-search-4df2772dddce
  Key facts: dish search = retrieval (millions of dishes down to hundreds) + ranking;
  deep-learning ranker over relevance, restaurant, popularity, distance and delivery
  time; multilingual challenge (chicken = Murgh in Hindi, Kozhi in Tamil).

- Real-time ML Ranking for Autocomplete: Deploying Learning-to-Rank inside OpenSearch (Part 1)
  https://bytes.swiggy.com/real-time-ml-ranking-in-autocomplete-part-1-3cdbbd44f85a
  Key facts: OpenSearch LTR plugin runs model inference at query time inside the engine,
  no external service call / network hop; two-stage candidate generation then top-k
  rescore; feature store serves precomputed + streaming features; continuous feedback
  loop retrains from clicks/conversions/orders. Replaced hand-tuned heuristic ranking.

- Improving search relevance in hyperlocal food delivery using (small) language models (Adityakiran)
  https://bytes.swiggy.com/improving-search-relevance-in-hyperlocal-food-delivery-using-small-language-models-ecda2acc24e6
  Key facts: hyperlocal = ~5km radius of serviceable restaurants; queries mix dish /
  dietary / time-slot / mood intent; two-stage fine-tuning (unsupervised on historical
  queries+orders, then supervised on curated query-item pairs); TSDAE + Multiple
  Negatives Ranking Loss; hard 100ms latency budget.

- What Serviceability means at Swiggy (Somsubhra Bairi)
  https://bytes.swiggy.com/what-serviceability-means-at-swiggy-c94c1aad352a

- Designing the Serviceability Platform at Swiggy for High Scale (Part 1)
  https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-1-751a631f0379
  Key facts: geohash groups restaurants into grid cells for fast nearby search;
  separate index per geohash5 (~4.9km cell) so ANN carts are geographically close not
  just embedding-close; map lat/long to geohash key, fetch reduced cluster set, then run
  point-in-polygon (PIP) only on that reduced set; distance service handles nearly 200M
  source-destination pairs per minute at peak.

- Learning To Rank Restaurants (Ashay Tamhane, Jagrati Agrawal)
  https://bytes.swiggy.com/learning-to-rank-restaurants-c6a69ba4b330
  Key facts: GBDT (Spark MLlib) for non-linearity + easy production integration;
  features across 3 dimensions (customer-level, restaurant-level, customer-restaurant).

- Using embeddings to help find similar restaurants in Search (Anurag Mishra)
  https://bytes.swiggy.com/using-embeddings-to-help-find-similar-restaurants-in-search-1d1417dff304

- Building a mind reader at Swiggy using Data Science (Shreyas Mangalgi)
  https://bytes.swiggy.com/building-a-mind-reader-at-swiggy-using-data-science-5a5c38aa6c17

## Secondary / summaries

- InfoQ: Swiggy Improves Search Autocomplete Using Real Time Machine Learning Ranking
  https://www.infoq.com/news/2026/05/swiggy-autocomplete-rt-ranking/

- ZenML LLMOps Database: Two-Stage Fine-Tuning of Language Models for Hyperlocal Food Search
  https://www.zenml.io/llmops-database/two-stage-fine-tuning-of-language-models-for-hyperlocal-food-search-ff103

## The one-line takeaway

Food search is four keyed lookups in sequence (serviceability by geohash, autocomplete
in-engine LTR, retrieval by inverted index + embeddings, ranking by deep model),
matching split cleanly from ranking, all heavy work precomputed offline, the sort done
server-side, the phone just paints a finished list.
