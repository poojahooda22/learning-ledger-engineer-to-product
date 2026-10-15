# Daily Viral Tech Report | 2026-10-15

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. I skipped smaller Oct 15 merges (Rust #164332, a 0.8% instruction-count win from an early return; Postgres EPQ pruning fix) to keep the spread wide.

---

## 1. three.js: ClusteredLighting Gains Spot Lights and a WebGL2 Fallback

**Category:** Web Graphics / GPU

**The Technical Why**

PR #34978 (sunag, merged Oct 15 as commit c94d808, milestone r187, related issue #34931) extends `ClusteredLightsNode` to cluster `SpotLight`s that have no shadows and no `map` or `colorNode`; everything else uses default lighting. Cone attenuation is packed into one `vec4` with the direction pre-scaled by `1 / ( penumbraCos - coneCos )`, so point lights run the same branchless path with attenuation fixed at 1. Culling uses the smallest sphere enclosing the cone, not a sphere of radius `distance`. The cluster compute now also runs on WebGL2: each invocation fills `chunksPerInvocation` chunks (one on WebGL, the whole cluster on WebGPU), and the light index buffer is an `instancedArray` with `setPBO( true )` so WebGL computes get a valid `instanceIndex`.

What is hard: clustered shading splits the view frustum into a 3D grid and stores, per cell, which lights touch it, so a fragment loops over a handful of lights instead of all 1,536. The grid must be rebuilt every frame if lights move, and WebGL2 has no compute shaders, so the same logic has to be expressed in a form WebGL can run (inference: via a render-to-buffer path, the page does not say). The author's benchmark (1,024 point + 512 spot lights, RTX 4070, Chrome 147, 3 runs): at 1280x720 the static-grid version ran 753 fps vs 708 fps clustered; at 2560x1440 DPR 2 it was 319 vs 342 fps. The grid rebuild costs about 0.33 ms CPU and 0.14 ms GPU at low resolution. Bundle cost: +136 B gzipped on WebGPU.

**Why It Matters**

Many dynamic lights is the case where forward rendering falls over, and dropping the `light.static` requirement means animated lights work. A WebGL2 path means the same scene runs on browsers and devices without WebGPU. For a shader or effects tool, the lesson is that "roughly on par" performance plus fewer constraints is a real win.

**Go Deeper**

- [three.js PR #34978, ClusteredLighting: Add spot lights and WebGL2 fallback support (primary source)](https://github.com/mrdoob/three.js/pull/34978)
- [three.js releases](https://github.com/mrdoob/three.js/releases)

---

## 2. llama.cpp: Support for Muse Spark Open, an 816B-Parameter Latent-Space MoE (Text, Vision, Audio)

**Category:** AI / ML Infrastructure

**The Technical Why**

PR #30616 (ServeurpersoCom, approved by aldehir, CISC, ggerganov and ngxson, merged Oct 15 as commit ba3c05f) adds the model. The PR calls it a latent space MoE: a residual stream of width 8192, but 256 experts that run in a 4096-wide space, with projections down and up around them; routing is top-8 with sigmoid scoring and a correction bias. Per the GGUF repo name it is 816B total, 42B active. Attention uses three sliding-window layers (2048-token window, RoPE) per one global layer with no RoPE, which the PR names as the source of the 1M context. Audio is a causal convolutional pyramid with no attention and no spectrogram, folding 16, 6, 5 and 4 samples per stage into channels, giving one embedding per 80 ms of 24 kHz audio. The latent projections and attention gate (about 1% of weights) stay at Q8_0 because the author says lower-bit types caused repetitive loops.

What is hard: a new architecture means re-implementing every quirk exactly (norm-preserving residuals with the gate folded in at conversion time, weightless QK-norm, a third output channel `to=commentary` for text between tool calls). The author checked cosine similarity against the transformers reference: 1.000000000 on text, 0.999999926 on the audio tower.

**Why It Matters**

Day-one llama.cpp support is what turns a released checkpoint into something people can run on a Mac Studio or one workstation GPU. It also shows where MoE design is going: cheaper expert compute by shrinking the dimension experts work in. I could not verify the model's publisher from this page, so I make no claim about who released it.

**Go Deeper**

- [llama.cpp PR #30616, model: add Muse Spark Open (primary source)](https://github.com/ggml-org/llama.cpp/pull/30616)
- [GGUF weights: ggml-org/Muse-Spark-Open-816B-A42B-GGUF](https://huggingface.co/ggml-org/Muse-Spark-Open-816B-A42B-GGUF)

---

## 3. PostgreSQL: Foreign-Key Fast Path Stops Calling Cast Functions Directly (Backpatched to 19)

**Category:** Databases / Systems

**The Technical Why**

Commit 6aecabc (amitlan, reported by Nikolay Samokhvalov and Amit Langote, reviewed by Matheus Alcantara, 3 files, +372/-165, backpatch through 19) routes referential-integrity checks to SPI when the FK column type needs a cast function to reach the equality operator's input. The RI fast path in `ri_triggers.c` probes the referenced index directly and skips per-row query setup. But when a cast was needed it called the cast function itself, skipping what a SQL query gets for free: argument and collation handling, NULL results, volatility, replanning. Observed failures: a NULL cast result raised an internal error instead of an FK violation, and replacing a cast function produced "cache lookup failed for function". Now types that are not binary-coercible go through SPI, while cross-type operators, domains and binary-compatible types stay fast. Eligibility is decided in `ri_check_fastpath_index()` and cached in `fastpath_state`, with invalidation now covering `pg_cast`.

What is hard: a fast path is a copy of the slow path's semantics, and every shortcut has to match all of them, including cache invalidation. Cost of getting it wrong shows up as wrong error codes, not wrong speed. Simplest safe rule: take the fast path only for cases you can prove are equivalent.

**Why It Matters**

Anyone with FKs across mismatched types (int to bigint is fine, but custom casts are not) on Postgres 19 gets correct constraint errors again. For engineers, it is a clean case study in the optimization-vs-correctness boundary.

**Go Deeper**

- [postgres commit 6aecabc, Use SPI for RI checks whose FK values need a cast function (primary source)](https://github.com/postgres/postgres/commit/6aecabc)
- [Discussion thread](https://postgr.es/m/CAM527d9BgPjeOOYmbCBTd57R145qHCk-dzw9qNq+nOrDq1j__A@mail.gmail.com)

---

## 4. llama.cpp Vulkan: Copying the Strided f16 KV Cache Once on AMD, Up to 15.8x Faster Prefill at 64k Context

**Category:** GPU / Systems Performance

**The Technical Why**

PR #27703 (Nathanw1014, opened Aug 25, merged Oct 15 by ggerganov as commit 733f57a, approved by 0cc4m, reviewed by jeffbolznv) fixes a memory-channel problem. With an f16 KV cache, consecutive KV rows of one head sit `n_kv_heads x head_dim x 2` bytes apart. When that stride is an even multiple of the 256 B channel-interleave block, reads land on only a subset of memory channels. The fix copies each layer's KV once into a scratch buffer so later reads are sequential. In review the dedicated shader was dropped for the generic copy shader (same speed), written in source order so the strided access falls on the writes, which stream. It is gated to AMD (an RTX 3070 did not benefit) and later disabled on RDNA4 after a regression.

What is hard: this is a hardware-address-mapping effect, invisible in the code and dependent on model shape and GPU. Author numbers on a Radeon 8060S: Qwen3-Coder-30B-A3B pp512 at depth 65536 went from 72.42 to 186.54 t/s, while pp512 at depth 0 was unchanged (1147 vs 1145). Another tester on a Radeon 8065S saw 5.59 to 88.44 (15.82x) at d65536. A third model (Qwen3.6-35B-A3B) showed no gain, and an RX 7900 XTX saw slight regressions at shallow depths (0.9547x at d32768). Cost: scratch space for one layer of KV.

**Why It Matters**

Long-context local inference on AMD iGPUs and consumer cards is where prefill becomes unusable; a layout-aware copy makes 64k+ prompts practical. The transferable point for any GPU runtime: stride and bank/channel conflicts are a performance cliff, and the fix can be a cheap one-time re-layout.

**Go Deeper**

- [llama.cpp PR #27703, vulkan: copy strided f16 KV once on AMD (primary source)](https://github.com/ggml-org/llama.cpp/pull/27703)
- [llama.cpp releases](https://github.com/ggml-org/llama.cpp/releases)

---

## Thread to Watch

Layout and fast-path correctness: a Postgres shortcut that diverged from SQL semantics, a KV cache whose stride starves memory channels, and a lighting grid rebuilt per frame to drop a `static` constraint. Tomorrow, watch whether three.js r187 gets a release date and whether the Vulkan KV scratch path gets tuned for RDNA4.
