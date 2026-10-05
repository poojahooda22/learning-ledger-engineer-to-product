# Daily Viral Tech Report | 2026-10-05

Every story below is checked against a primary GitHub page (release, PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. I found no verifiable business or platform story from the last day, so all four are engineering stories.

---

## 1. vLLM v0.31.0: A Balanced All2All Backend for MoE Serving, and a Weight Cache That Survives Restarts

**Category:** AI / ML Infrastructure

**The Technical Why**

vLLM v0.31.0 (Oct 5, 717 commits from 307 contributors) adds `--all2all-backend moonep` (PR #52101). In a mixture-of-experts model the router sends each token to a few experts spread over GPUs, so a skewed router overloads some ranks. MoonEP keeps exactly S x K tokens per expert rank by planning a few redundant experts online and prefetching their weights before expert compute. Dispatch returns tokens already grouped per expert, so the experts run as grouped GEMMs over `cu_seqlens` segments. The PR is a proof of concept: BF16 only, eager mode (no CUDA graphs yet), and replicated expert weights per rank. It reports Qwen3-30B-A3B QPS going from 4.0 to 6.7 with gsm8k accuracy 0.8923 versus 0.8870 baseline. It also reports only 10 of 16 token-identical generations on OLMoE-1B-7B, so output is not bit-equal to the baseline.

The release also ships `vllm preload` (PR #56680), a daemon that keeps post-quantized, tensor-parallel-sharded weights resident in GPU memory across engine restarts, and FlashMLA attention with an NVFP4 compressed KV cache as the SM100 default (PR #56935).

Why it is hard: expert load is data-dependent, so you cannot balance it statically. Redundant experts cost memory and a weight prefetch that must hide behind compute (my inference on the overlap goal; the PR describes the prefetch, not its hiding).

**Why It Matters**

MoE decode throughput is often set by the slowest rank, so removing skew raises throughput without new hardware. The weight cache attacks restart time, which matters for autoscaling and rolling deploys of large quantized models. Both are early: read the PR's own caveats before planning around them.

**Go Deeper**

- [vLLM v0.31.0 release notes (primary source)](https://github.com/vllm-project/vllm/releases/tag/v0.31.0)
- [PR #52101, MoonEP balanced all2all backend](https://github.com/vllm-project/vllm/pull/52101)

---

## 2. llama.cpp: Head-Parallel Flash Attention on Qualcomm Hexagon, +58% Prompt Throughput on a Phone

**Category:** AI / ML on the Edge

**The Technical Why**

llama.cpp PR #29974 (merged Oct 5, release b11430) changes how the Hexagon DSP backend splits flash attention across cores. Before, each core could read the entire KV cache. Now, when the KV head count divides evenly by the core count N, core i handles heads [i*n_kv_heads/N, (i+1)*n_kv_heads/N) and reads only its shard of the KV cache. The PR also gates the HMX matrix unit better for long contexts, spreads MoE expert work across devices, and adds F16 activation support. On a Galaxy S24 Ultra (Hexagon v75), Qwen3-0.6B prompt processing went from 6,977 to 11,026 tokens/s (+58%), and Qwen3.5-2B gained about 11% on average.

Why it is hard: on a phone, attention is bound by memory traffic, not arithmetic. Splitting by sequence position makes every core touch all heads' K and V. Splitting by head makes the work disjoint, but it only works when heads divide evenly across cores (the PR states the divisibility condition; the bandwidth explanation is my inference).

**Why It Matters**

Faster prefill on the NPU is what makes on-device assistants feel responsive on mid-range phones, and it avoids sending prompts to a server. Apps that embed llama.cpp get the gain by updating, with no model change.

**Go Deeper**

- [llama.cpp PR #29974, Hexagon matmul and flash-attention scalability (primary source)](https://github.com/ggml-org/llama.cpp/pull/29974)
- [llama.cpp releases](https://github.com/ggml-org/llama.cpp/releases)

---

## 3. PostgreSQL: Stop Sleeping in `vacuum_delay_point()` While Holding Locks

**Category:** Databases / Systems

**The Technical Why**

Commit 96c9995 (Oct 5, by Heikki Linnakangas) fixes a throttling hazard. VACUUM and ANALYZE call `vacuum_delay_point()` to sleep and respect the cost-based delay, which keeps maintenance from starving queries. Some callers slept while holding buffer or index locks, so a throttled vacuum could block another backend wanting the same lock (the commit message says exactly that). The diff (5 files, +15/-9) moves the call in GIN fast-list cleanup to after locks are released, moves it in `hashbulkdelete()` out of `hashbucketcleanup()`, moves it in `acquire_sample_rows()` to between pages, and makes `vacuum_delay_point()` return immediately when `INTERRUPTS_CAN_BE_PROCESSED()` is false.

Why it is hard: throttling is a cross-cutting concern, and a sleep is only safe at points where no one can be waiting on you. Auditing every call site for held locks is the work, and a mistake shows up as latency spikes on unrelated queries, not as an error (my inference on the symptom).

**Why It Matters**

Anyone running autovacuum with a nonzero cost delay on GIN or hash indexes could see unrelated queries stall behind a sleeping maintenance worker. This is master, not a release, so it lands in a future version and possibly back-branches (not stated on the page). As a pattern: never block while holding a resource others may need.

**Go Deeper**

- [PostgreSQL commit 96c9995 (primary source)](https://github.com/postgres/postgres/commit/96c99955b62994b2cb47380d84f463456cf16ee5)
- [PostgreSQL commit history](https://github.com/postgres/postgres/commits/master)

---

## 4. three.js: Reversed Depth Buffers Get Fixed Across WebGPU Shadows, Render Targets and TAA

**Category:** Web Graphics and GPU

**The Technical Why**

On Oct 5 three.js merged a run of reversed-depth fixes by Mugen87. PR #34801 fixes `ShadowNode`, which did not apply the reversed projection in the first frame; it now sets `_reversedDepth` directly in setup, as `SunShadowNode` already did. PR #34803 adds the missing `DEPTH32F_STENCIL8` format to the WebGPU backend and makes the default render-target depth texture work with `reversedDepthBuffer`. Related commits cover ArrayCamera depth clears (#34804) and TAA depth support (#34802).

Why it is hard: reversed-Z maps near to 1 and far to 0, which spends float precision where perspective projection needs it (standard technique; not stated in these PRs). But every place that clears, compares or reads depth must agree on the convention: clear value, compare function, projection, shadow maps, post passes. One missed site gives a first-frame glitch or broken shadows only in one backend.

**Why It Matters**

Large scenes with a long far plane show z-fighting with a normal depth buffer. Reversed depth fixes that, but only if the whole pipeline honors it. These PRs close the gaps so WebGPU matches WebGL.

**Go Deeper**

- [three.js PR #34801, ShadowNode reverse depth fix (primary source)](https://github.com/mrdoob/three.js/pull/34801)
- [three.js PR #34803, WebGPU depth format and default depth texture](https://github.com/mrdoob/three.js/pull/34803)

---

## Thread to Watch

Work that hides behind a throttle or a split: vLLM balancing experts, llama.cpp splitting attention by head, Postgres moving a sleep out of a critical section. Watch whether MoonEP gets CUDA graph support and sharded expert weights, since the PoC's replicated weights are its main memory cost.
