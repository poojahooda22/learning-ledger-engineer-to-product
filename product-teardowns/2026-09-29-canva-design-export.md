# Canva: Design Export (turning a live design into a shippable PNG, PDF, or MP4)

Date: 2026-09-29
Product: Canva
Feature: Design export (the Download button, and the render pipeline that produces the file)

A note on scope. This is not the live editor canvas that redraws while you drag a
photo. That is the interactive renderer, closer to the Figma canvas engine teardown
(06-27). This is the other half of Canva: the moment you press Download and Canva must
turn your editable design into a single frozen file that looks identical everywhere,
opens on any phone, and prints at 300 DPI. It is a compile step, not an edit step. That
is exactly why it is the best Rare.lab fit in this ledger so far.

---

## 1. The user

Meet Aarav. He runs a small home bakery in Pune called Aarav Bakes. It is 9:40pm. He
just finished a batch of red velvet jars and wants to post them before he sleeps so the
morning crowd sees them. He opens Canva on his three year old Android phone, picks an
Instagram post template, drops in two photos he shot on the kitchen counter, changes the
text to "Fresh red velvet jars, order by 11am," and adds a little confetti sticker that
loops. He is happy with it. Now he taps the one button that actually matters to him:
Download.

He is not a designer. He does not know what a codec is. He knows one thing. He wants a
clean image on his phone in a few seconds that he can post to Instagram and forward on
WhatsApp to his regulars. If it takes 40 seconds, or if the text looks slightly
different from what he saw on screen, or if the confetti is a still frame instead of a
loop, he loses trust in the tool.

## 2. The real problem

Told like a friend: what Aarav sees on his screen is not a file. It is a live document.
It is a bag of instructions. "Put this photo here at this size. Put this text on top in
this font at this color. Put a looping confetti animation in the corner." That bag of
instructions only means something inside Canva, running on his phone, with Canva's fonts
and Canva's photo library loaded.

Instagram cannot read a bag of Canva instructions. WhatsApp cannot. A print shop cannot.
They all want a plain file. A PNG. A PDF. An MP4. Something self contained, with the
pixels already drawn, the fonts already baked in, and nothing left to look up.

So export is a translation job. Take the live editable design and freeze it into a flat
file that carries everything with it and needs nothing from Canva to open. And it has to
look pixel for pixel like what Aarav saw, on a device that is not his, at whatever
resolution he asked for. That last part is the trap. The design he sees is rendered by
his phone's browser. The file must be rendered by Canva's servers. Two different
machines must agree on the exact same picture, down to the last shadow.

## 3. The feature in one sentence

Design export takes your live, editable design and renders it, on Canva's servers, into
a single self contained file (PNG, JPG, PDF, GIF, PPTX, or MP4) that looks identical to
what you saw and opens anywhere without Canva.

## 4. Jobs to be done

What is Aarav really hiring export to do:

- "Give me a real file I can post and forward." The editable design is useless outside
  Canva. He hires export to hand him something Instagram and WhatsApp accept.
- "Make it look exactly like what I made." He tuned the layout. He does not want the
  server to move the text or drop the confetti.
- "Do it fast enough that I do not wander off." A still image should feel instant. A one
  minute video he will tolerate waiting for, if Canva tells him it is working.
- "Get the quality right for where it is going." A crisp image for Instagram. A high
  resolution, print ready PDF at 300 DPI for the sticker labels on his jars. Same design,
  two very different files.
- "Do not surprise me at the checkout." If his design used a premium sticker he has not
  paid for, tell him before he has posted a watermarked mess, not after.

## 5. How it works for the user

Aarav taps Download at the top right. A little menu opens. It asks two things: the file
type (PNG, JPG, PDF Standard, PDF Print, MP4 Video, GIF) and, for images, the size or
quality. He picks MP4 Video because of the looping confetti. He taps Download again.

A progress state appears. It does not freeze. It says the export is being prepared, and
often shows a moving bar. A few seconds later, for a still image, or a bit longer for his
one minute video, his phone receives the finished file and the share sheet pops up. He
sends it straight to his Aarav Bakes Instagram.

If his design had a premium element he had not bought, Canva would have stopped him here
with a clear message: buy this element or upgrade, otherwise the export cannot complete.
No silent watermark surprise.

## 6. The actual flow, step by step

1. Aarav taps Download. The editor gathers the current design state. This is a
   structured document, effectively JSON: a list of pages, and inside each page a list of
   elements with their type, position, size, rotation, color, font, opacity, z order, and
   for the confetti a reference to a Lottie animation.
2. He picks MP4 and confirms. The client sends an export request to Canva's servers. The
   request names the design, the version, the format (MP4), and the settings (resolution,
   frame rate).
3. The server does not render on the spot and make him hold the line. It creates an
   export job. The job gets an id and a status of in_progress. This is the documented
   Canva Connect behavior: a newly created export job is in_progress and will eventually
   become success or failed (Canva Connect API, Create design export job).
4. The job lands in a queue. A render worker picks it up. For a video, the worker walks
   every frame: it resolves each element, draws the photos and text and the confetti
   frame for that moment, and hands the finished frame to a video encoder that packs the
   frames into an H.264 MP4.
5. Meanwhile Aarav's phone polls the job (or listens on a live channel). You poll the Get
   design export job endpoint until status flips from in_progress to success or failed
   (Canva Connect API, Get design export job).
6. On success the server returns download URLs. There is a download URL for each page,
   sorted by page order, and these URLs are short lived (the Connect API states they
   expire after 24 hours). Aarav's single page video comes back as one URL.
7. His phone downloads the file. Share sheet. Posted. If the job had failed (say, an
   unpurchased premium element), the response would carry the error reason instead, and
   Canva would show him the upgrade prompt.

Two real details worth pinning. First, multi page designs in a format that cannot hold
multiple pages (a PNG cannot) come back as a ZIP of one file per page, or as one URL per
page. Second, the file is rendered server side on purpose, so the output is identical no
matter whether Aarav exported from an old Android, an iPhone, or a laptop.

## 7. Under the hood, like the engineer

This is the heart. Canva does not publish a full architecture diagram of its export
service, so I will be strict about labeling. Facts are things Canva or its partners have
stated publicly. Inference is the well grounded "this is how this class of problem is
always solved" version, and I will say so each time.

### 7a. The design is a tree, not a picture

Fact, from the data model that any such editor must use and that Canva's own export
behavior implies: the design is a scene graph. A tree. At the top is the design. Under it
are pages. Under each page is an ordered list of elements. A group is an element that
holds child elements, so the tree nests.

Walk Aarav's post concretely. Page 1 holds, in z order from back to front:

- Element 0: a background rectangle, soft cream, full bleed.
- Element 1: photo A (red velvet jar close up), a frame at x=40 y=120, width 640, some
  corner rounding.
- Element 2: photo B (the counter shot), smaller, top right.
- Element 3: a text element, "Fresh red velvet jars, order by 11am," font Canva Sans,
  weight bold, size 48, color dark brown.
- Element 4: a Lottie confetti sticker, looping, pinned bottom left.

The z order is the whole reason a tree with an ordered child list is the right structure.
Rendering is the painter's algorithm: draw back to front, each element painted over what
is already there. The cream rectangle first, then the photos, then the text on top, then
the confetti on top of that. An array in the correct order gives you z order for free. A
plain hash map of elements would lose the order and you would have to sort every frame.
So: ordered tree, painted back to front. That is the core structure.

### 7b. The two halves of export: resolve, then rasterize and encode

Search teardowns in this ledger split into matching then ranking. Export splits into a
different two halves, and keeping them separate is just as important.

Half one, resolve. Turn the tree of references into a self contained render plan. Every
element in the design is full of pointers to things that are not in the design file.
Photo A is a reference to an image in Aarav's uploads. The text uses Canva Sans, a font
stored on Canva's servers. The confetti is a reference to a Lottie file (a JSON
description of a vector animation). Before you can draw a single pixel you must fetch all
of it: pull the real image bytes, load the exact font file and the exact glyphs, load the
Lottie JSON. This is also where the licensing check lives: a premium element that has not
been purchased fails here, before any rendering, which is why Canva can warn Aarav up
front (fact: the Connect API lists unpurchased premium elements as a documented export
failure reason).

Half two, rasterize and encode. Now that everything is local and concrete, draw it, then
pack it into the target container. Rasterize means turn shapes and text and vectors into
a grid of pixels. Encode means wrap those pixels in the file format: PNG compression for
an image, a page tree for a PDF, an H.264 stream for a video.

Why split them. The resolve step is about correctness and completeness (did we fetch
everything, is everything licensed, is the layout exactly as authored). The rasterize and
encode step is about raw compute (draw millions of pixels, compress, encode). They fail
for different reasons and they scale for different reasons. A missing font is a resolve
problem. A slow video encode is a rasterize and encode problem. Mixing them makes both
harder to reason about, the same way mixing matching and ranking makes search harder.

### 7c. Rasterization, and why the engine choice is the performance

Rasterizing text and vectors is not free. For Aarav's confetti, the Lottie file describes
the animation as vector paths that move over time. To make a video, the server must draw
each frame: evaluate the animation at that moment, turn the vector paths into pixels
(scanline coverage, anti aliasing at the edges), composite over the frame, repeat.

Here is the strongest public fact in this whole teardown, and it proves the point that
the rasterizer choice is not a detail, it is the performance. Canva rendered Lottie on
iOS using rLottie, a C++ Lottie renderer, across its video export and playback pipelines.
When Canva let users import their own Lottie animations into designs, the load on the
render pipeline jumped. Canva benchmarked rLottie against ThorVG (an open source C++
vector graphics engine that renders SVG and Lottie) in its iOS video export pipeline, on
a one minute video design containing only Lottie elements. ThorVG rendered about 80%
faster and used about 70% less peak memory, and it also threw fewer Lottie load and
render errors because it supported more Lottie variants. Canva switched to ThorVG.
(Fact, LottieFiles case study on Canva and ThorVG.)

Read that number again. Same design, same frames, same output. Just a different
rasterizer underneath. 80% faster and 70% less memory. That is what "the engine choice is
the performance" means in practice. When your product's whole job is turning descriptions
into pixels, the thing that turns descriptions into pixels is the ballgame.

### 7d. The job queue, and why export is async

Inference (this is the standard shape for any heavy render workload, and Canva's own
async job API is the visible tip of it): export does not run inside the web request that
asked for it. A still PNG might, but a one minute MP4 cannot. Encoding a minute of video
can take many seconds of pure compute. If that ran inside the request, the connection
would hang, a load balancer would time it out, and one slow export would tie up a web
server that should be serving thousands of other people.

So export is a queue. The visible proof is the documented job lifecycle: create job
returns in_progress, you poll until success or failed. Behind that API is almost
certainly a message queue of export jobs and a fleet of render workers pulling from it.
The web tier's only job is to accept the request, enqueue it, and hand back a job id
instantly. The heavy work happens on a worker that nobody is waiting on directly. Aarav's
phone polls for the result. This is the same offline think, online lookup spine as the
rest of this ledger, applied to rendering: the expensive work is decoupled from the
request that triggered it.

### 7e. The scale story at three tiers

What grows here is not a catalog you search. It is the number and weight of render jobs.
Canva users create about 38.5 million designs per day, and a huge fraction of those get
exported, some as heavy video. (Fact, Canva usage statistics 2024 and 2025.)

Tier one, about 1,000 exports a day (a small tool, early days). Render inside the web
request. When Aarav taps Download, the same server that served the page draws the PNG and
streams it back. No queue, no workers, no job polling. It is simple and it is correct, and
building a render fleet at this tier would be pure over engineering. Ship it and go home.
Nothing breaks.

Tier two, about 100,000 exports a day, and now with video. The in request model breaks the
moment videos appear, because a one minute MP4 encode blocks a web server for many
seconds and a handful of them starve everyone else. The fix is the async job model you can
see in Canva's API today: accept the request, drop a job on a queue, return a job id, let
a separate worker fleet render, let the client poll. Add object storage (the finished file
lands in something like S3) and hand back a short lived download URL (the 24 hour expiry
Canva documents). This tier is where the architecture earns its keep. The user experience
becomes "queued, then a progress bar, then a download," which is exactly what Aarav sees.

Tier three, tens of millions of exports a day across images and video and PDF, worldwide,
with a sharp morning peak (inference for the fleet shape, fact for the scale). Two new
walls appear.

First wall: one render fleet is wrong, because a PNG worker and a video worker want
completely different machines. A PNG is a light, fast, CPU job. A one minute MP4 is a
heavy job that wants lots of memory and benefits from a GPU. The survival move is
specialized fleets. Tag each job by type and tier, route light raster jobs to lots of
cheap lightweight workers and heavy video jobs to fewer beefy GPU capable workers, and
autoscale each fleet on its own queue depth. When the morning content rush hits, you do
not want video jobs and quick image jobs fighting for the same box.

Second wall: the peak. A tool with a worldwide creative audience has a predictable daily
surge (people make and post in their mornings and lunch breaks). Autoscaling reacts after
the queue is already backing up, which means the first wave waits. The survival move is to
pre warm: bring capacity up before the known peak rather than after, so the queue never
gets deep enough for Aarav to notice. This is the same discipline as pre computing an
index so the hot path stays a lookup, just applied to compute capacity instead of data.

A third lever that helps at every tier: dedupe by content. If the exact same design
version is exported to the exact same format and settings, the output is byte for byte
identical. Hash the tuple (design version, format, settings) and cache the result. A
template that a thousand people export unchanged should be rendered once, not a thousand
times. (Inference, but it is the obvious and standard optimization, and it matters most
exactly when a template goes viral.)

What never changes across all three tiers: the design is still a tree, you still resolve
then rasterize then encode, and the output is still a self contained file. Only the
plumbing around the render grows. The core compile step is the same at 1,000 and at tens
of millions.

## 8. The retention and habit mechanic

Export is where the value of everything else gets collected. Aarav did not open Canva to
admire a design inside Canva. He opened it to end up with a post on Instagram and a file
in WhatsApp. The Download button is the moment the tool pays off. Every earlier feature
(templates, drag and drop, the confetti) is setup. Export is the payoff.

The loop is simple and strong. Need a graphic, open Canva, make it in a few minutes,
download a clean file, post it, get the likes and the orders. Tomorrow he needs another
one, so he opens Canva again, because last time it worked and the file came out looking
right. The retention is built on export never betraying him. A wrong font, a dropped
animation, a blurry image, a watermark he did not expect: any one of those makes the next
graphic something he does somewhere else. So the entire quiet machinery of resolve and
rasterize and encode exists so that the file always looks exactly like what he made. It
is invisible craft retention, the same shape as Netflix picture quality (09-11) and
Spotify loudness (08-28): nobody thanks the rasterizer, they just keep coming back because
"Canva always gives me a clean file."

Which metric does it move. Primarily activation and retention. The single biggest
activation moment in a design tool is a new user's first successful export, because that
is the moment they got a real thing out of the tool. And the format menu is also where
revenue lives: high resolution downloads, transparent PNGs, and some export options are
paid, and the premium element gate at export time is a direct, well timed upgrade prompt
(you already made the thing, now pay to take it clean). Canva's roughly 4 billion dollar
ARR and 31 million plus paid users sit partly on that gate. (Fact, Canva statistics
2025.) The real observed mechanic: the export dialog is the calmest, highest intent place
to ask a happy user to upgrade, because they are one tap from the value they came for.

## 9. The lesson for Rare.lab

Rare.lab is a node based editor that compiles to shippable code, plus an embeddable
runtime. Canva export is that exact same shape: an editable authored thing gets compiled
into a frozen, self contained, portable artifact. This is the closest structural twin in
the ledger. Take these carry overs, biased toward scalability and performance.

1. Split the authoring representation from the shipping artifact, hard. Aarav's live
   design is a tree of references to fonts and photos and animations. The exported MP4
   carries none of that, it carries baked pixels. Rare.lab must do the same: the node
   graph in the editor is rich, live, and full of references, but the compiled output for
   the embeddable runtime must be self contained and carry nothing it needs to look up at
   runtime. Do the resolving (fetch, bake, inline, validate) at compile time, once, so the
   runtime is a flat lookup, never a resolve. This is the offline think, online lookup
   spine applied to a shader compiler.

2. Make export two clean halves: resolve, then rasterize and encode. For Rare.lab that is
   resolve (gather every texture, LUT, sub graph, and constant into a self contained
   plan, and run every validation and license and safety check here) then codegen and pack
   (emit the actual shader code and the packed runtime bundle). Keep them separate so a
   missing asset fails loud at resolve time and a slow codegen is a compute problem you can
   throw hardware at. Do not let a runtime meet a missing texture for the first time on a
   user's phone.

3. The engine choice is the performance, so benchmark rasterizers, do not argue about
   them. Canva got 80% faster rendering and 70% less peak memory on the same one minute
   Lottie video purely by swapping rLottie for ThorVG. Rare.lab's whole job is turning a
   graph into pixels, so the pixel pushing core (the rasterizer, the shader compiler
   backend, the math library) is not a detail, it is the product's ceiling. Build a real
   benchmark (a representative heavy scene, like Canva's one minute all Lottie video) and
   measure candidate backends on speed and peak memory before you commit. Peak memory
   matters as much as speed on the low end phones your embeddable runtime will actually run
   on.

4. Compile is a queued, async, specialized fleet job, never an inline one. Canva does not
   render a video inside the web request, it enqueues a job and returns a job id. Rare.lab
   compiles (especially heavy ones: many shader permutations, a big graph, a video export
   of an effect) should be async jobs with a status of in_progress then success or failed,
   with progress events for the waiting human. And when you scale, specialize the fleet:
   light shader compiles on cheap CPU workers, heavy video or GPU heavy bakes on GPU
   workers, each autoscaled on its own queue depth, and pre warmed against your known peak
   rather than scaled up only after the queue backs up.

5. Dedupe compiles by content hash. Canva can render an unchanged template once and serve
   it to a thousand exporters. Rare.lab should hash (graph version, target, quality
   settings) and cache the compiled bundle, so a popular effect that a thousand sites embed
   unchanged compiles once, not a thousand times. This matters most exactly when something
   goes viral, which is the worst time to be recompiling.

One line to keep: export is a compile step that freezes a live authored tree into a self
contained artifact by resolving everything then rasterizing and encoding it, run as a
queued async job on a specialized, pre warmed, content deduped render fleet, and the
rasterizer you pick is the performance you get, so Rare.lab should build its compiler the
same way and prove its rendering core with a real benchmark, not an opinion.

---

## Sources

- LottieFiles case study, "Canva Enhances iOS Lottie Rendering: 80% Faster and 70% More
  Efficient with ThorVG": https://lottiefiles.com/case-studies/canva and
  https://lottiefiles.com/blog/working-with-lottie-animations/canva-enhances-ios-rendering-faster-and-efficient-with-thorvg
- ThorVG (open source C++ vector graphics engine for SVG and Lottie), overview:
  https://docs.lottiefiles.com/en/runtimes/overview/thorvg and https://www.thorvg.org
- Canva Connect API, Create design export job (async job, in_progress then success or
  failed): https://www.canva.dev/docs/connect/api-reference/exports/create-design-export-job/
- Canva Connect API, Get design export job (poll for status, per page download URLs,
  24 hour URL expiry): https://www.canva.dev/docs/connect/api-reference/exports/get-design-export-job/
- Canva Apps SDK, Exporting designs (formats, multi page ZIP behavior, backend download
  guidance): https://www.canva.dev/docs/apps/exporting-designs/
- Canva usage and revenue statistics (30 billion designs by Dec 2024, ~38.5M designs/day,
  260M+ MAU in 2025, 31M+ paid users, ~$4B ARR): https://backlinko.com/canva-users and
  https://en.wikipedia.org/wiki/Canva
