# References: Instagram Reels recommendation and ranking (2026-08-07)

## Primary sources (Meta engineering, confirmed)
- Scaling the Instagram Explore recommendations system, Engineering at Meta, Aug 2023:
  https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/
  - Multi stage funnel: "starting with thousands of candidates and narrowing down... to hundreds."
  - Retrieval: "multiple candidate retrieval sources," real-time (recent interactions) + pre-generated (long-term interests).
  - Two tower NN chosen for "its cacheability property"; item embeddings stored in a service supporting "online approximate nearest neighbors search"; user tower "generates a user embedding on the fly to find the most similar items."
  - First stage ranker trained by distillation, label PSelect = media in top K of the second stage (small model imitates big model).
  - Second stage = multi-task multi-label (MTML) NN predicting engagement probabilities, "much heavier than the two towers model."
- Journey to 1000 models: Scaling Instagram's recommendation system, Engineering at Meta, May 2025:
  https://engineering.fb.com/2025/05/21/production-engineering/journey-to-1000-models-scaling-instagrams-recommendation-system/
  - Over 1,000 ML models in production.
  - Model registry built on Configerator; automated deploy pipeline cut launch from days to hours.
  - Unified health metrics = calibration + normalized entropy to auto-detect prediction issues.
- Instagram Feed Ranking System Card, Meta AI: https://ai.meta.com/tools/system-cards/instagram-feed-ranking/
  - Predicted actions PLIKE, PCOMMENT, PFOLLOW combined into a single relevance score.
- Instagram Explore ranking, Meta Transparency Center: https://transparency.meta.com/features/explaining-ranking/ig-explore/

## Reels signals and scale (Meta statements + reporting)
- Zuckerberg, Meta Q3 2025 earnings: >200B daily Reels plays (up from ~100B late 2024); $50B annual ad run-rate.
  Coverage: https://www.tubefilter.com/2025/10/30/meta-reels-ad-revenue-q3-2025-earnings-report/
- CNBC, Jan 2026: most of Instagram's ads ran on Reels in 2025 (>half, up from ~35% in 2024):
  https://www.cnbc.com/2026/01/20/most-of-instagrams-ads-ran-on-reels-in-2025-data-shows.html
- Mosseri (2026) top Reels signals: watch time, likes per reach, DM shares; completion rate = strongest reach signal;
  DM shares ~3-5x a like; negatives = skip rate (>40% clear negative), "Not interested," mutes/hides.
  Aggregated: https://blog.hootsuite.com/instagram-algorithm/ , https://later.com/blog/how-instagram-algorithm-works/
- Reels ~46% of US IG app time in 2025 (up from 37% in 2024); 4.5B reshares/day (Meta).

## Explainers / background
- How Instagram Reel Uses Recommender Systems, GeeksforGeeks:
  https://www.geeksforgeeks.org/machine-learning/how-instagram-reel-uses-recommender-systems/
- The Engineering behind Instagram's Recommendation Algorithm, Quastor:
  https://blog.quastor.org/p/engineering-behind-instagrams-recommendation-algorithm-dc9c

## Key technical spine (for reuse)
- No query, near-infinite refreshing pool, one-video-at-a-time delivery, a grade after every swipe. The feedback atom is WATCH TIME (passive), not a like.
- Funnel: billions -> thousands (retrieval) -> hundreds (first-stage) -> handful (second-stage) -> reranked batch of 10-20 sent to phone. Cheap stages on many, expensive on few.
- Two tower = user tower embedding + item tower embedding, score = dot product. Reason for two towers = CACHEABILITY: item embeddings precomputed once at upload (user-independent), only the user embedding is fresh per request -> ANN lookup (HNSW / FAISS / IVF, Lesson 33) instead of a full scan.
- Retrieval is multi-source: real-time (this session's watches) + long-term interests + trending-near-you + friends-resent.
- First stage = distillation (predict what second stage would keep, label PSelect). Second stage = MTML predicting many engagement probabilities. Value model = weighted linear sum of those probs; weights = editorial policy, tuned for long-term retention.
- Reels-specific: time-weighted (watch time, completion) over taps; DM share ~3-5x a like; strong negatives for fast skip / Not interested.
- Final rerank = diversity (no six same-topic clips in a row) + ads (Netflix-homepage sequence logic, 2026-08-06).
- Sorting is 100% server-side; phone plays + preloads + reports honestly (offline-think/online-lookup).
- Scale: 1k = score-all no funnel; 100k = funnel + ANN index + streaming features (scoring cost broke); 10M+/2B users/millions req/sec = fleet is the bottleneck, not any model (1000+ models, registry on Configerator, auto-deploy, calibration+entropy health metrics, streaming embeds for freshness).
- Retention = variable-ratio reward schedule (slot machine), near-zero-cost swipe + unpredictable payoff + never-ending feed; retention and revenue on the same line.
- Fact/inference: funnel + two-tower + ANN + distillation + MTML + value model CONFIRMED for Instagram recommenders as a class (Explore/Feed). Exact Reels internals and per-clip weights = grounded inference, labeled.
