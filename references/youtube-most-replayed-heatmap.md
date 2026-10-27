# References: YouTube "Most replayed" heatmap

Saved for the 2026-10-27 teardown.

## Launch and feature behavior (CONFIRMED)
- TechCrunch, launch to all users 18 May 2022:
  https://techcrunch.com/2022/05/18/youtubes-player-gains-new-features-including-most-replayed-video-chapters-single-loop-and-more
- 9to5Google: https://9to5google.com/2022/05/18/youtube-most-replayed/
- MobileSyrup (rolling out to all users): https://mobilesyrup.com/2022/05/18/youtubes-most-replayed-experimental-feature-is-now-rolling-out-for-all-users/
- iMore overview: https://www.imore.com/youtube-highlighting-popular-parts-videos-new-most-replayed-feature

## The public data structure (CONFIRMED via tooling that parses YouTube's InnerTube)
Path in player response:
`playerOverlays > decoratedPlayerBarRenderer > multiMarkersPlayerBarRenderer > markersMap`
entry key `heatmap` > `heatmapRenderer > heatMarkers[]`, each a `heatMarkerRenderer` with:
- `timeRangeStartMillis`   segment start
- `markerDurationMillis`   segment length
- `heatMarkerIntensityScoreNormalized`   float 0..1
About 100 markers per video (yt-dlp test fixture asserts `count:100`). Segment duration
= videoLength / 100. Rendered client-side as `svg.ytp-heat-map-svg` (1000x100 box), a
cubic-Bezier path.
- yt-dlp: https://github.com/yt-dlp/yt-dlp
- youtubei.js HeatMarker: https://ytjs.dev/api/namespaces/YTNodes/classes/HeatMarker
- youtubei.js MacroMarkersListEntity: https://ytjs.dev/api/namespaces/YTNodes/classes/MacroMarkersListEntity

## Creator-side (same raw data)
- YouTube Help, audience retention (absolute vs relative retention, >100% rewatch spikes):
  https://support.google.com/youtube/answer/9314415
- vidIQ guide (Shorts has no heatmap; creator retention visible even when public graph hidden):
  https://vidiq.com/blog/post/most-replayed-scenes-youtube-video

## Key facts to reuse
- Peak is always normalized to 1.0; curve shows shape of attention within one video, not an
  absolute count across videos.
- Threshold on view count before the graph is shown (noise control at low view counts).
- Measures re-watching, not enjoyment (a confusing mumbled word can be the peak).
- Internal aggregation pipeline is NOT public: offline bucket-and-normalize is INFERENCE,
  grounded in standard log-rollup practice; the served artifact and client rendering are
  CONFIRMED.
