# Netflix Open Connect: how a show is sitting inside your ISP before you press play

Date: 2026-07-13
Product: Netflix
Feature: Open Connect (the private CDN, proactive fill, and play-time steering)

A quick note on where this fits. On 2026-06-30 this ledger tore down Netflix
adaptive bitrate: how a title is encoded per shot into a ladder of files, and how
the player on your phone picks a rung from its buffer. That teardown stopped at
the edge of the network. This one is the other half. Once those files exist, how
do the actual bytes get from Netflix to a TV in a flat in Pune fast enough that
"Stranger Things" starts in under a second and never stalls? The answer is not
"a big data center." It is thousands of Netflix's own servers physically living
inside internet providers, filled overnight with the shows the neighbourhood is
about to watch. That system is Open Connect.

---

## 1. The user

Meet Rohan. It is 9pm on a Friday in Pune. He has had a long week, he has just
finished dinner, and he flops onto the sofa and opens Netflix on his TV. The new
season of a show he has been waiting for dropped at 1:30pm today. He taps it. He
expects it to start playing more or less instantly, in sharp 4K, and to keep
playing without that spinning circle, for the next three hours, while roughly ten
million other people in India do the exact same thing tonight.

Rohan has no idea that any of this is hard. That is the whole point. He is not
thinking about servers or bandwidth. He is thinking about whether Eleven is okay.

## 2. The real problem

Here is the pain, described like a friend would.

A single episode of that show, in 4K, is a few gigabytes. Netflix has hundreds of
millions of subscribers. On a normal evening, Netflix alone is a large fraction of
all the internet traffic on Earth. At peak it has historically been on the order
of 15 percent of global downstream traffic. That is not a website. That is a
firehose the size of a small country's entire internet usage.

Now think about the naive way to serve it. Netflix keeps all its video in a few
big data centers (or in Amazon's cloud, which is where Netflix actually runs). When
Rohan presses play, his TV reaches across the internet, through his ISP, through
several other networks, all the way to that data center, and pulls the bytes back
the same long way. Multiply that by ten million people in India tonight, and by
hundreds of millions worldwide, and two things break at once.

First, cost and congestion. Every one of those streams has to cross expensive
long-distance internet links, the same links Rohan's ISP pays other networks for.
The ISP hates this. Netflix hates this. The bytes travel thousands of kilometres
when the same movie is being watched by the guy next door.

Second, quality. Distance is latency, and shared long links are congestion, and
congestion is the spinning circle. The further the bytes travel and the more
networks they cross, the more likely Rohan's stream stutters, drops to blurry
480p, or refuses to start. And Netflix has measured, repeatedly, that a stutter
or a slow start is exactly what makes someone stop watching and, eventually,
cancel.

So the problem is not "store the video." Storing it is easy. The problem is
delivering a country-sized firehose of bytes to hundreds of millions of specific
sofas, cheaply, and so smoothly that nobody ever notices the machinery.

## 3. The feature in one sentence

Open Connect is Netflix's own delivery network of purpose-built video servers
placed for free inside internet providers and at internet exchanges, pre-loaded
every night with the exact titles that region is about to watch, so that pressing
play pulls the bytes from a box a few kilometres away instead of from a data
center on another continent.

## 4. Jobs to be done

What is Rohan really hiring Open Connect to do (without knowing it exists)?

- "When I press play, start now." A sub-second start, not a five-second spinner.
- "Never make me wait mid-show." No rebuffering during the binge.
- "Give me the good picture." Steady 4K, not a drop to blurry when the network
  gets busy at 9pm.
- "Work on Friday night when everyone else is watching too." Survive the
  synchronized crush of a new-season launch.

And what the ISP is hiring it to do:

- "Stop dragging Netflix across my expensive upstream links." Keep the heavy
  traffic local so peering and transit bills fall.

Those two customers, the viewer and the ISP, are why Open Connect is shaped the
way it is.

## 5. How it works for the user

For Rohan, there is no feature to see. He taps a title. It plays. That is the
entire visible experience, and its invisibility is the achievement.

The one thing he might notice, if he were looking, is the "video has loaded
instantly" feeling even on a mediocre home connection, and the way quality holds
up on a Friday night when other streaming apps get flaky. That steadiness is Open
Connect doing its job. The magic is that the show he chose was very probably
already sitting on a Netflix server inside his own ISP's building, put there in
the small hours of this morning, before he had decided to watch it.

## 6. The actual flow, step by step

Two flows run at completely different speeds. One happens overnight. One happens
the instant Rohan taps play.

The overnight flow (the fill), roughly 2am to 6am local time:

1. Netflix's brain in the cloud predicts what Rohan's region will watch in the
   next day: new releases, trending titles, the long tail people keep returning to.
2. It ranks every video file by how popular it will be in that specific region.
3. During the ISP's off-peak window, it pushes those files down to the Netflix
   appliances sitting inside Rohan's ISP, newest and most popular first, until the
   boxes are full.
4. By morning, the neighbourhood's Netflix servers are pre-loaded and waiting.

The play flow, the moment Rohan taps (2am work is done, this is the live path):

1. Rohan taps the new season on his TV.
2. The TV asks Netflix's control plane in the cloud: "I want this title, where do
   I get the bytes?"
3. The control plane looks at three things: which nearby appliances actually have
   these files, which of them are healthy right now, and which are closest to
   Rohan on the network.
4. It hands the TV a short ranked list of URLs, best server first.
5. The TV opens a connection to the top server (very likely the box inside Rohan's
   own ISP, a few kilometres away) and starts pulling the video.
6. As it streams, the player picks bitrate rungs from its buffer (that is the ABR
   teardown from 06-30). If the chosen server ever misbehaves, the TV quietly
   fails over to the next URL on the list.

Notice the split. The overnight flow moves petabytes and takes hours. The play
flow is a single quick question to the cloud, then a fat local download. The cloud
never touches the actual video bytes.

## 7. Under the hood, like the engineer

This is the heart. Open Connect is best understood as a strict separation between
a control plane and a data plane, plus a prediction engine that pre-positions
data so the live path is trivial.

### The two planes

The control plane runs in Amazon's cloud (AWS). It handles everything except the
video bytes: sign-in, the recommendation homepage, billing, the "what files make
up this title" lookup, the health telemetry from every appliance, and the steering
decision. This is the thinking, coordinating, low-bandwidth half.

The data plane is Open Connect: thousands of Netflix's own servers, called Open
Connect Appliances (OCAs), whose only job is to store video files and stream them
out over HTTPS. This is the heavy, dumb, high-bandwidth half.

The rule is clean and it is the single most important design decision: AWS runs
everything except the video streaming; OCAs do nothing but stream video. Roughly
95 percent and up of Netflix's traffic is served from OCAs, not from the cloud.
The expensive petabyte firehose lives entirely in the data plane, close to users;
the cloud only ever sends tiny control messages.

This is the same offline-think, online-lookup spine that runs through this whole
ledger, but stretched across physical geography. The "offline" work is literally
trucking data into buildings overnight. The "online" work is a keyed lookup that
returns a URL.

### The appliance itself: a server tuned to do one thing at absurd speed

An OCA is not a general server. It is a box built to read files off disk and push
them onto the wire as fast as physically possible, and nothing else. Concrete
shapes Netflix has published:

- A storage OCA: a chassis packed with tens to a few hundred terabytes of hard
  disk, tuned for capacity, serving on the order of tens of Gbps. Good for holding
  the long tail (the huge catalog of things a few people watch).
- A flash OCA: all solid-state, smaller capacity (tens of TB), but able to serve
  in the range of ~100 to 190 Gbps because flash has no seek penalty. Good for the
  red-hot popular content that everyone hammers at once.
- Storage-dense NVMe designs packing hundreds of TB into a 2U chassis as the
  technology matured.

The software is deliberately boring and deliberately hand-tuned: a custom fork of
FreeBSD and NGINX serving files over HTTP/HTTPS. The interesting engineering is
below the application, in the kernel. Netflix's landmark result was "Serving 100
Gbps from a single Open Connect Appliance." The tricks that got them there:

- sendfile and asynchronous sendfile, so a video file goes from disk straight to
  the network card without being copied up into application memory and back down.
  The bytes never take a detour through NGINX's own buffers.
- Kernel TLS (kTLS). Almost all Netflix traffic is encrypted now. Encrypting in
  the application would mean copying every byte into user space to encrypt it,
  destroying the sendfile win. So Netflix pushed TLS encryption down into the
  kernel, on the same data path as sendfile. Result: they could serve close to 90
  Gbps of 100 percent TLS traffic on the default stack, and push toward and past
  100 Gbps with tuning. kTLS from this work was upstreamed into FreeBSD.
- NUMA awareness: on a two-socket machine, memory attached to the wrong CPU is
  slower, so they pin work so bytes are read and sent by the CPU that owns that
  slice of memory and that network card.

Why this matters at the scale story below: the whole point of one box doing 100
Gbps of encrypted video is that you need far fewer boxes to serve a city, which is
what makes it economical to give them away to ISPs.

### The data structure of "where is everything"

At its core the live path is a hash-map lookup, twice.

First, on the appliance: an OCA holds thousands of large files (each title is many
files, remember: multiple resolutions times codecs times audio tracks times
languages, so one show is often hundreds of files). Serving a request is an
exact-match "give me file X, byte range Y to Z." That is a point lookup and a
sequential disk read, which is why the storage is optimized for large sequential
reads, not random tiny ones. No tree, no scan, no ranking on the box. The box is
dumb on purpose.

Second, in the cloud: the control plane keeps a live map of which OCA holds which
files and how healthy each OCA is right now, keyed so that "who has this file near
this client" is a fast filtered lookup, not a search over the whole fleet.

### Matching then ranking, the CDN version: steering

Even delivery has the ledger's two halves. When Rohan taps play:

Matching (the candidate set). The control plane filters to the OCAs that (a)
actually hold the requested files and (b) are reporting health status OK right
now. A box that is overloaded, mid-fill, or degraded is dropped from the set. This
is the cheap "who is even eligible" pass.

Ranking (order the survivors). The survivors are ordered mainly by network
proximity to Rohan. Proximity is not guessed from a geo-IP database; it is
computed from BGP, the internet's own routing protocol. Every ISP that hosts an
OCA runs a BGP session with it and advertises which blocks of customer IP
addresses live behind that appliance. So when Rohan's request arrives, the
steering service knows, from real routing data, that Rohan's IP sits directly
behind the OCA in his own ISP. That box gets rank one. The list continues outward:
the OCA at the nearest internet exchange, then a regional site, and so on. This
ordered list is the "proximity rank."

The output is a short ranked list of URLs handed to the TV. First choice is the
closest healthy box that has the file. The TV tries it, and can fail over down the
list if needed. This is exactly matching-then-ranking, the same shape as search,
except the catalog is "servers that can feed you" and the ranking signal is
network distance and health instead of relevance.

Concrete walk: Rohan in Pune taps the new season. Matching finds three OCAs
holding those episode files: the one embedded in his ISP in Pune, one at the
Mumbai internet exchange, and a regional site further out. All three report OK.
Ranking uses BGP proximity: the Pune box is directly behind Rohan's IP, so it is
rank one. The TV streams from a server maybe five kilometres away. The bytes never
leave the city.

### The prediction engine: proactive fill

Here is the part that makes the live path so cheap. Open Connect does not wait for
Rohan to request the show and then fetch it (that is how a normal reactive cache
works, and it means the first viewer pays a slow cache miss). Open Connect fills
proactively, ahead of demand, because Netflix can predict what a region will watch
with unusual accuracy. It knows the release calendar (it made the show), it knows
regional taste, it knows the daily rhythm of when people watch.

So every night, during a fill window agreed with each ISP that lines up with their
off-peak hours (say 2am to 6am), Netflix computes a popularity ranking for that
region and pushes content down to the local OCAs. Two subtle, important choices:

- Ranking is per file, not per title. Each individual file is ranked on its own
  predicted popularity, and files from the same title can land in different parts
  of the rank. Why: the 1080p English file of a hit is far more requested than its
  obscure-audio-track 4K variant. Ranking per file means the boxes fill with
  exactly the bytes people will ask for, which measurably lifts cache efficiency
  versus filling whole titles as a unit.
- Filling at off-peak is not just about network politeness. It also avoids disk
  contention: writing new content to the disks while they are busy serving the
  evening peak would make both slower. Fill when the disks are otherwise idle, so
  the serving reads at peak are clean and fast.

The target that falls out of all this: cache-hit ratios above 95 percent at a
well-provisioned site. Meaning: more than 95 times out of 100, the file Rohan
wants is already on a nearby box. The rare miss is served from a larger site
upstream and classified and studied (Netflix has written about classifying cache
misses precisely so they can drive that number even higher).

### How the fill itself scales: tiered fill, not a thundering herd

There is a trap hiding in "push tonight's content to every OCA." If ten thousand
appliances all pull the new season directly from the origin at 2am, you have just
built the exact long-distance firehose you were trying to kill, only pointed
inward. Netflix avoids this with a fill hierarchy, a cascade:

- Cache fill: content comes from the origin (Netflix's storage in the cloud) into
  the network.
- Tier fill and peer fill: OCAs fill from other OCAs. A designated fill-master OCA
  downloads a title first. Once it reports "I have it," other OCAs in the same
  cluster fill from that master (peer fill, staying within the same manifest
  cluster or subnet). OCAs outside that cluster fill across from a filled tier
  (tier fill). When that second tier finishes, it becomes a source for a third
  tier, and so on.

So content spreads outward like a tree, each filled layer becoming the source for
the next, instead of everyone hammering the origin. OCAs are grouped into fill
clusters that share a content region and a popularity feed, so the right content
flows to the right geography. The result is that moving petabytes into thousands
of buildings every night does not itself melt the backbone.

### The scale story at three tiers

Tier one, 1,000 titles, a small service. You do not need any of this. Put the
files on one origin server or one commercial CDN and serve them. At this size a
cache miss to origin is cheap and rare. The whole Open Connect apparatus would be
absurd overkill.

Tier two, 100,000 titles, national scale, millions of viewers. Now origin
bandwidth and distance start to hurt. You add caching layers and regional points
of presence. A reactive CDN (fetch on first request, cache for the next viewer)
gets you most of the way. The pain that appears here is the cold cache miss: the
first person to watch a new release in a region pays a slow cross-country fetch,
and a synchronized launch means many "first" viewers at once.

Tier three, a global catalog measured in petabytes, hundreds of millions of
viewers, a firehose that is a double-digit percentage of the entire internet. This
is where reactive caching breaks and Open Connect's specific answers are forced:

- Reactive caching is not enough, because a new-season launch is a coordinated
  spike where everyone wants the same brand-new files in the same hour and there
  is no "previous viewer" to have warmed the cache. Answer: proactive predictive
  fill. Warm the cache before the spike, using the fact that Netflix knows the
  release date.
- Long-distance links break, because you cannot pull petabytes across the ocean
  for every viewer. Answer: put the boxes physically inside ISPs (embedded OCAs)
  and at internet exchanges (peering OCAs), so the bytes are local. Netflix gives
  the embedded appliances to qualifying ISPs for free, because a box in the ISP's
  rack is cheaper for everyone than traffic on the ISP's upstream links.
- The origin melts if every box fills from it, so fill is tiered into a cascade
  (above).
- One box is not enough for a city, so you shard content across a cluster of OCAs
  and steer each client to the right one by BGP proximity and health, with
  hot content on fast flash boxes and the long tail on high-capacity disk boxes.
- Serving hardware is the cost floor, so each box is tuned to do ~100 Gbps of
  encrypted video (kTLS plus sendfile) to minimize how many boxes a city needs.

The through-line: at tier three the only way to win is to stop moving the firehose
at request time. Move it in advance, overnight, in a cascade, to boxes that are
already next to the viewer. Then the live request is a one-line steering lookup
and a local read.

### Fact versus inference

Fact, from Netflix's own engineering writing and Open Connect documentation: the
control-plane-in-AWS versus data-plane-in-OCA split; proactive fill during an
off-peak window; the peer-fill and tier-fill cascade with a fill-master; per-file
popularity ranking; BGP-based proximity steering with health and file-availability
filtering; the FreeBSD plus NGINX plus kTLS plus sendfile stack and the 100 Gbps
result; the >95 percent hit-ratio target; the two appliance families (storage vs
flash); embedded vs internet-exchange deployment.

Inference, labeled as such: the exact per-region popularity model, the precise
math that turns predicted views into a fill ranking, and the exact steering
scoring formula are not fully public. The description above is the well-grounded
"this is how this class of predictive-fill and proximity-steering system is built"
version, consistent with what Netflix has published but not a leak of the exact
weights.

## 8. The retention and habit mechanic

Open Connect is not a feature Rohan clicks, so its retention mechanic is not an
engagement loop like a Monday playlist or a vanishing story. It is defensive:
performance as retention. The loop is invisible and it runs like this. Bytes are
local, so playback starts fast and never stalls, so Rohan finishes the season, so
Rohan keeps watching, so Rohan does not cancel. Quality of experience is the
switching cost.

This is not a hand-wave. Netflix has long treated startup delay and rebuffering as
first-class metrics precisely because they correlate with people watching less and,
over time, churning. Every reduction in the spinning circle is a reduction in the
reasons to leave. The metric Open Connect moves is retention (through engagement),
and secondarily cost (it slashes the bandwidth bill, which is what lets Netflix
afford 4K for everyone).

The real observed example is a tentpole launch. When a huge new season drops
worldwide on a Friday, Open Connect has spent the preceding nights pre-filling
every embedded appliance in every ISP with all the episodes, in every resolution
and language that region needs. So at 1:30pm when it unlocks, and again at 9pm when
the whole country sits down at once, the synchronized global binge is served almost
entirely from boxes already inside each ISP. The internet does not melt, nobody
gets a spinner, and the event feels effortless. That effortlessness on the biggest
nights is exactly what keeps the subscription feeling worth it.

## 9. The lesson for Rare.lab

Rare.lab is an AI shader and visual-effects tool: a node-based editor that compiles
to shippable code, plus an embeddable runtime that many websites and apps will pull
effects from. The Open Connect lesson is direct and it is about the runtime, not
the editor.

Split the control plane from the data plane, and pre-position the heavy compiled
artifacts close to where they run, predictively, so the live path is a keyed local
read and never a compile.

Concretely:

- Never compile a shader on the end user's device at first paint. Compilation is
  your slow "cross-continent fetch." Compile offline in the editor and produce a
  content-addressed bundle (the compiled shader or effect, plus its assets). The
  runtime's live job should be a lookup-and-load of a pre-built artifact, not a
  build. This is the exact offline-think, online-lookup move: the OCA does not
  encode video, it only serves it; your runtime should not compile, it should only
  run.

- Give the compiled bundles their own edge distribution and warm it ahead of
  demand. If a customer is launching a campaign page on Friday with your effect,
  push that effect's compiled bundle to the CDN edges near that audience before
  the launch, the same way Open Connect fills OCAs the night before a premiere. A
  first visitor should hit a warm cache, not a cold compile-and-fetch. Design a
  cache key that is per-artifact and content-addressed, and let popularity (which
  effects are trending) drive what stays hot at the edge.

- Steer clients to the best source by real signals, and always ship a fallback
  chain. Hand the runtime a short ranked list: nearest warm edge first, regional
  cache next, origin last, and let it fail over silently, exactly like the TV
  walking down the OCA URL list. One healthy source failing should never be a
  visible glitch in the effect.

- Tier your artifacts by heat like Netflix tiers boxes. Hot, common effects belong
  on the fast, always-warm path; the long tail of rarely-used custom effects can
  live on cheaper, slower storage and be fetched on demand. Do not pay to keep
  everything hot.

The scalability payoff is the same one Open Connect proves at internet scale: when
the expensive work (compile, encode) is done in advance and pushed close to the
user, the number of live requests you can serve stops being bounded by how fast you
can build things and becomes bounded only by how fast you can read a file. That is
the difference between a runtime that gets slower as you get popular and one that
does not.

---

## Sources

- Netflix Technology Blog, "Netflix and Fill" (proactive fill, peer fill and tier
  fill, fill clusters, fill-master cascade, off-peak fill windows):
  https://netflixtechblog.com/netflix-and-fill-c43a32b490c0
- Netflix Technology Blog, "Content Popularity for Open Connect" (per-file
  popularity ranking, regional prediction, cache efficiency):
  https://netflixtechblog.com/content-popularity-for-open-connect-b86d56f613b
- Netflix Technology Blog, "Serving 100 Gbps from an Open Connect Appliance"
  (FreeBSD, NGINX, sendfile, kernel TLS, NUMA, ~90 Gbps all-TLS result):
  https://netflixtechblog.com/serving-100-gbps-from-an-open-connect-appliance-cdb51dda3b99
- Netflix Technology Blog, "Driving Content Delivery Efficiency Through Classifying
  Cache Misses":
  https://netflixtechblog.com/driving-content-delivery-efficiency-through-classifying-cache-misses-ffcf08026b6c
- Netflix Open Connect, "Open Connect Overview" (control plane in AWS vs OCA data
  plane, steering, appliance families):
  https://openconnect.netflix.com/Open-Connect-Overview.pdf
- Netflix Open Connect Partner Help Center, "Fill patterns" (fill window, peer/tier
  fill mechanics):
  https://openconnect.zendesk.com/hc/en-us/articles/360035618071-Fill-patterns
- Netflix Open Connect Partner Help Center, "Network configuration" (BGP session
  requirement for steering and delivery):
  https://openconnect.zendesk.com/hc/en-us/articles/360035533071-Network-configuration
- FreeBSD Foundation, "Case Study: Netflix Open Connect" (FreeBSD serving stack,
  kTLS upstreaming, throughput):
  https://freebsdfoundation.org/wp-content/uploads/2021/03/Netflix-Open.pdf
- APNIC Blog, "Netflix content distribution through Open Connect" (proximity rank,
  BGP-based steering, deployment models):
  https://blog.apnic.net/2018/06/20/netflix-content-distribution-through-open-connect/
- Netflix Open Connect, appliance specifications:
  https://openconnect.netflix.com/en/appliances/
