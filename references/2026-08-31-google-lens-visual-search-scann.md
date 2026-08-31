# References: Google Lens visual search / ScaNN nearest-neighbor retrieval (2026-08-31)

Keeper links for the visual-search teardown. Vector search over billions of image embeddings.

## Primary / official
- ScaNN paper: Guo, Sun, Lindgren, Geng, Simcha, Chern, Kumar, "Accelerating Large-Scale Inference with Anisotropic Vector Quantization," ICML 2020. https://proceedings.mlr.press/v119/guo20h.html | arXiv:1908.10396 https://arxiv.org/abs/1908.10396
- ScaNN open-source library (google-research/scann): https://github.com/google-research/google-research/tree/master/scann
- SOAR paper: "SOAR: Improved Indexing for Approximate Nearest Neighbor Search," 2024. arXiv:2404.00774 https://arxiv.org/abs/2404.00774
- Google Research blog, "Announcing ScaNN: Efficient Vector Similarity Search": https://research.google/blog/announcing-scann-efficient-vector-similarity-search/
- Google Research blog, "SOAR: New algorithms for even faster vector search with ScaNN": https://research.google/blog/soar-new-algorithms-for-even-faster-vector-search-with-scann/
- Google Developers Blog, "See the Similarity: Personalizing Visual Search with Multimodal Embeddings": https://developers.googleblog.com/see-the-similarity-personalizing-visual-search-with-multimodal-embeddings/
- Google patent US11782998B2, "Embedding Based Retrieval for Image Search."
- Google Cloud, Vertex AI Vector Search (built on ScaNN): https://cloud.google.com/vertex-ai/docs/vector-search/overview

## Deep-dives / secondary
- RESONEO, "How Google Lens works" (patent-based teardown): https://think.resoneo.com/google-lens/
- ann-benchmarks.com, public ANN benchmark suite (glove-100 dataset): https://ann-benchmarks.com

## Key facts to remember
- ScaNN pipeline = partition (k-means, prune clusters) + anisotropic vector quantization (compress to ~bytes, score-aware loss penalizes parallel residual) + rescore top candidates with full vectors.
- Anisotropic loss beats plain product quantization because it shapes error to be harmless where the query is dissimilar; ~2x faster than other libs, SOTA on ann-benchmarks.
- SOAR = redundant assignment to a primary and orthogonality-chosen backup cluster; boundary points stay reachable, recall up for small index-size cost.
- MIPS = Maximum Inner Product Search: find database vectors with the largest inner product against the query.
- Brute-force k-NN scan cost is linear in catalog size; it breaks past a few million vectors and is impossible at billions inside a 2s answer.
