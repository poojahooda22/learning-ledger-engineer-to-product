# Daily Viral Tech Report | 2026-10-20

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. Skipped as thin: three.js #35121 (analytic uv gradients on TextureNode; I did not read the page) and vLLM #61718 (Mamba2 chunk scan NaN on sm90; merged Oct 17, outside the window).

---

## 1. three.js: LightProbeGrid Bakes Get About 4x Faster by Capturing Eight Probes per Render With One 48-Camera ArrayCamera

**Category:** Web Graphics / GPU

**The Technical Why**

PR #34909 (author bhouston, merged Oct 20 by Mugen87 as c396f20, milestone r187) changes how the WebGPU `LightProbeGrid` bakes. Before, each probe cost six `CubeCamera` face renders plus one spherical harmonics (SH) projection pass: seven passes and seven queue submits per probe. Now one `ArrayCamera` with 48 sub-cameras (six faces times eight probes) draws into a 2D face atlas, and one pass projects the whole batch to SH. The projection no longer samples 512 Fibonacci directions. It integrates every face texel, weighted by its exact solid angle.

What is hard: the author profiled first and found the bake was CPU-bound on submits and per-object render setup, not GPU-bound. Cutting SH samples from 512 to 64 changed nothing. So the win comes from fewer submits, not faster shading. Batching also leaks per-camera state. Mugen87 found two regressions that `CubeCamera` handled correctly: `RectAreaLightNode` took its orientation from the parent camera (identity view matrix at the origin), and `ClippingNode` tested planes in each face's view space after they were transformed with the parent camera. The author fixed both separately (#35095, #35096, both merged), arguing they were latent ArrayCamera bugs. Author's numbers on `webgpu_lightprobes_complex` (cubemap 32, NVIDIA Pascal, Chrome): 790 ms to 188 ms (4.2x) at resolution 8, 5564 ms to 1238 ms (4.5x) at resolution 12 with one bounce. Max SH error against a 16k-sample reference fell from about 2.4% to 0.08%. Cost: the `sampleCount` option is removed, and WebGPU grows about 71 B gzipped.

**Why It Matters**

Baked global illumination is how web scenes get believable indirect light without a per-frame cost. A bake that drops from 8.6 s to 3.1 s (the author's Sponza example, which also changed to 16 probes per frame) makes live re-baking while editing plausible. The general lesson for a shader or scene editor: when a loop is CPU-bound on submits, batch the work into one pass and expect per-camera state to be the thing that breaks. Caveats on the page: one GPU and browser only, no per-sub-camera culling yet, and light leaking through walls remains.

**Go Deeper**

- [three.js PR #34909, LightProbeGrid: Batch probe capture with ArrayCamera (primary source, includes the review thread)](https://github.com/mrdoob/three.js/pull/34909)
- [three.js repository](https://github.com/mrdoob/three.js)

---

## 2. llama.cpp: A Realtime Transcription Server Lands Over an OpenAI-Compatible /v1/realtime WebSocket

**Category:** AI / ML Inference

**The Technical Why**

PR #30636 (author ngxson, merged Oct 20 by the author as 1242be6, approved by ServeurpersoCom) adds a `/v1/realtime` WebSocket endpoint to llama-server, shaped like OpenAI's Realtime API. Clients stream 16-bit PCM with `input_audio_buffer.append` events. Each connection gets a reader thread, and writes go through a mutex-guarded send. A session occupies a slot, audio is pushed into a stream in the multimodal library, and the slot decodes in steps (the first emits `1 + 32 + n_delay` rows). Support is gated per model; the test model was Voxtral-Mini-4B-Realtime-2602 as GGUF. The WebSocket support in the vendored HTTP library already existed, so no new dependency.

What is hard: a transcription session is long-lived and incremental, but the server's slot scheduler was built for request-response. Human review on the PR found real scheduling and security bugs. A slot waiting for more audio sat in the generating state and blocked other slots from batching (fixed in c135f02). Early appends were lost because `session.created` went out before the slot existed (fixed in 4e8679a). In router mode with `--api-key`, the WebSocket was unauthenticated (fixed in 3e6145a). An automated review also listed a possible use-after-free with `--sleep-idle`, an abort on a too-small batch, an unbounded audio backlog and no tests. The page does not show those four being fixed; the approval says "Everything above is fixed" but the listed commits do not obviously cover them (open question, not a confirmed bug). Audio is handled in 80 ms units. A reviewer reported a clean transcript "way faster than real time" on an RTX PRO 6000, with no figure.

**Why It Matters**

It lets a local, self-hosted model serve the same client protocol as a hosted realtime API, so an app can swap backends by changing a URL. The lesson for anyone adding streaming to a stateless server: the first bugs will be in slot lifecycle and auth on the new path, not in the model. Deferred to a follow-up: 24 kHz input with server-side resampling (OpenAI expects 24 kHz, this server takes the model's rate).

**Go Deeper**

- [llama.cpp PR #30636, server: add realtime transcription API (primary source, includes the review findings)](https://github.com/ggml-org/llama.cpp/pull/30636)
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

---

## 3. vLLM: Cooperative TopK Now Falls Back to Exact FP32 Selection When the Threshold Bin Overflows

**Category:** AI Serving / GPU Kernels

**The Technical Why**

PR #59645 (author LopezCastroRoberto, merged Oct 20 by LucasWilkinson as 612e928, approved by simon-veitner-redhat) fixes an inexact top-k. The cooperative TopK kernel bins FP32 scores into coarse buckets (a radix-histogram select), then gathers candidates from the bucket that contains the threshold. That bucket has a bounded candidate buffer. If more scores land in it than fit, some scores that belong in the result are dropped. The fix detects the overflow before scattering candidates and reruns only that row with exact FP32 selection. Rows that fit take the old fast path unchanged.

What is hard: the error is silent and data-dependent. The author argues the dropped and kept scores share a radix prefix, so their values are close, and overflow is rare. That is a claim, not a measurement: the page says overflow frequency in real workloads was not measured, and the accuracy impact is unknown. Cost on B300 microbenchmarks (250K width, 64 rows, k=2048): about 40 us at 2,048 candidates in the threshold bin, about 54 us at 2,049 (entering recovery), about 75 us when the whole row is concentrated in [1.0, 1.01]. End to end on DeepSeek-V4.1-Flash (tp 4, fp8 KV cache, topk=512), time per output token moved 0% to +1% against main. Open items: rows where every value in the overflow bin has the same key still enter recovery, roughly 1.5 to 2x slower on those rows, and a min == max check could skip it; the `persistent_topk` path is not covered; several test gaps remain.

**Why It Matters**

Top-k over long contexts picks which tokens a sparse-attention indexer attends to, so an approximate pick changes model output without any error. Making it exact for about 1% latency trades a speed shortcut for a correctness guarantee. The pattern to keep: a fast path with a bounded buffer needs an overflow detector and an exact slow path, and the slow path must be measured on the adversarial distribution, not the average one.

**Go Deeper**

- [vLLM PR #59645, Make cooperative TopK exact on candidate overflow (primary source, includes the benchmark tables and review)](https://github.com/vllm-project/vllm/pull/59645)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 4. PostgreSQL: EXPLAIN Printed the Wrong OLD/NEW Aliases for a RETURNING SubPlan (Backpatched to 18)

**Category:** Databases / Query Planning

**The Technical Why**

Commit 1999fe2 (patch by Henson Choi, reviewed by Tom Lane, committed by tglsfdc, 3 files, +107/-11, backpatched through 18) fixes `EXPLAIN` output in `ruleutils.c`. PostgreSQL 18 lets `RETURNING` refer to `OLD` and `NEW`, and a statement can rename them with aliases. `set_deparse_context_plan()` set those aliases only when EXPLAIN started printing a ModifyTable. But `push_ancestor_plan()` can jump to an ancestor ModifyTable to print a SubPlan's Param inside its RETURNING list, without refreshing the aliases. So the aliases from a previously printed ModifyTable, such as a data-modifying CTE, could be used. Also, `get_variable()` accepts a NULL refname silently, so an alias could vanish from the output.

What is hard: this is state that belongs to a plan node but was stored on the traversal context, so it went stale whenever the walk moved sideways in the tree. The fix moves the assignment into `set_deparse_plan()`, which runs on every plan-node change, sets both aliases to NULL for non-ModifyTable nodes, and adds assertions in `get_variable()` so a missing alias fails loudly. The new regression tests use `EXPLAIN (VERBOSE, COSTS OFF)` with a data-modifying CTE that sets different aliases than the outer statement. The commit describes an EXPLAIN display bug; it does not say query results were wrong (inference: execution does not use these names).

**Why It Matters**

EXPLAIN is what engineers use to understand a plan, so output that names the wrong alias sends debugging in the wrong direction. It only affects PostgreSQL 18 and later, since OLD/NEW in RETURNING is new there. The pattern worth keeping: keep per-node state with the node, not with the cursor that walks the tree, and turn "silently NULL" into an assertion.

**Go Deeper**

- [postgres commit 1999fe2, Fix OLD/NEW aliases that EXPLAIN prints in RETURNING SubPlans (primary source)](https://github.com/postgres/postgres/commit/1999fe2)
- [Mailing list discussion linked from the commit](https://postgr.es/m/CAAAe_zAUb_XydapJqUk7O8o2crJa7xbxX=KvSi3nY9h98OcN7g@mail.gmail.com)

---

## Thread to Watch

Fast paths with a bounded buffer or shared state and a slow exact path behind them: vLLM's TopK overflow (#59645), three.js ArrayCamera per-camera state (#34909) and llama.cpp slot lifecycle (#30636). Tomorrow, watch for the vLLM follow-up covering `persistent_topk` and the min == max skip, and whether llama.cpp lands tests and the 24 kHz resampling for `/v1/realtime`.
