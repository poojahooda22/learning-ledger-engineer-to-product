# References: Amazon item-to-item collaborative filtering

Keepers for the 2026-07-12 teardown on Amazon's "Customers who bought this also
bought" recommendation row.

## Primary sources

- **Linden, Smith, York (2003). "Amazon.com Recommendations: Item-to-Item
  Collaborative Filtering." IEEE Internet Computing 7(1):76-80.**
  DOI: 10.1109/MIC.2003.1167344
  https://dl.acm.org/doi/10.1109/MIC.2003.1167344
  The founding paper. Flips the axis from user-based to item-based CF; builds the
  similar-items table offline via cosine over item-vectors (each item = a vector
  over customers); online recommendation cost is independent of catalog size N
  and customer count M. States ~29 million customers and several million catalog
  items at the time. Worst case O(N^2 M), in practice much cheaper due to
  purchase-data sparsity.

- **Smith, Linden (2017). "Two Decades of Recommender Systems at Amazon.com."
  IEEE Internet Computing.**
  https://assets.amazon.science/76/9e/7eac89c14a838746e91dde0a5e9f/two-decades-of-recommender-systems-at-amazon.pdf
  20th-anniversary retrospective. IEEE Internet Computing picked the 2003 paper
  as the single "test of time" paper. Candid about weaknesses (recommends the
  obvious, ignores time/sequence, category loops) and the evolution toward latent
  factors, temporal dynamics, and deep/sequence models.

- **Amazon Science. "The history of Amazon's recommendation algorithm."**
  https://www.amazon.science/the-history-of-amazons-recommendation-algorithm

## Secondary / mirrors

- Paper mirror (UMD course): https://www.cs.umd.edu/~samir/498/Amazon-Recommendations.pdf
- Wikipedia, "Item-item collaborative filtering": https://en.wikipedia.org/wiki/Item-item_collaborative_filtering

## External estimate (label as such, not Amazon-published)

- McKinsey (2013): ~35% of Amazon purchases attributed to its recommendation
  engine. Widely cited, treat as external estimate.

## The one-line takeaway

Flip the axis (item-vectors over customers, not customer-vectors over items),
precompute the item-to-item similarity table offline, and the live
recommendation becomes a cheap keyed lookup whose cost tracks the few items the
user touched, not the millions in the catalog. Offline-think, online-lookup.
