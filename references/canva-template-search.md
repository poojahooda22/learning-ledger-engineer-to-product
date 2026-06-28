# References: Canva template search and ranking

Saved for the 2026-06-28 teardown on Canva template search.

## Primary (Canva Engineering)

- Search Pipeline: Part I : https://www.canva.dev/blog/engineering/search-pipeline-part-i/
  - ~50 distinct search entry points itemized; at least 4 separate search systems grown over time; motivation for one shared pipeline and a ubiquitous shared vocabulary.
  - Pipeline stages: query understanding/annotation -> candidate generation -> re-ranking -> interleaving.
- Search Pipeline: Part II : https://www.canva.dev/blog/engineering/search-pipeline-part-ii/
  - Componentized architecture; pipelines deployable in different topologies (query understanding as a remote call, candidate generators as microservices). Overfetching for precision/recall tradeoff. Post-fetch re-rankers reused across content types. Interleaving for ranking experiments.
- Migrating from Solr to Elasticsearch, and their differences : https://www.canva.dev/blog/engineering/migrating-from-solr-to-elasticsearch-and-their-differences/
  - Self-managed Solr needed a team to keep patched; moved to Elasticsearch 7.10 on AWS OpenSearch. Migration unlocks dense/sparse vector fields (binary doc_values). Both are Lucene underneath.
- How to improve search without looking at queries or results : https://www.canva.dev/blog/engineering/how-to-improve-search-without-looking-at-queries-or-results/
  - LLM-generated synthetic designs + queries for a static, reproducible, privacy-preserving offline eval. 1000+ test cases in under 10 minutes; 300+ offline evals in the time of one online interleaving experiment. Ranking test trick: degrade copies of a known-relevant design (lower text match, worse freshness) and check the ranker still surfaces it.

## Scale numbers

- Better Stack, "How Canva Scaled Their Search to Handle 1M+ Searches Per Minute" : https://newsletter.betterstack.com/p/how-canva-scaled-their-search-to
  - ~20,000 requests/second, 1,000,000+ searches/minute.
- The Social Shepherd, Canva statistics : https://thesocialshepherd.com/blog/canva-statistics
  - ~600,000 templates; 100M+ stock images/video/graphics; ~30 billion designs created since 2012.

## Background concepts

- Inverted index + BM25 (Lucene/Elasticsearch) : https://www.elastic.co/search-labs/blog/hybrid-search-elasticsearch
- Canva Design School, "Using search and Magic Design" (user-facing) : https://www.canva.com/design-school/resources/using-search-and-magic-design

## Notes on fact vs inference

- FACT (Canva-stated): pipeline stages, overfetching, interleaving, Solr->ES migration and vector-field unlock, synthetic offline eval and its numbers, scale numbers.
- INFERENCE (well grounded, labeled in the report): the exact hybrid lexical+vector blend used for templates in production, and the specific behavioral/quality features inside the re-rankers. Canva describes post-fetch re-rankers and the vector capability; precise production weights are not public. Standard solution-class reasoning used and labeled.
</content>
