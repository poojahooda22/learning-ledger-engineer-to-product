# References: YouTube Content ID (2026-07-08)

Keeper links for the Content ID fingerprint-matching teardown.

## Primary / canonical algorithm

- Avery Li-Chun Wang, "An Industrial-Strength Audio Search Algorithm," ISMIR 2003.
  The published, canonical version of the fingerprint-and-align class of problem:
  spectrogram peaks -> constellation map -> anchor/target combinatorial hashes
  (freq1, freq2, delta-t) -> inverted index (hash -> posting list of song, offset)
  -> match by histogram of time-offset differences (the diagonal line). Reports
  identifying a short noisy phone clip out of 1M+ tracks.
  https://www.ee.columbia.edu/~dpwe/papers/Wang03-shazam.pdf

- Google/YouTube patents, "Methods for identifying audio or video content":
  US 8,688,999 / 9,292,513 / 8,868,917. Describe the audio/video content
  identification family behind Content ID.
  https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/8688999

## Content ID facts, scale, and history

- Content ID, Wikipedia. Launch ~2007 ("Video Identification" trials June 2007),
  cost ($60M by 2016, $100M+ by 2018), payouts (~$2B by 2016), the three match
  types (audio, video, melody), policies (block/monetize/track), criticism.
  https://en.wikipedia.org/wiki/Content_ID

- VentureBeat, "YouTube: We've invested $100 million in Content ID and paid over
  $3 billion to rightsholders." Investment and payout figures.
  https://venturebeat.com/mobile/youtube-weve-invested-100-million-in-content-id-and-paid-over-3-billion-to-rightsholders/

- YouTube Help, "Content eligible for Content ID" and Google scale figures:
  100M+ active reference files, 500+ hours uploaded per minute, 2.2 billion
  Content ID claims in 2024 with over 99% automated.
  https://support.google.com/youtube/answer/2605065

- jdhao, "How Does The YouTube Content ID System Work?" Fingerprint overview and
  match/policy walkthrough.
  https://jdhao.github.io/2021/08/02/the_youtube_content_id_system/

## Video fingerprinting (the class-of-problem for the visual matcher)

- "Fast distributed video deduplication via locality-sensitive hashing with
  similarity ranking," ACM ICIMCS. Per-keyframe perceptual hash + LSH buckets for
  candidate retrieval, then verification by actual distance and temporal order.
  https://dl.acm.org/doi/10.1145/3007669.3007725

## The other side (false claims, the honest-uploader cost)

- EFF, "Content ID and the Rise of the Machines." The tradeoff toward
  over-claiming and its cost to legitimate creators (the "Arjun" side).
  https://www.eff.org/deeplinks/2016/02/content-id-and-rise-machines
