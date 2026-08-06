# References: Netflix personalized homepage (rows + page generation)

Saved for the 2026-08-06 teardown.

## Primary sources (Netflix)

- Gomez-Uribe & Hunt, "The Netflix Recommender System: Algorithms, Business Value,
  and Innovation," ACM TMIS, 2015.
  - https://dl.acm.org/doi/pdf/10.1145/2843948
  - Mirror: https://ailab-ua.github.io/courses/resources/netflix_recommender_system_tmis_2015.pdf
  - The canonical source. Describes the row-based homepage and the named algorithms:
    Personalized Video Ranker (PVR, per-member per-genre catalog ordering, blends
    personalization with popularity), Top-N Video Ranker (tuned for the head only),
    Trending Now (short-window, seasonal + event bursts), Continue Watching (resume
    likelihood), Because You Watched (BYW, seeded from a finished title), and
    video-video similarity. Also the business numbers: ~80% of hours streamed start
    from a recommendation; personalization + recommendations worth >$1B/year; churn
    reduced by "several percentage points"; the 60 to 90 second browsing window.

- Netflix Tech Blog, "Learning a Personalized Homepage," April 2015.
  - http://techblog.netflix.com/2015/04/learning-personalized-homepage.html
  - The 2D page problem. Page generation selects and orders rows, dedups, keeps
    diversity. Stage-wise (order rows on row-level scores, then fill) vs page-wise
    (score a row in the context of the partial page). 2D page-level metrics: extend
    recall / NDCG / MRR / Expected Reciprocal Rank to a cascade browse model (across
    a row, then down), so lower positions get lower see-probability. Rule-based (red
    line) vs personalized layout (blue line) recall comparison.

- Netflix Tech Blog, "GenPage: Towards End-to-End Generative Homepage Construction
  at Netflix," June 2026.
  - https://netflixtechblog.com/genpage-towards-end-to-end-generative-homepage-construction-at-netflix-77146fba8a08
  - arXiv HTML mirror: https://arxiv.org/html/2606.31031
  - Replaces the modular greedy pipeline with an autoregressive generative model:
    represent the page as a token sequence (rows + titles), compute a value for each
    candidate next token, greedily pick the max-value token, append, repeat token by
    token conditioned on the whole prefix. Trained with reinforcement learning on an
    aggregate page-level reward (diversity, stopping power, page-level business
    constraints), so cross-row interactions are learned instead of hand-coded.

- Netflix Tech Blog, "Innovating Faster on Personalization Algorithms at Netflix
  Using Interleaving."
  - https://netflixtechblog.com/interleaving-in-online-experiments-at-netflix-a04ee392ec55
  - Team-draft interleaving inside one row (coin toss for first pick, then alternate
    contributing next-best not-yet-used title), credit the algorithm that contributed
    the played title. Over 100x fewer members than the most sensitive A/B metric.
    Two-stage: interleaving prunes, A/B confirms.

## Secondary (for the attention-window numbers)

- Quartz, "Netflix knows exactly how long you'll look for something to watch before
  giving up": https://qz.com/622316
- NBC News: https://www.nbcnews.com/business/business-news/netflix-knows-how-long-you-ll-search-they-lose-you-n521766
  - 60 to 90 seconds before giving up; ~10 to 20 titles reviewed, ~3 in detail, on
    one or two screens.

## Widely reported UI figures (not guaranteed constants)

- ~40 rows per homepage, up to ~75 titles per row. Reported across multiple
  secondary write-ups of the Netflix UI; treat as approximate, not a Netflix-stated
  invariant.

## Not public (labeled as inference in the report)

- Exact model sizes, exact candidate-row count per member, serving latency budget,
  and the precise offline/online split (especially for GenPage). The claim that most
  scoring is precomputed offline and the live path is a keyed read + assembly is
  grounded inference from the system shape and consistent with the rest of the
  ledger's features at this scale.
