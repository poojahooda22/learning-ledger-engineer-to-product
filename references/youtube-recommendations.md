# References: YouTube recommendations (candidate generation + ranking)

Saved 2026-06-22 for the YouTube recommendations teardown.

## Primary papers

- **Deep Neural Networks for YouTube Recommendations** (Covington, Adams, Sargin,
  RecSys 2016). The foundational two-stage funnel: candidate generation as extreme
  multiclass softmax with negative sampling, serving as approximate nearest-neighbor
  search in dot-product space, ranking via weighted logistic regression to predict
  expected watch time, the "example age" freshness feature, predicting the next watch
  (forward in time) rather than a random held-out watch.
  https://research.google/pubs/deep-neural-networks-for-youtube-recommendations/
  Mirror PDF: https://cseweb.ucsd.edu/classes/fa17/cse291-b/reading/p191-covington.pdf

- **Recommending What Video to Watch Next: A Multitask Ranking System** (Zhao et al,
  RecSys 2019). Modern ranking: Multi-gate Mixture-of-Experts (MMoE) to predict many
  engagement and satisfaction goals at once, plus a shallow tower that models and
  removes position/selection bias (dropped at serving).
  https://daiwk.github.io/assets/youtube-multitask.pdf

- **Top-K Off-Policy Correction for a REINFORCE Recommender System** (Chen, Beutel,
  Covington, et al, WSDM 2019). Candidate generation as reinforcement learning over an
  action space of millions of videos, off-policy correction for logged data, top-K
  correction for recommending a slate. Reported as one of YouTube's most impactful
  engagement launches.
  https://arxiv.org/abs/1812.02353

## Secondary / context

- The Morning Paper walkthrough of the 2016 paper (Adrian Colyer).
  https://blog.acolyer.org/2016/09/19/deep-neural-networks-for-youtube-recommendations/

- Neal Mohan (then CPO) at CES Jan 2018: recommendations drive ~70% of watch time;
  average mobile session over an hour.
  https://www.tubefilter.com/2018/01/11/youtube-most-watch-time-driven-by-recommendations/
  https://qz.com/1178125/youtubes-recommendations-drive-70-of-what-we-watch

## The one idea to remember

Recommendation is search with the same two halves: matching (candidate generation,
billions down to hundreds, cheap, high recall, served as an ANN lookup over offline
embeddings) and ranking (hundreds in the best order, rich features, optimizing watch
time not clicks). The whole-corpus work happens offline; the live path is one user
vector, one nearest-neighbor lookup, one small sort.
