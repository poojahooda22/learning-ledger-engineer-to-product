# Figma: the canvas rendering engine (the smooth infinite canvas)

Date: 2026-07-09
Product: Figma
Feature: the canvas rendering engine (WebGL/WebGPU tile renderer, C++ compiled to WebAssembly)

A note on granularity. The ledger already covered Figma's multiplayer cursors and
live presence (2026-06-18). That was the network and sync half: how edits stay
conflict free and how cursors fly around. This teardown is a completely different
feature: the thing that actually draws pixels on the screen. When you drag a
rectangle and it moves at 60 frames a second, when you zoom from 10% to 6400% on a
single icon and the edges stay razor sharp, when a design file with 40,000 layers
still pans without stutter, that is the rendering engine. It runs entirely on your
GPU, inside a browser tab, and almost nobody notices it, which is the whole point.

---

## 1. The user

Meet Ananya. She is a product designer at a mid-size startup. It is 3pm and she has
the company's design system file open: a single Figma document with about 12,000
layers. Buttons, input fields, icons, color styles, a hundred component variants of
"Button/Primary", and forty full screen mockups pinned around the canvas like photos
on a wall.

She is doing the most ordinary thing in the world. She scrolls sideways to get from
the login screen to the checkout screen. She pinch-zooms out to see all forty screens
at once, then zooms all the way in to nudge one 16-pixel icon by one pixel. She drags a
card with a soft drop shadow across the canvas. She has three teammates' cursors moving
in the same file.

She is not thinking about any of this. She expects it to feel like moving paper on a
desk. The moment it stutters, she notices, and she gets annoyed, and some part of her
brain quietly files "this tool is slow."

---

## 2. The real problem

Here is the honest version, the way you would explain it to a friend.

A design file is not a document. It is a drawing. A big Google Doc is maybe a few
thousand paragraphs of text laid out top to bottom. Ananya's design file is 12,000
independent shapes floating on an infinite 2D plane, each with its own position, fill,
stroke, corner radius, shadow, opacity, and blend mode. Some overlap. Some are inside
frames that clip them. Some have a blur.

Now she zooms. When you zoom a text document, the browser just makes the font bigger,
which it is very good at. When you zoom a drawing, every single shape has to be
redrawn at the new scale, and a curve that looked fine at 100% needs more detail at
400% or the edges go jagged. A rectangle at a slight rotation needs its edges
re-smoothed. Do that for 12,000 shapes, sixty times a second, while she is actively
pinching. That is the problem.

The browser gives you three built-in ways to draw: HTML with CSS, SVG, and the 2D
canvas API. Figma's own engineering write-up is blunt about why none of them worked.
HTML and SVG "contain a lot of baggage" and are "often much slower than the 2D canvas
API due to the DOM." They are optimized for scrolling, not zooming, so geometry gets
"re-tessellated after every scale change." There is no guarantee that anything is GPU
accelerated. And support for the exact features designers need every day, masking,
blurring, and blend modes, "varies wildly between browsers." (Source: Figma, "Building
a professional design tool on the web.")

Translation: if Figma had built on the normal web stack, Ananya's 12,000-layer file
would crawl, and her drop shadow would look different in Chrome than in Safari. A
professional tool cannot ship that.

---

## 3. The feature in one sentence

Figma draws the entire canvas itself, from scratch, on the GPU, using a rendering
engine written in C++ and compiled to WebAssembly that talks straight to WebGL (and
now WebGPU), so a huge design file pans, zooms, and edits at a smooth frame rate
inside a plain browser tab.

---

## 4. Jobs to be done

What is Ananya really hiring this engine to do?

- "Let me move around a big file like it is paper on a desk, no stutter, no waiting."
- "Keep my edges crisp at any zoom, from the whole board down to one pixel."
- "Make my drop shadow, my blur, my blend mode look exactly the same for me and for
  the developer who opens this in a different browser."
- "Do all of this in a browser tab I can open on any machine, no install, so my whole
  team is in the same file."

That last one is the quiet business job. The reason Figma could beat Sketch (a
desktop-only Mac app) was not a better pen tool. It was being a real, fast design tool
that happened to live at a URL. None of that works if the canvas feels like a website.

---

## 5. How it works for the user

From Ananya's seat, there is nothing to "use." That is the feature. She sees:

- An infinite canvas she can pan forever in any direction.
- Buttery zoom from 1% to 6400%. At every zoom, curves are smooth and text is sharp.
- Shapes that redraw instantly when she changes a fill or drags a corner.
- Effects that render live: a Gaussian blur on a background card, a soft drop shadow
  under a button, a "Multiply" blend mode on an overlay, all updating as she drags a
  slider.
- The same picture, pixel for pixel, on her Mac in Chrome and on the developer's
  Windows machine in Edge.

The success condition is that she never thinks about it. The failure condition is a
single dropped frame.

---

## 6. The actual flow, step by step

Follow one real gesture: Ananya pinch-zooms out from the checkout screen to see all
forty screens.

1. Her trackpad fires a stream of zoom events. The browser hands them to Figma's
   JavaScript layer.
2. That layer forwards each event into the WebAssembly module, the compiled C++
   engine, which holds the real document in its own memory.
3. The engine updates one number: the viewport transform (a pan offset and a zoom
   scale). It does not touch the 12,000 shapes. They did not change. Only the camera
   moved.
4. The engine figures out which parts of the canvas are now visible, and which of
   those it has already drawn and cached.
5. For anything not cached, it builds GPU geometry: it turns each visible shape into
   triangles the GPU can rasterize, batches many shapes into few draw calls, and hands
   them to WebGL/WebGPU.
6. The GPU rasterizes everything, runs the shader programs that do the fills,
   gradients, blurs, shadows, and blend modes, and produces the final frame.
7. That frame goes to the screen. Steps 3 through 7 happen in under 16 milliseconds so
   the next frame is ready in time for 60fps.

Notice what did not happen. No 12,000 DOM nodes were created or moved. No geometry was
thrown away and rebuilt from scratch on every zoom tick. The JavaScript garbage
collector was never invoked in the hot loop. All the heavy lifting is inside the C++
engine and on the GPU.

---

## 7. Under the hood, like the engineer

This is the heart of the report. There are four real engineering ideas stacked on top
of each other. I will go through each, mark clearly what Figma has published as fact
versus what is well-grounded inference about how this class of problem is solved.

### 7.1 The document is a scene graph, not a DOM (fact + inference)

Fact: Figma reimplemented "everything from scratch using WebGL" as a "highly-optimized
tile-based engine with support for masking, blurring, dithered gradients, blend modes,
nested layer opacity, and more," with "all rendering done on the GPU" and "fully
anti-aliased." (Source: Figma, "Building a professional design tool on the web.")

Inference (standard for this class of tool): the document lives as a tree of nodes. A
frame contains children, a group contains children, each leaf is a shape. This is the
same shape as the block tree the ledger saw in Notion (2026-06-25) and the object tree
in the Figma multiplayer teardown (2026-06-18): a node is an ID plus a property map
plus an ordered list of child IDs plus a parent pointer. Ananya's "Button/Primary"
component is one node with a rectangle child, a text child, and an icon child.

Why a tree and not a flat array? Because clipping, opacity, and transforms cascade. A
frame at 50% opacity dims everything inside it. Rotating a group rotates its children.
A tree lets you multiply transforms down the branches and inherit properties, which a
flat list cannot express cleanly.

Why not the browser's own tree, the DOM? Because the DOM is the exact baggage Figma
threw out. Every DOM node carries layout, style resolution, event handling, and
accessibility machinery that Ananya's rectangle does not need, and touching the DOM is
slow. Twelve thousand DOM nodes that must all re-layout on every zoom is the thing that
kills the naive approach.

### 7.2 Own the memory: C++ compiled to WebAssembly, no garbage collector in the hot path (fact)

This is the piece with the hardest published numbers, and it matters most for 60fps.

Figma's editor is written in C++ and cross-compiled to run in the browser, originally
via Emscripten to asm.js, then to WebAssembly. Two facts from Figma's own posts:

- WebAssembly "parses around 20x faster than asm.js," and switching to it "cut Figma's
  load time by 3x" regardless of document size. Reported large-document load times fell
  from roughly 12 seconds to under 4. (Source: Figma, "WebAssembly cut Figma's load
  time by 3x.")
- On memory: "the generated code is completely in control of allocation," so "all C++
  objects are just reserved ranges in a pre-allocated typed array" and "the JavaScript
  garbage collector is never involved," which "makes it much easier to hit 60fps by
  avoiding GC pauses." (Source: same post.)

Unpack that last one, because it is the deep idea. In normal JavaScript, when you
create thousands of little objects per frame (a point here, a matrix there), the
garbage collector eventually has to stop the world and clean up. A GC pause of even 10
milliseconds in the middle of a pinch-zoom is a visible stutter. Figma sidesteps the
GC entirely: the whole C++ heap is one big typed array that JavaScript sees as an
opaque block of bytes. When the engine "allocates" a shape, it just carves out a range
inside that array. The JS GC has nothing to collect because, as far as it knows, there
is only one giant object. This is manual memory management smuggled inside a managed
runtime, and it is what buys a predictable frame budget.

Concrete example: as Ananya drags a card across the canvas, the engine recomputes its
bounding box and transform matrix every frame. In a GC language that is a fresh matrix
object 60 times a second times however many shapes. In Figma's arena it is writes into
a pre-owned range of bytes. Zero garbage produced, zero collection triggered.

### 7.3 Tile-based rendering: cache what did not change, redraw only the dirty part (fact + inference)

"Tile-based" is the word Figma uses for its engine, and it is the scalability trick.

The idea: divide the canvas into a grid of fixed-size tiles, say 256 by 256 pixels
each. Render each tile once into a GPU texture and keep it. On the next frame, if a
tile's contents did not change, do not redraw it, just re-blit the cached texture. Only
tiles that actually changed (the "dirty" tiles) get re-rendered.

A community reconstruction of the approach describes tiles classified by visibility:
tiles that intersect the current view, plus a ring of cached tiles just outside the
viewport that are "potentially visible," so a small pan reveals already-drawn content
instead of a blank edge. (Inference, consistent with Figma's "tile-based" description;
the exact tile size and eviction policy are not public.)

Walk Ananya's two gestures through it:

- She pans sideways from checkout to login. Most of the canvas that scrolls into view
  was already rendered and cached as tiles. The engine mostly copies textures. Cheap.
  Only the fresh strip at the leading edge is new work, and even some of that was
  pre-rendered in the "potentially visible" ring.
- She drags one card. Only the tiles the card overlaps are marked dirty. The other
  11,999 shapes sit untouched in their clean tiles. The engine redraws a handful of
  tiles, not the whole document.

This is the same recurring spine the ledger keeps finding, in a new costume: keep the
expensive work off the hot path and serve the frequent path from a cache. Discover
Weekly precomputes recommendations offline; YouTube precomputes embeddings; Figma
precomputes tiles. The live action is a cheap lookup or a cheap copy.

### 7.4 Draw it right on the GPU: analytic anti-aliasing, closed-form shadows, textureless text (fact)

Getting pixels on the GPU is only half the fight. Making them look good, crisp edges,
soft shadows, sharp text, at any zoom and with no blurry mush, is the other half.
Figma's engineers published two beautiful tricks.

Text (fact, Evan Wallace, "Easy Scalable Text Rendering on the GPU"). Most GPU text
engines pre-bake each glyph into a signed distance field texture. That eats memory and
gets soft at large sizes. Figma's approach renders the glyph outline directly. A
TrueType glyph is built from quadratic Bezier curve segments. The engine fills the
glyph by computing a winding number for the pixels inside the glyph's bounding box (the
classic triangle-fan fill from Haines 1994), but instead of using the stencil buffer it
accumulates coverage through color blending, and it evaluates the curved edges using
the Loop and Blinn (2005) implicit form of a quadratic Bezier. The payoff: it is
resolution independent (sharp at any zoom, because it is drawing the real curve, not a
baked bitmap), uses much less memory than distance fields (no per-glyph texture cache),
and still supports subpixel anti-aliasing. When Ananya zooms into the word "Sign up" on
a button, the letters stay perfectly crisp because the GPU is re-evaluating the actual
letter outlines, not scaling a picture of them.

Drop shadows (fact, Evan Wallace, "Fast Rounded Rectangle Shadows"). A soft shadow is
a Gaussian blur of a shape. The slow way is to actually blur, sampling many neighboring
pixels per output pixel. Wallace's insight: a box shadow is the product of two
perpendicular 1D blurred edges, and the convolution of a 1D Gaussian with a 1D step
edge has a closed-form answer, the error function (erf). So for a plain box, a shadow
pixel is one constant-time formula, no sampling at all. For a rounded rectangle there
is no exact closed form, so the engine uses the closed form along one axis and a little
sampling along the other. Ananya's button shadow updates live as she drags the blur
slider because each shadow pixel is a cheap math expression, not a heavy blur pass.
(This exact technique was later adopted by the Zed editor's GPUI framework, which is a
nice outside confirmation that it is real and good.)

Anti-aliasing everywhere (fact). Figma states all rendering is "fully anti-aliased" on
the GPU. Rather than draw at higher resolution and shrink (expensive), the shaders
compute how much of each edge pixel is actually covered by the shape and blend
accordingly. That is why a rotated rectangle at 100% zoom has smooth edges without a
4x-resolution buffer behind it.

### 7.5 The scale story at three tiers

Tier one, about 1,000 layers (a single app screen). Almost anything works here. Even a
naive SVG or 2D-canvas renderer can push 1,000 shapes. Figma's engine is not breaking a
sweat: the whole document might fit in a handful of tiles, and a full redraw is cheap.
The tile cache barely matters yet.

Tier two, about 100,000 layers (a real design system or a busy team file). Now the
naive approaches die. A DOM with 100,000 nodes is unusable, layout alone chokes. Even a
2D-canvas full redraw every frame blows the 16ms budget. This is the tier where
Figma's choices start paying rent: draw only what is on screen (viewport culling by the
scene tree), cache everything else as tiles, redraw only dirty tiles, batch thousands of
shapes into a few GPU draw calls, and keep zero garbage so no GC pause interrupts a
pan. What would break at the next tier: even building GPU geometry for every visible
shape, every frame, starts to cost, and the per-draw-call CPU overhead adds up.

Tier three, 1,000,000 to 10,000,000-plus layers (a giant org file, a whole product's
design history in one document). Here you cannot afford to even walk every node each
frame. Survival techniques, some published, some standard inference:

- Spatial indexing so "what is visible" is a fast range query, not a scan of ten
  million nodes. (Inference: a quadtree or R-tree over the canvas is the textbook tool;
  Figma has not published its exact index.)
- Tile caching becomes mandatory, not an optimization. The visible set is tiny relative
  to the document, so you render a few dozen tiles and cache-copy the rest.
- Draw-call batching becomes the bottleneck, and this is exactly why Figma moved from
  WebGL to WebGPU. Fact (Figma, "Figma Rendering: Powered by WebGPU"): they redesigned
  the graphics interface to make draw-call arguments explicit, wrote a custom processor
  to translate their GLSL shaders to WGSL, and switched uniform handling so that "uniform
  data for all encoded draw calls is uploaded to a single buffer," with each draw call
  reading its uniforms by offset into that one buffer. WebGPU's explicit pipelines and
  bind groups "greatly reduce CPU overhead" and scale more smoothly "to large numbers of
  objects." They also plan to move blur onto compute shaders and use WebGPU's MSAA and
  RenderBundles to cut CPU overhead further.
- Compute shaders (WebGPU only) let heavy effects like large blurs run on the GPU in
  parallel instead of pinning a CPU core.

The through-line across all three tiers: the cost of a frame should track what is on
screen and what changed, never the total size of the document. A ten-million-layer file
and a one-thousand-layer file should pan at the same frame rate as long as the viewport
shows the same amount. Tiling plus culling plus batching plus manual memory is how you
make frame cost depend on the viewport, not the catalog. This is the exact same "the
candidate set size is the real dial" lesson the ledger found in Amazon search and Canva
templates, here applied to pixels instead of results.

---

## 8. The retention and habit mechanic

This feature does not have an engagement loop like Instagram Stories or a weekly habit
like Discover Weekly. Its retention mechanic is subtler and, for a tool, more powerful:
performance is the product, and the product is the switching cost.

Here is the loop. Figma runs fast enough in a browser that a designer stops missing the
native desktop app. Because it is in the browser, it is one URL, so the whole team can
be in the same file at once (this is where the multiplayer teardown connects). Because
the team is in the file, the file becomes the source of truth. Because the file is the
source of truth, leaving Figma means moving the whole company. That is lock-in built
out of raw frame rate.

The metric this moves is retention and expansion, not a daily-active vanity number. The
observed real-world example is the market itself: Figma took the professional design
market from Sketch, a fast native Mac app, by being a browser tool that felt native. If
the canvas had stuttered, none of the collaboration story would have mattered, because
designers would not have tolerated it as their primary tool. The rendering engine is
the price of admission that made every other Figma advantage reachable. A designer who
tries a slower browser-based competitor and feels the lag has just been reminded why
they stay.

There is a smaller, immediate loop too. Live effects (blur, shadow, blend mode updating
in real time as you drag a slider) make editing feel like direct manipulation, like
touching the thing itself. That tight feedback keeps Ananya in a flow state, and flow
is what makes a tool feel good enough to open first thing tomorrow.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor for AI shaders and visual effects that compiles to
shippable code, plus an embeddable runtime. Figma's rendering engine is almost a
blueprint, because it is the same problem: draw complex visual content on the GPU, fast,
predictably, inside someone else's environment.

The one concrete lesson: make frame cost depend on what changed, not on graph size, by
caching the output of unchanged subgraphs the way Figma caches unchanged tiles.

Here is the direct mapping. In Rare.lab's node editor, when a user tweaks one node (say
they bump the intensity on a bloom node near the end of the chain), you do not need to
re-evaluate the entire graph. Everything upstream of the edited node produced the same
output as last frame. Cache each node's output as a GPU texture or buffer, mark only the
edited node and its downstream descendants "dirty," and recompute only that path. This
is Figma's dirty-tile idea moved from screen space to graph space. A user editing one
node in a 300-node graph should pay for a handful of nodes, not 300, exactly like Ananya
dragging one card pays for a handful of tiles, not 12,000 shapes.

Three supporting moves, all straight from this teardown:

1. Own your memory in the runtime. If Rare.lab's embeddable runtime hits the GPU every
   frame, do what Figma did: pre-allocate arena buffers and reuse them, never allocate
   per-frame in a way that triggers a GC pause. A single 10ms GC hitch is a dropped
   frame in a visual effect, and users feel it instantly.

2. Prefer closed-form GPU math over sampling loops when you can. Wallace's erf drop
   shadow replaced a many-tap blur with one formula. When Rare.lab's compiler lowers a
   node to a shader, teach it to recognize effects that have analytic solutions and emit
   the closed form instead of a naive sampling kernel. That is a real, measurable
   framerate win in the shipped code, which is your actual product.

3. Keep the graph IR backend-agnostic so you can move substrates. Figma spent real
   effort migrating WebGL to WebGPU precisely because their rendering was theirs to
   move, and WebGPU unlocked compute shaders. Rare.lab's promise is "compiles to
   shippable code," so design the intermediate representation so the same node graph can
   emit WGSL today and whatever comes next tomorrow, without rewriting the editor. The
   backend should be a swappable target, not baked into the graph.

The meta-lesson, and it is the ledger's oldest one: the expensive work goes off the hot
path (compile and cache offline or on edit), and the live path is a cheap lookup, copy,
or single forward pass. Figma proves it works for pixels. Rare.lab should make it work
for nodes.

---

## Sources

- Figma, "Building a professional design tool on the web" (Evan Wallace): https://www.figma.com/blog/building-a-professional-design-tool-on-the-web/
- Figma, "WebAssembly cut Figma's load time by 3x" (Evan Wallace): https://www.figma.com/blog/webassembly-cut-figmas-load-time-by-3x/
- Figma, "Figma Rendering: Powered by WebGPU": https://www.figma.com/blog/figma-rendering-powered-by-webgpu/
- Evan Wallace, "Easy Scalable Text Rendering on the GPU": https://medium.com/@evanwallace/easy-scalable-text-rendering-on-the-gpu-c3f4d782c5ac
- Evan Wallace, "Fast Rounded Rectangle Shadows": https://madebyevan.com/shaders/fast-rounded-rectangle-shadows/
- Andrew Chan, "Notes From Figma II: Engineering Learnings": https://andrewkchan.dev/posts/figma2.html
- Haines, "Point in Polygon Strategies," Graphics Gems IV (1994) (triangle-fan fill referenced by Wallace's text technique)
- Loop and Blinn, "Resolution Independent Curve Rendering using Programmable Graphics Hardware," SIGGRAPH 2005 (quadratic Bezier implicitization used by Wallace's text technique)
- Raph Levien, "Blurred rounded rectangles" (independent treatment of the erf-based shadow math): https://raphlinus.github.io/graphics/2020/04/21/blurred-rounded-rects.html
- Zed, "Leveraging Rust and the GPU to render user interfaces at 120 FPS" (adopts Wallace's Gaussian-blur-in-SDF shadow technique): https://zed.dev/blog/videogame
