# References: Uber batched matching and dispatch (DISCO)

Keeper links for the 2026-07-02 teardown.

## Primary (Uber / Lyft engineering and research)

- Matt Ranney, "Scaling Uber's Real-time Market Platform" (InfoQ). The classic
  account of the Supply Service, Demand Service, and DISCO, and of sharding the
  world with Google's S2 cells so dispatch is geographically partitioned.
  https://www.infoq.com/presentations/uber-market-platform/
- Uber Engineering, "H3: Uber's Hexagonal Hierarchical Spatial Index." The
  hexagonal grid used for pricing and marketplace features; k-ring neighbor
  expansion; why hexagons (all six neighbors equidistant, low quantization error
  as people move).
  https://www.uber.com/en/blog/h3/
- Uber Engineering, "DeepETA: How Uber Predicts Arrival Times Using Deep
  Learning." A transformer post-processes a routing-engine estimate; ETA is the
  edge weight that drives matching, served fast enough to fit the batch budget.
  https://www.uber.com/en/blog/deepeta-how-uber-predicts-arrival-times/
- Uber Engineering, "Reinforcement Learning for Modeling Marketplace Balance."
  Driver value functions so an edge cost reflects not just this trip but where
  the assignment leaves the marketplace next.
  https://www.uber.com/blog/reinforcement-learning-for-modeling-marketplace-balance/
- Uber Engineering, "Engineering More Reliable Transportation with Machine
  Learning and AI at Uber." Batch matching that optimizes globally; the reported
  serving shape of tens of thousands of predictions within about 100ms per batch.
  https://www.uber.com/us/en/blog/machine-learning/
- "A Better Match for Drivers and Riders: Reinforcement Learning at Lyft"
  (arXiv 2310.13810). Same problem class from the other big US player: batch
  matching plus learned value functions.
  https://arxiv.org/pdf/2310.13810

## Secondary / background

- Cornell INFO 2040 course blog, "Uber and Lyft's Batch-Matching Markets." Clean
  intuition for why batching beats greedy first-dispatch.
  https://blogs.cornell.edu/info2040/2019/10/17/uber-and-lyfts-batch-matching-markets/
- Hungarian (Kuhn-Munkres) algorithm for the assignment problem, the O(n^3)
  exact solver that frames the min-cost bipartite matching.
  https://en.wikipedia.org/wiki/Hungarian_algorithm

## The one-line takeaway

Match is two halves. Candidate generation is a spatial lookup (cell + k-ring, plus
forward dispatch for soon-to-be-free cars), so cost tracks the neighborhood, not
the city. Assignment is a min-cost bipartite matching over a batch collected in a
short on-purpose window, so the whole street is optimized at once instead of
greedily one request at a time. Geo-shard for linear scale, cap and sub-shard the
hot cell, serve ETAs from cache. Retention is the liquidity flywheel, not a click
loop.
