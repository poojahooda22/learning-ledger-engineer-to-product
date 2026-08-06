# Day 49, How does live voice and video for 2.6 million concurrent people not fall over the moment a fourth friend joins a call?

**Date:** 2026-08-06
**Difficulty:** Expert
**Topic:** Real-time audio and video at scale: why peer-to-peer mesh WebRTC calls die around 4 to 6 participants, why a naive central mixing server (an MCU) just moves that same wall onto the server's CPU, and how Discord's Selective Forwarding Unit (SFU) architecture, simulcast, and receiver-driven stream selection serve millions of concurrent voice users by forwarding bytes instead of reprocessing them.
**Stack relevance:** Rare.lab's runtime shares one WebGL context across an embedded session. The exact question this lesson answers for audio and video, how do you serve many simultaneous consumers off one expensive shared resource without recomputing that resource once per consumer, is the same question Rare.lab will hit the moment more than one viewer needs to watch the same live scene at once.

---

## 1. The company and the breaking number

**Discord.** In a 2018 engineering post titled for a 2.5 million concurrent voice user milestone, Discord's own numbers by the time of writing had already climbed past that: more than **2.6 million concurrent voice users**, served off more than **850 voice servers** spread across **13 regions** in over **30 data centers**, moving more than **220 Gbps** of egress traffic and **120 million packets per second** at peak. Those are Discord's own reported figures, not a secondary estimate.

The number that breaks the naive design is much smaller and much easier to reproduce on a single laptop: **peer-to-peer mesh bandwidth per participant grows as K times (N minus 1)**, where K is the bitrate of one stream and N is the number of people on the call. At a conservative 500 kbps per video stream, a 6-person mesh call needs each participant to simultaneously upload 5 separate encoded streams and download 5 more, roughly 2.5 Mbps up and 2.5 Mbps down for video alone, on top of audio. Across the whole call that is 30 individually managed streams for 6 people. Industry consensus documented across WebRTC engineering writing (bloggeek.me, among others) puts mesh calls as genuinely fine for a 1:1 call or a 3-person standup, and reliably falling apart by the time a 5th or 6th person joins, and it fails on CPU as much as on bandwidth: a laptop that was encoding video once is now running multiple simultaneous encoder instances, and video encoding, not decoding, is the expensive half of a codec.

---

## 2. Why the naive (demo) design dies

The demo version of group video calling is `RTCPeerConnection` per remote participant, wired directly client to client: everybody calls everybody, no server in the media path except for signaling. This is the standard first WebRTC tutorial, and it genuinely works for two people. It fails for three specific, compounding reasons once a third or fourth person joins.

**Uplink bandwidth grows linearly with participants, and consumer uplink is the scarce side.** Most home and mobile connections are asymmetric: download capacity is generous, upload capacity is not. A mesh call forces every participant's outbound bandwidth to scale as K*(N-1), so the connection's weakest link, upload, is exactly the one the naive design multiplies fastest. A participant with a perfectly fine 10 Mbps download and a modest 2 Mbps upload can watch four incoming streams without trouble but cannot send four outgoing ones.

**CPU cost grows the same way, and it hits the expensive half of the codec.** Sending N-1 separate encoded streams generally means running N-1 encoder passes (even where some optimization shares work across them, the cost still scales with N, not staying flat). Video encoding is CPU-intensive by nature, motion estimation, quantization, rate control, all running per outgoing stream. Documented accounts of mesh calls under load report senders getting capped around 300 kbps of outgoing video even on connections with plenty of headroom, because the device's CPU, not its network link, is the bottleneck, burning cycles encoding the same camera frame multiple times over for different peers.

**Connection setup cost is quadratic, not linear, and each pair can independently need a relay.** N participants in full mesh means N*(N-1)/2 unique pairwise connections: 2 people is 1 pair, but 8 people is 28 pairs, each running its own ICE negotiation to find a viable network path, and each independently falling back to a TURN relay server if a direct path cannot be established (common behind symmetric NATs, corporate firewalls, and some mobile carrier networks). A demo with 2 or 3 participants never notices this. A real call with 8 people is suddenly negotiating 28 independent paths and potentially provisioning 28 independent relay allocations for what should have been one shared session.

The obvious server-side fix, route everyone's media through one central server, has its own naive failure mode if that server tries to be helpful: decode every incoming stream, mix a custom composite for every listener (so each person hears everyone except themselves, or sees a custom video grid), and re-encode a personalized output per recipient. This is a Multipoint Control Unit, an MCU, and it moves the wall rather than removing it: decode N streams, then mix and re-encode up to N distinct personalized outputs, is CPU work that scales roughly with N times N on a single server, the exact same quadratic shape that killed the client-side mesh, just relocated from N laptops onto one box.

---

## 3. The architecture

The shape that Discord's voice infrastructure, and essentially every large-scale group video product, converged on:

```
[Client: mic/camera capture, encode ONCE at one or more
 fixed simulcast quality layers]
   analogy: recording your voice once onto a few different
   quality cassette tapes, not re-recording it separately
   for every listener
   |
   v
[ICE / STUN: discover a usable network path, try direct
 connectivity first]
   analogy: two phones on the same street trying to shout
   directly to each other before going through an operator
   |
   v
[TURN relay, only if direct connectivity fails, e.g. behind
 a symmetric NAT or restrictive firewall]
   analogy: the switchboard operator patching the call through
   when the direct line cannot be made
   |
   v
[Regional voice server selection: client measures latency to
 nearby candidate servers, joins the one that is closest]
   analogy: dialing the nearest local exchange instead of a
   long-distance operator on the other side of the country
   |
   v
[SFU, Selective Forwarding Unit, one stateful server owns this
 voice channel's session: receives ONE encoded upload per
 client (possibly at several simulcast layers), does NOT
 decode or re-encode, just forwards packets]
   analogy: a mail sorting office that reroutes envelopes to
   the right addresses without ever opening and rewriting them
   |
   v
[Receiver-driven selection: each subscribing client tells the
 SFU which remote streams it currently wants and at what
 quality (Discord's own protocol calls this a Media Sink
 Want); the SFU also weighs active-speaker detection and each
 receiver's current available bandwidth]
   analogy: each listener at a big table telling the waiter
   which conversations to relay to them, instead of every
   word at the table being repeated to everyone
   |
   v
[Closed-loop congestion control: receivers report loss and
 round-trip time back to senders; senders lower their encoder
 bitrate in real time when the network cannot carry more]
   analogy: a driver slowing down the instant they feel the
   road get slick, not waiting for a crash to react
   |
   v
[Client: jitter buffer absorbs arrival-time variance, forward
 error correction and packet-loss concealment paper over small
 gaps, LATE packets are discarded rather than waited for]
   analogy: a radio DJ who skips a scratched second of a
   record rather than stopping the whole show to fix it
   |
   v
[Local decode and render, once per incoming stream, which is
 far cheaper than encoding]
```

The structural decision underneath all of this is the same one this ledger has named before in a different form: separate the expensive, shared step from the cheap, per-consumer step. Encoding is expensive and happens once, at the source. Forwarding is cheap and happens per recipient, but it is pure packet routing, not reprocessing. Decoding is comparatively cheap and happens locally on each receiving device, spending the receiver's own CPU instead of the shared server's. Discord's own official developer documentation for its voice gateway protocol confirms this shape concretely: clients negotiate SSRCs (synchronization source identifiers) for each simulcast layer they publish, and a receiving client sends an explicit request, documented as opcode 15, telling the SFU exactly which remote SSRCs and quality level it wants forwarded right now, which is the server-side mechanism that makes "forward only what is actually needed" possible instead of blind fan-out to everyone.

A voice channel's session is pinned to one server, chosen by measured latency at join time, the same category of problem as Day 48's H3-cell-to-node ownership, just operating in latency space instead of geographic space: the session's live state (who is in the channel, which streams are active) lives in memory on the one server that owns that channel, not scattered across a stateless pool.

---

## 4. The transferable mechanisms

**a. Forward, don't transcode.** The single biggest lever in this whole architecture is refusing to decode and re-encode media that is only being relayed. An SFU treats each incoming stream as opaque bytes after the initial decode-time codec negotiation; it never touches the pixels or the audio samples. Every time a system is tempted to "process" data on its way through a hot path purely to reshape it for each consumer, ask whether the consumers could instead accept the same shared representation directly, the way SFU clients accept the same encoded packets the sender already produced.

**b. Simulcast: precompute a small number of fixed quality tiers, pick cheaply at serve time.** A publishing client encodes 2 or 3 quality layers once (say, a low, medium, and high bitrate/resolution variant), and the SFU picks which tier to forward to each subscriber based on that subscriber's available bandwidth, without ever transcoding between tiers itself. This is the identical shape as H3's fixed resolution levels from Day 48 and the cheap-recall-then-narrow pattern from search and feed ranking on Days 7 and 18: do the expensive work a fixed, small number of times up front, then make the per-consumer decision a cheap lookup, not a fresh computation.

**c. Closed-loop congestion control as proactive backpressure.** Receivers continuously report loss and round-trip time back to senders, and senders cut their own encode bitrate the instant that feedback shows the network cannot carry the current load, before a human notices anything is wrong. This is backpressure applied at the media layer: the signal to slow down comes from the actual consumer of the data, not from a fixed rate limit picked in advance, and the correction happens before failure, not after.

**d. Receiver-driven, not blind, delivery.** Discord's Media Sink Wants mechanism lets each client tell the server which of the many available streams it actually needs right now. Bandwidth and server forwarding effort get spent on what is currently being listened to or watched, not fanned out uniformly to every participant regardless of whether anyone is looking at them. This is the same principle as only ranking the short list of geo-candidates in Day 48, or only computing the top-k suggestions in Day 16's typeahead example: push the cost of "what do I actually need" as close to the consumer as possible.

**e. Accept loss at the transport layer instead of retransmitting.** Real-time media rides over UDP (wrapped in SRTP for encryption), which makes no delivery guarantee at all, by design. A jitter buffer and forward error correction absorb small amounts of loss and reordering, and a packet that arrives too late to be useful is simply dropped rather than waited for. This is a deliberate rejection of TCP's model, where a single lost packet blocks everything queued behind it until it is retransmitted and reordered; for a live call, a half-second of silence while waiting for one retransmitted packet is a worse user experience than losing that one packet outright.

**f. Stateful session pinned by measured proximity, not by a stateless load balancer.** Because the SFU holds live per-channel state in memory (who is publishing, what simulcast layers exist, current subscriptions), a request for channel X must land on the specific node that owns channel X, exactly the same constraint DISCO/Ringpop hit in Day 48. Here the assignment key is measured client latency to nearby servers rather than a spatial hash, but the underlying requirement, this is a stateful service and needs session affinity, not round-robin routing, is identical.

---

## 5. The trade-offs

**Availability of the live stream versus completeness of every packet, decided explicitly per data type.** Media packets are allowed to simply vanish: a lost video frame or a dropped audio sample is gone forever, never retransmitted, because the alternative, waiting for guaranteed delivery, costs more in freshness than the missing data costs in quality. This is not the same choice Discord makes for every kind of data in the same product: text chat messages and the signaling data that establishes who is in a voice channel do need reliable, ordered delivery, and correspondingly do not ride the same lossy UDP transport as the audio and video itself. The lesson generalizes past this one product: the right delivery guarantee is a property of what a specific stream of data means to the user, not a single policy applied uniformly across an entire system.

**Server egress cost versus server CPU cost, and why SFU wins that trade at scale.** An SFU forwards every subscribed simulcast layer to every subscriber that wants it, which costs more total network egress than an MCU's single mixed stream per listener would. It accepts that cost specifically to avoid the MCU's decode-mix-encode CPU bill, which scales quadratically with participants. Bandwidth is the resource that scales cheaply and close to linearly, more NICs, more servers, more peering; CPU-bound transcoding does not scale that way on a single box. That asymmetry is exactly why virtually every large group-calling product that reached serious scale, Discord included, ended up on the SFU side of this trade rather than the MCU side, reserving MCU-style mixing for narrower cases like producing one flattened recording or broadcast output, not the live interactive path.

**Jitter buffer size: smoothness versus latency, a knob turned per network condition, not a fixed constant.** A larger buffer absorbs more arrival-time variance and packet reordering before it runs dry, producing smoother playback on a jittery network. It also adds fixed delay before anything the buffer is holding gets played, which is directly perceptible as lag in a live conversation. There is no buffer size that is simply "correct"; it is a live trade between two things a user can both feel, choppiness versus delay, and production systems adapt it dynamically rather than fixing it once.

---

## 6. The systems-thinking lens

**The feedback loop: congestion collapse, the media-layer version of a retry death spiral.** Imagine a design that tried to be more careful, retransmitting any lost media packet the way a reliable protocol normally would. Trace what happens on a real, imperfect network path, a mobile handoff between cell towers, a saturated home wifi link during a video call: a packet is lost, the naive design retransmits it, that retransmission adds more traffic onto a link that was already too congested to deliver the original packet on time, which increases the odds the retransmission is also delayed or lost, which triggers another retransmission, all while newer packets keep arriving and queuing up behind the ones still being chased. The jitter buffer, built to smooth over ordinary variance, runs dry waiting for data that is stuck behind its own retry traffic, and the call stutters or drops audio entirely, sometimes for longer than the original network hiccup lasted, because the retransmission backlog is now the dominant traffic on the link, not the original media stream.

**The senior fix breaks the loop structurally: never retransmit at the media layer, and let old data die.** Real-time transport does the opposite of TCP on purpose. Congestion control (mechanism c) cuts the sender's bitrate the moment receiver feedback shows loss or rising round-trip time climbing, before the link fully saturates, proactive load shedding rather than a reaction to failure. The jitter buffer (mechanism e) discards a packet that arrives too late to be useful instead of stalling everything behind it. Neither of these adds capacity to the network; both change what the system does when capacity runs short, exactly the same family of fix this ledger has named repeatedly for other domains: the correct response to congestion is to shed load and back off at the source, not to try harder to deliver everything that was already sent. Discord's operational answer to the adjacent failure mode, one server or data center going down and every affected client reconnecting at once, is the same idea applied to infrastructure rather than packets: 850-plus servers spread across 13 regions and 30-plus data centers means a single failure displaces a bounded slice of channels, and clients reconnecting land on the next-nearest healthy server by the same latency measurement used at initial join, not all funneled onto one designated fallback that would itself become the next bottleneck.

---

## Map to Rare.lab's stack

**Where the same shape shows up, stripped of the video-calling wrapper.** Rare.lab's runtime holds one shared WebGL context per embedded session. The moment more than one viewer needs to watch the same live scene at once, collaborators watching a shared canvas, or an embedded runtime serving multiple simultaneous consumers of the same rendered output, the naive move is exactly the MCU mistake from section 2: spin up a private render pass, or a private encode, per viewer, so cost scales with viewer count times render cost. The SFU move is the one to take instead: render or compute the shared, expensive step once, and forward its output (or a small number of precomputed variants, the simulcast equivalent) to every consumer, spending only cheap, per-viewer work, picking a variant, applying a small client-side delta, on the parts that genuinely differ per viewer.

**The concrete ceiling and the move to make before hitting it.** The wall in the video-calling story is the instant a receiver needs something genuinely personal, their own custom video grid layout, their own audio mix minus their own voice, which is exactly what forces MCU-style mixing to exist at all, just scoped narrowly (one flattened recording, not the live interactive path for everyone). Rare.lab will hit the same wall the moment a viewer needs a truly personalized frame: their own highlight overlay, their own camera angle into a 3D scene, a per-user debug view. The move that keeps the shared render path at O(1) per frame regardless of viewer count is to keep personalization as a small delta applied client-side on top of a shared broadcast state, a highlight layer composited in the browser, a client-local camera transform applied to a shared scene payload, rather than the server re-rendering a bespoke frame per viewer. And for the multiplayer scene-state fan-out itself, not the GPU render but the state that keeps every connected client's view of the scene in sync, the same forward-don't-retranscode principle applies directly: broadcast one shared diff or patch stream and let each client apply it locally, instead of the server computing and sending a personalized full-state payload to every connected client.

---

## References and summaries

**Discord Engineering Blog (official): "How Discord Handles Two and a Half Million Concurrent Voice Users using WebRTC"**
https://discord.com/blog/how-discord-handles-two-and-half-million-concurrent-voice-users-using-webrtc (mirrored on Medium: https://medium.com/discord-engineering/how-discord-handles-two-and-half-million-concurrent-voice-users-using-webrtc-ce01c3187429)
Discord's own primary account of their voice infrastructure: a client-server (SFU-based) architecture chosen explicitly because full peer-to-peer mesh becomes prohibitively expensive as participant count grows, more than 2.6 million concurrent voice users served off more than 850 voice servers across 13 regions in over 30 data centers, more than 220 Gbps of egress and 120 million packets per second at peak, and a single C++ media engine built on the WebRTC native library shared across Discord's desktop, iOS, and Android clients.

**Discord Developer Documentation (official): "Voice Connections"**
https://docs.discord.com/developers/topics/voice-connections
The primary source for this lesson's protocol-level details: the voice gateway's opcode set (Identify, Select Protocol, Ready, Session Description, Speaking, and others), how SSRCs are assigned per simulcast stream a client publishes, and the receiver-driven mechanism, documented as opcode 15, Media Sink Wants, that a subscribing client uses to tell the SFU exactly which remote streams and quality layers it currently wants forwarded, rather than the server fanning out everything to everyone by default.

**BlogGeek.me (Tsahi Levent-Levi): "What is WebRTC P2P mesh and why it can't scale?"**
https://bloggeek.me/webrtc-p2p-mesh/
Industry-standard explanation of the mesh topology's bandwidth math, uplink and downlink per participant scaling as K times (N minus 1), the source for this lesson's framing that mesh is genuinely fine for a 1:1 call or a small standup and reliably breaks down by the time a 5th or 6th participant joins, on CPU (multiple simultaneous encoders) as much as on bandwidth (asymmetric consumer uplink).

**BlogGeek.me: "WebRTC SFU Explained: Technology, Pros, Cons & Use Cases"**
https://bloggeek.me/webrtcglossary/sfu/
Reference definition used in this lesson for the SFU versus MCU distinction: an SFU receives one stream per publisher and forwards it without decoding or re-encoding, while an MCU decodes every incoming stream, mixes a composite, and re-encodes a distinct output per recipient, the quadratic-cost design this lesson identifies as the server-side version of the same wall that breaks client-side mesh calling.

**Secondary corroborating sources for the mesh math used in section 2 (clearly labeled as secondary, not primary)**
https://dev.to/christosalexiou/the-multiple-faces-of-webrtc-n-peer-calling-mesh-mcu-and-sfu-39dg and https://antmedia.io/webrtc-scalability/
Used only to corroborate two specific, widely repeated illustrative figures: a 6-participant mesh call producing 30 total individually managed streams across the call, and senders in mesh calls commonly getting capped near 300 kbps of outgoing video by encoder CPU cost rather than by available network bandwidth. Treat the exact figures as representative, well-corroborated examples rather than a single audited benchmark.
