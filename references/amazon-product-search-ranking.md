# References: Amazon product search ranking (2026-06-23)

Keeper links for the Amazon search teardown. Primary sources first.

## Primary (Amazon's own papers)

- Daria Sorokina and Erick Cantu-Paz, "Amazon Search: The Joy of Ranking Products", SIGIR 2016.
  - Amazon Science: https://www.amazon.science/publications/amazon-search-the-joy-of-ranking-products
  - ACM DL: https://dl.acm.org/doi/10.1145/2911451.2926725
  - What it gives us: matching vs ranking split; gradient boosted trees as the
    main ranking model; pairwise objective with NDCG; heavy reliance on
    behavioral features because product text is near-identical; day-to-day
    position-bias correction; per-category rankers blended in All Product Search;
    statement that search powers the majority of Amazon's sales.

- Priyanka Nigam et al., "Semantic Product Search", ACM SIGKDD (KDD) 2019.
  - ACM DL: https://dl.acm.org/doi/10.1145/3292500.3330759
  - PDF: https://arxiv.org/pdf/1907.00937
  - What it gives us: two-tower neural model (query tower + product tower in a
    shared embedding space); cheap averaged token embeddings chosen for latency;
    unigram/bigram/character-trigram tokenization; OOV tokens hashed into fixed
    buckets; trained to push purchased products above random and impressed-not-
    purchased negatives; product vectors precomputed and served via approximate
    nearest neighbor; merged with the lexical (inverted-index) candidate set.

## Secondary / context

- "Exploring Query Understanding for Amazon Product Search" (2024): https://www.arxiv.org/pdf/2408.02215
- Amazon A9 history and the COSMO (intent knowledge graph) and Rufus (conversational) layers:
  https://www.amalytix.com/en/knowledge/seo/amazon-alogrithm-a9/
  https://feedvisor.com/university/a9-search-engine/

## Background (the general way this class of problem is solved; labeled inference in the report)

- Inverted index and posting lists: https://milvus.io/ai-quick-reference/how-does-an-inverted-index-work
- BM25, inverted index, and dynamic pruning at scale: https://www.systemoverflow.com/learn/search-ranking/ranking-algorithms/bm25-implementation-inverted-index-and-dynamic-pruning-at-scale

## Scale figures used

- 350M+ products listed, ~60,000 searches/second: https://wifitalents.com/amazon-search-statistics/
- Majority of US product searches start on Amazon: https://www.emarketer.com/content/do-most-searchers-really-start-on-amazon
