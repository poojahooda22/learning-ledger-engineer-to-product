# References: Netflix Downloads (offline viewing)

Keeper links for the 2026-10-08 teardown on Netflix Download for offline viewing
(Downloads, Smart Downloads, Downloads for You).

## Netflix official

- How to download titles to watch offline (expiry, per-title caps, Smart Downloads,
  ad-plan 15/device/month): https://help.netflix.com/en/node/113287
- How to change the video quality of a download (Standard = SD/less storage,
  High/Higher = up to 1080p/more storage; cannot change on non-HD Android/Fire):
  https://help.netflix.com/node/116071

## Smart Downloads (auto next episode)

- VentureBeat, Netflix launches Smart Downloads, 2018 (auto-download next episode on
  Wi-Fi, delete watched, Android first):
  https://venturebeat.com/media/netflix-launches-smart-downloads-to-queue-your-next-offline-episode-automatically/
- TechSpot, Smart Downloads enable offline binge-watching, 2018:
  https://www.techspot.com/community/topics/netflixs-smart-downloads-enable-offline-binge-watching.247711/

## Real storage measurements

- Digital Trends test (iPhone 15 Pro Max): 53-min episode "Eric" = 885.4MB High vs
  259.9MB Standard; 75-min film "Inside the Mind of a Dog" = 1.25GB High vs 290.8MB
  Standard (~3.4x to 4.4x smaller at Standard): https://www.digitaltrends.com/

## Codec (AV1)

- TVBEurope, Netflix adopts AV1 on Android, 2020 (about 20% more efficient than VP9,
  Save Data only, goal to roll out to all platforms, 10-bit, dav1d decoder):
  https://www.tvbeurope.com/media-management/netflix-adopts-av1-codec-for-android-users
- SiliconRepublic, Netflix AV1 data-saving on Android, 2020:
  https://siliconrepublic.com/comms/netflix-streaming-data-av1-android

## DRM (Widevine / persistent offline licenses)

- Bitmovin, Widevine security levels (L1 = TEE/secure hardware, up to 4K; L3 =
  software-only, typically SD):
  https://developer.bitmovin.com/playback/docs/widevine-security-levels-in-web-video-playback
- VdoCipher, Widevine overview (license expiration, device binding, persistent keys):
  https://www.vdocipher.com/blog/widevine-drm-hollywood-video/

## India / mobile-first context

- Telecoms.com, Netflix India mobile-only plan and heavy downloading, 2019
  (Indian members watch more on mobile than anywhere; Smart Downloads for low-signal
  areas): https://www.telecoms.com/wireless-networking/netflix-india-looks-for-growth-in-mobile-only-subscriptions
- Business Standard via PressReader, 2016 (high data costs drove download-then-watch
  behavior): https://www.pressreader.com/india/business-standard/20161205/281629599890269

## Limits

- Beebom, Netflix download limits (device caps by plan 1/2/4; ~100 titles/device;
  per-title caps set by licensors; ad-plan 15/month): https://beebom.com/netflix-download-limit/

## Cross-references within this ledger

- Netflix per-title / per-shot encoding (2026-09-11): the encode ladder reused to
  source downloads.
- Netflix Open Connect CDN (2026-07-13): delivers the download bytes; after that the
  phone serves itself.
- Netflix adaptive bitrate streaming (2026-06-30): the live ladder a download cannot
  walk.
- Netflix recommendations / item-to-item (2026-06-22, 2026-07-12): the ranking
  plausibly reused by Downloads for You.

Notes on confirmed vs inference: the DRM mechanism, security-level resolution caps,
AV1 efficiency, Smart Downloads behavior, the Standard/High setting, storage numbers,
expiry/limits, and India mobile context are confirmed by the sources above. Netflix
has NOT published its exact offline license durations, which codec each download uses
per device, the Downloads for You model, or the internal download pipeline; those are
clearly labeled as grounded inference in the report.
