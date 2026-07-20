# References: Instagram Explore recommender

Saved for the 2026-07-20 teardown (candidate generation + multi-stage ranking).

## Primary (Meta / Instagram engineering)

- "Powered by AI: Instagram's Explore recommender system" (Ivan Medvedev, Instagram
  Engineering, 2019). The foundational post: ig2vec account embeddings (word2vec on
  sequences of interacted accounts), seed accounts, candidate generation vs ranking,
  sampling 500 candidates, distillation model for the first pass, MTML ranking.
  https://instagram-engineering.com/powered-by-ai-instagrams-explore-recommender-system-7ca901d2a882

- "Scaling the Instagram Explore recommendations system" (Engineering at Meta,
  2023-08-09). Multi-stage funnel (retrieval, first-stage ranking, second-stage
  ranking, final reranking), Two Towers retrieval model, user tower live + item
  towers precomputed into ANN, caching/precomputation to afford heavier models,
  headline scale: 65 billion features extracted, 90 million predictions per second.
  https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/

- "How Instagram suggests new content" (Engineering at Meta, 2020-12-10). Account
  embeddings, suggested content sourcing.
  https://engineering.fb.com/2020/12/10/web/how-instagram-suggests-new-content/

- "Journey to 1000 models: Scaling Instagram's recommendation system"
  (Engineering at Meta, 2025-05-21). Model-serving scale.
  https://engineering.fb.com/2025/05/21/production-engineering/journey-to-1000-models-scaling-instagrams-recommendation-system/

- Meta Transparency Center, "Instagram Explore AI system." Public plain-language
  description of the engagement predictions Explore combines (like, save, comment).
  https://transparency.meta.com/features/explaining-ranking/ig-explore/

## Infrastructure

- FAISS (Facebook AI Similarity Search): the ANN library behind embedding retrieval.
  https://github.com/facebookresearch/faiss

## Third-party recaps (useful cross-checks, not primary)

- VentureBeat, "Facebook details the AI technology behind Instagram Explore."
  https://venturebeat.com/ai/facebook-details-the-ai-technology-behind-instagram-explore/

## Key numbers to remember

- Funnel: billions of posts -> a few thousand candidates -> 500 sampled -> 150
  (distillation pass) -> 50 (MTML pass) -> ~25 shown.
- 65 billion features extracted and 90 million model predictions every second (2023).
- Retrieval: ig2vec (word2vec-style account embeddings) generalized to Two Towers
  (separate user tower and item tower, similarity trained to predict engagement).
- Ranking value model: weighted sum of predicted actions (like, save, share,
  watch-through) minus negative signals (See Less, hide, report). Exact weights
  are private.
