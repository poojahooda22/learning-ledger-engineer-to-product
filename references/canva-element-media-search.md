# References: Canva element and media search (2026-10-20)

Keeper links for the teardown of Canva's in-editor search (query understanding,
BM25 plus dense-vector candidate fetch, re-ranking) over a 100 million-plus
asset library.

## Primary (Canva Engineering Blog)

- Search Pipeline: Part I (why the rebuild; at least four separate search
  systems with their own codebases; query representation decoupled from the
  engine so vector stores can be used):
  https://www.canva.dev/blog/engineering/search-pipeline-part-i/
- Search Pipeline: Part II (the new pipeline: Query Understanding, Candidate
  Generation, Re-ranking; validate then rewrite the query; multiple candidate
  generators under coordination with time or result-count deadlines; results
  merged into an in-memory structure; "allows us to use BM25 search, while
  re-ranking through dense vector spaces"; business rules including result
  diversity and free-versus-Pro; personalization and ML models; migrated audio,
  font, video and media search, templates next):
  https://www.canva.dev/blog/engineering/search-pipeline-part-ii/
- Image replacement in Canva designs using reverse image search (metadata misses
  nuanced visual features like subject count and colours; image embeddings and a
  vector index; 50,000 library images in an in-memory Faiss store; several
  models compared on a 200-image sample; a GPT-4o-description-to-CLIP-index
  approach performed worst):
  https://www.canva.dev/blog/engineering/image-replacement-in-canva-designs-using-reverse-image-search/

## Background on the class of problem

- Faiss (Facebook AI Similarity Search), the vector index library Canva used:
  https://github.com/facebookresearch/faiss
- HNSW approximate nearest neighbour (Malkov and Yashunin, 2016):
  https://arxiv.org/abs/1603.09320
- CLIP (Radford et al., 2021), multimodal text-and-image embeddings:
  https://arxiv.org/abs/2103.00020
- Reciprocal rank fusion (Cormack, Clarke, Buettcher, SIGIR 2009):
  https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf

## Scale figures (secondary, company-reported claims)

- Canva Pro premium-content page (more than 100 million stock photos, videos,
  graphics and audio): https://www.canva.com/pro/premium-content/
- Canva Create 2025 coverage: around 230 million monthly active users (up from
  about 180 million a year earlier); users created one billion designs in March
  2025; roughly 34 billion cumulative designs since 2013.

## Notes on confidence

- CONFIRMED by Canva's own posts: the pipeline shape (query understanding,
  candidate generation, re-ranking), BM25 retrieval with dense-vector
  re-ranking, the four-legacy-systems problem, the migrated content types, and
  the reverse-image-search prototype details (50,000 images, Faiss, 200-image
  comparison, GPT-4o approach worst).
- INFERENCE (grounded in standard practice, labeled as such in the report):
  two-tower dual encoders, embeddings precomputed at upload, ANN via IVF or HNSW
  at the 100 million tier, reciprocal rank fusion for merging, a heavier
  cross-encoder re-ranker on the few hundred survivors, sharding of both the
  inverted index and the vector index, and caching of hot queries.
- SECONDARY: all company scale numbers (users, designs, library size).
