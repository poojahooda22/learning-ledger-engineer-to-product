# Daily Viral Tech Report | 2026-10-07

Every story below is checked against a primary GitHub page (PR, commit or release). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Two candidates (PyTorch 2.14.1 and Deno 2.9.7) were dropped as too incremental, and I could not reach postgresql.org docs from this environment, so nothing here relies on them.

---

## 1. llama.cpp: A GPU Cache for MoE Experts That Live in Host RAM Gives 1.6x to 2.2x Decode Speed on One GPU

**Category:** AI / ML Inference

**The Technical Why**

llama.cpp PR #29887 (am17an, approved by Georgi Gerganov, merged Oct 7) adds `llama_moe_cache`. In a Mixture-of-Experts model only a few experts fire per token, but all experts must live somewhere. When the experts sit in host RAM, the new cache keeps hot expert weights in VRAM and runs `MUL_MAT_ID` for those experts on the GPU. Only cache misses are uploaded over PCIe. It uses LRU eviction, applies to batches of 32 tokens or fewer (decode), and is sized with `--moe-cache-mib`. The PR is a port of the qvac-fabric MoE cache. Author numbers on Qwen3.8-Flash-Next Q4_0: RTX 4090 25.0 to 40.7 t/s (77% hit rate), RTX 5090 30.8 to 67.8 t/s (89% hit rate).

What is hard: the win depends entirely on hit rate against PCIe bandwidth. The thread reports an AMD R9700 gaining nothing at 85% hits (11.9 vs 11.8 t/s) and a 2x RTX 5090 setup getting slower (about 13 to 14 t/s vs 18 t/s) because PCIe saturated. It is single-device only; multi-GPU is split into PR #30112. Commenters argued plain LRU churns and that frequency-based eviction would do better (not implemented). My inference: expert routing has skew, so a small cache catches most traffic, but skew varies per model and prompt.

**Why It Matters**

Large MoE models become usable on a single consumer GPU with big system RAM, which is the cheapest way to run them locally. The lesson is generic: a cache turns a bandwidth problem into a hit-rate problem, and you must measure hit rate before trusting it.

**Go Deeper**

- [llama.cpp PR #29887, MoE expert GPU cache (primary source)](https://github.com/ggml-org/llama.cpp/pull/29887)
- [llama.cpp PR #30112, multi-GPU follow-up](https://github.com/ggml-org/llama.cpp/pull/30112)

---

## 2. vLLM v0.31.0: A Weight-Cache Daemon Keeps Quantized Weights in GPU Memory Across Engine Restarts

**Category:** AI / ML Infra

**The Technical Why**

vLLM v0.31.0 (Oct 5; 717 commits from 307 contributors) adds `vllm preload`, a daemon that "keeps post-quantized weights resident in GPU memory across engine restarts" (#56680). v0.30.0 earlier added a "Fast Start" daemon that loads weights over CUDA IPC instead of disk. The new release also lists experimental `vllm snapshot create/restore` (#51360), which uses CRIU to restore an initialized TP1 engine. Other items: FlashMLA with the DeepSeek-V4.1 NVFP4 compressed KV cache becomes the SM100 default (#56935), and prefill context parallelism now works with data parallelism (#57075). The release page gives no benchmark numbers for these.

What is hard (inference, the page does not explain it): the expensive part of a restart is loading and quantizing weights, so you move weight ownership out of the engine process into a long-lived one. That means a second process owning GPU memory, handing out handles via CUDA IPC, and handling its lifetime, health (#58552) and readiness (#58370). Also note a security change: per-request multimodal kwargs are now gated behind `--trust-request-mm-kwargs`.

**Why It Matters**

Restart time is downtime in autoscaling and crash recovery for big models. Treating weights as a separate long-lived service is the same pattern as separating a database from the app tier. Winners are teams running large MoE models in production.

**Go Deeper**

- [vLLM v0.31.0 release notes (primary source)](https://github.com/vllm-project/vllm/releases/tag/v0.31.0)
- [vLLM releases page, v0.30.0 Fast Start context](https://github.com/vllm-project/vllm/releases)

---

## 3. three.js: MaterialXLoader Gets Pluggable Node Resolvers and Spec-Correct Noise, Funded by Needle

**Category:** Web Graphics / Shaders

**The Technical Why**

On Oct 7 three.js merged two MaterialXLoader PRs by hybridherbst (reviewed and merged by Mugen87, milestone r187). #34832 lets an app supply a resolver that builds the TSL node for a MaterialX node before the loader's built-in library is consulted; it returns a node or `null` to fall back. Later commits turned it into a parse option, `options.nodeResolver`, alongside `interfaceValidator` and `uvSpace`. #34833 fixes a correctness bug: vector and color noise "repeated the scalar noise in every channel". Now vector2, vector4 and color4 noise sample real vector noise, following MaterialX's GLSL reference. The fourth channel of vector4 comes from scalar noise at an offset, and `mx_noise_vec4()`'s 3D offset changed from (19, 73, 0) to (19, 73, 29) to match. Cost: about 14 bytes gzipped.

What is hard: MaterialX is a node-graph interchange format, and three.js maps it onto TSL, which compiles to WGSL or GLSL. Matching a reference implementation per channel is subtle because a visual match depends on exact hash constants and offsets (the PR's offset fix shows this).

**Why It Matters**

A node graph authored in one tool now renders the same in a browser, and apps can plug in custom nodes without forking the loader. For any node-based shader editor, this is the interop and conformance story: a spec plus a reference implementation beats screenshots.

**Go Deeper**

- [three.js PR #34832, node resolver (primary source)](https://github.com/mrdoob/three.js/pull/34832)
- [three.js PR #34833, per-channel vector noise](https://github.com/mrdoob/three.js/pull/34833)

---

## 4. PostgreSQL 19: Concurrent REPACK Now Strips Dropped-Column Values From Every Replayed Tuple

**Category:** Databases / Systems

**The Technical Why**

On Oct 7 Álvaro Herrera committed e40b509 (author Shihao Zhong, backpatch through 19), "Concurrent REPACK: Clear out dropped-column values from all tuples". `REPACK (CONCURRENTLY)` rewrites a table into a new heap while writes continue, then replays the changes made meanwhile (the commit's own wording is "replayed tuples"; that it uses logical decoding is my inference). An earlier commit (e5d2595) nulled dropped-column values only for UPDATE new tuples. But an INSERT can also carry dropped-column data, for example when a BEFORE INSERT trigger returns a copy of an existing row. Those values survived, wasting space and possibly failing with "row is too big" if the new heap has no TOAST table. The fix moves the clearing into `restore_tuple()` so it covers all replayed tuples. It adds an injection-point isolation test (`repack_dropped.spec`) with a new insert permutation.

What is hard: online table rewrite has to be correct for every kind of concurrent change, and each change type is a separate path. This bug is a missed path, and it was found by a test that forces a specific interleaving.

**Why It Matters**

REPACK is the in-core path toward pg_repack-style bloat removal without long exclusive locks. Reading these fixes shows what "online" really costs: every write path must be replayed correctly. Watch the 19 release notes.

**Go Deeper**

- [Postgres commit e40b509 (primary source)](https://github.com/postgres/postgres/commit/e40b509c08f34e245bfab4124e24297ddaf4d535)
- [Postgres master commit log](https://github.com/postgres/postgres/commits/master)

---

## Thread to Watch

Caches and daemons that decouple expensive state from the process that uses it: llama.cpp's expert cache, vLLM's weight daemon, Postgres's concurrent rewrite. Tomorrow, watch llama.cpp PR #30112 (multi-GPU MoE cache) and whether anyone lands a frequency-based eviction policy.
