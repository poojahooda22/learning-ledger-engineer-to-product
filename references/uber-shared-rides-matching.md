# References: Uber shared rides matching (Pool / UberX Share)

Keeper sources for the 2026-08-24 teardown on ride-pooling / shared-rides matching.

## Primary research

- **Santi, Resta, Szell, Sobolevsky, Strogatz, Ratti. "Quantifying the benefits
  of vehicle pooling with shareability networks." PNAS 2014.**
  https://www.pnas.org/doi/10.1073/pnas.1403657111
  Origin of the shareability network: node = trip request, edge = two requests
  can be shared within a delay bound; pairing = maximum matching on that graph.
  New York taxi data; a large share of trips shareable in pairs with only a few
  minutes of delay, cutting trips needed by roughly 40%.

- **Alonso-Mora, Samaranayake, Wallar, Frazzoli, Rus. "On-demand high-capacity
  ride-sharing via dynamic trip-vehicle assignment." PNAS 2017, 114(3):462-467.**
  DOI 10.1073/pnas.1611675114
  https://www.pnas.org/doi/abs/10.1073/pnas.1611675114
  PubMed: https://pubmed.ncbi.nlm.nih.gov/28049820/
  The RV-graph (request-vehicle prune) -> RTV-graph (request-trip-vehicle, with
  the cliques-only rule for building bigger trips) -> ILP assignment pipeline.
  Anytime-optimal: valid greedy answer first, improve toward optimum under a
  deadline. Headline: 2,000 vehicles of capacity 10 (about 15% of NYC's taxi
  fleet), or 3,000 of capacity 4, serve 98% of demand at 2.8 min mean wait and
  3.5 min mean in-car delay.

- **Unofficial reference implementation (MetaZuo/RideSharing) on GitHub.**
  https://github.com/MetaZuo/RideSharing
  Code for RV-graph, RTV-graph, ILP assignment, and idle-vehicle rebalancing.
  Useful for seeing the data structures made concrete.

## Uber engineering / marketplace framing

- **Uber Engineering. "Reinforcement Learning for Modeling Marketplace Balance."**
  https://www.uber.com/us/en/blog/reinforcement-learning-for-modeling-marketplace-balance/
  How Uber frames matching as a marketplace-efficiency optimization rather than a
  per-request greedy assignment.

## Surveys and secondary explainers

- **"Increasing Shareability in Ride-Pooling Systems," in Reengineering the
  Sharing Economy (Cambridge University Press).**
  https://www.cambridge.org/core/books/reengineering-the-sharing-economy/increasing-shareability-in-ridepooling-systems/43C6E5E26B6E0A54332A692A0F3DE2F4
  Survey of detour caps, shareability, and the pairing-then-assignment structure.

- **Secondary explainer on production matching (batching windows, Hungarian vs
  Jonker-Volgenant on sparse cost matrices, greedy fallback above ~400 requests
  in a zone). Not an Uber primary source; treat as color.**
  https://www.frugaltesting.com/blog/how-uber-prepares-its-ride-matching-app-for-high-demand

## Key numbers to remember

- Detour factor cap: commonly ~1.25 (shared trip at most 25% longer than solo),
  plus a maximum extra wait.
- All-pairs shareability at 100k open requests is about 5 billion pairs, so
  spatial (H3/S2 cell) plus time-bucket pruning is mandatory before building the
  graph.
- Set packing (a car assigned a rider-set with order-dependent cost) is NP-hard,
  which is why the pipeline prunes hard first, then runs an anytime-bounded ILP.
