# Daily Viral Tech Report | 2026-10-27

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. Skipped: vLLM #63101 and llama.cpp #29882 (covered yesterday), and Postgres 34d52e6 (hash_mem cap on hash table buckets; I saw only the title).

---

## 1. vLLM: TMA Loads for the MoE Up and Down Projections, 15 to 20% Faster Kernel, 3.1 to 3.4% More Throughput on GLM-5.3

**Category:** AI Serving / GPU Kernels

**The Technical Why**

PR #63820 (author yewentao256, opened Oct 26, approved by sfeng33, merged Oct 27, merge commit c56706e, tracked under issue #57406 "GLM 5.3 Performance Optimization") switches the MoE up and down projection kernels to use TMA, the Tensor Memory Accelerator. TMA is a Hopper-class hardware unit that copies tiles between global and shared memory from a descriptor, so a single thread issues the copy and the other threads do not spend instructions computing addresses and loading. The page gives no implementation detail beyond the title and the core commit d62a1a1, so the description of how TMA helps is general background (inference), not something the PR spells out.

What is hard: the expert matmuls in an MoE layer are small and ragged (each expert gets a different number of rows), so they are bound by how fast weights and activations reach the tensor cores, not by math. Kernel microbenchmark, main vs TMA: M=24 78.4 to 62.8 us (19.9%), M=96 125.5 to 101.6 us (19.1%), M=1024 223.3 to 189.1 us (15.3%). End to end on GLM-5.3, H200, TP=8, MTP speculative decoding with 5 tokens, guidellm with 8,000 prompt and 1,000 output tokens: concurrency 4 went 297.7 to 307.8 tok/s (+3.4%), concurrency 16 went 497.3 to 512.9 tok/s (+3.1%). Latency moved less: -0.3% at concurrency 4, -1.9% at 16.

**Why It Matters**

A 15 to 20% kernel win becomes about 3% end to end because the up and down projections are only one slice of a decode step, which also has attention, routing, communication and sampling. That ratio is the useful lesson: profile first, then expect the end-to-end gain to be the kernel gain times the kernel's share. Caveats on the page: concurrency 1 results were lost and the author asked others to validate, accuracy is described only as "covered in unit test" with no numbers, the five requested code owners had not reviewed at the snapshot, and it is one model on one GPU type. Who wins: operators serving large MoE models such as GLM-5.3 on Hopper-class GPUs.

**Go Deeper**

- [vLLM PR #63820, [Perf] Use TMA for MOE up/down proj (primary source, kernel table, end-to-end table)](https://github.com/vllm-project/vllm/pull/63820)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 2. llama.cpp: Sparse Attention Selection No Longer Borrows the Shape of the Dense KQ Mask

**Category:** Inference Runtimes / Sparse Attention

**The Technical Why**

PR #31243 (author am17an, opened and merged Oct 27, commit 27b7a8a, approved by ggerganov and sanmai) decouples the sparse key selection used by the qwen4exp (QSA) and glm5-next models from the layout of the dense KQ mask. Sparse attention here means an indexer picks the top-k cache cells per query, and attention reads only those. After #31009 those models no longer read mask values, but their selection was still built in the mask's row layout, and both model files asserted the shapes matched. The fix builds the selection as `[n_kv, n_tokens]` over cache cells and splits it into the K view's streams just before attention.

What is hard: this is a layout contract between two components. If the selection is shaped like the mask, the mask cannot change layout without breaking every sparse model. The author says the motivation is PR #29510, where the unified cache attends through per-sequence K/V row lists and the mask follows those lists. Without this change, qwen4exp needed an opt-out that also skipped its MTP graph. Testing reported: random models with `indexer.top_k` set to 8 so selection really happens, perplexity matched master across 16 configurations (unified and split cache, flash attention on and off, CUDA and CPU). On Qwen3.8-Flash-Next IQ4_NL, 2k perplexity matched and greedy outputs were identical. At 32k the difference was within run-to-run noise, which the author attributes to top-k ties.

**Why It Matters**

Sparse attention only pays off if the selection step stays cheap and independent of how the cache is laid out. Separating the two means the cache work in #29510 can proceed without touching each model. Caveats: the GLM comparison used master from before #31009, so it tests this PR and part of #31009 together; results at 32k and one GLM chunk are within noise, not bit identical; #29510's status is not described on the page. The PR discloses AI assistance. Who wins: local-inference users running new sparse-attention models, and maintainers adding multi-sequence cache features.

**Go Deeper**

- [llama.cpp PR #31243, llama : build the k-pool sparse selection independently of the kq_mask shape (primary source, test matrix, review thread)](https://github.com/ggml-org/llama.cpp/pull/31243)
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

---

## 3. PostgreSQL: Base Backups Could Capture a Replaced File Truncated or Zero-Padded

**Category:** Databases / Backup Correctness

**The Technical Why**

Commit 51d7bf5 (author David Steele, committed by Michael Paquier, `basebackup.c` +24 -9, backpatched through 15, v14 excluded as near end of life) fixes a time-of-check to time-of-use gap. The base backup scans the data directory and calls `stat()` on each file, then later opens it and sends exactly that many bytes. If a file is swapped by a concurrent `rename()` in between, the backup holds the new content cut short or padded with zeros to the old size. For relation files that is harmless, because WAL replay repairs them. For files written to a temp file and renamed into place it is not. The commit names `postgresql.auto.conf`, `pg_logical/replorigin_checkpoint` (a damaged copy fails its CRC check and blocks startup), pg_prewarm's `autoprewarm.blocks`, and `pg_stash_advice.tsv`.

The fix: for non-relation files only, call `fstat()` on the already-open file descriptor, and use those stats for the tar header, the manifest entry and the byte count. Because `rename()` is atomic, an opened file is always a complete version. Relation files keep the earlier `stat()`, since WAL replay restores them and skipping the refresh is cheaper.

**Why It Matters**

The general rule: size and content must come from the same handle. Stat by path, then read by path, is two lookups that can resolve to two different files. The distinction in the fix is also instructive: it pays the extra check only where nothing downstream can repair the result. Who wins: anyone taking base backups with `pg_basebackup` or tools built on the same protocol, and restores of the listed files. I did not verify the minor release schedule, and the page shows no date for the commit.

**Go Deeper**

- [PostgreSQL commit 51d7bf5, Avoid truncated or zero-padded copies of replaced files in backups (primary source, full message and diff)](https://github.com/postgres/postgres/commit/51d7bf5edcaf4f8a95bb10f96b7af76d00a2f83f)
- [PostgreSQL repository](https://github.com/postgres/postgres)

---

## 4. three.js: WebGL Instanced Matrices Above the Uniform Limit Rendered Distorted, Fixed in One Day

**Category:** Web Graphics / Instancing

**The Technical Why**

PR #35227 (author sunag, opened, approved by Mugen87 and merged into `dev` Oct 26, commit f2a0982, milestone r187) fixes a regression from #35007, which shares builds between instanced meshes. When the instance count passes the uniform buffer limit, instance matrices are bound as vertex attribute columns. Those columns were `InterleavedBufferAttribute` objects whose `data` was an `InstancedBufferAttribute`, not an `InterleavedBuffer`. `WebGLBindingStates` assumed `data` was always interleaved, so it used a stride of 0 and never set a vertex attribute divisor. The result was distorted rendering in the `webgl_tsl_instancing` example when `count` exceeded 1024.

The fix mirrors what the WebGL 2 fallback path (`WebGLVertexArrayUtils.js`) already did: use the attribute's `itemSize` as stride when the buffer is not interleaved, compute the offset from the attribute, and call `vertexAttribDivisor` with `meshPerAttribute` for instanced attributes.

**Why It Matters**

The divisor is what tells the GPU to advance an attribute once per instance instead of once per vertex. Without it, every vertex reads a different matrix, which looks like distortion, not a crash. Lesson: when one code path is reused for a new data layout, every place that assumed the old layout needs a check, and two backends with two copies of the logic drift. Caveats: no automated test is described, only one reviewer confirmed it by running the example, and the linked discussion is a placeholder. Bundle cost was about 68 B minified. Who wins: apps drawing thousands of instances through three.js on WebGL.

**Go Deeper**

- [three.js PR #35227, WebGLRenderer: Fix instanced attributes viewing a buffer attribute (primary source)](https://github.com/mrdoob/three.js/pull/35227)
- [three.js PR #35007, the change that exposed it](https://github.com/mrdoob/three.js/pull/35007)

---

## Thread to Watch

Contracts between layers: llama.cpp stops shaping sparse selection like the mask (#31243), Postgres stops trusting a stale size (51d7bf5), three.js trusts a buffer type it should have checked (#35227). Tomorrow, watch for llama.cpp #29510 (unified cache with per-sequence row lists) and the concurrency 1 validation of vLLM #63820.
