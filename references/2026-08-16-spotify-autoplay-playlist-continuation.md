# References: Spotify Autoplay / Automatic Playlist Continuation (2026-08-16)

Saved sources for the Autoplay teardown. Autoplay is real-time session continuation, distinct from Discover Weekly (batch) and instant playback (cache/CDN).

## The task framing (playlist continuation as sequential recommendation)

- Spotify Research, "Recsys Challenge 2018: Automatic Music Playlist Continuation." Defines APC as a form of sequential recommendation: given a playlist of arbitrary length, recommend up to 500 tracks that fit. 1,000,000 playlists; main track 113 teams / 1,228 runs; creative track 33 teams / 239 runs; top main-track R-precision 0.2241, NDCG 0.3946, avg recommended-song clicks 1.784.
  https://research.atspotify.com/publications/recsys-challenge-2018-automatic-music-playlist-continuation
- Spotify Research, "An Analysis of Approaches Taken in the ACM RecSys Challenge 2018 for Automatic Music Playlist Continuation."
  https://research.atspotify.com/publications/an-analysis-of-approaches-taken-in-the-acm-recsys-challenge-2018-for-automatic-music-playlist-continuation
- ACM RecSys Challenge 2018 details page.
  https://recsys-challenge.spotify.com/details
- Spotify Million Playlist Dataset (Zenodo). 1,000,000 playlists created Jan 2010 to Oct 2017; re-released as the Million Playlist Dataset Challenge in Sept 2020.
  https://zenodo.org/records/6425593
- arXiv 1808.04288, "Automatic Playlist Continuation through a Composition of Collaborative Filters."
  https://arxiv.org/abs/1808.04288
- arXiv 1901.00450, "Automatic playlist continuation using a hybrid recommender system combining features from text and audio."
  https://arxiv.org/pdf/1901.00450

## Ranking brain: bandits (BaRT)

- McInerney et al., "Explore, Exploit, Explain: Personalizing Explainable Recommendations with Bandits" (RecSys 2018). The foundational BaRT paper. Multi-armed / contextual bandits balance exploiting known preferences vs exploring uncertain items; reward is user satisfaction. New users lean explore, established users lean exploit.
  https://research.atspotify.com/publications/explore-exploit-explain-personalizing-explainable-recommendations-with-bandits
- BaRT = "Bandits for Recommendations as Treatments"; governs Home ordering, items within, and explanations. Autoplay and Radio use audio-similarity + session-continuation signals and share the same underlying machinery (secondary summary, useful for the mental model).
  https://dynamoi.com/learn/spotify-algorithm/what-is-spotify-bart-algorithm

## Matching: track embeddings + approximate nearest neighbor

- spotify/annoy (GitHub). Approximate Nearest Neighbors in C++/Python, memory-mapped, forest of random-projection trees. Built by Erik Bernhardsson at Spotify (~2013 to 2015). Powered Discover Weekly, Home, Radio.
  https://github.com/spotify/annoy
- Spotify Engineering, "Introducing Voyager: Spotify's New Nearest-Neighbor Search Library" (2023). HNSW-based successor to Annoy; higher recall + speed; production Java/Python bindings; now Spotify's recommended ANN library. NOTE: engineering.atspotify.com was egress-blocked during the run; cited from public title + repo.
  https://engineering.atspotify.com/2023/10/introducing-voyager-spotifys-new-nearest-neighbor-search-library
- InfoQ coverage of Voyager (secondary, was egress-blocked this run).
  https://www.infoq.com/news/2023/11/spotify-ann-voyager/
- Track embeddings background: 128- or 256-dim vectors from matrix factorization (collaborative) plus audio features (tempo, key, energy, loudness, valence, danceability; Echo Nest lineage). Secondary explainer.
  https://python.plainenglish.io/what-spotify-hears-before-you-do-ca7a86be3e20

## Notes on fact vs inference

- CONFIRMED (Spotify / primary): APC as sequential recommendation and the RecSys 2018 dataset/results; BaRT bandit approach and explore/exploit-by-certainty; Annoy design and authorship; Voyager as HNSW successor to Annoy.
- GROUNDED INFERENCE (labeled in the report): that production Autoplay specifically uses the BaRT machinery plus session-continuation and audio-similarity signals (Spotify says Autoplay/Radio share Home's machinery, but exact Autoplay serving code is not published); the exact candidate-set sizes and the per-request pipeline order; the hot-seed caching strategy for global album drops.
- Exact Voyager-vs-Annoy speed/recall/memory numbers live on the Spotify engineering blog, which could not be fetched this run; only the architectural claim (HNSW replacing random-projection trees) is asserted.
