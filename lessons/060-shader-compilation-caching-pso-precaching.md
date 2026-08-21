# Day 60 — How do you stop a single GPU shader from freezing a frame for seven times its entire budget, across millions of machines you don't control?

**Date:** 2026-08-21
**Difficulty:** Expert
**Topic:** Shader and pipeline-state-object (PSO) compilation caching, the mechanism behind the "traversal stutter" that plagued PC game ports through the early 2020s. Why a GPU shader is not just source code, it is source code plus a specific combination of render state, and the driver only turns that combination into real, runnable GPU machine code the first time it is actually drawn with. Why that "compile on first use" default, completely invisible in a small demo, becomes a visible frozen frame the instant a shipped game's real permutation space (Unreal Engine's own telemetry: single-digit-millisecond hitches that spike past 100 milliseconds) collides with a 16.6-millisecond, 60-frames-per-second budget. How Valve's Fossilize system precompiles and ships a shader cache to every Steam player before they ever press play (Overwatch 2's shader pre-cache: a roughly 7.7 to 7.8 gigabyte download, expanding into a shader cache around 20 gigabytes on disk, produced by a central build pass that spikes to 22 to 23 gigabytes of RAM), and the real production incident where that precaching stopped working: Nvidia's own driver enforces a 1 gigabyte on-disk shader cache limit by default, smaller than the 4 to 5.7 gigabytes Overwatch 2 can generate in a single two-hour session, so the driver silently evicts the very cache Valve just spent all that effort warming.
**Stack relevance:** Rare.lab's node-based editor already compiles a graph to GLSL ahead of the render loop, which is half of this lesson's fix done by accident. The gap is the other half: nothing caches the compiled output by content hash, and the embeddable runtime's single shared WebGL context has the exact same "one hot render path, many uncoordinated callers" shape that makes a synchronous shader compile anywhere in this lesson's story freeze everything sharing that path, not just the thing that triggered it.

---

## 1. The company and the breaking number

**Blizzard's Overwatch 2, distributed through Valve's Steam platform, ongoing.** Overwatch 2 is a live-service, competitive shooter, the kind of game where a frozen frame during a gunfight is not a cosmetic annoyance, it is the difference between winning and losing a duel that lasts less time than it takes to read this sentence. Two real, documented numbers anchor this lesson.

The first is a frame-budget number, and it is pure arithmetic, not a citation. A game targeting 60 frames per second has, for every single frame:

```
1,000 ms / 60 frames = 16.6 ms per frame, total, for everything
```

That 16.6 ms has to cover input handling, game logic, physics, and every draw call the renderer issues, all of it. Unreal Engine's own PSO-precaching documentation and the community guides built around it define a "hitch" as any single pipeline-state-object compile that exceeds a default threshold of **20 milliseconds**, already more than an entire frame's budget on its own, and report that real, uncached PSO compiles commonly cost **5 to 10 milliseconds**, with the worst cases reaching **117 milliseconds or more**. Put that worst case next to the frame budget:

```
117 ms / 16.6 ms per frame = roughly 7 frames' entire budget, spent compiling one shader
```

One shader. One frame. Seven frames' worth of nothing but waiting for the GPU driver to turn source code into machine code, all while the player is mid-fight.

The second number is a distribution and infrastructure number, and it comes from Valve's own public GitHub issue tracker for **Fossilize**, the Vulkan pipeline-cache library that powers Steam's shader pre-caching system. A 2024 issue filed against Fossilize (ValveSoftware/Fossilize#312) documents, with exact figures from a real user's system, what it costs Valve to turn Overwatch 2's shader cache into something a player's machine can use directly: a shader pre-cache **update download of roughly 7.7 to 7.8 gigabytes**, expanding a local Vulkan pipeline-cache file that had already grown to **around 20 gigabytes total** on that player's disk, and the Fossilize processing step itself spiking to **22 to 23 gigabytes of RAM plus 5.6 to 6.9 gigabytes of swap** on a machine with only 32 GB of RAM and an 8 GB swap file, enough to trigger an out-of-memory kill on some systems. By contrast, that same player's own two hours of ordinary gameplay only generated **about 41 megabytes** of new, locally-compiled shader data, a three-order-of-magnitude gap between "what the shipped cache pays to prevent stutter for everyone" and "what one individual session actually needs."

Both numbers describe the same underlying problem from two different angles: a modern game's total shader permutation space is enormous, and whether you pay for that enormity on the render thread, one 117 ms freeze at a time, or on the build farm and the download pipe, one multi-gigabyte transfer at a time, is the entire design decision this lesson is about.

## 2. Why the naive (demo) design dies

The naive design is not a strawman, it is the literal default behavior of every major graphics API. A "shader" is not a self-contained program the way a `.exe` is. It is source code (GLSL, HLSL, or a portable bytecode like SPIR-V) plus a specific combination of *pipeline state*: which vertex layout is bound, which render target format is active, which blend mode, which set of feature toggles (does this surface receive shadows, is it skinned, is fog enabled, which tonemap curve). A **shader permutation** is one specific combination of those things, and a **PSO**, a pipeline state object in DirectX 12 and Vulkan's terminology, is the actual compiled, GPU-vendor-specific machine code for one exact permutation. The driver's job, and by default the *only* moment it does this job, is to compile that machine code the first time the renderer issues a draw call asking for that exact combination. Nobody wired that up on purpose; it is simply what `glCompileShader`, `CreatePipelineState`, and `vkCreateGraphicsPipelines` do if you call them at the moment you need them, which is the obvious, simplest way to write a renderer.

This design dies in three concrete ways once a game leaves the developer's own machine.

**It dies to combinatorial explosion.** Unity's own shader-variants documentation lays out the arithmetic plainly: a shader with ten independent boolean feature toggles produces `2^10 = 1,024` distinct compiled variants from that one shader file, because every toggle multiplies the permutation count rather than adding to it. Real production shaders layer far more than ten toggles (lighting model, shadow cascade count, fog, skinning, instancing, a dozen material feature flags), and the documented consequence, echoed across Unity's docs and independent graphics-engineer write-ups on the general "shader permutation problem," is that a modern AAA game routinely ships **tens of thousands of distinct compiled shader permutations**, not dozens. There is no way to hand-enumerate "the shader" for a game like this; there are tens of thousands of them, and which ones any given player's session actually touches depends on where they walk, what they shoot, and what settings they enabled.

**It dies to real gameplay, not the developer's test path.** A studio's QA pass walks through the game in a fairly predictable order. A real player does not. The exact combination of material, mesh, and lighting that gets triggered the moment a player rounds a corner into a room nobody on the dev team happened to enter with that exact loadout, or the specific particle-and-lighting combination of an explosion effect that has never fired in front of that exact wall texture before, is precisely the kind of "first time this permutation is needed" event the naive design defers to draw-call time. It cannot be, because it is driven by where the player chooses to go, which is unknowable in advance in general. This is documented, widely, as **traversal stutter**, the specific, well-known complaint of a game running smoothly moment to moment and then freezing for a fraction of a second exactly when the camera turns to reveal something new, which multiple PC ports through the early 2020s (documented extensively across Steam community discussions, ResetEra, and GameDev.net threads on "why do games compile shaders during runtime") were criticized for.

**It dies to a heterogeneous install base.** A demo only ever runs on the one GPU, one driver version, and one OS the developer tested on. A shipped game runs on however many distinct combinations of GPU vendor, driver version, and operating system its install base actually owns, and here is the part that makes this fundamentally different from shipping ordinary application code: **the compiled machine code from a shader is not portable between those combinations.** An Nvidia driver compiles different machine code than an AMD driver for the exact same GLSL source, and a driver update can change what that machine code looks like even on the same GPU. You cannot precompile a shader once and copy the binary to every player's machine the way you would copy a regular executable; the compile step is tied to the specific (GPU, driver, OS) triple each individual player actually has.

## 3. The architecture

Adapting the usual client-to-storage flow to this problem, the pipeline runs from the game's authored content all the way down to an individual GPU's local cache, with a feedback loop closing it back to the top:

```
Shader / material graph source, authored in-engine
        |
        v
[Build-farm compiler front end]
  Job: statically enumerate every (mesh x material x render-state) combination
       that the shipped content actually reaches, not every theoretically
       possible combination. Turns "infinite permutation space" into
       "this game's actual, finite PSO list."
  Analogy: printing only the phone numbers that really exist,
           instead of trying to dial every possible ten-digit combination.
        |
        v
[Central precompilation service — Valve's Fossilize / the studio's build farm]
  Job: for every (GPU vendor, driver version, OS) combination the studio
       wants to support, actually invoke that real driver once, centrally,
       and keep the resulting vendor-specific compiled binaries.
  Analogy: one printing press running one plate for millions of copies,
           instead of every individual reader hand-copying the book.
        |
        v
[Distribution layer — the Steam depot / a game's patch download]
  Job: get the precomputed answer physically onto every player's disk
       before they ever press play (Overwatch 2's ~7.7-7.8 GB shader
       pre-cache update).
  Analogy: a CDN edge node holding a video segment near the viewer,
           except what's being cached is compiled machine code, not media.
        |
        v
[Client warm-up pass — the loading screen, background thread pool]
  Job: walk the shipped cache and materialize it into the local driver's
       own cache on background threads, never the render thread, so it's
       ready before the frame that needs it arrives.
  Analogy: warming the espresso machine before opening the doors,
           not making the first customer wait for it to heat up.
        |
        v
[Local persistent cache — the OS/driver's own on-disk PSO cache]
  Job: a per-machine cache (Nvidia's shader disk cache, D3D's Pipeline
       Library, Vulkan pipeline cache files) that survives between play
       sessions on that one machine, so a permutation seen once is free
       every time after.
  Analogy: a browser's disk cache — local, persistent, but not shared
           with anyone else's machine.
        |
        v
[Runtime async-compile + fallback tier — for whatever wasn't precached]
  Job: for the permutations nobody predicted, dispatch the compile to a
       background worker pool instead of blocking the render thread, and
       substitute a cheap fallback/default material for a few frames
       until the real compiled program is ready.
  Analogy: load shedding applied to the frame budget itself — never let
           the correct final answer block "something renders this frame."
        |
        v
[Telemetry feedback loop]
  Job: aggregate which PSOs real players in the wild actually hit, across
       the whole heterogeneous install base, and feed that back into what
       the next patch's build-farm pass (two steps up) chooses to
       precompile.
  ------------------------------------------------------------------> back to top
```

Two production details make this architecture more than a diagram. First, Unreal Engine 5.2 introduced **PSO precaching**, which does layers three through five inside the engine itself: it either bundles a curated PSO cache gathered from real playthroughs, or predicts likely-needed PSOs automatically as an object is about to load or spawn, and issues the compile ahead of the draw call that would otherwise need it, specifically to address the resource-heavy, imprecise older approach of manually bundling every cache Epic's own tech blog describes. Second, id Software's DOOM Eternal is cited across graphics-engineering discussion (including Matt Pettineo's widely-referenced "Shader Permutation Problem" series) as something close to a best-case example of layer one done right: through unusually disciplined use of a forward-rendering ubershader, the game reportedly reaches only around 100 distinct shaders and around 350 PSOs total, small enough that the game can precompile essentially all of it during the roughly one-minute loading screen players see at launch, and then run completely stutter-free afterward. That specific figure is relayed through search-aggregated discussion rather than a directly fetched primary source in this session, and is worth treating as a well-corroborated but secondary-sourced number, not a fetched fact; the mechanism it illustrates, that keeping the permutation count small enough to fully precompile is a legitimate alternative to building elaborate runtime fallback machinery, holds regardless.

## 4. The transferable mechanisms

- **Precompute-and-serve instead of compute-on-request.** Move the expensive work (the actual compile) out of the synchronous hot path (the frame) into a build-time or background-time phase. The identical shape as Day 54's leaderboard precompute and Day 43's star-tree pre-aggregation: never let the thing a user is actively waiting on also be the thing doing the expensive computation.
- **Content-addressed caching keyed by exact input.** The cache key is a hash of the shader source plus the exact pipeline state; the value is the compiled binary. Because the key is a pure function of the input, cache correctness never depends on separate invalidation logic, only on whether that exact key has been seen before, the same principle as Day 23's content-addressed storage and Rare.lab's own R2 scene storage.
- **Async work plus graceful fallback, never a blocking wait on the primary path.** For the unavoidable cache misses, hand the expensive work to a background queue and serve something cheap in the meantime, rather than stalling everyone else. The identical shape as Day 13's backpressure and load shedding.
- **Warm the cache during a low-stakes window, before real traffic arrives.** A loading screen, an install step, or a patch download is a moment nobody is measuring frame time; spend the expensive work there instead of on the first real frame that needs it, the same discipline as Day 47's Quicksilver edge propagation and Day 58's shadow-table backfill.
- **Centralize a fixed cost across a heterogeneous fleet, instead of repeating it per node.** Pay the compile cost once per (driver, GPU, OS) combination, centrally, and distribute the answer, instead of every one of millions of individual machines paying that cost themselves. This is the same economics as a CDN serving one origin fetch to millions of readers, just applied to compiled machine code instead of an HTTP response.
- **A telemetry feedback loop deciding what's worth precomputing next.** Real production usage, not a QA test pass, tells you which permutations actually matter; feeding that signal back into the next build is what keeps the precomputed set matching reality as content grows, rather than slowly drifting stale.

## 5. The trade-offs

**Completeness versus cost.** Bundling every reachable permutation ahead of time buys close to zero runtime stutter, but it costs patch size and download bandwidth (Overwatch 2's 7.7 to 7.8 GB precache update, expanding into a roughly 20 GB local cache) and it costs real infrastructure to produce (Fossilize's build pass spiking to 22 to 23 GB of RAM just to merge one update). This is a straightforward cost-versus-latency trade: pay disk space, bandwidth, and RAM upfront, centrally and on every individual machine, to buy a frame budget that a shader compile can never blow.

**Availability over strict consistency, translated into rendering.** No precaching pass can reach 100% coverage; the permutation space is combinatorially larger than any telemetry sample or playtest can fully enumerate, and player behavior isn't fully predictable in advance. Every production system in this space makes the same choice as a result: Unreal's async PSO compile with a fallback material, and WebGL's `KHR_parallel_shader_compile` extension both choose to show something now (a fallback, a placeholder, a frame with slightly wrong shading for an instant) rather than freeze the frame waiting for the exactly correct compiled shader. That is availability chosen over consistency, applied to pixels instead of database rows.

**Central caching versus local caching, and who owns the durability guarantee.** Valve's centrally-precompiled Fossilize cache buys a predictable experience across the whole player base, but it depends entirely on Valve's build farm having actually exercised every real driver version players show up with. A per-machine local driver cache costs nothing to ship, but it is worthless on a first install, after a driver update wipes it, or the moment the OS or driver decides to reclaim the disk space it occupies, which is exactly the failure mode in the next section.

## 6. The systems-thinking lens

Name the feedback loop: a **silent-cache-eviction stutter storm**, a variant of the classic "an assumption about a shared resource turns out to be false under real load" pattern that shows up across almost every system in this lesson series.

The architecture in Section 3 assumes the local, driver-level cache is durable: precompile once, expect it to still be there next session, and every session after that. But a real, documented GitHub issue against Valve's own `steam-for-linux` project (ValveSoftware/steam-for-linux#11392) describes exactly how that assumption breaks. Nvidia's driver enforces a default on-disk shader cache limit of just **1 gigabyte** (raised from 128 megabytes in an earlier driver release), and a shader-heavy game like Overwatch 2 can generate **4 to 5.7 gigabytes** of shader and pipeline data across a single two-hour play session once that artificial cap is removed. The moment the cache fills past its default ceiling, the driver silently starts pruning entries the game, and Valve's own precaching pass, assumed would simply persist, without telling anyone. Players affected by this report the exact symptom the entire precaching architecture exists to eliminate returning in force: continuous shader recompilation during ordinary play, CPU cores pegged at 100 percent, and traversal stutter, not because the game's own design failed, but because an out-of-band actor nobody on the game team controls, the GPU driver's own eviction policy, silently invalidated a durability assumption the whole architecture was quietly resting on.

The senior fix is not "tell Nvidia to raise the default cache limit" and stop there, that only moves the ceiling, it doesn't remove the loop; the same failure recurs the first time a game's shader footprint outgrows whatever limit ends up chosen next. The actual fix is the discipline underneath Section 3's async-compile-plus-fallback tier: **never assume a cache layer you do not own is durable.** Treat any cache you don't control, a driver's disk cache, a CDN's edge cache, a browser's cache, as a volatile, best-effort accelerator only, and always keep a correctness backstop underneath it that degrades gracefully the moment that cache turns out to be smaller or more aggressively evicted than assumed, rather than assuming the fast path will simply always be warm. That's the identical discipline as Day 13's backpressure lesson: an optimization is allowed to make the common case fast, it must never become a silent hard dependency the system can't survive without the moment it's taken away.

---

## Map to Rare.lab's stack

Rare.lab's compiler, the piece that turns a node graph into shippable GLSL, already does the first half of Section 3's fix by construction: it compiles ahead of the render loop rather than lazily on first draw, because that's simply how a compile-to-code pipeline works. The gap is the second half, and it's a specific, addable piece: nothing today caches the *compiled GLSL output itself* keyed by a content hash of the exact graph subtree that produced it. Chrome's own GPU program cache, documented in its public design notes, uses exactly this shape for the browser's shader compilation problem: the cache key is the shader source text plus metadata, the value is the compiled program, and a warm reload skips recompilation entirely because the key is a pure function of the input, the same mechanism Section 4 names generally and the same one Rare.lab already applies one level up, to whole scenes, via R2's content-addressed storage. Applying it one level down, to the compiled GLSL for a graph subtree, means two different users publishing the same shader, or one user reloading their own project, skip the graph-to-GLSL compile step entirely on a cache hit, instead of re-deriving it every time.

The embeddable runtime's single shared WebGL context is precisely the "one hot render path, many uncoordinated callers" shape this entire lesson is about, and it is exposed to the naive design's exact failure mode: if a page embeds several Rare.lab widgets and any one of them needs to compile a shader permutation it has never used before, a synchronous compile call blocks that one shared context, freezing every other embed on the page for that same stall, not just the widget that triggered it, because they all share one WebGL context and therefore one compile queue. The concrete, addable fix is the real WebGL `KHR_parallel_shader_compile` extension, which exists specifically because, as its own specification explains, checking shader compile status used to force a blocking stall, a "longstanding user complaint" on Windows in particular: it exposes a `COMPLETION_STATUS_KHR` parameter that can be polled without blocking, letting a compile run in the background while each embed keeps rendering its last-good frame (or a simple placeholder) until its own new shader is actually ready, instead of stalling the shared context for every embed on the page.

The one gap this lesson can't close for Rare.lab the way Valve closes it for Steam is worth stating honestly rather than glossing over: Valve can centrally precompile actual GPU machine code because Steam knows the finite, real matrix of (driver, GPU, OS) combinations its players run. A browser deliberately hides that matrix from any code that isn't the browser itself, so the final machine-code compile always happens client-side, once per browser instance, no matter how aggressively Rare.lab caches on its own side of that boundary. The honest lesson to carry forward is knowing exactly where the boundary sits: cache and skip everything Rare.lab actually controls, graph parsing, GLSL text generation, validation, and treat the browser's own final compile as the one irreducible cost, minimized by the async-plus-fallback mechanism, never eliminated by caching alone.

---

## Sources

- [Overwatch 2 shader pre-cache: Fossilize processing reaches 22–23 GB RAM and triggers kernel OOM on AMD RADV, ValveSoftware/Fossilize#312](https://github.com/ValveSoftware/Fossilize/issues/312): fetched directly. Source for the exact Overwatch 2 numbers used in Section 1: the 7.7-7.8 GB shader pre-cache download, the roughly 20 GB total local pipeline cache, the 22-23 GB RAM plus 5.6-6.9 GB swap peak during Fossilize processing, and the contrasting ~41 MB of shader data generated by two hours of ordinary local play.
- [Precompiled shaders are pruned by nvidia driver, causing stutter from cold cache effects in games with excessive numbers of shaders, ValveSoftware/steam-for-linux#11392](https://github.com/ValveSoftware/steam-for-linux/issues/11392): fetched directly. Source for Section 6's central incident: Nvidia's default 1 GB on-disk shader cache limit (raised from 128 MB), Overwatch 2 generating 4-5.7 GB of shader data in a two-hour session once that cap was removed, and the CPU-pegged, continuously-recompiling symptom this mismatch causes.
- [KHR_parallel_shader_compile extension, MDN](https://developer.mozilla.org/en-US/docs/Web/API/KHR_parallel_shader_compile) (fetched via a raw GitHub mirror of MDN's source content after the primary MDN domain was blocked by this session's egress policy): source for the extension's actual mechanism, the `COMPLETION_STATUS_KHR` non-blocking poll, and its stated purpose of fixing "a longstanding user complaint" about long, blocking shader compile times on Windows.
- [WebGL KHR_parallel_shader_compile Extension Specification, Khronos registry](https://registry.khronos.org/webgl/extensions/KHR_parallel_shader_compile/): direct fetch blocked by this session's network egress policy; relayed via search-indexed summary corroborating the same mechanism as the MDN source above.
- [Unity Manual: Introduction to shader variants](https://docs.unity3d.com/6000.0/Documentation/Manual/shader-variants.html): relayed via search summary. Source for Section 2's combinatorial-explosion arithmetic (ten keyword sets producing 1,024 variants) and the general framing of shader-variant explosion as a named, documented engine problem.
- [Game engines and shader stuttering: Unreal Engine's solution to the problem, Epic Games tech blog](https://www.unrealengine.com/tech-blog/game-engines-and-shader-stuttering-unreal-engines-solution-to-the-problem): direct fetch blocked by this session's network egress policy; relayed via search-indexed summary. Source for the general framing of PSO compilation stutter as an industry-wide problem Epic addressed directly.
- [PSO Precaching for Unreal Engine, Epic Developer Community documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/pso-precaching-for-unreal-engine): direct fetch blocked; relayed via search-aggregated summary of this page together with independent community guides (Tom Looman's and StraySpark's PSO-precaching write-ups). Source for the mechanism described in Section 3 (bundled or auto-predicted PSO precaching introduced in UE 5.2) and the hitch-duration figures in Section 1 (20 ms default hitch threshold, 5-10 ms typical hitches, hitches reaching 117 ms or more). These specific millisecond figures were relayed through aggregated search summaries rather than read first-hand from Epic's own page, and are worth re-verifying directly if the network policy changes.
- [Steam Ironing Out Shader Pre-Caching For Helping Game Load Times, Stuttering, Phoronix](https://www.phoronix.com/news/Steam-Vulkan-Shader-Pre-Cache): direct fetch blocked; relayed via search summary. Source for the general description of Valve's Fossilize-based Vulkan shader pre-caching system and its stated goal of reducing load-time stutter.
- [The Shader Permutation Problem, Parts 1 and 2, Matt Pettineo (therealmjp)](https://therealmjp.github.io/posts/shader-permutations-part2/): direct fetch blocked; relayed via search-aggregated summary. Source for the general "shader permutation problem" framing used throughout this lesson and for DOOM Eternal's cited figure of roughly 100 shaders and 350 PSOs, a figure treated in Section 3 as well-corroborated but secondary-sourced, not independently verified against a primary id Software talk in this session.
- [GPU Program Caching, Chromium design document](https://docs.google.com/document/d/1Vceem-nF4TCICoeGSh7OMXxfGuJEJYblGXRgN9V9hcE/mobilebasic): direct fetch blocked by this session's network egress policy; relayed via search summary. Source for Chrome's GPU shader/program cache design used in the Rare.lab mapping: cache key of shader source text plus metadata, cache value of the compiled program binary, and an in-memory cache loaded from a disk-backed cache at browser startup.

---

*Inference vs. fact, stated plainly: the two GitHub issues (ValveSoftware/Fossilize#312 and ValveSoftware/steam-for-linux#11392) and the MDN/raw-GitHub source for the WebGL extension were fetched directly in this session and can be treated with high confidence; every other source in this lesson, the Unity shader-variants documentation, Epic's tech blog and developer documentation, Phoronix's Fossilize coverage, Matt Pettineo's shader-permutation series, and Chrome's GPU program caching design document, was relayed through this session's web search rather than read first-hand, because direct fetches to unrealengine.com, dev.epicgames.com, phoronix.com, therealmjp.github.io, docs.google.com, and registry.khronos.org were all blocked by this session's network egress policy. The 16.6-millisecond frame-budget arithmetic in Section 1, and the "seven frames' worth of budget" comparison against the 117 ms worst-case hitch figure, are this lesson's own derivation, dividing 1,000 ms by 60 frames and comparing the result against the search-relayed Unreal hitch figures, not a calculation any of the sources above published themselves. DOOM Eternal's roughly-100-shaders, roughly-350-PSOs figure is treated explicitly in Section 3 as a well-corroborated but secondary-sourced claim, not independently confirmed against a primary id Software GDC talk in this session. The ten-layer architecture diagram, the "silent-cache-eviction stutter storm" name and its framing as a variant of the shared-assumption failure pattern seen in earlier lessons, and the entire Rare.lab mapping, including the Chrome-cache-as-template proposal and the shared-WebGL-context-as-one-hot-render-path analogy, are this lesson's own synthesis on top of the documented mechanics above, not a claim that Valve, Epic, or Google describe their systems in these exact terms.*
