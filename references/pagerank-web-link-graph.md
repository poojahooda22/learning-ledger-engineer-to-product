# References: PageRank and the web link graph

Saved for the 2026-07-18 Google Search PageRank teardown.

## Primary sources (papers)

- Sergey Brin, Lawrence Page. "The Anatomy of a Large-Scale Hypertextual Web Search
  Engine." WWW7, 1998. The founding paper. Has the PageRank formula, the random-surfer
  model, damping factor d = 0.85, and the empirical result that PageRank converged in
  52 iterations on a 322 million link database.
  http://infolab.stanford.edu/pub/papers/google.pdf
  (mirror: https://snap.stanford.edu/class/cs224w-readings/Brin98Anatomy.pdf)

- Lawrence Page, Sergey Brin, Rajeev Motwani, Terry Winograd. "The PageRank Citation
  Ranking: Bringing Order to the Web." Stanford InfoLab Technical Report, 1999.
  Fuller treatment: dangling nodes, teleport, convergence.
  http://ilpubs.stanford.edu:8090/422/

- Jeffrey Dean, Sanjay Ghemawat. "MapReduce: Simplified Data Processing on Large
  Clusters." OSDI 2004. PageRank as iterated Map/Reduce jobs is the canonical example
  of iterative graph batch processing.
  https://research.google.com/archive/mapreduce-osdi04.pdf

- Grzegorz Malewicz, Matthew Austern, Aart Bik, James Dehnert, Ilan Horn, Naty Leiser,
  Grzegorz Czajkowski. "Pregel: A System for Large-Scale Graph Processing." SIGMOD
  2010. Vertex-centric "think like a vertex" supersteps, combiners for hot nodes,
  checkpoint-based fault tolerance. The system that ran PageRank at Google scale;
  inspired Apache Giraph.
  https://15799.courses.cs.cmu.edu/fall2013/static/papers/p135-malewicz.pdf

## Books / references

- Amy N. Langville, Carl D. Meyer. "Google's PageRank and Beyond: The Science of
  Search Engine Rankings." Princeton University Press, 2006. The linear-algebra view:
  PageRank as the stationary distribution / dominant eigenvector, power iteration, why
  convergence rate is governed by the damping factor.

## Key facts pulled (fact vs inference)

Fact (primary-sourced):
- PageRank formula: PR(A) = (1-d) + d * sum over in-linkers Ti of PR(Ti)/C(Ti), where
  C(Ti) = out-degree of Ti, d = 0.85. (Brin-Page 1998)
- 52 iterations to converge on 322M links; convergence rate scales with d. (Brin-Page)
- Damping/teleport solves dangling nodes and spider traps and guarantees a unique
  stationary distribution. (Page et al. 1999; Langville-Meyer 2006)
- MapReduce (2004) and Pregel (2010) are Google's published batch/graph systems;
  PageRank is a stated example workload for both. (Dean-Ghemawat; Malewicz et al.)
- Web graph stored as sparse adjacency lists (never a dense matrix); power iteration
  is repeated sparse matrix-vector multiply. (standard, Langville-Meyer)

Inference / not public:
- The EXACT link-based scoring Google uses in 2026 is not published. Google has
  publicly said classic PageRank is one signal among hundreds and that it moved off
  the original algorithm years ago. The durable, sourced idea is: a query-independent,
  graph-derived quality score computed offline and read O(1) at query time. This
  teardown claims the algorithm and its scaling lineage as fact, NOT that the 1998
  formula ranks a live query today.
