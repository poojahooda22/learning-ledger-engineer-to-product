# Spotify instant playback: how a song starts in a quarter of a second

Date: 2026-07-07
Product: Spotify
Feature: Instant playback and streaming (the tap-to-sound path: local cache, the old P2P overlay, and today's CDN)

---

## 1. The user

Meet Riya. It is 9:10 on a Tuesday morning and she is on the Mumbai local,
packed shoulder to shoulder, one hand on the overhead rail and the other holding
her phone. She wants one specific thing: Queen, "Bohemian Rhapsody", right now,
because she has exactly 42 minutes of commute and she has decided this is a
Queen morning. She opens Spotify, types "bohem", taps the first result, and the
piano intro is already playing before her thumb has fully left the glass.

She does not think about any of this. She thinks the song "just played". The
whole point of the feature is that Riya never once wonders where the audio came
from, whether it came at all, or why it did not stutter when the train dived into
the tunnel between Dadar and Matunga.

## 2. The real problem

Here is the thing your friend would tell you over chai. A song is a big file. At
Spotify's normal quality a three-minute track is roughly 5 to 6 megabytes. At
high quality it is more like 11 or 12. If Riya's phone had to download that whole
file before the first note, she would be staring at a spinner for several seconds
on a good connection and much longer on a train. Several seconds of nothing is
death for a music app. People tap a song because they want the feeling *now*, not
after a progress bar.

So there are really three separate pains hiding inside "play this song":

- **The start has to be instant.** Not fast. Instant. The gap between tap and
  sound is the entire product. Spotify decided the target was a couple hundred
  milliseconds, the point where a human stops perceiving delay.
- **It cannot stutter.** Once the music is playing, a half-second of silence in
  the middle of the guitar solo is worse than a slow start. The buffer must
  survive the tunnel.
- **It cannot bankrupt the company.** In 2010 Spotify was a startup streaming to
  millions. Every byte that leaves a Spotify server costs Spotify money. Serving
  the full song to every listener from the datacenter, over and over, was a
  bandwidth bill that scaled straight up with popularity. The more users loved
  it, the more it cost. That is a trap.

Solving any one of these is easy. Solving all three at once, cheaply, for a
catalog of millions of songs and tens of millions of listeners, is the feature.

## 3. The feature in one sentence

Spotify starts the first few seconds of your song from a server so playback
begins almost instantly, then quietly fills the rest of the track from your own
local cache and (in the 2009 to 2014 era) from other listeners' computers, so
the music never stops and the servers barely sweat.

## 4. Jobs to be done

What is Riya really hiring this feature to do?

- "When I tap a song, give me sound before I can doubt it." (Instant start.)
- "Do not embarrass me in the tunnel." (No stutter.)
- "Let me line up the next three songs and have them play gaplessly." (Prefetch.)
- "Do not eat my mobile data replaying songs I already heard today." (Cache.)

And what is Spotify the company hiring it to do?

- "Serve 8 million tracks to millions of people without a bandwidth bill that
  grows as fast as our user count." (Offload the servers.)

Those last two jobs are the ones that shaped the whole design. The user wants
instant and smooth; the company wants cheap. The clever part is that the same
architecture delivers both.

## 5. How it works for the user

From Riya's seat, the experience is almost aggressively boring, which is the
compliment. She taps "Bohemian Rhapsody". Sound. She adds "Don't Stop Me Now" and
"Under Pressure" to the queue. When the first song ends, the second begins with
no gap and no spinner, even though the train is now underground and her bars have
dropped to one. She skips halfway through "Under Pressure" to hear the David
Bowie bit again; the seek lands instantly because that chunk is already on her
phone.

The only time she would ever see the machinery is if things went badly: a brief
"buffering" if she jumped to a brand-new song with no signal at all. On a normal
day she never sees it. Below one percent of playbacks stutter, per Spotify's own
measurements.

## 6. The actual flow, step by step

1. Riya taps "Bohemian Rhapsody".
2. The client checks its **local cache** first. Has this phone played this exact
   track recently? If yes, most or all of the file is already on disk. Playback
   starts from local storage in milliseconds and needs the network for nothing.
   This is the fastest path and, it turns out, the most common one.
3. If it is not fully cached, the client immediately asks a **Spotify server**
   for the *beginning* of the track. Just the head, the first few seconds. This
   is the urgent request, and the server always answers it. The piano intro
   starts playing off these first bytes.
4. At the same instant, the client starts hunting for the rest of the track from
   cheaper sources: its own partial cache, and (2009 to 2014) the **peer-to-peer
   overlay**, other Spotify desktop clients nearby in the network who have this
   track cached.
5. As those cheaper sources deliver bytes, the client **throttles down its use of
   the server**. It only leans on the datacenter for the part it needs right now
   and cannot get elsewhere in time. Everything past the opening it prefers to
   pull from peers and cache.
6. Meanwhile the client watches the play queue. When "Bohemian Rhapsody" is
   getting close to done, it **prefetches** the head of "Don't Stop Me Now" so the
   next song starts with zero gap.
7. If the buffer ever runs dangerously low mid-song (P2P peers dropped, train
   went dark), the client hits the **emergency button**: it re-requests the
   missing pieces urgently from the server. The server is the safety net that
   guarantees the music does not stop.
8. Everything Riya plays lands in the local cache, and (in the P2P era) her
   client then quietly served those cached songs to other listeners.

That is the whole loop: **cache first, server for the urgent head, peers and
cache for the bulk, server again only in emergencies.**

## 7. Under the hood, like the engineer

This is the heart of it. Spotify's streaming had two very different eras, and the
contrast is the whole lesson, so I will walk both. The deep primary source is
Gunnar Kreitz and Fredrik Niemelae's paper "Spotify: Large Scale, Low Latency,
P2P Music-on-Demand Streaming" (IEEE P2P 2010). The modern CDN design is described
in Spotify's own engineering posts and their 2016 move to Google Cloud. Where I go
past what is published, I label it inference.

### The file, and why it is cut into pieces

A track is stored as **Ogg Vorbis**, normal quality around 160 kbps, high quality
around 320 kbps. "Bohemian Rhapsody" is 5 minutes 55 seconds, so at 160 kbps that
is roughly 7 MB. The client does not treat it as one 7 MB blob. It treats it as a
sequence of small **pieces** (16 kB each in the paper's design). This is the first
important data-structure decision, and it is the same one BitTorrent made: if the
unit of transfer is a small piece, then you can fetch different pieces from
different places at the same time, and you can start playing piece 1 while piece
40 is still in flight.

Think of the track as an **array of pieces**, indexed 0, 1, 2, and so on. Playback
is a cursor walking that array left to right. The job of the network layer is
simply to make sure the piece under the cursor is present before the cursor
arrives. Everything else is bookkeeping about *where* to get each piece from
cheapest.

### Path 1: the cache (the fastest and most common source)

Every Spotify client keeps a **local cache** of the music it has played. In the
2010 design the default cache size was 10 percent of free disk space, bounded
between roughly 56 MB and 10 GB, and it evicted with a policy close to **Least
Recently Used** but weighted so that recently and frequently played tracks
survive longest. On a phone this is why replaying "Bohemian Rhapsody" an hour
later uses zero network: the array of pieces is already on disk.

The single most striking number in the whole paper: **55.4 percent of all music
data played came straight from the local cache.** More than half the bytes never
touched the network at all. The cheapest request is the one you never make. If
you remember one thing about how Spotify felt fast, it is this: the design bets
that people replay music, and it makes the replay free.

At the data-structure level the cache is a **hash map** from track id to the set
of pieces held on disk, plus an LRU ordering (a linked list or heap) for eviction.
A cache lookup is O(1): does this phone have piece 0 of this track id? If yes, play
it. No server, no peers, no waiting.

### Path 2: the server head (how the start becomes instant)

When the track is not cached, the trick for an instant start is to **split the
work by urgency, not by source.** The client sends an urgent request to a Spotify
server for only the *opening* of the track, enough to start playing and cover the
first several seconds. The server is fast and reliable, so this head arrives
quickly and the piano intro plays. That urgent head is the reason the median
**time from tap to sound was 265 milliseconds**, with the 75th percentile at 515 ms
and the 90th at 1047 ms. A quarter of a second, median, including all the cached
plays.

Crucially the client does **not** keep pulling the whole song from the server. It
gets the head, then as cheaper sources warm up it **throttles the server down**.
This is the money-saving move. The server only ever paid for the part that had to
be instant.

### Path 3: the P2P overlay (2009 to 2014, the bandwidth trick)

Here is where the old Spotify was genuinely clever. To get the *bulk* of the track
cheaply, the desktop client joined an **unstructured peer-to-peer overlay**: a big
graph where each node is a Spotify client and edges are live TCP connections to
other clients. No supernodes, no DHT, everyone equal. A client kept at most about
**60 neighbors**.

To find peers who have "Bohemian Rhapsody", the client used **two mechanisms at
once**:

1. **A tracker**, exactly like BitTorrent. A Spotify server kept a list, per
   track, of clients that recently played it, and handed back up to about 10 of
   them. This is a **hash map from track id to a short list of recent peers.**
2. **A broadcast query in the overlay.** The client asked its own neighbors "who
   has this track?", and they asked their neighbors, out to a small radius. This
   is a bounded **breadth-first search over the peer graph**, capped so it never
   floods the network.

The two together mean: the tracker finds peers anywhere in the world who played
the song, and the local broadcast finds peers close to you in the graph who might
be faster. The client then downloaded different **pieces** of the array from
different peers in parallel, the same way a torrent does, while a peer's own
cached tracks were simultaneously uploaded to others.

The payoff: **35.8 percent of all played data came from other peers, and only 8.8
percent came from Spotify's servers.** Read that again. Over 91 percent of the
music bytes Spotify served in 2010 did not come from Spotify. By 2011 Spotify
reported that of tracks not played through the web player, around **80 percent**
went through the P2P network. The bandwidth bill was mostly paid by the users'
own machines, and the servers only had to cover the urgent heads and the
emergencies.

### The safety net (why it never stuttered)

Peers are flaky. They log off mid-song, their upload dies, the train enters a
tunnel. So the client constantly watched its **buffer**, the runway of pieces
already downloaded ahead of the play cursor. If that runway shrank toward
dangerous (a few seconds left), the client stopped trusting peers for those pieces
and sent an **urgent re-request to the server**. The server always answered urgent
requests. This is why below **1 percent** of playbacks stuttered even though a
third of the data came from unreliable strangers: the datacenter was always
standing behind the P2P layer, ready to cover a gap in milliseconds. Peers for
cheap, server for certainty.

### Prefetch (why the next song has no gap)

The queue is known in advance, so the client cheated on the future. When the
current track was near its end (roughly the last chunk, on the order of the final
10 to 30 seconds), it **prefetched the head of the next track** in the queue. So
when "Bohemian Rhapsody" ended, the opening of "Don't Stop Me Now" was already on
disk and playback was gapless. Prefetch is just the same instant-start trick moved
earlier in time: pay the 265 ms cost while the user is not looking.

### The scale story at three tiers

**1,000 tracks, a few thousand users.** You do not need any of this. One server
with the files on disk streams every song directly. Cache helps replays, but P2P
and CDN are pure overhead. At this size, simplicity wins. Just serve the file.

**100,000 tracks, hundreds of thousands of users.** Now the server bandwidth bill
is real and popularity is spiky. A hit single gets hammered while the long tail
sits idle. This is where **cache-first plus the urgent-head split** starts paying
off: replays go free, and the server only pays for the openings. A small CDN or a
P2P overlay begins to make economic sense because it moves the repeated bulk bytes
off your origin. What breaks at this tier is naive "stream the whole file from
origin every time"; the fix is to stop serving bytes you have already served.

**10 million plus tracks, tens of millions of users (Spotify, 2010 to 2014).**
Origin bandwidth is now the dominant cost and it grows with your success. This is
the tier where Spotify reached for **P2P**: push 90 percent of the bytes onto the
users themselves, keep only the urgent 8.8 percent on the servers. It worked
because the workload is perfect for it: the same few thousand popular tracks are
requested by huge overlapping crowds, so for any track you want there are many
peers who just played it. The thing that breaks is the origin bandwidth curve; P2P
flattened it by making every listener also a server.

### Why they killed P2P (the second era)

And then, in April 2014, Spotify **shut the P2P network down** and went fully
server plus CDN. This is the part engineers should sit with, because the earlier
design was clever and they threw it away on purpose.

Three things changed underneath the feature:

- **Mobile ate everything.** Phones were the majority of listening, and phones
  never ran the P2P overlay (battery, data caps, NAT, backgrounding). So P2P was
  optimizing a shrinking slice of traffic.
- **CDN and cloud economics collapsed.** By the mid 2010s, pushing bytes through
  a global **Content Delivery Network** (edge servers caching encrypted audio
  chunks close to the user) had become cheap and dead simple. A CDN gives you the
  same win P2P gave you, repeated bulk bytes served from somewhere that is not
  your origin, but without shipping a fragile peer protocol inside every desktop
  client. Spotify later moved its whole backend to Google Cloud in 2016.
- **Complexity has a cost.** A P2P overlay is a lot of code: NAT traversal, peer
  discovery, choke and unchoke, security, abuse. Every one of those is a bug
  surface and a support ticket. Once a CDN did the job, that complexity was
  pure liability.

So the modern flow is the same *shape* with a boring middle: **cache on the
device, urgent head and bulk from the nearest CDN edge, origin only on a miss.**
The 265 ms instant-start idea survived. The cache-first idea survived. The peer
overlay, the genuinely novel part, was retired the moment a simpler thing did its
job. (Inference, clearly labeled: the exact modern chunk sizes and cache
parameters on today's mobile client are not fully public, but the cache-first plus
edge-CDN structure is confirmed by Spotify's engineering writing and their GCP
migration.)

## 8. The retention and habit mechanic

The habit here is subtle because the feature's whole job is to be invisible. But
invisibility *is* the mechanic. Every time Riya taps a song and it plays before
she can doubt it, the app pays off instantly. There is no friction, no spinner, no
"is my connection bad?" moment of hesitation. That reliability is what makes
Spotify feel like a utility, like tap water, rather than an app you have to
negotiate with.

The metric it moves is **retention**, and it moves it by removing the small
frustrations that would otherwise accumulate into churn. A music app that stutters
on the train twice a week teaches you to distrust it. A music app that never
stutters becomes the default, the muscle-memory tap. Spotify measured this
obsessively; the entire P2P paper is organized around one number, playback
latency, because they understood that the 265 ms was not a vanity stat. It was the
retention lever.

The cache is also a quiet daily loop: the songs you love get faster the more you
play them, because replays come free from disk. Your favorites literally reward you
for repeat listening with instant starts. That is a habit loop built out of
physics, not notifications.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader and visual-effects code, plus
an embeddable runtime. The Spotify lesson maps almost one to one, and it is about
**splitting a load by urgency and serving the hot path from the cheapest place.**

Concretely:

1. **Cache-first, always.** More than half of Spotify's bytes never hit the
   network because people replay music. Rare.lab users will re-run the same graph,
   tweak one node, re-run again, thousands of times an hour. Make the compiler and
   runtime cache-first at the artifact level: content-address every compiled
   shader and every intermediate node output by a hash of its inputs, so an
   unchanged subgraph recompiles to nothing and re-renders from a warm cache. The
   cheapest compile is the one you skip.

2. **Split by urgency, not by source.** Spotify served only the *urgent head* from
   the expensive server and pulled the bulk from cheap places. In the runtime, do
   the same with frames: get *something* on screen instantly (a fast low-cost
   first pass or a cached previous frame) while the expensive high-quality pass
   fills in behind it. Never make the user stare at nothing while the perfect
   result computes. A 265 ms "something is happening" beats a 2-second "here is
   the perfect thing".

3. **Prefetch the known future.** Spotify prefetched the next track because the
   queue was known. In an editor, the next thing the user will do is often
   predictable: they are dragging a slider, so the next parameter values are a
   small neighborhood; they just added a node, so its default preview is coming.
   Precompile and pre-warm those before they are asked for, so the interaction
   feels instant.

4. **Keep a boring safety net behind the clever fast path.** Spotify's P2P was
   brilliant and fragile, so a reliable server always stood behind it and covered
   every gap in milliseconds. If Rare.lab ships a fancy fast approximate renderer,
   always keep a plain, correct, slower path one step behind it, ready to take over
   the instant the fast path cannot deliver. The clever thing earns trust only
   because the boring thing guarantees correctness.

5. **Be willing to delete your cleverest system when the boring one catches up.**
   This is the deepest one. Spotify's P2P overlay was the most impressive part of
   the 2010 design, and they deleted it in 2014 the moment a CDN did the same job
   with less code. Do not fall in love with a clever subsystem. If a simpler
   thing (a CDN, a cache, an off-the-shelf compiler pass) can carry the load, ship
   the simpler thing and retire the clever one. Complexity you do not need is not
   an asset, it is a bug surface with a maintenance bill.

The through-line, the same one that runs through this whole ledger: **do the
expensive thinking rarely and somewhere cheap, and make the live path a fast
lookup.** Spotify's live path was a cache hit or a 16 kB piece from a nearby peer.
Rare.lab's live path should be a cache hit or a cheap first pass. Same spine.

---

## Sources

- Gunnar Kreitz, Fredrik Niemelae. "Spotify: Large Scale, Low Latency, P2P
  Music-on-Demand Streaming." IEEE 10th International Conference on Peer-to-Peer
  Computing (P2P 2010). https://ieeexplore.ieee.org/document/5569963/
- Author's copy and slides: https://kreitz.se/spotify-p2p10/
- Mirror (PDF): https://people.cs.umass.edu/~phillipa/CSE390/spotify-p2p10.pdf
- TechCrunch. "Spotify Removes Peer-To-Peer Technology From Its Desktop Client"
  (April 17, 2014). https://techcrunch.com/2014/04/17/spotify-removes-peer-to-peer-technology-from-its-desktop-client/
- TorrentFreak. "Spotify Starts Shutting Down Its Massive P2P Network"
  (April 16, 2014). https://torrentfreak.com/spotify-starts-shutting-down-its-massive-p2p-network-140416/
- Engadget. "Spotify moves away from delivering music through peer-to-peer
  networks" (April 17, 2014). https://www.engadget.com/2014-04-17-spotify-phases-out-peer-to-peer.html
- Spotify Engineering / Google Cloud. Spotify's migration to Google Cloud Platform
  (2016), background on the CDN-plus-cloud backend.
