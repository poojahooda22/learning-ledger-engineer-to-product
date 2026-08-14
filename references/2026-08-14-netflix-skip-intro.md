# References: Netflix Skip Intro (2026-08-14)

Keeper links for the Skip Intro teardown. The core mechanic (find the intro by its repetition across episodes) is a fingerprint-and-match problem, so the Shazam paper is the load-bearing primary source, backed by Netflix's own media-ML engineering writing.

## Primary / canonical

- Avery Li-Chun Wang, "An Industrial-Strength Audio Search Algorithm," ISMIR 2003. The Shazam paper. Spectrogram peaks, constellation map, combinatorial anchor-target hashing, and the histogram-of-time-offsets matching trick reused here for within-season self-similarity.
  https://www.ee.columbia.edu/~dpwe/papers/Wang03-shazam.pdf
  Reproduction repo: https://github.com/arjunbahuguna/repoducing-shazam

- Netflix Technology Blog, "Match Cutting: Finding Cuts with Smooth Visual Transitions Using Machine Learning" (2023). Shot segmentation, starting from millions of shot pairs, image/video/audio/audio-visual feature extractors, classification + metric learning. Intro detection is a constrained cousin of this.
  https://netflixtechblog.com/match-cutting-at-netflix-finding-cuts-with-smooth-visual-transitions-31c3fc14ae59
  Paper: https://arxiv.org/pdf/2210.05766

- Netflix Technology Blog, "Scaling Media Machine Learning at Netflix." Precompute expensive media features once, store them, let many models read them; shot boundaries as a single source of truth.
  https://netflixtechblog.com/scaling-media-machine-learning-at-netflix-f19b400243

- Netflix Technology Blog, "Simplifying Media Innovation at Netflix with Archer." Chunked parallel media processing on Titus containers.
  https://medium.com/netflix-techblog/simplifying-media-innovation-at-netflix-with-archer-3f8cbb0e2bcb

- Netflix Technology Blog, "The Making of VES: the Cosmos Microservice for Netflix Video Encoding." Cosmos as a workflow-driven media-microservice platform with queue-depth autoscaling.
  https://netflixtechblog.com/the-making-of-ves-the-cosmos-microservice-for-netflix-video-encoding-946b9b3cd300

## Third-party deep-dives (Skip Intro specifics; treat as inference)

- "Netflix's Skip Intro Feature, How the Hell do They do That?" audio-sample query across episodes, first-2-minutes constraint, computer-vision title verification, manual review.
  https://medium.com/an-attempt-at-writing/netflixs-skip-intro-feature-how-the-hell-do-they-do-that-7c5db9408f82
- "Feature Teardown: Netflix's Skip Intro," Eugene Leychenko, The Startup.
  https://medium.com/swlh/feature-teardown-netflixs-skip-intro-15cac114a136
- Ask HN: "What is the engineering behind Netflix's Skip Intro button?"
  https://news.ycombinator.com/item?id=20850537

## Fact vs inference note

Confirmed by Netflix's own writing: shot-boundary detection as a shared primitive, the Match Cutting multimodal pipeline, precomputed reusable media features, and the Archer/Cosmos/Titus batch infrastructure. NOT confirmed in a single end-to-end Netflix blog: the exact Skip Intro pipeline (audio fingerprint by repetition -> CV title verify -> human edge-review). That specific chain is well-grounded inference from credible deep-dives plus how-this-class-of-problem-is-solved. The Shazam mechanics are exact and public. Reclaimed-time figures are Netflix marketing estimates.
