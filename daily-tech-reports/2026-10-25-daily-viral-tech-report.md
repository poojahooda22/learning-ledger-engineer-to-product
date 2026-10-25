# Daily Viral Tech Report | 2026-10-25

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. Skipped: vLLM #63088 (daemon-to-daemon warmup cache seeding; the page has no mechanism write-up or numbers, and calls the protocol "initial"), vLLM #61976 (priority self-preemption scheduler fix; the end-to-end accuracy data was still unposted on the page) and the other llama.cpp backend PRs merged today (Vulkan, OpenCL, Hexagon; I only saw titles).

---

## 1. llama.cpp: CUDA Prompt Processing Fuses the FFN Gate and Up Matmuls Plus the GLU Into One MMQ Kernel, Up to +10.7%

**Category:** AI Inference / GPU Kernels

**The Technical Why**

PR #29948 (author anujj, opened Oct 4, approved by JohannesGaessler Oct 24, merged by ggerganov Oct 25) extends an existing gate/up + GLU fusion, which only covered small batches, to the MMQ path used for prompt processing. A transformer FFN computes `act(x * W_gate) * (x * W_up)`. Before, that was two matmuls and one elementwise op, three launches and two round trips of the intermediate through VRAM. Now one `mul_mat_q` launch does all of it. The loaders interleave up and gate rows in chunks of 8, so each thread holds the up value and the matching gate value for the same output element (rows `r` and `r + 8` of the MMA tile). The MMA loop, tile config and accumulator `sum[]` are unchanged. The write-back computes GLU straight from registers and writes 64 output rows per tile. Supported: SWIGLU, GEGLU, SWIGLU_OAI, SWIGLU_CLAMP on Q4_K, Q5_K, Q6_K, Q8_0 and IQ4_XS, NVIDIA Turing and newer.

What is hard: fusion is not free. Review feedback showed the tile order matters. With the other order the fused kernel re-read the gate and up weights from VRAM once per column tile, so the author put column tiles innermost, letting neighbouring blocks share weight tiles. Fusion also only fires when master's tiling rule (at least 90% wave efficiency) would pick tiling for the fused grid, and only when gate and up share type, shape and strides. Numbers (llama-bench, pp16384, tok/s, author's runs): on a DGX Spark (GB10), Llama-3.1-8B IQ4_XS went 2933.9 to 3249.0 (+10.7%) and Qwen3.8-27B Q4_K_M 840.1 to 899.9 (+7.1%). On an RTX 5090, Llama-3.1-8B Q4_K_M went 12071.5 to 12625.9 (+4.6%). Perplexity is bit-identical with `GGML_CUDA_DISABLE_FUSION=1` on all five types.

**Why It Matters**

Prompt processing (prefill) is compute-bound, and the FFN is the biggest block of it, so cutting launches and memory traffic there speeds up time to first token for long prompts on local GPUs. The honest limits are on the page: dense FFN only (MoE is a promised follow-up), five quant types, and per-layer mixed quants are not fused (that is #30489). At small ubatch sizes on the 5090 the gate skips fusion and gains are near zero (+0.6% to +0.1% at ub 128 to 512 versus +4.9% to +6.3% at ub 1024 to 2048). On the RTX 5090 the fused matmul alone was about 2% slower than two stock matmuls; the whole block still won. Who wins: anyone running long-context prefill on consumer or edge NVIDIA hardware.

**Go Deeper**

- [llama.cpp PR #29948, CUDA: fuse FFN gate/up matmuls and GLU in MMQ with interleaved gate/up rows (primary source, mechanism, benchmark tables, review thread)](https://github.com/ggml-org/llama.cpp/pull/29948)
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

---

## 2. vLLM on ROCm: One Big KV Cache Allocation Silently Pushed Attention to 64-Bit Addressing, Costing About 18% Throughput

**Category:** AI Serving / GPU Compilers

**The Technical Why**

PR #60084 (author Uros Markovic of AMD, opened Oct 5, merged Oct 25 into `main`, commit a55937f) restores 32-bit buffer addressing for the `TRITON_ATTN` backend. Per the PR, Triton's AMD backend uses 32-bit buffer loads only when a pointer's `ptr_range()`, or else its whole backing storage, fits in 2 GiB. An earlier change (#51718) moved the KV cache into one backing allocation (about 36.6 GiB in the author's setup), so each layer's K and V view failed the check even though one layer's region is small. The attention kernel `kernel_unified_attention` then compiled with 64-bit addressing, its VGPR use rose from 205 to 256, and occupancy on gfx1100 fell from 7 to 5 waves per SIMD. The fix: on ROCm, `set_triton_ptr_range` gives each K/V view a `ptr_range()` equal to the bytes from its `data_ptr()` to the end of its layer's region, called in `forward` and `do_kv_cache_update`. Triton still checks the 2 GiB limit at each launch.

What is hard: nothing broke. Outputs stayed correct, so only a profile reveals it. A memory layout choice upstream (one allocation, simpler to manage) changed an addressing mode downstream in the compiler, which changed register pressure, which changed how many wavefronts the GPU can keep in flight to hide memory latency. Author's numbers on a Radeon Pro W7900 (gfx1100): Whisper-large-v3 5.34 to 6.31 req/s (+18.2%), Qwen2.5-7B-Instruct 2.70 to 3.19 req/s (+18.4%), median attention call 1,843 us to 1,379 us on Whisper cross-attention, and a bisect shows #51718 at 5.40 req/s versus 6.40 for its parent. Transcripts were token-identical on all 2,703 LibriSpeech validation-clean clips; GSM8K moved within noise (stderr 1.1).

**Why It Matters**

AMD serving performance is judged on exactly this kind of detail, and a quiet 18% regression sat in the tree until someone bisected it. Caveats from the page: CI has no RDNA runners, so no test was added; on CDNA (the datacenter parts) AMD's afriedri measured only about 1%; layers above 2 GiB and block-outermost layouts still use 64-bit addressing; a TRITON_MLA follow-up was offered, and ROCM_ATTN saw no benefit. Who wins: RDNA3 users and anyone building a GPU runtime, since "one big arena" allocators have a hidden price in addressing width.

**Go Deeper**

- [vLLM PR #60084, [ROCm][Perf] Restore 32-bit buffer addressing for TRITON_ATTN KV cache views (primary source, root cause, benchmarks)](https://github.com/vllm-project/vllm/pull/60084)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 3. PostgreSQL: pg_listening_channels() Leaked a Hash Scan When a Query Stopped Reading Early, Risking a PANIC

**Category:** Databases / Systems

**The Technical Why**

Commit dfd7ede ("Fix leaked hash scans in pg_listening_channels()", author Joel Jacobson, reviewed by Tom Lane, committed to `master` Oct 25) fixes a bug introduced by commit 282b1cd. That commit made `pg_listening_channels()` a value-per-call set-returning function: it opened a `HASH_SEQ_STATUS` scan on `localChannelTable` on the first call and resumed it on each later call. But the executor is not required to call a set-returning function until it is exhausted. `SELECT pg_listening_channels() LIMIT 1` calls it once and abandons it. The scan stays registered with the hash table machinery until transaction end, where it is reported as a leak; per the commit message, repeated leaks eventually lead to a PANIC. The fix uses `InitMaterializedSRF()`, writes every channel name into the result tuplestore in one call, and finishes the `hash_seq_init`/`hash_seq_search` walk inside that call. The `FuncCallContext` and `SRF_*` multi-call machinery is gone. 3 files, +32 -35, plus a regression test that runs exactly the `LIMIT 1` query.

What is hard: Postgres hash tables track open sequential scans so that an insert during a scan cannot silently corrupt iteration, and the bookkeeping is capped. A scan held across calls is a resource with a lifetime the caller controls, and the caller (the executor under LIMIT) may never end it. This is the same class of bug as an unclosed file handle, except the cleanup is a transaction-end sweep. The materialize-in-one-call pattern trades a small tuplestore for removing the cross-call lifetime entirely. Backpatch: "through 19", the release containing 282b1cd (from the commit message; I did not see the release schedule).

**Why It Matters**

LISTEN/NOTIFY is used as a lightweight pub/sub inside Postgres, and introspection functions get called from monitoring queries that often add LIMIT. The lesson for anyone writing iterators: if your API lets the consumer stop early, never hold a scarce resource open between `next()` calls. Either materialize, or give the scan a destructor that runs on abandonment. Who wins: Postgres 19 users on the development branch, before release.

**Go Deeper**

- [PostgreSQL commit dfd7ede, Fix leaked hash scans in pg_listening_channels() (primary source, full message and diff)](https://github.com/postgres/postgres/commit/dfd7ede)
- [PostgreSQL repository](https://github.com/postgres/postgres)

---

## 4. three.js: LDrawLoader smoothNormals() Was Quadratic in Face Count Because `for..in` Rebuilt the Key List, Fixed With a Map (About 5x on 80 Models)

**Category:** Web Graphics / JavaScript Performance

**The Technical Why**

PR #35200 (author AvaLovelace1, opened Oct 24, approved by Mugen87 and merged to `dev` Oct 25, milestone r187) replaces a plain object `halfEdgeList` with a `Map` in `LDrawLoader.smoothNormals()`. Per the PR, the loop was a `for..in` over the object that took the first entry and then `break`ed, repeated until the list drained. That looks O(1) per pass, but each `for..in` over an object rebuilds the object's full key list, which grows with the number of faces, so the whole pass is quadratic. A `Map` iterates once and deleting entries during iteration is safe because deleted entries are skipped. The commit title: "Make smoothNormals() linear instead of quadratic in the number of faces."

What is hard: the cost was hidden in the language, not the algorithm. The author's explanation relies on V8 internals: V8 caches an object's enumerable keys only while the object is in "fast mode", and adding more than 1020 keys or deleting an entry drops it out of that mode, so every `for..in` pays the full key-collection cost. The author only skimmed SpiderMonkey and JavaScriptCore and says other engines are affected "to some extent" without verifying it. Benchmark: about 80 LDraw models parsed on the author's own site, total parse time from about 80 s to about 15 s (the page gives approximate values with "~"; no published methodology; the review was a short approval).

**Why It Matters**

LDraw models (LEGO instructions) are big, so load time, not rendering, was the wait. The broader rule: a dictionary used as a work queue (insert, peek first, delete) is a trap in JavaScript. `Map` has insertion-ordered iteration and cheap deletes by design. Who wins: anyone loading large LDraw files in three.js, and any loader author using objects as hash maps with churn.

**Go Deeper**

- [three.js PR #35200, LDrawLoader: Make smoothNormals() linear instead of quadratic (primary source, explanation and timings)](https://github.com/mrdoob/three.js/pull/35200)
- [three.js repository](https://github.com/mrdoob/three.js)

---

## Thread to Watch

Hidden cost models that only a profile reveals: a single KV arena changing the compiler's addressing mode (vLLM #60084), a `for..in` rebuilding its key list (three.js #35200), and a fusion that wins as a block while its kernel loses (llama.cpp #29948). Tomorrow, watch whether llama.cpp lands the MoE version of the gate/up fusion promised in #29948 and whether the vLLM TRITON_MLA follow-up to #60084 appears.
