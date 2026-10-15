# Instagram "Suggested for you" account recommendations (who to follow)

Keeper links for the 2026-10-15 teardown on how Instagram picks which accounts to
suggest you follow. Distinct from the Explore CONTENT recommender
(instagram-explore-recommendations.md): this is about recommending ACCOUNTS (a
follow edge), not posts to view.

## Primary sources (Meta's own)

- Meta AI, "Powered by AI: Instagram's Explore recommender system" (2019)
  https://ai.meta.com/blog/powered-by-ai-instagrams-explore-recommender-system/
  The foundational writeup. Two stages: candidate generation (sourcing) then
  ranking. IGQL declarative query language with a combiner that blends candidate
  generators by weight. ig2vec = word2vec over the sequence of accounts a user
  interacts with (accounts as "words", sessions as "sentences"). FAISS for fast
  approximate nearest-neighbour similarity search over account embeddings, loaded
  offline. First-pass ranking uses a lightweight distillation model.

- Meta Engineering, "Scaling the Instagram Explore recommendations system"
  (engineering.fb.com, 2023-08-09)
  https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/
  The modern funnel: retrieval, first-stage ranking, second-stage ranking, final
  reranking; thousands of candidates narrowed to hundreds. Two Towers retrieval
  network with caching on both user and item side. Super-lightweight distillation
  model as first-stage (NDCG ranking loss over the main model's output). MTML
  (multi-task multi-label) heavy ranker. Value model fuses predicted engagement
  probabilities into one score. Heavy reliance on caching and precomputation.

- Meta Transparency Center, "Instagram Suggested Accounts AI system"
  https://transparency.meta.com/features/explaining-ranking/ig-suggested-accounts/
  Account-specific signals: followers of accounts you already follow (two-hop
  graph), popular accounts in your region, topics you recently engaged with,
  synced contacts. Models learn from likes, comments, DMs, and engagement. Signals
  change frequently. Users can remove a suggestion.

- Meta Transparency Center, "Instagram Explore AI system"
  https://transparency.meta.com/features/explaining-ranking/ig-explore/
  Funnel numbers: up to 1500 media fetched in retrieval, top ~100 passed on after
  early-stage ranking. Newer config than the 2019 numbers.

## Supporting

- "How Personal is Machine Learning Personalization?", arXiv:1912.07938 (2019)
  https://arxiv.org/pdf/1912.07938
  Concrete ranking cascade example: ~500 candidates, first pass to 150, neural net
  to 50. Numbers are from 2019 and have likely drifted.

- VentureBeat, "Facebook details the AI technology behind Instagram Explore"
  https://venturebeat.com/ai/facebook-details-the-ai-technology-behind-instagram-explore/
  Confirms ig2vec treats "the accounts a user interacts with as word sequences in
  a sentence" and the account-embedding retrieval pipeline.

- LinkedIn Engineering, "People You May Know"
  https://engineering.linkedin.com/teams/data/artificial-intelligence/people-you-may-know
  Friends-of-friends "triangle closing" as the canonical first candidate signal
  for any who-to-connect system. Names relevance and scale as the two hard parts.

## The one-line takeaway

Recommending an account is matching (find a few hundred candidates out of
billions, via a two-hop follow-graph walk + ig2vec/FAISS nearest-neighbour +
contacts/region, stitched by IGQL) then ranking (a cheap distillation model thins
the herd, a heavy MTML model + value model orders the survivors). All the
expensive work (graph walks, embedding training, candidate caching, both Two
Towers) is precomputed offline; the live tap is a cached lookup + one ANN query +
a light score, finishing inside the Follow-button animation.
