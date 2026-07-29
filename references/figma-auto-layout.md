# References: Figma Auto Layout (2026-07-29)

## Figma primary sources
- Behind the feature: the making of the new Auto Layout: https://www.figma.com/blog/behind-the-feature-the-making-of-the-new-auto-layout/
- Use auto layout with CSS Flexbox in mind (Figma Learn): https://help.figma.com/hc/en-us/articles/42031586813719-Use-auto-layout-with-CSS-Flexbox-in-mind
- How We Rebuilt the Foundations of Component Instances (dependency graph, push-based invalidation, Materializer, subtree recompute + caching): https://www.figma.com/blog/how-we-rebuilt-the-foundations-of-component-instances/
- Figma Rendering: Powered by WebGPU: https://www.figma.com/blog/figma-rendering-powered-by-webgpu/
- Keeping Figma Fast: https://www.figma.com/blog/keeping-figma-fast/
- Speeding Up File Load Times, One Page At A Time: https://www.figma.com/blog/speeding-up-file-load-times-one-page-at-a-time/
- Improving Performance with Incremental Frame Loading: https://www.figma.com/blog/incremental-frame-loading/

## Talks
- Joey Liaw, "A Tour of the C++ Engine which Powers That Design MMO Called Figma", Config 2022: https://www.youtube.com/watch?v=opGoe7yHHkk

## Flexbox layout algorithm (for the two-pass inference)
- MDN, Basic concepts of flexbox: https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Flexible_Box_Layout/Basic_Concepts_of_Flexbox
- tchayen, How to Write a Flexbox Layout Engine (multi-pass measure/layout): https://tchayen.com/how-to-write-a-flexbox-layout-engine

## Key extracted facts
- Canvas engine: C++ compiled to WebAssembly via Emscripten; draws to GPU (WebGL, now WebGPU), bypasses browser HTML/CSS layout. 2018 renderer rewrite ~3x faster.
- Invalidation: explicit dependency graph, push-based; mark dependents dirty, recompute lazily; Materializer auto-records dependencies as nodes read data; only affected subtree recomputes; subtree computations cached.
- Auto Layout model: direction (row/column), gap, padding, alignment (incl. space-between); child sizing modes Fixed / Hug contents / Fill container; documented to mirror CSS flexbox.

## Fact vs inference
- Fact: C++/WASM scene graph, GPU rendering, 3x speedup, dependency-graph/dirty-flag/Materializer model, the Fixed/Hug/Fill model, flexbox parity claim.
- Inference (grounded): the specific two-pass "measure bottom-up (Hug), place top-down (Fill/flex-grow)" layout algorithm. Figma has not published its exact pass; this is how all flexbox-class engines are built.
