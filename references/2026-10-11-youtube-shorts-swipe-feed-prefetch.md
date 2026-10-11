# References: YouTube Shorts swipe feed and prefetch (2026-10-11 teardown)

Keeper links for the Shorts teardown. Primary sources first. The Shorts feed is not documented at the model level by YouTube, so the recommendation pipeline is confirmed from YouTube's published papers and the Shorts-specific tuning plus the exact prefetch depths are clearly labeled inference from the short-form-video research literature.

## Primary / confirmed

- **Covington, Adams, Sargin, "Deep Neural Networks for YouTube Recommendations," RecSys 2016.** The two-stage funnel (candidate generation shrinks the catalog to hundreds, a separate ranking model scores only those), approximate-nearest-neighbor serving of learned embeddings, watch-time objective. The spine of Engine A.
  https://research.google/pubs/pub45530/

- **Zhao, Hong, Chi et al, "Recommending What Video to Watch Next: A Multitask Ranking System," RecSys 2019.** The real ranking engineering: Multi-gate Mixture-of-Experts (MMoE) for competing objectives, and a "shallow tower" that learns and removes position/selection bias. Final ranking is a hand-weighted blend of the predicted objectives.
  https://research.google/pubs/pub49380/
  DOI: 10.1145/3298689.3346997

- **YouTube Help, "Search & discovery on Shorts" (official).** Names the signals: whether viewers watch/skip/mark not interested, how long they stay, likes, post-watch surveys, percentage who chose to view, average view duration, average percentage viewed, plus past Shorts/channels enjoyed and trending sounds.
  https://support.google.com/youtube/answer/11914225
  (Note: support.google.com was blocked by the egress proxy during this run; signal list is from Google's indexed summary of the page. Re-verify the live page when possible.)

## Short-form delivery / prefetch research (Engine B; grounded class-level, not YouTube-confirmed internals)

- **Nguyen et al, "Network-aware Prefetching Method for Short-Form Video Streaming," 2022.** Separate buffer thresholds for the current video vs the next videos in the recommended list; the core trade-off that prefetching too little causes rebuffering and prefetching too much in a fixed order/size causes "significant rebuffering and bandwidth wastage" because skipped videos discard their prefetched segments.
  https://arxiv.org/abs/2209.02927

- **"Bandwidth-Efficient Multi-video Prefetching for Short Video Streaming," 2022.** Prefetching across multiple upcoming videos while bounding wasted bandwidth.
  https://arxiv.org/pdf/2206.09839

- **"Digital Twin-Assisted Adaptive Preloading for Short Video Streaming," 2023.** Adaptive preload sizing based on network and viewing behavior; reports substantial startup-latency and data-waste reductions versus fixed baselines (treat exact percentages as indicative of the research class, not YouTube-published figures).
  https://arxiv.org/pdf/2307.07836

- Context from this ledger: adaptive bitrate was torn down for long-form Netflix (2026-06-30) and CDN edge delivery as Netflix Open Connect (2026-07-13); both reappear here under a much tighter per-swipe deadline.

## Scale figures (as reported)

- **200 billion+ average daily Shorts views** stated by YouTube CEO Neal Mohan at Cannes Lions, June 2025, repeated in YouTube's August 2026 Partner Program update. Earlier official figure was **70 billion daily views in 2023**.
  https://blog.youtube/

- **View-counting change, effective March 31, 2025:** a Shorts view now counts when the Short starts to play or is replayed, rather than only after a watch threshold. This makes the 70B and 200B figures not directly comparable.

- **Shorts length cap raised from 60 seconds to 3 minutes, effective October 15, 2024** (square or vertical uploads only); any upload of 3 minutes or less now registers as a Short.
  https://www.socialmediatoday.com/news/youtube-expands-3-minute-shorts-to-all-users/736967/
  https://www.engadget.com/entertainment/youtube/youtube-shorts-can-now-run-up-to-three-minutes-160002081.html
