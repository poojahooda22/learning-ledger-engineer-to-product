# WhatsApp voice and video calling (the green call button, the relay, and the codec)

Date: 2026-07-26
Product: WhatsApp
Feature: One-to-one and group voice/video calling (call setup, the relay, encrypted media, the codec)

---

## 1. The user

Priya is 29, lives in a rented flat in Andheri, Mumbai. It is 9:40pm on a
Tuesday. Her mother is in Pune, 150km away, sitting in a kitchen with weak
2-bar 4G. Priya opens WhatsApp, taps her mother's chat, taps the phone icon,
and by the time she has put the phone to her ear it is ringing. Her mother
picks up. They talk for eleven minutes about nothing. The call does not drop
when Priya walks from her living room (WiFi) into the lift (4G). Her mother's
voice stays clear even though the Pune signal is bad enough that a webpage
would not load.

Priya is not the only one. WhatsApp carries more than 5.5 billion voice calls
and 2.4 billion video calls every month, and users spend more than 2 billion
minutes on calls every single day (Business of Apps / SQ Magazine, 2026
compilations). The average voice call runs about 9.7 minutes. This runs on top
of a base of roughly 2 billion users.

## 2. The real problem

A phone call over the internet is a fight against physics that a text message
never has to have.

A WhatsApp text can take three seconds to arrive and nobody notices. A voice
packet that arrives 300 milliseconds late is garbage. You cannot "resend it and
wait" the way TCP resends a lost web-page byte, because by the time the resend
arrives, that slice of your mother's sentence is already in the past. The
conversation has moved on.

So the real pain is four things stacked on top of each other. One, the two
phones usually cannot even reach each other directly, because both sit behind
home routers and mobile carrier NATs that hide their real address. Two, the
network jitters: packets arrive early, late, out of order, or not at all, and
the ear is brutally sensitive to it. Three, one side is often on a terrible
connection (the Pune kitchen), and the call has to survive that without dragging
the good side down. Four, all of it has to be private, so nobody in the middle,
not even WhatsApp, can listen. Solve one and ignore the others and the call
still feels broken.

## 3. The feature in one sentence

WhatsApp voice and video calling sets up a live, end-to-end encrypted media
stream between two or more phones, usually routed through a WhatsApp relay
server that forwards the encrypted audio and video it cannot read, tuned so the
call starts in under a second and survives bad networks and old phones.

## 4. Jobs to be done

- "Let me hear my mother's actual voice, now, without her voice breaking up."
- "Start the call instantly. Do not make me wait through five seconds of
  connecting."
- "Do not drop the call when I walk from WiFi to mobile data."
- "Keep it working on her 6-year-old phone and her weak signal, not just my
  new one."
- "Keep it private. This is a family call, not a postcard."
- "Let me add my brother and my aunt to the same call without installing
  anything new."

## 5. How it works for the user

Priya taps the call button. She sees "Calling..." for a moment, then "Ringing."
Her mother's phone rings, she accepts, and they are connected. A small timer
counts up. If Priya taps the video icon mid-call, her mother's face appears,
with Priya's own face in a small floating tile. If the network gets bad, the
video may get softer or freeze for a second while the audio keeps going, because
WhatsApp will sacrifice picture before sound. For a group call, Priya taps the
call button in a group or adds people one by one, up to 32 participants on voice
or video. On the newest builds she can turn on a privacy setting, "Protect IP
address in calls," which routes every call through WhatsApp servers so the other
person can never see her home IP address, at the cost of a small quality dip.

## 6. The actual flow, step by step

1. Priya taps the phone icon in her mother's chat.
2. Her phone gathers "how can I be reached" candidates: its local WiFi address,
   its address as seen from the outside world (found by asking a STUN server),
   and a relay address it can fall back to. This is the ICE process.
3. Those candidates, plus the encryption setup, travel to her mother's phone as
   a signaling message over WhatsApp's normal always-on connection (the same
   store-and-forward channel that carries chat messages, covered in the 2026-06-15
   ticks teardown). Signaling and media are two separate planes.
4. Both phones already share, or now establish, a Signal Protocol session. From
   it they derive the keys that will encrypt the actual audio and video (SRTP
   keys). No media is ever sent in the clear.
5. To make the call start fast, WhatsApp begins by sending media through a relay
   server (called the "conf bridge" internally) rather than waiting to negotiate
   a direct path. Sound can begin almost immediately.
6. For a one-to-one call, the phones may in parallel try to open a direct
   peer-to-peer path. If they succeed and it is better, media can move to it. If
   they are behind strict NATs and cannot, the relay stays as the path.
7. Audio is captured, cleaned (echo cancellation, noise suppression, automatic
   gain), compressed by the codec, encrypted, chopped into small RTP packets,
   and sent roughly every 20 milliseconds.
8. On the receiving phone, packets land in a jitter buffer that reorders them
   and smooths timing, the codec decodes them, and the audio plays out. Lost
   packets are concealed or recovered so the gap is not heard.
9. When Priya walks into the lift and her phone switches from WiFi to 4G, ICE
   re-checks paths and the media flow migrates to the new network without
   dropping the call.
10. Either party taps the red button. A hangup signaling message ends the
    session and the relay tears down its forwarding state.

## 7. Under the hood, like the engineer

This is where the real work lives. A call is two separate problems wearing one
coat: get a path between the phones (connectivity), and keep the media usable
once the path exists (resilience). Then group calling piles a third problem on
top: fan the streams out to many people through a server that is not allowed to
understand them.

### Two planes: signaling and media

Keep these separate in your head. The signaling plane is small, reliable, and
occasional: "I want to call you," "here are my addresses and keys," "accepted,"
"hang up." It rides WhatsApp's existing persistent messenger connection, the
post office from the ticks teardown. The media plane is large, continuous, and
loss-tolerant: tens of packets per second of encrypted audio and video, sent
over UDP as SRTP. Public analyses of WhatsApp's stack describe exactly this,
media as SRTP over UDP to WhatsApp voip/relay servers, with the transport
endpoints exchanged over signaling.

Why UDP and not TCP for media? Because TCP guarantees delivery by retransmitting
and waiting in order, and for live voice that guarantee is the enemy. A late
packet is worthless, and TCP's head-of-line blocking would freeze everything
behind it. UDP lets a lost packet simply be lost, and the media engine decides
what to do about it. This is the single most important design choice in
real-time media.

### The connectivity problem: NAT, STUN, TURN, ICE

Priya's phone does not have a public address the world can dial. It sits behind
her home router (one NAT) and, on 4G, behind her carrier's NAT (a second one).
Her mother's phone is the same. Two devices that both hide behind NATs usually
cannot just send each other packets.

The toolkit that solves this is ICE (Interactive Connectivity Establishment),
which uses two helpers:

- STUN: the phone asks a STUN server "what address do you see me coming from?"
  and learns its own public-facing address and port. Often that is enough for
  the other side to reach it. This is a tiny request, a few bytes.
- TURN: when both NATs are too strict for any direct path, a TURN relay in the
  middle accepts packets from each side and forwards them. Neither phone ever
  talks to the other directly; they both talk to the relay. This always works,
  at the cost of an extra hop.

ICE gathers all candidate addresses (local, STUN-derived, relay), the two sides
trade the lists over signaling, and they probe pairs until one works, preferring
the cheapest. The data structure here is just a small ranked list of candidate
pairs with priorities, checked in order. WhatsApp's twist, per public write-ups,
is that it does not wait for that probing to finish before making sound: it
starts on the relay for a fast connect, then upgrades to peer-to-peer for a
one-to-one call if a better path appears. Start reliable, optimize later.

### Why WhatsApp leans on the relay even when P2P is possible

A pure peer-to-peer call leaks Priya's IP address to her mother's phone, and to
anyone who can see that traffic. IP reveals rough location. So in November 2023
WhatsApp shipped "Protect IP address in calls," which forces every call through
its relays so the other party only ever sees a WhatsApp server address, not
yours (Meta engineering, 2023). The published tradeoff is a slight quality dip
from the extra hop. This flips the usual WebRTC instinct: P2P is not always the
goal, because privacy can be worth a relay hop. Group calls go through the relay
regardless, because meshing every pair of 32 people is hopeless (more on that
below).

### The resilience problem: the jitter buffer

Once packets are flowing, the receiver faces jitter: her mother's packets,
sent evenly every 20ms, arrive unevenly (18ms, then 35ms, then 12ms, one out of
order, one missing). If you played them the instant they arrived, the audio
would stutter and click.

The fix is a jitter buffer. WhatsApp's audio path builds on the WebRTC media
engine, whose jitter buffer is NetEQ. It is worth knowing as a data structure
because it is a small masterpiece:

- Incoming RTP packets go into a packet buffer, keyed and ordered by their
  timestamp/sequence number. A packet that arrives too late to be useful is
  discarded on insert. Everything else waits its turn. (The public NetEQ docs
  describe exactly this: an InsertPacket path that drops the hopelessly late and
  stores the rest, and a GetAudio path that asks for 10ms of sound at a time.)
- The buffer computes a target delay from recent network behavior. Hold too
  little and a late packet causes a gap; hold too much and the call feels
  laggy. NetEQ constantly retunes this target as the network changes.
- If the buffer is running fuller than target, it speeds playout up slightly
  (acceleration). If it is running low, it stretches playout (preemptive
  expand). These are done by resampling in a way the ear does not notice. So
  the buffer silently trades a few milliseconds of delay against smoothness,
  packet by packet.

The lesson buried here: the buffer is a shock absorber. Sparse, jittery truth
comes in; smooth sound goes out. This is the same "sparse truth, smooth fiction"
idea as Swiggy interpolating a rider's position at 60fps over GPS fixes that
truly land every few seconds (2026-06-24 teardown), applied to sound.

### Fighting packet loss: FEC, NACK, PLC

The Pune kitchen drops packets. Three tools, cheapest first:

- FEC (forward error correction): the codec tucks a low-bitrate copy of the
  previous audio frame inside the next packet. If packet N is lost but packet
  N+1 arrives, the receiver reconstructs N from the redundant copy. Opus's
  in-band FEC recovers cleanly up to roughly 15% packet loss. It costs a little
  bandwidth always, but it needs no round trip, which is why it is the first
  line for audio.
- NACK (retransmission): the receiver notices a gap and asks the sender to
  resend that specific packet. This costs a full round trip, so it is only
  worth it when the trip is short relative to how much buffer you are holding.
  Used heavily for video, sparingly for low-latency audio.
- PLC (packet loss concealment): if a packet is simply gone and cannot be
  recovered in time, the decoder invents a plausible replacement from the sound
  around it, so you hear a soft smear instead of a click.

The engineering judgment is which tool for which stream: proactive FEC for
audio (no round trip), reactive NACK for video (can afford it), PLC as the last
resort for both.

### The codec: MLow, and why an old phone matters

Compression is where a call lives or dies on a weak pipe. WhatsApp long used
Opus, the open standard. In June 2024 Meta shipped MLow (Meta Low bitrate), a
new codec now rolling out across WhatsApp, Messenger, and Instagram calls
(Meta engineering, 2024).

The numbers are the story. At 6 kbps wideband, MLow scores about a POLQA MOS of
3.9 against Opus's 1.89, which Meta frames as roughly twice the quality at the
same tiny bitrate, while using about 10% less compute than Opus. Two wins that
usually fight each other: better sound and cheaper to run.

Why build a whole codec for this? Because of who is actually calling. Meta
reports that more than 20% of its calls happen on ARMv7 devices, and tens of
millions of WhatsApp calls a day happen on phones more than ten years old. For
that phone in the Pune kitchen, "twice the quality at 6 kbps with 10% less CPU"
is the difference between a usable call and a dropped one. Cheaper compute also
means the phone's battery lasts and the phone does not overheat. And encoding
good audio at a very low bitrate leaves room to pack more FEC in the same
budget, so low-bitrate audio is also more loss-resistant. The codec choice is a
product decision about your worst-off user, not your best-off one.

### Group calls: the hard part, because the relay is blindfolded

A one-to-one call can go peer-to-peer. A 32-person call cannot: full mesh means
every phone sends its stream to 31 others and decodes 31 incoming streams,
which melts the phone and the network. So group calls run through a server that
receives each person's stream once and forwards it to the others. In generic
WebRTC this server is called an SFU (Selective Forwarding Unit): it forwards,
it does not mix.

Here is WhatsApp's genuinely hard constraint, the one most video platforms do
not have. The media is end-to-end encrypted, so the relay cannot read it. It
cannot decode a stream, it cannot mix voices, and crucially it cannot transcode
video down to a smaller size for the person on a weak network. A normal video
server handles a slow viewer by re-encoding the stream smaller. WhatsApp's relay
is blindfolded; it can only forward the ciphertext it was given.

So how does a group call survive one person on bad WiFi when a phone on great
WiFi is blasting high-bitrate video? WhatsApp's answer, described in its At
Scale relay talk and echoed in Meta writeups, is video simulcast. When the
relay detects mixed network quality in a call, it tells the phones on good
networks to encode and send two versions of their video at once, one
high-bitrate and one low-bitrate. Both are separately encrypted and sent up.
The relay, without ever decrypting anything, simply chooses which already-made
version to forward to each recipient: the high one to people on good networks,
the low one to the person in the Pune kitchen. The intelligence moves from the
server (which is not allowed to think about the pixels) to the sender (which is)
plus a dumb selection at the relay. Encryption forced the design, and the design
is elegant: precompute both qualities at the trusted edge, let the untrusted
middle pick.

Group-call key management follows the same spirit as WhatsApp's group messaging.
Rather than encrypt each frame separately for all 31 others, the sender uses a
shared group session so the media is encrypted once, and the relay fans it out.
That keeps the sender's cost roughly constant instead of scaling with group
size, the same "encrypt once, server fans out" move as the Sender Keys design
in the 2026-07-01 encryption teardown. (Exact WhatsApp group-call key internals
are not fully public; this is the well-grounded inference from its published
group-messaging design and general E2E group-call practice.)

### The scale story at three tiers

Tier 1, about 1,000 concurrent calls. Almost anything works. A handful of relay
machines in one region, plenty of one-to-one calls going peer-to-peer and never
touching a server at all. The jitter buffer and codec on each phone do the real
work. Nothing here forces cleverness. This is where a small startup ships and
wrongly concludes calling is easy.

Tier 2, about 100,000 concurrent calls. Now relay capacity is a real resource.
You need many relay servers spread across regions so that Priya in Mumbai lands
on a relay near Mumbai, not one in Virginia, because every extra 100ms of
distance is heard. You need to place users on the right relay, balance load
across relays, and handle a relay dying mid-call by moving flows. Group calls
are now common, so the encrypted-forwarding-plus-simulcast machinery has to be
solid, not a demo. The bottleneck has shifted from the phone to the relay fleet
and its placement.

Tier 3, 10 million or more concurrent calls, which is a normal evening for
WhatsApp. Two things dominate. First, the media is huge and continuous, so the
relays must be many, regional, and cheap to run, and the whole system is built
for graceful degradation rather than perfection. WhatsApp's stated engineering
philosophy for this layer is to keep the relay simple enough to reason about
under stress, favoring simplicity and resilience over cleverness. Second, the
load is not smooth. The worst case WhatsApp calls out is New Year's Eve: as the
clock strikes midnight in each country, calling volume spikes extremely steeply
and stays elevated for about 20 minutes, then the wave moves to the next time
zone an hour later. You cannot autoscale fast enough for a spike that steep, so
the resilience has to be baked into the relay architecture ahead of time:
provision for the peak, shed and degrade gracefully rather than fall over, and
localize the blast radius so one hot region does not take down the rest. This
is the same shape as the hot-key and cell-based-architecture lessons in the
ledger, applied to a global synchronized rush.

Notice the recurring spine. The expensive, careful thinking is pushed to where
it is cheap and safe: the codec and jitter buffer live on the phone, the
simulcast decision lives at the sending edge, capacity is pre-provisioned before
the spike. The live server path is kept deliberately dumb and fast: receive an
encrypted packet, look up who wants it, forward it. Offline-think, online-lookup,
in a real-time coat.

## 8. The retention and habit mechanic

Calling is not a slot-machine engagement loop like a feed. Its retention power
is defensive, and it is one of the strongest moats there is: the network effect
plus switching cost.

The habit is simple: when calling your mother is free, instant, and works on her
old phone on a bad Pune signal, you stop using anything else. Her contacts are
here. Your contacts are here. The call quality is good enough that the
carrier's own phone line feels pointless and expensive by comparison. Every good
call quietly deepens the assumption that "to call family, you open WhatsApp."
Over 5.5 billion voice calls a month is not a campaign, it is a reflex.

The metric it moves is retention and defensiveness, not a vanity click count.
This is why Meta spent real engineering on MLow for ten-year-old phones and on
surviving New Year's Eve. The marginal user they protect is exactly the one a
competitor could steal: the person on the weak network and the cheap phone. Keep
that call from dropping and you keep the whole family locked to the platform,
because a family group call only moves to a rival app if it works for everyone,
including grandmother's phone. The weakest device in the group is the retention
battleground, and WhatsApp engineers to win it.

A concrete example of the loop hardening: the "Protect IP address in calls"
setting. It costs a little quality, but it turns "I can call safely without
leaking my location" into another reason never to use a plain phone call or a
lesser app. Trust becomes switching cost.

## 9. The lesson for Rare.lab

Rare.lab is an AI shader and visual-effects product: a node-based editor that
compiles to shippable code, plus an embeddable runtime. The runtime ships to a
brutally uneven field of devices, exactly like a WhatsApp call: flagship phones,
mid-range Androids, six-year-old GPUs, weak thermal budgets. WhatsApp's calling
design gives you three concrete, scalability-first moves.

1. Engineer for the weakest device in the room, and make it the headline metric.
   MLow exists because 20% of Meta's calls are on ARMv7 and tens of millions
   run on ten-year-old phones, so "twice the quality at 6 kbps with 10% less
   CPU" was worth building a codec for. Do the same for your runtime: pick your
   p10 device (old integrated GPU, tight power budget) and make "does this effect
   hold 60fps there" the number you optimize, not the RTX demo. In a shared
   scene, the weakest client caps everyone, just like the weakest phone caps a
   group call.

2. Precompute multiple quality tiers at the trusted edge, and make the runtime
   pick, not compute. WhatsApp cannot transcode encrypted video, so it makes
   the sender emit a high and a low stream (simulcast) and the relay just
   selects. Mirror that at compile time: for each effect, your compiler should
   emit a small ladder of prevalidated variants (full, reduced, minimal:
   different sample counts, LOD, precision) keyed by device capability. The
   runtime's hot path is then a cheap lookup and select against a measured
   frame-time budget, never a live "compile and hope" on the user's GPU. Push
   the expensive thinking offline; keep the online path a dumb fast selection.

3. Separate the control plane from the media plane, and never let control block
   frames. WhatsApp splits small reliable signaling from large loss-tolerant
   media, and it drops a late media packet rather than stall. Your runtime
   should treat parameter and graph updates (control) as one channel and the
   per-frame render loop (media) as another, with a hard per-frame millisecond
   cap and graceful degradation: if a node cannot finish this frame, drop
   detail or reuse last frame, do not freeze the whole scene. Like a call
   sacrificing video before audio, decide in advance what your runtime sheds
   first when the budget is blown, so it degrades instead of hitching.

The through-line: the trusted edge does the smart, expensive work ahead of time
and ships tiers; the untrusted or constrained runtime does the cheap, fast,
never-blocking selection. That is how a call stays clear on a ten-year-old phone
in a weak-signal kitchen, and it is how a shader stays at 60fps on a laptop from
2018.

---

## Sources

- Business of Apps, WhatsApp Revenue and Usage Statistics (2026): https://www.businessofapps.com/data/whatsapp-statistics/
- SQ Magazine, WhatsApp Statistics 2026 (call volumes and minutes): https://sqmagazine.co.uk/whatsapp-statistics/
- Meta Engineering, "MLow: Meta's low bitrate audio codec" (June 13, 2024): https://engineering.fb.com/2024/06/13/web/mlow-metas-low-bitrate-audio-codec/
- Meta Engineering, "Enhancing the security of WhatsApp calls" (Nov 8, 2023): https://engineering.fb.com/2023/11/08/security/whatsapp-calls-enhancing-security/
- At Scale Conference, "Calling Relay Infrastructure at WhatsApp Scale": https://atscaleconference.com/calling-relay-infrastructure-at-whatsapp-scale/
- webrtcHacks, "What's up with WhatsApp and WebRTC?" (media plane, SRTP over UDP, relay/conf bridge): https://webrtchacks.com/whats-up-with-whatsapp-and-webrtc/
- webrtcHacks, "How WebRTC's NetEQ Jitter Buffer Provides Smooth Audio": https://webrtchacks.com/how-webrtcs-neteq-jitter-buffer-provides-smooth-audio/
- WebRTC NetEQ design docs (InsertPacket/GetAudio, packet buffer, target level, accelerate/expand): https://github.com/zhaoxiu-zeng/webrtc/blob/master/modules/audio_coding/neteq/g3doc/index.md
- GetStream, "Media Resilience in WebRTC" (FEC, NACK, PLC, RED): https://getstream.io/resources/projects/webrtc/advanced/media-resilience/
- Google Research, Holmer/Shemer/Paniconi, "Handling Packet Loss in WebRTC": https://research.google.com/pubs/archive/41611.pdf
- The Hacker News, "WhatsApp Introduces New Privacy Feature to Protect IP Address in Calls" (Nov 2023): https://thehackernews.com/2023/11/whatsapp-introduces-new-privacy-feature.html
- Deccan Herald, "WhatsApp gets Communities, group video call limit increased to 32": https://www.deccanherald.com/technology/whatsapp-gets-communities-group-video-call-limit-increased-to-32-1158991.html
- WhatsApp Blog, "Group Video and Voice Calls Now Support 8 Participants": https://blog.whatsapp.com/group-video-and-voice-calls-now-support-8-participants
