# References: Spotify recommendations and Discover Weekly

Keepers worth returning to.

## Primary / engineering
- spotify/annoy GitHub repo (random-projection forest ANN, mmap, build-once serve-many): https://github.com/spotify/annoy
- Erik Bernhardsson on Annoy (origin, how the trees and hyperplanes work): https://erikbern.com/2013/04/12/annoy.html
- ANN benchmarks (how Annoy compares to other nearest-neighbor libs): https://erikbern.com/2018/06/17/new-approximate-nearest-neighbor-benchmarks.html
- van den Oord, Dieleman, Schrauwen, "Deep content-based music recommendation," NeurIPS 2013 (CNN predicts latent factors from audio, solves cold start, mel-spectrogram 599x128, 7-8 layer CNN): http://papers.neurips.cc/paper/5004-deep-content-based-music-recommendation.pdf
- Sander Dieleman, "Recommending music on Spotify with deep learning" (readable companion to the paper): https://sander.ai/2014/08/05/spotify-cnns.html
- Chris Johnson (Spotify), "Algorithmic Music Recommendations at Spotify" slides (matrix factorization, implicit feedback, ANN, Hadoop/Spark): https://www.slideshare.net/MrChrisJohnson/algorithmic-music-recommendations-at-spotify

## Explainers / deep-dives
- Sophia Ciocca, "How Does Spotify Know You So Well?" (canonical three-pillar explainer: collaborative filtering + NLP + raw audio): https://medium.com/@sophiaciocca/spotifys-discover-weekly-how-machine-learning-finds-your-new-music-19a41ab76efe

## Concepts to reuse in future teardowns
- Matrix factorization: turn a giant sparse user-by-item matrix into small dense user vectors and item vectors; dot product = predicted affinity.
- Approximate Nearest Neighbors (ANN): trade exactness for speed, O(log N) per tree vs O(N) brute force. Annoy uses a forest of random-hyperplane binary trees.
- Cold start: new items have no interaction data; fix with content-based features (audio CNN, text, thumbnails) that predict the missing vector.
- Matching vs ranking: matching cheaply narrows millions to thousands, ranking scores those thousands precisely. Two different halves.
- Precompute + memory-map + cache: build heavy artifacts offline, mmap a read-only file shared across processes, serve cheap lookups live. Survives the 10M+ tier.
