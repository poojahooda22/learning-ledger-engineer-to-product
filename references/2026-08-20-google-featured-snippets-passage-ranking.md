# References: Google Featured Snippets and passage ranking (2026-08-20)

Keeper links behind the 2026-08-20 teardown.

## Primary (Google)

- Google, "Understanding searches better than ever before" (BERT launch, Pandu Nayak, Oct 25 2019). BERT affects ~1 in 10 English US queries at launch; applied to ranking AND featured snippets; expanded to 70+ languages Dec 2019; example queries ("2019 brazil traveler to usa need a visa", "can you get medicine for someone pharmacy", "parking on a hill with no curb"); served on Cloud TPUs for the first time. https://blog.google/products-and-platforms/products/search/search-language-understanding-bert/
- Google Search Help, "How Google's featured snippets work". Selection, formats (paragraph/list/table), quality raters, removal policy, not paid. https://support.google.com/websearch/answer/9351707
- Google Search Central, "Featured snippets and your website" (developer docs). https://developers.google.com/search/docs/appearance/featured-snippets

## Secondary (dates, numbers, technical framing)

- Search Engine Land, "FAQ: All about the BERT algorithm in Google search". https://searchengineland.com/faq-all-about-the-bert-algorithm-in-google-search-324193
- Search Engine Land, "Could Google passage indexing be leveraging BERT?" (512-token limit forces passage-sized chunks; DeepRank codename). https://searchengineland.com/could-google-passage-indexing-be-leveraging-bert-342975
- Search Engine Land, "Google algorithm updates 2020 in review: passage indexing". https://searchengineland.com/google-algorithm-updates-2020-in-review-core-updates-passage-indexing-and-page-experience-345070
- Sixth City Marketing, "Google Launches Passage Ranking in the U.S." (announced Search On Oct 2020; live English-US Feb 10-11 2021; ~7% of queries). https://www.sixthcitymarketing.com/2021/02/16/google-launches-passage-ranking/
- SiliconANGLE, "Google deploys new NLP models, Cloud TPUs to make its search engine smarter" (Oct 2019). https://siliconangle.com/2019/10/25/google-deploys-new-nlp-models-cloud-tpus-make-its-search-engine-smarter/
- MediaPost, "Meet BERT, Google's Latest Neural Algorithm For Natural-Language Processing" (Oct 28 2019). https://www.mediapost.com/publications/article/342485/meet-bert-googles-latest-neural-algorithm-for-na.html

## Engineering pattern (retrieve-then-rerank, well-grounded inference for Google's exact internals)

- MS MARCO passage ranking dataset (~8.8M passages from real Bing queries) and the standard bi-encoder retrieve then cross-encoder re-rank recipe. Sentence Transformers docs. https://sbert.net/examples/sentence_transformer/training/ms_marco/README.html
- Two-stage retrieval: bi-encoder (dual tower, independent query/passage vectors, ANN recall, fast) for matching; cross-encoder (joint query+passage attention, slow, accurate) for ranking the shortlist. Standard IR practice; mirrors the offline-embed/online-ANN pattern used across this ledger.

## Notes / caveats

- Google's exact production stack for featured snippet extraction and passage ranking is not fully public. The inverted-index + dense-retrieval + cross-encoder-rerank + extractive-span architecture is labeled inference in the report, grounded in confirmed facts (BERT applied to snippets, 512-token limit, TPU serving) plus the public IR literature and MS MARCO.
- "Zero-click" share (~half of searches) comes from third-party analyses (e.g. SparkToro), not Google; treat as estimate, not official.
