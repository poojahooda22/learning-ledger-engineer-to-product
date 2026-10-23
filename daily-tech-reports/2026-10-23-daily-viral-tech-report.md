# Daily Viral Tech Report | 2026-10-23

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. Skipped: vLLM #56430 (P/D disaggregation KV leak under fault tolerance; solid, but the page reports no performance numbers and I kept the AI slot for the batch invariance fix) and llama.cpp #28725 (SYCL graph record and replay; I did not read the page).

---

## 1. vLLM: Online FP8 MoE Experts Were Not Batch-Invariant Because One Dynamic Scale Covered the Whole Batch

**Category:** AI Inference / Numerics

**The Technical Why**

PR #57092 (author ybwbqg9379, merged Oct 23 by yewentao256 as 6d30b7a, fixes issue #57016) makes MoE models served with online FP8 give the same output for a request no matter which other requests share its batch, when `VLLM_BATCH_INVARIANT=1`. Dense layers were already invariant; the experts were not. Two causes, per the author. First, online per-tensor FP8 used one dynamic scale over the whole activation tensor, so a request's quantization scale depended on every co-batched token. Second, the Triton MoE launcher passed a zero expert stride for 1-D `[E]` weight scales and mishandled the singleton dimension of `[E, 1, 1]` scales, so once per-token activations were used the kernel read the wrong scales. The fix switches to per-token activation scales under the flag, passes the expert stride unconditionally and broadcasts singleton scale dimensions. A review comment found the same stride bug in the batched Triton launcher, and a follow-up commit (b32f222) fixed it with a regression test.

What is hard: the bug is numerical, not logical. Nothing crashes. A scale computed across the batch is a hidden coupling between unrelated users (my inference on the general mechanism; the page states the two causes above). Evidence from the page: on an RTX 5090, request A alone versus A co-batched with B flipped its first token at position 11 with a logprob gap of 0.166 on main, and was bitwise identical on the PR. The new e2e test compares logprobs bitwise between BS=1 and BS=8; main failed it with a max abs diff of 2.223e-01. Independent H20 run by CarrotSwordsman: the clean 0.28.0+cu129 wheel failed with 2.8e-01 and passed with the PR files. Cost: gsm8k 5-shot (granite-3.1-1b-a400m-instruct) with the flag on went from 0.3275 / 0.3260 on main to 0.3192 / 0.3146 on the PR (flexible-extract / strict-match). The author's McNemar test gave p = 0.47 and p = 0.31, so the differences were treated as noise; 1,187 of 1,319 generations changed, which is expected.

**Why It Matters**

Batch invariance is what makes RL rollouts reproducible and lets you debug "why did this prompt answer differently at 3pm" without the load on the server being a variable. FP8 is the default way to fit large MoE models, so an invariance gap there blocks the feature for exactly the models people run. Open gap: per-channel weights with per-tensor activations (issue #58532) is still unfixed.

**Go Deeper**

- [vLLM PR #57092, Make online FP8 MoE experts batch-invariant under VLLM_BATCH_INVARIANT (primary source, includes repro, e2e test and lm_eval tables)](https://github.com/vllm-project/vllm/pull/57092)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 2. llama.cpp: CUDA Graphs Are Now Captured in Chunks of 512 Nodes to Amortize Launch Cost

**Category:** AI Inference / GPU Kernels

**The Technical Why**

PR #30804 (author am17an, idea credited to ORippler, merged Oct 23 as 280141e; approved by ORippler, aendk and IMbackK) splits the CUDA graph for a decode step into chunks. Each chunk is captured and launched as its own instance, and the code now stores a vector of instances. Chunk size comes from the `GGML_CUDA_GRAPH_CHUNK_SIZE` environment variable and defaults to 512. The PR overview states that CUDA graph cost grows with graph size and can be amortized by running smaller chunks. Why this works is not spelled out on the page; my inference is that graph instantiate and update cost is superlinear in node count, so several small graphs are cheaper than one large one.

What is hard: the win is small and the tuning is blind. The author reports "a 1-2% TG speed-up" from a sweep of 128, 256, 512 and 1024. ORippler's RTX Spark run: average prediction speed 40.564 to 40.744 t/s, average latency 9.9381 s to 9.9047 s (0.34%). Multi-GPU showed no gain, which ORippler expected because split subgraphs already pipeline across GPUs. HIP on CDNA was performance neutral (IMbackK). Reviewers asked whether the 512 default holds across hardware and OSes, and asked for a test that kernel fusion is respected inside chunks; the author said there is no CUDA-graph regression suite to add one to. After merge, CISC reported a CUDA build failure ("class 'ggml_cuda_graph' has no member 'instance'") and EditorConfig failures; PR #31053 by sreeram-11 fixed the build the same day.

**Why It Matters**

At batch size 1, decode on a fast GPU is bounded by how quickly the CPU can feed work, so launch overhead is a direct tokens-per-second tax on every local-inference user. A 1-2% gain is modest, but it is free of model changes and applies to every NVIDIA user of the project. The pattern: when a fixed cost scales with graph size, bound the graph.

**Go Deeper**

- [llama.cpp PR #30804, CUDA: chunk graphs to amortize graph launch (primary source, includes the RTX Spark table and review thread)](https://github.com/ggml-org/llama.cpp/pull/30804)
- [llama.cpp PR #31053, the same-day build fix](https://github.com/ggml-org/llama.cpp/pull/31053)

---

## 3. three.js: `LightMapper` Bakes Diffuse Light Into an Atlas From Any World-Space Irradiance Function

**Category:** Web Graphics / GPU

**The Technical Why**

PR #34991 (author bhouston, opened Oct 13, merged Oct 23 by Mugen87 as d19352c, milestone r187) adds `LightMapper`, a progressive diffuse light baker for `WebGPURenderer`. It takes a function `(position, normal) => irradiance`, so it works with any diffuse light calculator. The PR ships two: `lightProbeGridIrradiance()` and `vxgiIrradiance()`. Usage from the example: `unwrapUVs(scene, { resolution: 1024, padding: 3 })` builds a UV atlas (from the earlier PR #34944), `new LightMapper(renderer, atlas.resolution, irradiance)` is created, `baker.update(64)` accumulates samples, and the result is assigned as `mesh.material.lightMap`. Features listed: progressive accumulation, UV jitter, chart padding.

What is hard: baking is the inverse of rendering. You shade every texel of an atlas, not every pixel of a screen, so UV jitter and chart padding are needed to avoid seams and to converge across samples. The page does not describe the internal algorithm beyond that (so the explanation above is the PR's own listing, not a source read). The real quality limit is the input: reviewers found VXGI light bleeding depends on voxel resolution and normal offset. Mugen87 recommended 128 or 256; bhouston set 256. Follow-ups merged the same day or next: #35160 (VXGI bleeding mitigation), #35183, #35188. Validation: 1,973 core and 140 addon tests, zero failures, on a single NVIDIA GTX 1050 (VXGI completed 64 samples, Light Probe 16). Bundle cost: +39 B gzipped in WebGPU. A commenter, zalo, warned that BVH build cost "becomes catastrophic in some scenes" for a ray-based radiosity alternative (draft #35124).

**Why It Matters**

A baked light map costs one texture sample at runtime, versus per-frame global illumination. For web scenes aimed at phones and low-end laptops, moving lighting cost from every frame to build time is the standard way to get quality at a fixed frame budget. For anyone shipping a node-based visual tool, the design choice to separate "baker" from "irradiance source" is the part to copy.

**Go Deeper**

- [three.js PR #34991, LightMapper: Diffuse light baker, works with Light Probe and VXGI (primary source, includes the review thread)](https://github.com/mrdoob/three.js/pull/34991)
- [three.js repository](https://github.com/mrdoob/three.js)

---

## 4. PostgreSQL: Hash Join Emits Null-Keyed Outer Tuples Immediately Again, Removing a v18 Spill Regression (Backpatched Through 19)

**Category:** Databases / Query Execution

**The Technical Why**

Commit 126976a (author shihao zhong, co-authored by Tom Lane, committed by tglsfdc, 3 files, +47/-29, backpatched through 19) changes `nodeHashjoin.c`. Per the commit message, commit 1811f1a put outer tuples with null join keys into a tuplestore and emitted them at the end of the batch, "which risks expensive spills to disk if there are a lot of them." A null key can never match under `=`, so for outer joins these rows only need to be output with null-extended columns. The new code emits them immediately in the non-parallel and single-batch cases, "at the tiny cost of propagating the 'is-null' result flag one call level further up." The commit message says this "avoids a performance regression from v18 for such cases."

What is hard: in a multi-batch parallel join the commit message says it is "difficult to avoid the tuplestore," so that path still buffers. Why it is hard is not on the page; my inference is that parallel workers share batch state, so emitting a row early would break the ordering each worker assumes. The page gives no benchmark numbers and no listed reviewers, so how large the regression was is unknown from this source.

**Why It Matters**

Left joins on columns with many NULLs (optional foreign keys, sparse attributes) are common in real schemas. A buffering strategy that is fine at 100 rows spills to disk at 100 million. The lesson is about the fast path: when a row's fate is already known, do not carry it through a buffer built for rows whose fate is not.

**Go Deeper**

- [postgres commit 126976a, Emit null-keyed hash join outer tuples immediately when feasible (primary source)](https://github.com/postgres/postgres/commit/126976a)
- [Mailing list discussion linked from the commit](https://postgr.es/m/CAGRkXqTgugmYrtnP_11NjL_GCXwAfzVfm5WKUNFsfZOzxDF2rA@mail.gmail.com)

---

## Thread to Watch

Fast paths that quietly change behavior or cost: vLLM's batch-coupled scale (#57092) and Postgres's null-key buffering (126976a) are both shortcuts that looked free until measured. Tomorrow, watch whether llama.cpp gets a CUDA-graph regression test after the #30804 build break, and whether vLLM closes the per-channel weight gap in issue #58532.
