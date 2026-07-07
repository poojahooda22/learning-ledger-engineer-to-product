# References: Spotify instant playback and streaming

Saved 2026-07-07 for the Spotify streaming teardown.

## Primary source (the paper)

- Gunnar Kreitz, Fredrik Niemelae. "Spotify: Large Scale, Low Latency, P2P
  Music-on-Demand Streaming." IEEE 10th Intl Conf on P2P Computing (P2P 2010).
  - IEEE Xplore: https://ieeexplore.ieee.org/document/5569963/
  - Author copy + slides: https://kreitz.se/spotify-p2p10/
  - Mirror PDF: https://people.cs.umass.edu/~phillipa/CSE390/spotify-p2p10.pdf

## Key numbers to remember (from the 2010 paper)

- Median tap-to-sound playback latency: 265 ms. 75th pct: 515 ms. 90th pct: 1047 ms.
- Data source split of all music played:
  - Local cache: 55.4 percent
  - Peer-to-peer overlay: 35.8 percent
  - Spotify servers: 8.8 percent
- Stutter: below 1 percent of playbacks.
- Format: Ogg Vorbis, ~160 kbps normal, ~320 kbps high.
- Track handled as pieces of 16 kB (fetch/play different pieces from different sources in parallel).
- Local cache default: 10 percent of free disk, bounded roughly 56 MB to 10 GB, LRU-weighted eviction.
- P2P overlay: unstructured, no supernodes/DHT, max ~60 neighbors per client, TCP.
- Peer discovery: (1) a tracker server (per-track list of recent peers, returns ~10) AND (2) a bounded broadcast query to neighbors (a small-radius BFS over the peer graph).
- Server used only for the urgent head + emergency re-requests on low buffer; client throttles the server down once P2P/cache warm up.
- Prefetch: head of the next queued track prefetched near the end of the current one -> gapless.
- 2011 stat: of tracks not played via the web player, ~80 percent went through P2P.

## The second era (why P2P died)

- April 2014: Spotify shut down the desktop P2P network, went full server + CDN.
  - TechCrunch: https://techcrunch.com/2014/04/17/spotify-removes-peer-to-peer-technology-from-its-desktop-client/
  - TorrentFreak: https://torrentfreak.com/spotify-starts-shutting-down-its-massive-p2p-network-140416/
  - Engadget: https://www.engadget.com/2014-04-17-spotify-phases-out-peer-to-peer.html
- Reasons: mobile (which never ran P2P) became the majority; CDN + cloud got cheap
  and simple; P2P was a large fragile code/security surface. Spotify migrated its
  backend to Google Cloud Platform in 2016.
- Modern shape: cache-first on device, urgent head + bulk from nearest CDN edge
  (encrypted audio chunks cached at the edge), origin only on a cache miss.

## The reusable spine

Split the load by URGENCY, not by source: serve only the urgent head from the
expensive place, fill the bulk from the cheapest available source, keep a boring
reliable path as the safety net, and delete the clever subsystem the moment a
simpler one does its job. Offline-think / online-lookup: the live path is a cache
hit or a small piece from nearby.
