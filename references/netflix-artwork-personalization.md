# References: Netflix artwork personalization (contextual bandits)

Saved 2026-06-19 for the Netflix personalized-artwork teardown.

## Primary (Netflix engineering)

- "Artwork Personalization at Netflix" (Netflix Tech Blog, Dec 2017)
  https://netflixtechblog.com/artwork-personalization-c589f074ad76
  The core source. Contextual bandit framing, "take fraction" reward, Replay
  off-policy evaluation, epsilon-greedy to closed-loop exploration spectrum,
  "1 of N images from the title image suite," Good Will Hunting and Stranger
  Things examples.

- "AVA: The Art and Science of Image Discovery at Netflix" (Netflix Tech Blog)
  https://netflixtechblog.com/ava-the-art-and-science-of-image-discovery-at-netflix-a442f163af6
  Candidate generation side. Frame annotation into visual / contextual /
  composition metadata; the Archer MapReduce-style chunked media pipeline.

- "ML Platform Meetup: Infra for Contextual Bandits and Reinforcement Learning"
  (Netflix Tech Blog)
  https://netflixtechblog.com/ml-platform-meetup-infra-for-contextual-bandits-and-reinforcement-learning-4a90305948ef
  Serving infra and closed-loop logging.

## Academic / talk

- Fernando Amat et al., "Artwork Personalization at Netflix," RecSys 2018
  https://www.slideshare.net/slideshow/artwork-personalization-at-netflix-fernando-amat-recsys2018-118208854/118208854
- Semantic Scholar: "Artwork personalization at Netflix" (Gil, Chandrashekar et al.)
  https://www.semanticscholar.org/paper/Artwork-personalization-at-netflix-Gil-Chandrashekar/9aa3dd841b2dc52a68e281ffd5ae508e2162d416

## Secondary explainers

- Eppo: "How Netflix, Lyft, and Yahoo use Contextual Bandits for Personalization"
  https://www.geteppo.com/blog/netflix-lyft-yahoo-contextual-bandits
- Analytics India Magazine: "Understanding AVA"
  https://analyticsindiamag.com/ai-features/understanding-ava-image-discovery-tool-used-by-netflix-to-power-its-content-posters/

## Key takeaways worth remembering

- Two halves: candidate generation (AVA, offline, heavy CV on Archer) and
  selection (contextual bandit, live, cheap dot-products + argmax).
- Reward is "take fraction": a quality play, not a click.
- Replay = match logged uniform-random impressions against the new policy's
  pick, average reward over the matched subset. Unbiased, but only ~1/N of logs
  match, which is the real reason the candidate set N is kept small.
- Live path stays O(N) in candidates, constant in catalog size: precompute member
  context, cache the pick per session for stability, never compile/scan live.
</content>
