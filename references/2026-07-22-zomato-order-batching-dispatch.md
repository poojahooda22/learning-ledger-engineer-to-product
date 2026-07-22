# References: 2026-07-22 Zomato order batching and delivery-partner assignment

## Primary academic sources (the matching + batching framing on real Indian data)

- FoodMatch: Batching and Matching for Food Delivery in Dynamic Road Networks (Joshi, Singh, Rajkumar, Bhattacharya, et al., IIT Delhi). Maps vehicle assignment to minimum weight perfect matching on a bipartite graph; reduces batching to graph clustering; best-first search to build only the likely-optimal subgraph; angular distance to anticipate moving vehicles. Evaluated on real order data from a large Indian food delivery company.
  - arXiv: https://arxiv.org/abs/2008.12905
  - ACM Transactions on Spatial Algorithms and Systems: https://dl.acm.org/doi/10.1145/3494530
  - IEEE ICDE version: https://ieeexplore.ieee.org/document/9458617/
- Gigs with Guarantees: Achieving Fair Wage for Food Delivery Workers (fairness constraints layered on assignment): https://arxiv.org/pdf/2205.03530
- Crowdsourced on-demand food delivery: an order batching and assignment algorithm (Transportation Research Part C): https://www.sciencedirect.com/science/article/pii/S0968090X2300044X

## Primary engineering sources (how the class of problem is solved at scale)

- DoorDash Engineering, "Using ML and Optimization to Solve DoorDash's Dispatch Problem" (DeepRed): mixed integer program with binary dasher-to-order match variables, offer scoring, batching decisions, strategic delay, solved with Gurobi. https://careersatdoordash.com/blog/using-ml-and-optimization-to-solve-doordashs-dispatch-problem/
- DoorDash Engineering, "Next-Generation Optimization for Dasher Dispatch at DoorDash": https://careersatdoordash.com/blog/next-generation-optimization-for-dasher-dispatch-at-doordash/

## Zomato-specific signals

- Zomato Engineering, "The Deep Tech Behind Estimating Food Preparation Time" (FPT model that feeds edge costs / arrival timing): https://blog.zomato.com/food-preparation-time
- Zomato Engineering, "The elements of scalable machine learning": https://blog.zomato.com/elements-of-scalable-machine-learning
- Zomato Active Dispatch: Deep Q-Network based multi-agent reinforcement learning model that repositions idle delivery partners toward predicted demand using predicted supply (surfaced via Zomato engineering discussion; complementary to the live matching step).

## Supporting deep-dives

- NextBillion.ai, "Route Optimization for Food Delivery Apps: Lessons from Swiggy and UberEats" (batching threshold rule, ~8 to 10 minute added-time test): https://nextbillion.ai/blog/route-optimization-for-food-delivery-apps
- Ilya Zinkovich, "Evolution of Food Delivery Dispatching" (greedy nearest driver to bipartite matching to optimization): https://ilyazinkovich.github.io/2020/06/16/delivery-dispatching-evolution.html

## Key takeaways worth keeping

- Dispatch is an assignment problem, not a search problem. The winning move is to stop assigning on arrival and solve a whole window of orders together.
- Minimum weight bipartite matching is optimal but quadratic; the practical trick is best-first subgraph construction (never build an edge that cannot win), the same pruning idea as autocomplete top-k.
- Batching is graph clustering feeding into the matching: cluster orders with small combined detour, then match batches to partners. Club only under a time threshold with both deadlines intact.
- Scale survival: geo-shard the map so each zone's matching is independent and parallel; keep the live geo index in memory fed by GPS streaming; precompute and cache zone-to-zone route costs; let the batching window double as a load-shedding queue.
