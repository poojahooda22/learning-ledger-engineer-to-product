# YouTube: the upload transcoding pipeline (chunked parallel encoding + the Argos VCU)

Date: 2026-07-17
Product: YouTube
Feature: The upload-to-playable transcoding pipeline (split into chunks, encode in parallel, stitch back, on custom Argos video chips)

---

## 1. The user

Meet Arjun. He runs a small gaming channel from a bedroom in Pune. Tonight he
finished editing a 12 minute Valorant highlights video. His export is a single
big 4K60 file, about 6 GB, sitting on his laptop. It is 11pm. He opens
youtube.com, drags the file onto the upload box, types a title, and waits.

He does not care about codecs. He does not know what a GOP is. He wants two
things. First, he wants the upload to finish before his patchy Jio broadband
drops. Second, and this is the part he never thinks about but feels
immediately, he wants the video to play instantly and smoothly for everyone who
clicks it: his friend on a gigabit fiber TV in Bangalore, a viewer on a cracked
Android phone on 3G in a village, someone on an iPhone, someone on a Smart TV in
Germany. Same video, wildly different screens and pipes.

Arjun uploads one file. YouTube has to turn that one file into dozens of files
so that every one of those viewers gets a version that fits their screen and
their bandwidth, and it has to do this fast, because a video stuck "processing"
is a video nobody can watch.

## 2. The real problem

Here is the honest version, the way you would explain it to a friend.

The file Arjun uploaded is useless to almost everyone as-is. It is 4K, encoded
in whatever his editor spat out, at a bitrate that would buffer forever on a
phone. A phone needs 360p or 480p. A laptop on office wifi needs 720p or 1080p.
A TV wants 4K. And each of those needs to exist in more than one compression
format, because an old iPhone can only decode H.264 while a modern Android can
decode VP9 or AV1 and get the same picture at half the bytes.

So one upload has to become a whole ladder of outputs: many resolutions (144p
all the way to 4K and 8K), times several codecs (H.264, VP9, AV1), times several
bitrates. That is the job called transcoding, and it is one of the most
compute-hungry things a computer can do. Encoding a single 4K video into the
full ladder on one machine can take longer than the video itself runs.

Now multiply. YouTube receives about 500 hours of new video every single minute
([Google Cloud / YouTube, widely reported figure](https://blog.youtube/press/)).
That is not a queue you can grind through on one server. If transcoding one
video takes an hour of machine time, and 500 hours arrive every minute, the math
does not just get big, it gets impossible on general-purpose CPUs alone. This is
the problem: turn a firehose of huge single files into a mountain of small
playable files, fast enough that "processing" is measured in minutes, and cheap
enough that it does not bankrupt you.

## 3. The feature in one sentence

Take one uploaded video, split it into short independently-decodable chunks,
encode all the chunks in parallel across a fleet of custom video chips into every
resolution and codec the world needs, then stitch the pieces back into finished
streams.

## 4. Jobs to be done

What is Arjun really hiring this pipeline to do?

- "Make my video playable on every device without me knowing what a codec is."
- "Get it live fast. Do not leave me staring at a processing spinner."
- "Make it start instantly and never buffer for my viewers, even the ones on
  weak networks, because buffering makes them close the tab."
- "Do not silently wreck my quality. My 4K should still look like 4K."

And what YouTube is hiring it to do for itself:

- "Serve 2.5 billion users smooth video without setting fire to the entire data
  center budget." Transcoding at this scale is a cost problem as much as an
  engineering one. That cost pressure is exactly what led Google to build its own
  chip, which we will get to.

## 5. How it works for the user

From Arjun's seat it is almost invisible, and that is the point.

1. He drags the file in. A progress bar shows the upload.
2. The moment the upload finishes, the video page appears with a note that says
   it is still processing HD (or SD, then HD, then 4K).
3. Within a minute or so, a low resolution version is already watchable. He can
   share the link and it plays, just not yet in full 4K.
4. Over the next few minutes the higher resolutions "light up." Refresh the
   quality menu and 1080p appears, then 2160p (4K).
5. He never picks a codec, never picks a bitrate. The player decides per viewer,
   per second, which rung of the ladder to pull.

The clever bit he never sees: the video became watchable in low res before the
whole ladder finished. That is not an accident. The pipeline is built so the
cheap universal version ships first and the expensive versions catch up.

## 6. The actual flow, step by step

Tap by tap, then machine by machine.

1. Arjun drops the 6 GB file. The browser does a resumable, chunked upload: the
   file is cut into pieces (tens of MB each) and sent piece by piece, so a
   dropped connection resumes from the last acknowledged piece instead of
   restarting. (This upload chunking is a different thing from the transcoding
   chunking below. One is about surviving a flaky network; the other is about
   parallel encoding.)
2. The raw file lands in distributed storage. A processing job is queued.
3. A splitter reads the raw video and cuts it into short segments, each starting
   on a keyframe so it can be decoded on its own. Call it roughly a few seconds
   per segment.
4. Each segment becomes many independent encode tasks: segment 7 into 360p
   H.264, segment 7 into 720p VP9, segment 7 into 1080p VP9, and so on. These
   tasks fan out across a huge fleet of encoders.
5. Encoders (on YouTube's Argos video chips) chew through the tasks in parallel.
   Hundreds of chunks of one video can be in flight at once on different chips.
6. A stitcher collects the finished encoded segments for each output stream and
   concatenates them back into one continuous file per resolution-plus-codec,
   fixing up timestamps so playback is seamless across segment boundaries.
7. Each finished stream is packaged for adaptive streaming (broken into the
   small delivery segments the player fetches) and pushed out toward the CDN and
   edge caches so viewers pull it from nearby.
8. The video page flips each rung from "processing" to "available" as its stream
   finishes. Low, cheap rungs finish first; 4K AV1 finishes last.

That is the shape: split (map), encode in parallel (the heavy lifting), stitch
(reduce). A MapReduce for video.

## 7. Under the hood, like the engineer

This is the heart of it. Three ideas stacked: why you must chunk, what the chunk
actually is, and why Google went and built a chip.

### 7a. Why chunk at all, and why the chunk boundary is not arbitrary

A video is not a flat array of independent pictures. To save space, encoders
store a few full frames (I-frames, the complete picture, like a JPEG) and then
store most frames as differences from their neighbors (P-frames and B-frames,
which say "same as the last frame but this patch moved right by 4 pixels"). A run
that starts with an I-frame and is followed by the frames that depend on it is
called a GOP, a Group of Pictures.

This dependency is the whole reason chunking works and the whole reason you
cannot chunk carelessly. If you sliced the video at an arbitrary frame, that
frame might be a P-frame that references a picture in the previous slice. Now the
slice cannot be decoded alone; it is missing the thing it points at. So the
splitter must cut only on GOP boundaries, at I-frames. Each chunk then starts
with a full picture and contains only frames that reference material inside the
same chunk. That makes every chunk independently decodable, which makes it
independently encodable, which is what lets you throw a thousand chunks at a
thousand workers with no coordination between them.

Concrete walk. Arjun's 12 minute video at, say, a keyframe every 2 seconds gives
roughly 360 GOP chunks. Chunk 0 is seconds 0 to 2 (a full I-frame of the game
menu plus the differences as the cursor moves). Chunk 143 is a mid-match
sequence. Each chunk is a self-contained little movie. Encode chunk 143 into
1080p VP9 on one chip while chunk 0 becomes 360p H.264 on another chip and chunk
359 becomes 4K AV1 on a third. None of them wait on each other.

Data structures in play:

- The video is a sequence (an ordered array) of GOP chunks. Order matters only at
  stitch time, so each chunk carries its index. Stitching is "sort by index,
  concatenate, fix timestamps," an O(n) merge, not a re-encode.
- The set of encode jobs is a DAG (directed acyclic graph): split is the root,
  each chunk-times-format is a leaf task, stitch nodes depend on all leaves of
  their stream, package depends on stitch. A scheduler walks the DAG and runs any
  task whose inputs are ready. (The DAG framing is standard for this class of
  pipeline; YouTube's exact orchestrator internals are not public, labeled as
  inference.)
- A work queue holds the leaf tasks; stateless encoder workers pull from it. A
  worker dying just means its task goes back on the queue, because the input chunk
  is immutable in storage. This is the same "queue as shock absorber plus
  idempotent retries" pattern the ledger keeps meeting.

Why this is the right structure: it decouples the cost of one video from the wall
clock. A 4 hour movie and a 4 minute clip both finish in about the time it takes
to encode a few chunks, because you just use more workers for the longer one. The
knob that turns runtime into "constant-ish" is the number of parallel workers,
not the length of the video.

### 7b. The codec ladder is a spend decision keyed on predicted demand

Not every video earns every codec. AV1 gives the smallest files at the same
quality, which saves a fortune in delivery bandwidth over a video's life. But AV1
is brutally expensive to encode. Public measurements put AV1 at roughly 18 times
the encode cost of H.264, while VP9 is only about twice H.264
([Streaming Learning Center](https://streaminglearningcenter.com/codecs/which-codecs-does-youtube-use.html)).

So YouTube does not encode everything into everything on day one. The cheap
universal rung (H.264, low resolutions) is made first so the video is watchable
almost immediately and plays on the oldest devices. Then the pipeline spends more
compute on the better codecs as the video proves it deserves them: VP9 kicks in
as views climb (reported around the low thousands of views), and AV1 is reserved
for videos heading for very large audiences, where the per-view bandwidth saving
finally outweighs the one-time encode cost
([Streaming Learning Center](https://streaminglearningcenter.com/codecs/which-codecs-does-youtube-use.html)).

This is the ledger's offline-think, online-lookup idea wearing a cost hat. Do the
expensive work (a full AV1 encode of a viral video) only when the predicted
payoff justifies it. A cat video with 40 views never earns an AV1 pass. MrBeast's
next upload earns it in the first minutes because it will be watched hundreds of
millions of times and each of those streams is cheaper in AV1.

Concrete example. If a 1080p AV1 stream is 30% smaller than the VP9 version, and
a video gets 100 million views, that 30% multiplies into petabytes of egress
saved. That saving is what pays for the 18x encode. On a 40-view video the same
18x encode buys you nothing, so you skip it.

### 7c. Why Google built a chip: Argos, the VCU

Here is the deepest and best-sourced part. General-purpose CPUs are wasteful at
video encoding. Encoding spends most of its time on a few fixed operations
(motion estimation, transforms, entropy coding) that a CPU does with general
instructions and a lot of overhead. Put those exact operations into
fixed-function silicon and you win enormously on performance per dollar and per
watt. So Google built the VCU, the Video Coding Unit, internally named Argos, and
described it in the ASPLOS 2021 paper "Warehouse-scale video acceleration:
co-design and deployment in the wild"
([paper PDF](https://gwern.net/doc/cs/hardware/2021-ranganathan.pdf)).

The hard, real numbers:

- A VCU card carries two Argos ASICs on one full-length PCIe board with a passive
  aluminum heat sink
  ([ServeTheHome](https://www.servethehome.com/google-youtube-vcu-for-warehouse-scale-video-acceleration/)).
- Each Argos chip has 10 encoder cores. Each encoder core can encode 2160p (4K)
  in real time at up to 60 FPS using three reference frames
  ([ServeTheHome](https://www.servethehome.com/google-youtube-vcu-for-warehouse-scale-video-acceleration/)).
- The first VCU generation supports VP9 and H.264, with AV1 on the roadmap
  ([DataCenterDynamics](https://www.datacenterdynamics.com/en/news/google-develops-custom-argos-video-transcoding-chips-for-youtube-processing/)).
- Google reported 20x to 33x improvements in compute efficiency (performance per
  total cost of ownership) versus their previous CPU-based (Skylake) systems
  ([9to5Google](https://9to5google.com/2021/04/22/youtube-google-custom-chip/),
  [DataCenterDynamics](https://www.datacenterdynamics.com/en/news/google-develops-custom-argos-video-transcoding-chips-for-youtube-processing/)).
- Deploying VCUs was estimated to replace on the order of 10 million Intel CPUs
  (analyst range 4 to 33 million)
  ([SemiAnalysis](https://newsletter.semianalysis.com/p/google-new-custom-silicon-replaces)).

Now connect the chip back to the chunking. The paper's whole point is co-design:
the chip is fast at one video's worth of work, but a single big upload still has
to be spread across many chips to finish quickly. So videos are sharded into
chunks and processed in parallel across many VCUs, and a full VCU machine (paper
describes systems with many VCUs, for example a 20-VCU box) replaces multiple
racks of CPU-only servers for VP9 work
([ServeTheHome](https://www.servethehome.com/google-youtube-vcu-for-warehouse-scale-video-acceleration/)).
The chunk is the unit of parallelism; the chip is the muscle per unit. You need
both. A fast chip with no chunking still takes forever on a 4 hour movie; perfect
chunking on slow CPUs still bankrupts you at 500 hours a minute.

### 7d. The scale story at three tiers

Tier 1, about 1,000 videos (a single company's internal video library). One
machine with FFmpeg, encode each video into a handful of formats, sequentially.
No chunking needed. Wall clock per video is minutes to an hour, and nobody
cares. What breaks at the next tier: the single machine cannot keep up with the
arrival rate, and one 4 hour upload blocks everything behind it.

Tier 2, about 100,000 videos a day (a mid-size platform). Now you need a fleet
and a queue. Split into GOP chunks so long videos do not head-of-line-block the
queue, fan chunks across many workers, stitch. A stuck worker no longer stalls
the platform because tasks are idempotent and re-runnable from immutable input
chunks. What breaks at the next tier: CPUs become the cost. Encoding VP9 and AV1
for everyone on general-purpose cores means renting racks upon racks, and the
electricity bill and machine count grow faster than revenue.

Tier 3, YouTube scale, 500 hours per minute and billions of views. Two moves.
First, tier the codec spend by predicted demand so you are not paying 18x AV1
cost on videos nobody watches. Second, stop using CPUs for the hot kernel at all:
move encoding onto fixed-function silicon (the VCU) for the 20 to 33x
efficiency jump, and keep the chunk-parallel fabric on top so any one video still
finishes in minutes. Delivery scales separately, on the CDN and edge caches (the
Open Connect-style story from an earlier teardown), because encoding cost scales
with uploads while delivery cost scales with views, and those two firehoses are
different sizes.

The recurring shape: expensive, embarrassingly-parallel work pushed off the
critical path and onto specialized hardware, with a cheap universal output
shipped first so the user is never blocked waiting for the expensive one.

## 8. The retention and habit mechanic

Transcoding is not a button Arjun taps for fun, so the habit loop is indirect but
real, and it runs on both sides of the market.

Creator side (supply retention). The pipeline is why Arjun's video is live and
watchable within a minute instead of an hour. Fast processing plus "low res
available first" removes the dead time between finishing an edit and getting
views. Views are the reward that makes him upload again next week. A pipeline that
left videos "processing" for hours would quietly kill creator retention, and
creators are the supply the whole platform lives on. The loop: upload, live fast,
get views, upload again.

Viewer side (watch-time retention). The reason a viewer stays instead of bouncing
is that the video starts instantly and never buffers, on their exact device and
their exact network. That is the ladder doing its job: the phone on 3G gets a
small VP9 or AV1 stream that fits the pipe, the TV gets 4K. Buffering is one of
the strongest predictors of abandonment, so every rung the pipeline produces is a
few more viewers who do not close the tab. The metric it moves is watch time,
YouTube's north star.

Business side (revenue and margin). This is the unusual one. The clearest metric
the VCU moves is cost, which is margin, which is revenue kept. Replacing on the
order of 10 million CPUs and getting 20 to 33x better efficiency is a direct,
enormous saving that also unlocks shipping better codecs (AV1) to more videos,
which lowers delivery cost too. Cheaper delivery in bandwidth-poor markets means
more people can watch more, which loops back to watch time. The habit here is
structural: make it so cheap and smooth to watch that watching stays effortless
everywhere.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader code plus an embeddable
runtime, and cares about scalability and performance. YouTube's transcoding
pipeline hands you three concrete, specific moves.

1. Make the compile chunk-parallel with independent units, like GOP chunks. Do
   not compile a big effect graph as one monolithic job. Split it into
   independently-compilable subgraphs (a node island that only depends on its own
   inputs, the shader equivalent of a GOP that only references frames inside
   itself), fan them across workers, and stitch the results. Pick your split
   points at real dependency boundaries the way YouTube splits at I-frames, never
   mid-dependency, so a chunk never waits on another chunk. This turns a slow
   whole-graph compile into a fast parallel one and makes retries cheap: a failed
   subgraph re-runs from its immutable inputs without redoing the whole scene.

2. Ship the cheap universal target first, then let the expensive ones catch up.
   YouTube makes low-res H.264 watchable in a minute and finishes 4K AV1 later.
   Rare.lab should compile a guaranteed-to-run baseline variant first (a simple
   fallback shader, or WebGL over WebGPU, or a low-quality-but-correct path) so
   the user sees their effect running almost immediately, then stream in the
   optimized and higher-fidelity variants as they finish. Never block the first
   frame on the most expensive build.

3. Tier your most expensive compile targets by predicted demand, like AV1. AV1
   costs 18x but only earns its keep on videos with huge audiences, so YouTube
   spends it selectively. Rare.lab likely has a matrix of target variants
   (platforms, quality tiers, GPU feature levels, precompiled permutations). Do
   not eagerly compile the full matrix for every effect. Compile the expensive,
   heavily-optimized, or rarely-used permutations lazily, driven by which targets
   an effect is actually shipped to or how often it is embedded. And when a hot
   kernel dominates your runtime, remember the deepest YouTube lesson: at enough
   scale the win is not a better general-purpose loop, it is moving the fixed
   inner kernel onto fixed-function hardware. For Rare.lab that means pushing the
   repeated per-pixel work down to the GPU's native pipeline aggressively rather
   than leaving it in general-purpose compute, because performance per watt on
   the hot path is where scale is won or lost.

The through-line: do the heavy work once, off the critical path, split into
independent parallel units, spend the expensive quality only where it pays, and
give the user a cheap correct result instantly while the good one finishes.

---

## Sources

- Ranganathan et al., "Warehouse-scale video acceleration: co-design and
  deployment in the wild," ASPLOS 2021 (the Argos VCU paper):
  https://gwern.net/doc/cs/hardware/2021-ranganathan.pdf
- ACM listing for the paper:
  https://dl.acm.org/doi/abs/10.1145/3445814.3446723
- ServeTheHome, "Google YouTube VCU for Warehouse-scale Video Acceleration"
  (two Argos ASICs per card, 10 encoder cores per chip, 2160p60 with three
  reference frames, 20-VCU machine replacing racks):
  https://www.servethehome.com/google-youtube-vcu-for-warehouse-scale-video-acceleration/
- DataCenterDynamics, "Google develops custom 'Argos' video-transcoding chips
  for YouTube" (VP9 and H.264, AV1 roadmap, 20-33x efficiency):
  https://www.datacenterdynamics.com/en/news/google-develops-custom-argos-video-transcoding-chips-for-youtube-processing/
- 9to5Google, "Google-developed 'Argos' VCU chip helps YouTube process videos
  much more efficiently" (20-33x compute efficiency vs prior CPU systems):
  https://9to5google.com/2021/04/22/youtube-google-custom-chip/
- SemiAnalysis, "Google New Custom Silicon Replaces 10 Million Intel CPUs":
  https://newsletter.semianalysis.com/p/google-new-custom-silicon-replaces
- Streaming Learning Center, "Which Codecs Does YouTube Use?" (H.264 vs VP9 vs
  AV1 by view count; AV1 ~18x, VP9 ~2x the H.264 encode cost):
  https://streaminglearningcenter.com/codecs/which-codecs-does-youtube-use.html
