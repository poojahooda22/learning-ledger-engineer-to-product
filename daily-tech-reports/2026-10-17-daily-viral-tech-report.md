# Daily Viral Tech Report | 2026-10-17

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. I could not open Meta's launch post, The Register, or other non-GitHub sites from this environment (the network policy blocks them), so I dropped the reported Postgres 19 release delay rather than repeat a story I could not read. Skipped as thin: SGLang #44668 (Helion megakernels; the page has no description, no benchmarks, and CI is red).

---

## 1. llama.cpp: Muse Spark Open (816B MoE, 42B Active) Gets llama.cpp Support, but Only if the Routing Latents and Gate Stay at Q8_0

**Category:** AI / ML Infrastructure

**The Technical Why**

PR #30616 (author ServeurpersoCom, merged Oct 15 as ba3c05f, approved by ggerganov, CISC, aldehir and ngxson) adds Meta's Muse Spark Open for text, vision and audio. Per the PR, it is a latent-space MoE: the residual stream is 8192 wide, 256 experts run at width 4096, and a projection goes down and back up around the experts, with top-8 sigmoid routing and a correction bias. Attention mixes three sliding-window layers (2048-token window, RoPE) with one global layer without RoPE, which the author names as where the 1M context comes from. Audio is a causal convolutional pyramid with no attention and no spectrogram: four stages fold 16, 6, 5 and 4 samples into channels, so one embedding covers 80 ms of 24 kHz audio.

What is hard: quantization. The author keeps the latent projections and the attention gate at Q8_0 ("a little over 1%" of the weights) because low-bit types there produced a repeating loop. The rest goes to Q4_K_S or an imatrix Q2_K. Parity with the transformers reference is cosine 1.000000000 on text and 0.999999926 on audio. After merge, a CPU-only `muse-spark-moe` test missed its 1.0e-05 NMSE threshold (1.17e-05 and 1.15e-05); the author suspects top-8 routing flipping an expert on a near tie (the author's guess, not confirmed), and #30674 changed the random test model's tensor fill.

**Why It Matters**

Open weights only matter once runtimes can load them. This one landed in llama.cpp within the same window as SGLang's day-0 PR (#44530) and its cookbook, so people can run it on a Mac Studio M3 Ultra (the author's test machine) as well as on GPU servers. The lesson for anyone shipping a quantized model: a tiny fraction of tensors (routing, gates, latent projections) carries disproportionate risk, so quantize by sensitivity, not uniformly.

**Go Deeper**

- [llama.cpp PR #30616, model: add Muse Spark Open (primary source)](https://github.com/ggml-org/llama.cpp/pull/30616)
- [SGLang PR #44687, Muse Spark Open SWA ratio calculator (`min(1, 2048 / L)`; a 100,000-token request gives `--swa-full-tokens-ratio 0.02048`)](https://github.com/sgl-project/sglang/pull/44687)
- [Meta launch post on the PyTorch blog (found by search; I could not open it, so no claim above rests on it)](https://pytorch.org/blog/muse-spark-open/)

---

## 2. vLLM: Streaming Qwen3 Tool Calls No Longer Re-Parse the Whole Argument per Token (12 KB Call: 2.1 s to 11 ms)

**Category:** AI Serving / Algorithms

**The Technical Why**

PR #62027 (author deadstrobe5, merged Oct 17 by sfeng33 as 5c1c903) fixes quadratic work in the streaming tool-call parser. Each token delta re-ran `_qwen3_arg_converter` and `_safe_arg_prefix`, a per-character Python loop over the full JSON so far, on the API server. The fix reuses the `arg_value_extender` hook that GLM (#61253) and Gemma4 (#61494) already had: append the new delta to the open value instead of reconverting everything. It covers qwen3_coder, qwen3_xml, Seed-OSS, Step3p5, Nemotron-v3, Granite thinking and MiniCPM-V.

What is hard: the extender must know when to stop. It returns None when a delta completes `</parameter>` or the next `<parameter=`, even if the tag is split across deltas or has whitespace variants, so the old converter takes over at the boundary. It also refuses repeated keys, since a repeated key keeps its first position in the JSON. The author checked equivalence with a differential fuzz: 27,500 random calls streamed with and without the extender, about 194K deltas, every `DeltaMessage` identical. Numbers (Apple M5, Python 3.12, 7 alternating runs, parser only): a 12 KB `write_file` call (2,788 tokens) goes from 2.1 s to 11 ms; 25 KB from 8.4 s to 22 ms. No GPU end-to-end run.

**Why It Matters**

Coding agents stream whole files through tool calls, and the CPU parser, not the GPU, was the bottleneck on the API server for large arguments. The fix turns O(n^2) into O(n). Inference serving stacks win on the unglamorous Python between the GPU and the socket, and an incremental parser with a fuzz test against the batch parser is the pattern to copy.

**Go Deeper**

- [vLLM PR #62027, Stream Qwen3 tool argument values incrementally (primary source)](https://github.com/vllm-project/vllm/pull/62027)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 3. three.js: Occlusion Query Results Are Now Kept Per Camera (Same Root Cause as Yesterday's Build-Cache Fix)

**Category:** Web Graphics / GPU

**The Technical Why**

PR #35066 (author arodic, finished by Mugen87, merged Oct 17 as c46746f, milestone r187, fixes #35065) fixes `isOccluded()` returning another camera's result. Since #32546, render contexts are keyed by attachment state, MRT setup and call depth, not by scene and camera. Two cameras rendering one scene into the same target (or targets of the same format) therefore shared a context, and their occlusion queries overwrote each other. The fix adds `Backend.getOcclusionData()`, which keeps data in a `WeakMap` keyed by `renderContext.camera`, used by both the WebGPU and WebGL backends. Draw calls look it up only when the context has occlusion queries.

What is hard: a GPU occlusion query is asynchronous, so its result arrives frames later and must be filed under whoever issued it. Yesterday's #35014 removed `context.id` from the material cache key to share builds; this PR shows the cost of sharing a context: any mutable state that was implicitly per-context must now be keyed explicitly. A `WeakMap` on the camera lets the data die with the camera. The author's headless Chromium test (a sphere behind a plane, a front camera that cannot see it, a side camera that can) fails on `dev` for split screen, same-format render targets, and a split screen plus an overlay scene, and passes with the PR. Cost: +396 B minified, +94 B gzipped on the WebGPU full build; core and WebGL unchanged.

**Why It Matters**

Multi-camera setups (split screen, minimap, mirrors, XR eyes) are exactly where a shared-context optimization bites. For anyone building a renderer: when you dedupe state to save compile time, list which of that state is mutable and per-viewpoint, and key it explicitly.

**Go Deeper**

- [three.js PR #35066, Backend: Keep occlusion query results per camera (primary source)](https://github.com/mrdoob/three.js/pull/35066)
- [Yesterday's related change, PR #35014, shared node builds across call depths](https://github.com/mrdoob/three.js/pull/35014)

---

## 4. PostgreSQL: An Assertion Failure in nbtree Row Comparisons That Can Never Match (Backpatched to 18)

**Category:** Databases / Query Execution

**The Technical Why**

Commit 45fc51f (Peter Geoghegan, 1 file, +5/-6 in `src/backend/access/nbtree/nbtpreprocesskeys.c`, backpatch through 18) fixes a spurious assertion in `_bt_mark_scankey_required`. A row comparison such as `(a, b) > (1, NULL)` is unsatisfiable when its first member is NULL, so the index scan can return nothing. The assertion expects the first member key to share the header key's strategy. But `_bt_fix_scankey_strategy` returned `false` early on a NULL first member, before fixing (commuting) member strategies, so the assertion failed in the single-key fast path. The fix computes `satisfiable = !(subkey->sk_flags & SK_ISNULL)`, runs the member loop always, and returns `satisfiable` at the end. The commit message calls it an oversight in ec986020, which improved unsatisfiable RowCompare detection.

What is hard: detecting "this scan returns nothing" early is an optimization, but every early exit skips the normalization that later code assumes already happened. The invariant (header and members agree) must hold on every path, including the "empty result" path. Inference on impact: assertions are compiled out of normal builds, so this most likely affects assert-enabled builds and testing, not production results; the commit page does not state user-visible impact.

**Why It Matters**

Row comparisons power keyset pagination (`WHERE (created_at, id) > (...)`), which is the standard way to page through large tables without OFFSET. Postgres fixes in this area show how much preprocessing a B-tree scan does before it touches a page. Lesson for any engineer writing early returns: normalize first, then decide, or move the invariant checks next to the early exit.

**Go Deeper**

- [postgres commit 45fc51f, Fix nbtree unsatisfiable row compare assertion failure (primary source)](https://github.com/postgres/postgres/commit/45fc51f)
- [postgres commit history, master](https://github.com/postgres/postgres/commits/master)

---

## Thread to Watch

Shared state hiding per-owner state: a shared three.js render context that mixed up two cameras' occlusion results, and a shared Python re-parse loop that made every streamed token cost O(everything so far). Tomorrow, watch whether SGLang's five-part Muse Spark stack (#44664 to #44668) and the Helion verify megakernels (#44798) get benchmarks, and whether three.js r187 gets a date.
