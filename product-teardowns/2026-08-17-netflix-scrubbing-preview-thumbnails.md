# Netflix scrubbing preview thumbnails (the little filmstrip above the seek bar)

Date: 2026-08-17
Product: Netflix
Feature: Trickplay preview images (the thumbnail that pops up when you drag the timeline)

---

## 1. The user

Meet Sarah. It is 11pm on a Tuesday. She is halfway through Stranger Things
Season 4, Episode 1, on her living room TV. She dozed off around the point
where Eleven is at Rink-O-Mania, woke up, and the show is now somewhere in the
Hawkins Lab flashback. She has no idea how much she missed.

She grabs the remote, presses left, and the seek bar appears at the bottom. She
does not want to watch the whole thing again. She wants to land on the exact
moment she fell asleep, the roller rink, and pick up from there. So she starts
dragging the scrubber backward. As her thumb moves, a small picture appears just
above the bar and changes as she drags: a classroom, a hallway, then the bright
neon of the rink. She stops the second she sees neon, lets go, and the show
resumes right there.

That small changing picture is the feature. Sarah used it for maybe four
seconds and never thought about it again. That is the sign of a feature done
right.

The same person hits this feature in three other moods:
- Rewinding 8 seconds because a character mumbled a line in Peaky Blinders.
- Skipping forward past a slow stretch in a documentary.
- Coming back the next night and confirming, before pressing play, that Continue
  Watching really did remember the right spot.

---

## 2. The real problem

Dragging a video timeline blind is miserable. Without a preview, the scrubber is
a guess. You drag, let go, wait for the video to re-buffer and start decoding
from that point, realize you overshot, drag again, wait again. Three or four
round trips of "seek, buffer, nope" and you give up and just watch from wherever
you landed.

The pain has two parts. First, you cannot see where you are going, so you
overshoot. Second, every real seek is expensive: the player has to jump to a new
point in the video, find the nearest keyframe, download that chunk, and start
decoding. Doing that four times to find one scene is slow and it burns data.

Described like a friend would: "I know the good bit is around the 30 minute mark,
but every time I try to find it I either land too early or too late, and each
try takes forever to load." The preview thumbnail kills both problems at once.
You aim before you commit, and you only pay for the one real seek that matters.

---

## 3. The feature in one sentence

As you drag the seek bar, Netflix shows a tiny still frame of the exact moment
under your finger, pulled from a pack of pre-made images, so you can eyeball the
right spot before the real video ever moves.

---

## 4. Jobs to be done

What Sarah is really hiring this feature to do:
- "Let me find a specific scene without watching my way to it." (the rink)
- "Let me rewind a few seconds to catch a line I missed, precisely." (the mumble)
- "Let me skip the boring part and land on the good part." (the documentary)
- "Let me confirm where I am before I commit to a slow, data-heavy seek."
- "Do all of that instantly, even on my parents' weak DSL, without spinning."

Notice the common thread: every job is about aiming cheaply. The real video seek
is the expensive commit. The preview is the cheap aim.

---

## 5. How it works for the user

You bring up the player controls. You grab the scrubber (with a finger, a mouse,
or by holding the remote's arrow). A small box appears floating just above the
bar. Inside it is a thumbnail of the video at whatever timestamp your finger is
over. As you move, the thumbnail updates almost instantly, frame after frame,
like flipping through a filmstrip. On many Netflix clients a timestamp label
sits under the image ("0:31:14"). When the picture matches the scene you want,
you release. Only then does the actual video jump and start playing from there.

The key felt quality: the preview reacts with zero lag, even when the video
itself is streaming at 4K and would take a second or two to re-buffer on a real
seek. That gap in speed is the whole clue to how it works.

---

## 6. The actual flow, step by step

1. Sarah taps the screen (or presses a remote key). The transport controls fade
   in, including the seek bar.
2. She presses and holds the scrubber handle. The client enters "scrubbing"
   mode. Crucially, the video decoder does not follow her yet. It keeps holding
   the last frame or keeps playing muted underneath.
3. She drags. For every position of the scrubber, the client computes a
   timestamp t (for example, "she is 43% across a 51 minute episode, so t is
   about 21 minutes 55 seconds").
4. The client converts t into an image index and looks up the matching preview
   picture from a small pack of images it already downloaded. It crops that one
   picture out and draws it in the floating box. This is local. No network call,
   no video decode.
5. As she keeps dragging, steps 3 and 4 repeat many times a second. The box
   flips through neighboring thumbnails. It feels like scrubbing a real filmstrip.
6. She sees the neon rink and releases the handle. Now, and only now, the client
   issues the real seek: it maps t to the nearest keyframe in the actual video
   stream, requests that segment from the CDN, and starts decoding. This is the
   one expensive operation, and it happens exactly once.
7. Playback resumes at the rink. Total real seeks: one. Preview lookups: dozens,
   all free.

---

## 7. Under the hood, like the engineer

This is the heart of it. The single most important idea: **the preview you drag
through is not the video.** It is a completely separate, pre-made pack of tiny
still images. The player never decodes real video to show you a preview. If it
did, scrubbing would be as slow as a real seek, and the feature would be
pointless.

So there are two media objects for one title, built and served on different
paths:

- The video stream. Heavy. Adaptive bitrate ladder, chunked, decoded live. This
  is the subject of the earlier Netflix teardowns in this ledger (per-shot
  encoding, ABR, Open Connect).
- The trickplay image pack. Featherweight. A grid of small JPEGs, one every few
  seconds of the title, made once, offline, and cached.

### The data structure: an array indexed by time

The preview pack is, at its core, an **array of images indexed by time**. Frame
0 is at t=0, frame 1 is at t=10s, frame 2 at t=20s, and so on for a fixed
interval. To find the picture for any timestamp, the client does one division:

```
index = floor(t / interval)
```

That is O(1). It does not scan, it does not search, it does not sort. For t =
21 minutes 55 seconds (1315 seconds) at a 10 second interval, index = floor(1315
/ 10) = 131. Go get picture number 131. Done.

An array with a fixed stride is the right structure precisely because the access
pattern is "random access by position, constant time, no gaps." You never need a
tree or a hash map here. The index is the position, and the position is math.

### Packing: why one big sheet beats a thousand little files

Naively you could store each thumbnail as its own JPEG file. A 51 minute episode
at a 10 second interval is 306 pictures. Now the client has to make up to 306
separate HTTP requests to scrub one episode, one per thumbnail. Each request has
its own round-trip latency and overhead. Scrubbing would stutter, and the CDN
would groan under billions of tiny objects.

So the images are **packed together**. Two common shapes:

- **Sprite sheet (tile grid).** Many thumbnails laid out in a single big JPEG,
  say a 10 by 10 grid of 100 small frames in one image. To show frame 131, the
  client works out which sheet holds it and which cell: column = index mod 10,
  row = (index div 10) mod 10. Then it crops the rectangle at (column times
  width, row times height) and paints it. One download covers 100 previews. This
  is the same idea as a texture atlas in games.
- **BIF (Base Index Frames).** The format Netflix has long used on devices,
  originally from Roku and now widely supported. A BIF file is literally a packed
  array with an index table stapled to the front. Layout, well grounded from the
  public Roku spec:
  - A magic header identifying the file as BIF, plus a version.
  - A count of how many images are inside.
  - The framewise separation, meaning the interval in milliseconds between
    images (for example 10000 for one image every 10 seconds).
  - An **index table**: one entry per image, each entry storing that image's
    timestamp and its byte offset inside the file, ending with a sentinel entry.
  - Then the raw JPEG blobs, back to back.

  Reading BIF is therefore: jump to entry number 131 in the index table, read its
  byte offset, seek there, read the JPEG. The index table is an offset array,
  which is exactly the on-disk version of the O(1) time-to-image map. No decode
  of anything except the one small JPEG you want.

There is also a standards-track version of this for adaptive streaming, worth
naming because it is the modern path:
- **HLS I-frame playlists** (the `EXT-X-I-FRAME-STREAM-INF` tag) let a player
  reconstruct previews from the video's own keyframes. It works, but it means
  decoding real video I-frames, which is heavier than reading a ready JPEG.
- **Image-based trickplay** (an Image Media Playlist of JPEG tiles) was pushed by
  Roku, Disney, and WarnerMedia precisely because ready-made JPEGs are cheaper to
  show than decoded I-frames.
- **DASH thumbnail tiles** plus a WebVTT-style map that says "this rectangle in
  this sheet covers seconds 1300 to 1310" is the DASH equivalent.

All of them are the same pattern under different wrappers: pre-made small images,
packed, with a time-to-rectangle index.

### Where the work happens: offline generation, online lookup

Making the pack is an offline job that runs once per title in Netflix's encoding
pipeline. The heavy lifting is the same shape as the rest of Netflix's media
processing:

- Netflix rebuilt its video pipeline on **Cosmos**, a platform of media
  microservices. A Cosmos service has three layers: an API layer (**Optimus**), a
  workflow layer (**Plato**) that orchestrates the steps, and a serverless compute
  layer (**Stratum**) that runs the actual media functions, all talking over a
  priority message bus called **Timestone**. The video encoder itself is the
  **Video Encoding Service (VES)**. (These names are public from the Netflix tech
  blog.)
- Generating trickplay images is a natural fit for this shape, and this next part
  is **labeled inference**, since Netflix has not published a dedicated
  trickplay-internals post: decode the high quality master, sample a frame every
  N seconds, downscale each to a small size (on the order of 320 by 180 or
  smaller), JPEG-encode it, and pack the results into sheets or a BIF. Like video
  encoding, this fans out: split the title by chunks or shots, generate images for
  each chunk in parallel across Stratum functions, then stitch the parts into the
  final pack. Embarrassingly parallel, because chunk 5's thumbnails do not depend
  on chunk 4's.
- The finished packs are stored and pushed out to **Open Connect**, Netflix's own
  CDN, ahead of demand (proactive fill, covered in the Open Connect teardown). By
  the time you press pause, the images are already sitting on a server inside or
  near your ISP.

The live path is then almost nothing. When Sarah opens the player, the client
fetches the small trickplay pack (often by byte ranges, grabbing only the sheets
near where she is scrubbing). From then on, every drag is a local crop. The
expensive thinking was done once, offline, months before Sarah pressed pause.
This is the same **offline-think, online-lookup** spine that runs through
Discover Weekly, YouTube recommendations, and Skip Intro in this ledger.

### Sparse truth, smooth fiction

The images exist only every N seconds. But Sarah's finger moves continuously. So
the client shows the nearest available thumbnail and snaps between them as she
drags. The truth is sparse (one frame per interval), the feel is smooth (the box
updates on every pixel of drag by reusing the nearest frame). Newer players
densify the pack (an image every 1 to 2 seconds) or interpolate, which trades
more storage for a smoother filmstrip. This is the same sparse-truth,
smooth-fiction trick as the moving rider in the Swiggy and Uber teardowns, where
GPS truly updates every few seconds but the marker glides at 60fps.

### The scale story, three tiers

**1,000 titles.** Trivial. A 2 hour film at a 10 second interval is 720
thumbnails. At roughly 10 to 15 KB per small JPEG, the whole pack is a few
megabytes. Generate one pack per title, drop them on any CDN, done. No cleverness
required.

**100,000 titles.** Two costs grow. First, generation compute: you must decode
every master once and produce images, which is real CPU, so you fan it out
(chunked parallel generation on Cosmos/Stratum) and it stays a batch job, not a
bottleneck. Second, storage and delivery: you may need multiple sizes (phone,
tablet, TV) and multiple aspect handlings, multiplying the packs. The fix is the
CDN. Packs are static files, infinitely cacheable, and served from Open Connect
with essentially zero origin traffic. What breaks if you skip this: serving
100,000 titles of images from origin on demand would hammer your storage tier;
pushing them to the edge makes the origin load nearly flat.

**10 million plus, global, concurrent.** This is where the design earns its keep.
On a big night, well over 100 million people are watching, and a large fraction
of them scrub. If every drag hit a server, or worse, triggered a video decode,
you would need to serve billions of preview lookups per hour. That is impossible
and pointless. Instead, **every drag costs zero server work.** The only network
event is the one-time fetch of the small pack when the player opens, and that is
a cached static file at the edge. The billions of drags all happen locally in the
client as array lookups and crops.

Name what breaks at the next tier and what survives it:
- If you served the real video frame per seek: you would need a live decode per
  drag. At 100M users that melts. Survived by precomputing images.
- If you stored one JPEG per thumbnail as separate objects: billions of tiny
  files and one HTTP request per preview. The request overhead alone would stall
  scrubbing and bloat the CDN object count. Survived by packing into sheets or
  BIF, so one fetch covers minutes of runtime.
- If you generated packs on demand the first time someone scrubbed a title: the
  first viewer of every title pays a decode tax and waits. Survived by generating
  once, offline, ahead of release, and prefilling the CDN.

The one dial that makes all of this scale is the same dial as search ranking's
candidate-set size, just wearing a different hat: **move the cost from
per-interaction to per-title.** There are billions of scrubs but only millions of
titles. Pay once per title, offline, and the per-scrub cost falls to a local
crop. That is the entire game.

---

## 8. The retention and habit mechanic

This feature does not build a loop by itself. It is a friction remover, and that
is exactly its job. It protects the session you are already in.

The mechanic: scrubbing without previews is frustrating enough that people
abandon. You overshoot, wait through re-buffers, and some fraction of viewers
just close the app in irritation, especially when hunting for one specific moment.
The preview turns a five-try, thirty-second ordeal into a two-second glance. Fewer
rage-seeks, fewer abandoned sessions, more content actually finished.

Which metric it moves: **retention and completion**, indirectly, by cutting
abandonment on a very common interaction. A session that ends in frustration at
the seek bar is a session that did not finish the episode, and unfinished
episodes are the enemy of the binge. Netflix's whole retention model leans on
binge momentum (Continue Watching, autoplay next episode, Skip Intro). Trickplay
previews are the quiet enabler underneath all of them: they make "jump back to
where I was" and "skip to the good part" instant, so the momentum never breaks.

Real observed example: someone rewinding to catch a mumbled line in Peaky
Blinders finds it on the first try, in two seconds, and stays in the show. Without
the preview they overshoot twice, sit through two re-buffers, lose the thread, and
maybe drift to their phone. The preview is the difference between "quick rewind,
back to the show" and "lost the moment, closed the app."

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor with a timeline and an
embeddable runtime. Anything with a timeline invites scrubbing, and scrubbing a
live shader is the expensive version of Sarah's problem: if the user drags the
playhead across an animated effect and you re-run the full shader graph on the
GPU for every scrub position, you burn frames, stutter, and heat the device.

Do exactly what Netflix does. **Bake a preview atlas offline (or during idle) and
make scrubbing an O(1) texture-atlas lookup, not a re-render.**

Concretely:
- When a user finishes editing an effect, or on idle, render a low-resolution
  filmstrip of the effect over its timeline: one preview frame every N
  milliseconds, at a fraction of full resolution.
- Pack those preview frames into a **single texture atlas**, not N separate
  textures. This is the GPU version of "sprite sheet beats a thousand files." One
  atlas means one texture upload and one bind, so scrubbing switches previews by
  changing UV coordinates into an already-resident texture, not by uploading a new
  image and not by re-running the graph. Fewer binds, fewer draw calls, no shader
  recompute.
- Index by time with the same trick: `frameIndex = floor(t / interval)`, then map
  frameIndex to a cell rectangle in the atlas. Constant time, no graph traversal.
- Only when the user releases the playhead do you run the real, full-resolution
  shader at that single timestamp. Cheap aim, one expensive commit, exactly like
  Netflix's one real seek.

The scalability payoff is identical. When Rare.lab effects ship inside a customer
app viewed by millions, the runtime must never pay per-scrub GPU cost across all
those users. Bake the preview atlas once per effect at author time, ship it as a
static asset, and every end-user scrub becomes a UV lookup into a cached texture.
Cost moves from per-interaction (millions of scrubs) to per-effect (a one-time
bake), which is the same move that lets Netflix serve billions of drags for free.
For a performance-first product, that is the whole point.

---

## Sources

- Netflix Technology Blog, "Rebuilding Netflix Video Processing Pipeline with
  Microservices" (Cosmos platform, Optimus/Plato/Stratum layers, Timestone bus):
  https://netflixtechblog.com/rebuilding-netflix-video-processing-pipeline-with-microservices-4e5e6310e359
- Netflix Technology Blog, "The Making of VES: the Cosmos Microservice for Netflix
  Video Encoding":
  https://netflixtechblog.com/the-making-of-ves-the-cosmos-microservice-for-netflix-video-encoding-946b9b3cd300
- Roku Developer docs, BIF (Base Index Frames) file format for trick mode:
  https://developer.roku.com/docs/developer-program/media-playback/trick-mode/bif-file-creation.md
- AWS Elemental MediaPackage, "Working with trick-play" (I-frame playlists vs
  image-based trick play vs DASH thumbnail tiles; who pushed the image spec):
  https://docs.aws.amazon.com/mediapackage/latest/ug/trick-play.html
- Apple HLS authoring, I-frame playlists (`EXT-X-I-FRAME-STREAM-INF`):
  https://developer.apple.com/documentation/http-live-streaming
- Unified Streaming, "Adding trick play to a DASH or HLS stream":
  https://docs.unified-streaming.com/documentation/package/trickplay.html
- Bitmovin, "Thumbnail Generation Support for VOD Encoding" (sprite sheets +
  WebVTT mapping):
  https://developer.bitmovin.com/encoding/docs/thumbnail-generation-support-for-vod-encoding
