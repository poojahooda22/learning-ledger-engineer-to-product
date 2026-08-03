# References: Spotify Connect (device handoff)

Saved sources for the 2026-08-03 teardown. Primary-source-first.

## Spotify official
- Spotify Engineering, "Spotify's Player API: Your Toolkit for Controlling Spotify Programmatically" (2022): https://engineering.atspotify.com/2022/04/spotifys-player-api
  - Player API observes the player + Connect environment state; commands need state observation.
- Spotify for Developers, Web Playback SDK, "Spotify Connect": https://developer.spotify.com/documentation/web-playback-sdk/concepts/spotify-connect
- Spotify for Developers, Commercial Hardware, "Connect Basics": https://developer.spotify.com/documentation/commercial-hardware/implementation/guides/connect-basics
  - Embedded SDK callbacks incl. SpCallbackPlaybackNotify() with kSpPlaybackNotifyBecameActive.

## Open-source protocol implementations (documented, used by real hardware/projects)
- librespot (Rust Connect client), SPIRC source: https://github.com/librespot-org/librespot/blob/dev/connect/src/spirc.rs
  - SpircTask, ConnectState, notify()/send_state(), handle_cluster_update() (active_device_id, becomes inactive when another activates), handle_connection_id_update(), handle_transfer() (transfer_state), Reply::Success/Failure, UPDATE_STATE_DELAY 200ms, VOLUME_UPDATE_DELAY 500ms.
- librespot PR #1356, "Spirc: Replace Mercury with Dealer": https://github.com/librespot-org/librespot/pull/1356
- librespot DeepWiki overview: https://deepwiki.com/librespot-org/librespot
- go-librespot, "Network & Protocol Layer": https://deepwiki.com/devgianlu/go-librespot/4-network-and-protocol-layer
  - Dealer = WebSocket + JSON push for real-time commands; Spclient = REST + Protobuf; AP = Shannon + Mercury (legacy). URI-prefix message routing, request/reply via Request.Reply().
- librespot-golang SPIRC package docs: https://pkg.go.dev/github.com/anisse/librespot-golang/src/librespot/spirc
  - ConnectDevice (Name/Ident/Url/Volume), Hello/Notify/Load/Play/Pause/Goodbye frames, mDNS `_spotify-connect._tcp.`, login blob, ListMdnsDevices().

## Talks
- FOSDEM 2026, "Reverse Engineering the World's Largest Music Streaming Platform" (slides PDF): https://fosdem.org/2026/events/attachments/RNBQ8U-reverse-engineering-spotify/slides/267362/reverse_e_xy4vd0r.pdf
  - Post-2019 infra: Dealer (WebSocket+JSON), Spclient (REST+Protobuf), AP (Shannon/Mercury).

## Scale numbers
- TechCrunch (2024), Spotify crosses 600M MAU: https://techcrunch.com/2024/02/06/spotify-crosses-the-600m-monthly-active-users-mark/
- Backlinko, Spotify user stats (675M+ MAU, 293M premium, 100M+ tracks): https://backlinko.com/spotify-users

## Key takeaways worth reusing
- The one architectural decision: split control plane (kilobytes of state/commands) from data plane (megabytes of audio). The active device does the streaming; any device is the remote.
- Devices behind a NAT/router cannot accept inbound connections, so they dial OUT and hold a long-lived WebSocket (Dealer) open; the server pushes down it. Classic server-push-without-polling.
- One cluster per account with a single active_device_id = single source of truth. Fan-out is per-account (2-5 devices), never a global broadcast, which is why it scales.
- Debounce (200ms state / 500ms volume) is the cheap scalability win that tames write storms.
