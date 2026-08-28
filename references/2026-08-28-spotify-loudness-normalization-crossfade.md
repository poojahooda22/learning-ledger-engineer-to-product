# References: Spotify loudness normalization and crossfade (2026-08-28)

## Primary / official
- Spotify for Artists, "Loudness normalization on Spotify": https://support.spotify.com/us/artists/article/loudness-normalization/
  - Target -14 LUFS (ITU-R BS.1770). Listener "Volume level": Loud -11, Normal -14 (default), Quiet -19.
  - Gain applied at playback, file not re-encoded. Negative gain for loud masters (no distortion), positive gain for quiet masters.
  - Album normalization for album playthrough so inter-track dynamics are preserved.
  - Limiter: engages at -1 dB (sample values), 5 ms attack, 100 ms decay.
  - Headroom rule: leaves ~1 dB for lossy. Example: -20 LUFS track with -5 dBTP true peak is only lifted to -16 LUFS, not -14.
- Spotify support, "Tracks transitions" (crossfade 0-12 s, gapless toggle): https://support.spotify.com/us/article/tracks-transitions/

## Standards
- Recommendation ITU-R BS.1770-5 (11/2023) PDF: https://www.itu.int/dms_pubrec/itu-r/rec/bs/R-REC-BS.1770-5-202311-I!!PDF-E.pdf
  - K-weighting filter, 400 ms gating blocks with 75% overlap, mean-square power.
  - Two-stage gate: absolute -70 LUFS, then relative gate 10 dB below the absolute-gated loudness.
- Wikipedia LUFS (LKFS == LUFS): https://en.wikipedia.org/wiki/LUFS
- ReplayGain (store-a-gain-tag design): https://en.wikipedia.org/wiki/ReplayGain

## DSP background
- Equal-power crossfade: gainA = cos(t*pi/2), gainB = sin(t*pi/2); midpoint 0.707 = -3 dB; cos^2+sin^2=1 keeps power constant. Linear fade dips to -6 dB (half power) = audible hole.
  - Audacity Manual: https://manual.audacityteam.org/man/fade_and_crossfade.html
  - Signalsmith Audio: https://signalsmith-audio.co.uk/writing/2021/cheap-energy-crossfade/
- Fora Soft loudness normalization overview: https://www.forasoft.com/learn/audio-for-video/articles-audio/loudness-normalization-ebu-r128-bs1770-atsc-a85
- Sage Audio, mastering for streaming (per-platform targets): https://www.sageaudio.com/articles/mastering-for-streaming-platform-loudness-and-normalization-explained

## Key transfer to Rare.lab
1. Measure each asset's intensity once offline (BS.1770-style gated measure), store one normalization scalar in the compiled artifact, apply as a per-frame multiply at runtime. Guard positive gain with a peak limiter (white-point clamp) like Spotify's -1 dB limiter.
2. Cross-dissolve between shader states on an equal-power curve (cos/sin), never linear, so transition intensity never dips at the midpoint.
