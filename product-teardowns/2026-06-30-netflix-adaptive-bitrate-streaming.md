# Netflix: Adaptive Bitrate Streaming (per-title encoding plus the client picking quality on the fly)

Date: 2026-06-30
Product: Netflix
Feature: Adaptive bitrate streaming. The two halves that make the picture sharp and never freeze: how the video is encoded ahead of time (per-title and per-shot ladders), and how the player on your phone or TV picks which quality to download next, second by second.

A note on scope. An earlier ledger lesson covered Netflix Open Connect (the CDN that puts bytes near you). This teardown deliberately skips the CDN and goes after the two pieces it sits between: the offline encoding brain that decides what versions of the video even exist, and the live client algorithm that decides which version to pull right now. Those are the parts that turn one movie file into a smooth stream on a flaky train.

---

## 1. The user

Priya is on the 7:40pm local train out of Mumbai, standing, one hand on the rail, the other holding her phone. She has 35 minutes and she is two episodes into Stranger Things season 4. The train goes through three tunnels. Her signal drops from full LTE to almost nothing and back, four or five times, on the way home.

She is not thinking about codecs or bandwidth. She wants the next episode to start fast, look good when the signal is good, and not freeze with a spinning wheel when she goes into a tunnel. The moment a buffering spinner sits there for eight seconds, she feels it. She might just put the phone away.

The same feature is serving Raj on a 4K TV on fiber at home, and a student on hostel wifi shared by 60 people. Same movie, wildly different pipes. One feature has to make all three feel taken care of.

---

## 2. The real problem

Here is the honest version, friend to friend. The internet does not give you a steady pipe. Your bandwidth jumps around constantly, especially on mobile. One second you have 12 Mbps, the next you are in a tunnel with 200 kbps.

Video is heavy. A single 1080p movie at a fixed high quality might need 5 to 6 Mbps the whole time. If the network drops below that for even a few seconds and you have nothing saved up, the picture freezes. That freeze, the rebuffer, is the single most hated thing in streaming. Netflix has measured it: rebuffering is the QoE metric most tied to people abandoning a session.

So you cannot ship one fixed-quality file. You also cannot just "send the best the network can take right now," because the network a half second from now might be a tunnel, and by the time you find out, the screen is already frozen.

And there is a second, quieter problem hiding underneath. Not all video is equally hard to compress. A dark, grainy, fast action scene from Stranger Things needs a lot of bits to look clean. A flat, simple cartoon like an episode of a kids' show needs almost nothing to look perfect. If you encode both with the same fixed bitrate ladder, you waste bits on the cartoon (and Priya's tunnel suffers for no reason) and you starve the hard scene (blocky artifacts in the dark). One size fits nobody well.

---

## 3. The feature in one sentence

Netflix encodes every title into a custom set of quality rungs tuned to how hard that specific video is to compress, and the player on your device continuously picks the highest rung it can safely download given how full its buffer is, so the picture stays as sharp as possible and almost never freezes.

---

## 4. Jobs to be done

What Priya is really hiring this feature to do:

- "Start the episode fast, even on a weak signal." (low start-up delay)
- "Do not freeze when I hit a tunnel." (no rebuffering)
- "Look as good as my connection honestly allows, and get sharper the second it can." (quality tracks bandwidth)
- "Do not eat my whole data pack for a cartoon that did not need it." (no wasted bits)
- "Do not make me touch a quality setting. Just handle it." (zero config)

What Netflix is hiring it to do: protect retention by killing the spinner, and cut tens of percent off the bits shipped, which is real money at their CDN volume.

---

## 5. How it works for the user

Priya taps the episode. It starts in about two seconds, maybe a touch soft for the first moment. Within a few seconds it sharpens to crisp 1080p. The train is in open air, signal is strong, picture is clean.

Train enters a tunnel. Signal collapses. Priya notices nothing, or at most the picture gets slightly softer for a few seconds. No spinner. It comes out of the tunnel and re-sharpens. She never opens a menu. She never sees a number. The whole adaptation is invisible. That invisibility is the product.

On Raj's 4K TV on fiber, the same title plays at full 4K the entire time because the pipe never wavers. Same content, the system just walked further up the same ladder.

---

## 6. The actual flow, step by step

1. Priya taps play on Stranger Things S4E1.
2. Her client asks Netflix for the manifest for this title. The manifest is the menu: it lists every available stream (each resolution and bitrate rung) and where the chunks live. Video is cut into small segments, a few seconds each.
3. The client picks a starting rung. It starts conservative so playback begins fast, then climbs.
4. The client downloads segment 1, then segment 2, into a buffer (think of it as a small reservoir of already-downloaded seconds of video, maybe up to 240 seconds when things are healthy).
5. Before each next segment, the client decides which quality rung to request for that segment. This decision is the heart of the live algorithm.
6. Tunnel hits. Download speed craters. The buffer (the reservoir Priya built up in open air) drains while playback keeps going. The client sees the buffer falling and steps down to a lower rung so the next segments are small and arrive in time.
7. Out of the tunnel. Buffer refills. The client steps back up the ladder, segment by segment.
8. All of this is server-side-encoded but client-side-decided. Netflix's servers do not push a quality at her. Her player pulls the rung it judges safe. The intelligence at play time lives on the device.

The segments she is choosing between were created long before she pressed play, in a giant offline encoding pipeline. That pipeline is the other half of the story.

---

## 7. Under the hood, like the engineer

There are two engines here. One runs offline, once per title, and decides what rungs exist. One runs live, on the device, and decides which rung to pull. Keep them separate in your head; that separation is the whole architecture.

### Engine A: the offline encoder that builds the ladder

**The old way: one fixed ladder for everything.** For years (confirmed, Netflix Tech Blog 2015) Netflix used a single fixed bitrate ladder, the same rungs for every title:

| Bitrate | Resolution |
|--------:|------------|
| 235 kbps | 320 x 240 |
| 375 kbps | 384 x 288 |
| 560 kbps | 512 x 384 |
| 750 kbps | 512 x 384 |
| 1050 kbps | 640 x 480 |
| 1750 kbps | 720 x 480 |
| 2350 kbps | 1280 x 720 |
| 3000 kbps | 1280 x 720 |
| 4300 kbps | 1920 x 1080 |
| 5800 kbps | 1920 x 1080 |

Concrete failure of one-size-fits-all: at the top rung, 5800 kbps for 1080p, a flat simple cartoon looks perfect with bits to spare, while a noisy, film-grain-heavy action scene still shows blockiness in the dark. The ladder is too generous for one and too stingy for the other. Same ten rungs, two different injustices.

**Per-title encoding (confirmed, Netflix 2015).** Instead of guessing one ladder, measure. For each title, run a grid of test encodes: several resolutions, each at several QP values (QP is the quantizer; higher QP means more compression and lower quality). For every test encode, plot a point: bitrate on one axis, perceptual quality on the other. Netflix scores quality with VMAF (more on that below), not raw PSNR, because VMAF tracks what a human eye actually notices.

Plot all those points and the good ones trace a **convex hull**: the outer envelope where, for any given bitrate, you have the best achievable quality, and the resolution that achieves it. Below some bitrate, 720p encoded well beats 1080p encoded badly, because a clean 720p upscaled looks better than a blocky 1080p. The convex hull tells you exactly where to switch resolutions. The custom ladder for that title is a handful of points sampled along its own hull.

Result (confirmed): roughly **20% average bitrate reduction** for the same perceived quality versus the fixed ladder. For a simple animated title, the top rung might now be far below 5800 kbps because it simply does not need more. For a hard title, a rung might sit higher, or the resolution switch points move, so the dark action scene stops being blocky.

Concrete payoff for Priya: the cartoon her niece watches now tops out at a fraction of the old bitrate, so it sails through the tunnel; the Stranger Things dark scene gets a ladder that actually holds up at the rungs her train allows.

**Per-shot / Dynamic Optimizer (confirmed, Netflix 2018 and the 2020 4K rollout).** One ladder per title is still too coarse, because a two-hour film is not uniformly hard. The opening credits are easy. A grain-heavy night chase is brutal. The Demogorgon reveal in the dark is brutal. A quiet dialogue scene is easy.

So Netflix went finer: split the video at **shot boundaries** (a shot is a continuous camera take) and optimize each shot independently. A single episode of Stranger Things is processed as roughly **900 shots** (confirmed figure from Netflix). For each shot, the Dynamic Optimizer searches resolution and QP to find, for a target bitrate, the choice that maximizes VMAF. Then it stitches the per-shot choices into a global trajectory using a Lagrangian/trellis optimization so the whole stream hits its budget while spending bits where the eye benefits most. Per-scene optimization was cited as pushing savings toward 30%.

The data structure here is essentially a trellis: each shot is a column, each candidate (resolution, QP) is a node, and you find the lowest-cost path across all shots subject to a total-bitrate constraint. That is a clean dynamic-programming shape, which is why the Lagrangian relaxation works.

**VMAF, the metric the whole thing optimizes (confirmed, Netflix open-sourced it).** VMAF (Video Multimethod Assessment Fusion) is how the encoder knows "quality." It is not one formula. It extracts several elementary features per frame: VIF (visual information fidelity), DLM (detail loss metric), and a temporal/motion feature (frame-to-frame difference). Then a trained SVM (nuSVR) fuses those features into a single 0 to 100 score calibrated against real human opinion scores. So when the convex hull and the Dynamic Optimizer "maximize quality," they are maximizing a model of human perception, not a raw signal error. That is why a clean 720p can legitimately beat a blocky 1080p in the score: humans prefer it, so VMAF prefers it.

### Engine B: the live client algorithm that picks the rung

Now the device. Every few seconds it must answer: which rung do I request for the next segment?

**Why the obvious approach fails.** The naive method is throughput estimation: measure how fast the last few segments downloaded, predict the next, pick the highest rung under that prediction. Netflix's own research (confirmed, the "Using the Buffer to Avoid Rebuffers" paper, SIGCOMM 2014) showed this is fragile. Bandwidth estimates on mobile are noisy and lag reality. By the time your estimate notices the tunnel, your screen is already frozen. Throughput estimation alone overreacts to short spikes and underreacts to real drops.

**The buffer-based idea (BBA).** The insight: you already have a perfect, lag-free signal of your true sustained throughput sitting in front of you. It is the **buffer occupancy** itself. If the buffer is filling, you are downloading faster than you are playing, so it is safe to go up. If the buffer is draining, you are downloading slower than you play, so go down. The buffer integrates the network over time and cannot lie about it.

So BBA maps buffer level directly to bitrate with a function:

- A **reservoir** at the bottom: a protected band of seconds you never gamble. While the buffer is in the reservoir (say the first ~10% of capacity), always pick the lowest rung to refill fast and never risk a freeze. This is what saves Priya in the tunnel: when her buffer drops into the reservoir, the client slams to a low rung so segments are tiny and keep arriving.
- A **cushion** above it: as the buffer climbs through the cushion, the chosen rung rises smoothly along the map, up to the top rung once the buffer is comfortably full.
- A cap at the top so a giant buffer does not request beyond the best available rung.

Netflix reported (confirmed) that adding buffer-based logic cut the **rebuffer rate by about 20%** versus their previous best throughput-based approach, tested live across **more than 500,000 users**, while keeping the average video rate essentially the same. Less freezing for free.

**BOLA, the principled version (confirmed, Netflix/academic, now in the dash.js reference player).** BOLA reframes rung selection as an online optimization: maximize a utility that rewards higher bitrate and penalizes rebuffering, using Lyapunov control, with the buffer level as the control variable. It is provably near-optimal in that utility framework and needs no bandwidth prediction at all. Production players blend buffer-based and throughput signals (hybrid), but the spine is the buffer.

The live decision cost is tiny: read buffer level, evaluate a mapping function, request a URL from the manifest. Constant work per segment, independent of catalog size or title length. All the heavy thinking already happened offline in Engine A. This is the same offline-think / online-lookup pattern that runs through Discover Weekly, YouTube, and Razorpay routing in this ledger: do the expensive optimization once, ahead of time; keep the live path a cheap read.

### The scale story at three tiers

Encoding cost scales with content, not viewers. Delivery cost scales with viewers. They break at different tiers.

**1,000 titles.** Per-title encoding is easy. Even per-shot, a few hundred shots per title times a grid of test encodes is a manageable batch job. You could almost brute force it. Storage of the ladders is trivial. Client ABR is the same simple buffer map. Nothing is stressed.

**100,000 titles, millions of viewers.** Now the encoding bill bites. Per-shot Dynamic Optimization means a single Stranger Things episode is ~900 shots, each tried at multiple resolutions and QPs, each scored by VMAF. That is thousands of encode-and-score operations per episode, times a huge catalog, times every codec generation (H.264, then HEVC, then AV1) you support. What breaks: a single encoding farm cannot keep up. The survival move (confirmed pattern) is massive parallelism. Shots are independent, so they fan out across a media-processing platform (Netflix's Archer/Cosmos style chunked-parallel encoding, the same fan-out used for the artwork pipeline in the 2026-06-19 teardown). Embarrassingly parallel by shot is the property that makes per-shot encoding affordable at catalog scale. On the delivery side, millions of concurrent viewers would melt any central origin, which is exactly why Open Connect pushes the encoded rungs out to edge appliances near users. The client ABR does not change at all; it is already O(1) per segment.

**10 million plus concurrent streams (a global hit drop).** Stranger Things season finale night: tens of millions start within an hour. The encoding was done weeks ago, so Engine A is not in the live path at all; this tier is pure delivery plus client decisions. What breaks: any shared live coordination. The survival move is that there is almost none. Each client runs its own buffer-based ABR locally and independently, pulling pre-encoded segments from the nearest edge. There is no central server choosing quality for 10 million people; that would be a single point of contention. Decentralizing the decision to each device is the scale trick. The hard scene during a contended evening simply means more clients independently sit a rung or two lower; the system degrades smoothly instead of falling over. The convex-hull ladders make that graceful degradation look good rather than blocky, because the lower rungs were chosen to be the best possible quality at that bitrate, not arbitrary cuts.

The dial that decouples everything: **segment length and buffer size.** Short segments make adaptation responsive (the client can change its mind every few seconds) at the cost of more requests. A large buffer (Netflix can hold up to ~240 seconds) is the shock absorber that lets a client ride through a long tunnel without a single freeze, as long as it filled up beforehand. That buffer is the reservoir Priya unknowingly banked in open air and spent in the tunnel.

---

## 8. The retention and habit mechanic

The mechanic here is the absence of friction, not a flashy hook. Every rebuffer is a moment Priya might quit. Netflix's research explicitly ties rebuffering to session abandonment and lower engagement. By making freezes rare and start-up fast, the feature removes the exits. The loop it protects is the binge: episode ends, autoplay starts the next one (the 2026-06-19 ledger covered the artwork side of that auto-play card), and a smooth uninterrupted stream is what lets "just one more" actually happen. A single eight-second spinner at the wrong moment breaks the trance and Priya puts the phone away one episode early.

Which metric it moves: primarily **retention and engagement** (hours watched, sessions completed, churn avoided), and secondarily **cost/revenue** through the 20 to 30% bitrate savings, which at Netflix's delivery volume is enormous CDN and storage money. Better quality at lower bitrate also widens the addressable market: people on weak or metered connections in India and Southeast Asia can actually finish episodes, which is direct subscriber retention in exactly the markets Netflix is fighting for.

Real observed example: the per-title work was justified partly by users on constrained connections getting watchable, non-blocky video at bitrates their network could sustain. The smooth experience in Priya's tunnel is not a nicety; it is the reason she renews.

---

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph to shippable shader code and ships an embeddable runtime. The runtime will run on a GPU you do not control: a flagship phone, a five-year-old laptop with integrated graphics, a mid-range Android, a high-refresh desktop. That is exactly Priya's tunnel problem wearing a different hat. The network here is the frame budget.

Steal both engines.

**Engine A, offline: build a per-effect quality ladder, not one fixed effect.** Netflix's win was refusing one-size-fits-all and instead measuring each title's own complexity curve. Do the same at compile time. For each shader graph, the Rare.lab compiler should emit not one binary but a small ladder of variants at decreasing cost: full quality, then progressively cheaper rungs (fewer samples, lower-res render targets, half-precision, dropped sub-effects, a downsampled bloom). Pick the rungs along the effect's own quality-versus-cost convex hull, scored by a perceptual metric, not raw error, so a cheap rung that looks identical to the eye is preferred over an expensive one that does not. This is per-title encoding for shaders: spend the expensive search once, at compile time, where you have all day.

**Engine B, live: adapt on a buffer-like signal, not an instantaneous one.** The runtime should pick its rung from the equivalent of buffer occupancy, not from one frame's timing. The honest, lag-free signal is your **frame-time headroom integrated over a short window**: if you are consistently finishing frames under budget, climb a rung; if frame time is creeping toward the vsync deadline, drop a rung immediately, with a protected reservoir (never gamble quality when you are one bad frame from dropping below 60fps). Do not react to a single 18ms spike (that is throughput-estimation fragility); react to the trend, the way BBA reads the buffer. And keep the live decision O(1): read the rolling frame-time stat, index into the precompiled variant, swap. All the thinking happened at compile time.

The concrete payoff: a Rare.lab effect embedded on a marketing page holds a smooth 60fps on a cheap Android by quietly sitting two rungs down, and snaps to full quality on a gaming desktop, with zero developer configuration, exactly as Priya never touches a quality menu. The thing that makes that graceful instead of ugly is the same thing that saves Netflix: the lower rungs were chosen to be the best-looking option at that cost, not arbitrary cuts. Make degradation a designed product, not an accident.

---

## Sources

- Netflix Technology Blog, "Per-Title Encode Optimization" (2015): https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2
- Netflix Technology Blog, "Dynamic optimizer, a perceptual video encoding optimization framework" (2018): https://netflixtechblog.com/dynamic-optimizer-a-perceptual-video-encoding-optimization-framework-e19f1e3a277f
- Netflix Technology Blog / Research, "Optimized shot-based encodes for 4K: Now Streaming!" (2020): https://research.netflix.com/publication/optimized-shot-based-encodes-for-4k-now-streaming
- Streaming Learning Center, "How Netflix Pioneered Per-Title Video Encoding Optimization": https://streaminglearningcenter.com/encoding/how-netflix-pioneered-per-title-video-encoding-optimization.html
- Te-Yuan Huang et al., "A Buffer-Based Approach to Rate Adaptation / Using the Buffer to Avoid Rebuffers" (Netflix, SIGCOMM 2014): https://arxiv.org/pdf/1401.2209
- Spiteri, Urgaonkar, Sitaraman, "BOLA: Near-Optimal Bitrate Adaptation for Online Videos": https://dl.acm.org/doi/fullHtml/10.1145/3336497
- Netflix VMAF open-source repository and documentation: https://github.com/Netflix/vmaf
- Video Multimethod Assessment Fusion overview: https://en.wikipedia.org/wiki/Video_Multimethod_Assessment_Fusion
