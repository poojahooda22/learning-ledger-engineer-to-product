# References: Figma canvas rendering engine

Keepers for the 2026-07-09 teardown (the WebGL/WebGPU tile renderer, distinct from
the 2026-06-18 multiplayer/sync teardown).

## Primary (Figma engineering + Evan Wallace)

- Building a professional design tool on the web (Evan Wallace, Figma). The canonical
  source for the renderer. Why not HTML/SVG/2D-canvas: "a lot of baggage," "often much
  slower than the 2D canvas API due to the DOM," optimized for scrolling not zooming,
  geometry "re-tessellated after every scale change," no GPU guarantee, and masking/blur/
  blend "varies wildly between browsers." Figma built "everything from scratch using
  WebGL" as a "highly-optimized tile-based engine" with masking, blurring, dithered
  gradients, blend modes, nested layer opacity; "all rendering is done on the GPU" and is
  "fully anti-aliased." https://www.figma.com/blog/building-a-professional-design-tool-on-the-web/
  Mirror: https://madebyevan.com/figma/building-a-professional-design-tool-on-the-web/

- WebAssembly cut Figma's load time by 3x (Evan Wallace, Figma). Editor is C++
  cross-compiled via Emscripten; WASM "parses around 20x faster than asm.js"; load time
  cut >3x regardless of doc size (large docs ~12s -> <4s). Memory: "the generated code is
  completely in control of allocation," "all C++ objects are just reserved ranges in a
  pre-allocated typed array," "the JavaScript garbage collector is never involved,"
  "makes it much easier to hit 60fps by avoiding GC pauses."
  https://www.figma.com/blog/webassembly-cut-figmas-load-time-by-3x/
  Mirror: https://madebyevan.com/figma/webassembly-cut-figmas-load-time-by-3x/

- Figma Rendering: Powered by WebGPU (Figma, 2024). Migration WebGL -> WebGPU:
  redesigned graphics interface to make draw-call arguments explicit; custom shader
  processor translates GLSL -> WGSL; uniform handling batched so "uniform data for all
  encoded draw calls is uploaded to a single buffer," each draw call reading its uniforms
  by offset; explicit pipelines + bind groups "greatly reduce CPU overhead" and scale to
  large object counts; compute shaders move work CPU -> GPU; future: blur via compute
  shaders, WebGPU MSAA, RenderBundles. https://www.figma.com/blog/figma-rendering-powered-by-webgpu/

- Easy Scalable Text Rendering on the GPU (Evan Wallace). Renders glyph outlines
  directly, no baked SDF texture. TrueType glyph = quadratic Bezier segments; fill via
  winding number over the glyph bounding box (Haines 1994 triangle-fan), coverage
  accumulated through color blending instead of the stencil buffer; curved edges via
  Loop-Blinn 2005 quadratic implicitization. Resolution independent, much less memory
  than distance fields, supports subpixel AA.
  https://medium.com/@evanwallace/easy-scalable-text-rendering-on-the-gpu-c3f4d782c5ac

- Fast Rounded Rectangle Shadows (Evan Wallace). A 2D box shadow = product of two
  perpendicular 1D blurred edges; convolution of a 1D Gaussian with a 1D step has a
  closed form (the error function, erf), so a box-shadow pixel is one constant-time
  formula with no sampling. Rounded rects have no exact closed form: use the closed form
  along one axis + light sampling along the other.
  https://madebyevan.com/shaders/fast-rounded-rectangle-shadows/

## Secondary / outside confirmation

- Andrew Chan, "Notes From Figma II: Engineering Learnings." Independent engineer's
  notes on the renderer, WASM, and WebGPU migration.
  https://andrewkchan.dev/posts/figma2.html

- Raph Levien, "Blurred rounded rectangles." Independent derivation of the erf-based
  blurred-rounded-rect math; good cross-check on Wallace's shadow technique.
  https://raphlinus.github.io/graphics/2020/04/21/blurred-rounded-rects.html

- Zed, "Leveraging Rust and the GPU to render user interfaces at 120 FPS." Zed's GPUI
  framework adopts Wallace's Gaussian-blur-in-SDF shadow technique. Outside proof the
  method is real and worth copying. https://zed.dev/blog/videogame

## Foundational papers (cited by Wallace's text technique)

- E. Haines, "Point in Polygon Strategies," Graphics Gems IV (1994). Triangle-fan fill.
- C. Loop, J. Blinn, "Resolution Independent Curve Rendering using Programmable Graphics
  Hardware," SIGGRAPH 2005. Quadratic Bezier implicitization used to evaluate curved
  glyph edges on the GPU.

## The one-line takeaway

Draw the canvas yourself, on the GPU, with manual memory (no GC), and cache unchanged
tiles so a frame costs what the viewport shows, not what the document holds. Prefer
closed-form GPU math (erf shadows, textureless glyph fill) over sampling. Keep the
render backend swappable (WebGL -> WebGPU) so a new substrate is a migration, not a
rewrite.
