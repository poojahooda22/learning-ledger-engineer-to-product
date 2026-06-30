# References: Netflix adaptive bitrate streaming (per-title/per-shot encoding + client ABR)

Saved for the 2026-06-30 teardown. Primary sources first.

## Offline encoding (Engine A)

- Netflix Technology Blog, "Per-Title Encode Optimization" (2015). The original convex-hull, per-title bitrate ladder post. Introduces measuring complexity per title and ~20% savings.
  https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2
  Mirror: http://techblog.netflix.com/2015/12/per-title-encode-optimization.html

- Netflix Technology Blog, "Dynamic optimizer — a perceptual video encoding optimization framework" (2018). Per-shot optimization, VMAF as objective, trellis/Lagrangian trajectory across shots.
  https://netflixtechblog.com/dynamic-optimizer-a-perceptual-video-encoding-optimization-framework-e19f1e3a277f

- Netflix Research, "Optimized Shot-based Encodes for 4K: Now Streaming!" (2020). Shot-based 4K rollout; the ~900-shots-per-episode figure.
  https://research.netflix.com/publication/optimized-shot-based-encodes-for-4k-now-streaming
  Blog version: https://medium.com/netflix-techblog/optimized-shot-based-encodes-now-streaming-4b9464204830

- Streaming Learning Center, "How Netflix Pioneered Per-Title Video Encoding Optimization." Deep-dive recap with the old fixed ladder table (235 kbps/320x240 ... 5800 kbps/1920x1080).
  https://streaminglearningcenter.com/encoding/how-netflix-pioneered-per-title-video-encoding-optimization.html

## Quality metric

- Netflix/VMAF GitHub repo (open source). nuSVR fusing VIF + DLM + temporal/motion features against human opinion scores.
  https://github.com/Netflix/vmaf
- VMAF overview: https://en.wikipedia.org/wiki/Video_Multimethod_Assessment_Fusion

## Live client ABR (Engine B)

- Huang, Johari, McKeown, Trunnell, Watson, "A Buffer-Based Approach to Rate Adaptation: Evidence from a Large Video Streaming Service" (Netflix, SIGCOMM 2014). BBA reservoir+cushion, ~20% rebuffer reduction across 500k+ users, why throughput estimation lags.
  https://arxiv.org/pdf/1401.2209

- Spiteri, Urgaonkar, Sitaraman, "BOLA: Near-Optimal Bitrate Adaptation for Online Videos" / "From Theory to Practice: Improving Bitrate Adaptation in the DASH Reference Player." Lyapunov-optimization ABR, now in dash.js.
  https://dl.acm.org/doi/fullHtml/10.1145/3336497

## Key numbers to remember

- Old fixed ladder: 10 rungs, 235 kbps (320x240) up to 5800 kbps (1920x1080).
- Per-title savings ~20% at equal perceived quality; per-scene/per-shot pushes toward ~30%.
- Stranger Things episode processed as ~900 shots.
- Buffer-based ABR cut rebuffer rate ~20% vs throughput-based, tested on 500k+ users.
- Client buffer can hold up to ~240 seconds (the shock absorber for tunnels).
- VMAF: 0 to 100 perceptual score, SVM (nuSVR) over VIF, DLM, temporal features.

## Pattern links to other ledger entries

- Offline-think / online-lookup: same spine as Spotify Discover Weekly (2026-06-13), YouTube recommendations (2026-06-22), Razorpay routing (2026-06-26).
- Embarrassingly-parallel media fan-out: same shape as Netflix artwork pipeline (2026-06-19, Archer/Cosmos).
- CDN delivery (Open Connect) deliberately out of scope here; covered in the "CDN zero origin egress" lesson (Day 4).
