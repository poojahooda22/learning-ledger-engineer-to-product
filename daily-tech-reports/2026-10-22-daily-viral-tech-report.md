# Daily Viral Tech Report | 2026-10-22

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. Skipped as thin: three.js #35145 (LightProbeGrid multi-bounce; I only saw the title and the story overlaps the Oct 20 report) and vLLM #62784 (spec decode skips drafting; I did not read the page).

---

## 1. three.js TSL: A `toVar()` Used Inside a Branch Was Emitted Twice in Generated Shader Code, and the Fix Took Five Commits

**Category:** Web Graphics / GPU

**The Technical Why**

PR #35067 (author sunag, merged Oct 22 by the author as b0f6d52, approved by Mugen87, milestone r187) fixes a code generation bug in TSL, three.js's node-to-shader compiler. Since #34643, a `toVar()` created outside a `Fn()` and first used inside a branch was generated twice: once in the branch and again in the base stack. For a cheap value that is wasted ALU. For an expensive one it is a second full evaluation. The fix has two parts. `LoopNode.declareBefore()` now declares a stack-less node once before the loop, but only if its value is the same in every iteration; otherwise it is generated lazily where used. A later commit adds `NodeBuilder.createBlock()` and "Reuse external variables in branches of the same condition."

What is hard: the first version did not fully work. Mugen87 showed on Oct 17 that a `toVar()` used by two outputs, each inside a `select()` on the same condition, was still emitted twice. In his city demo, `room` appears in both `colorNode` and `emissiveNode` under `if ( isGlazing )`, so glass fragments raymarched the room interior twice. Where to declare a variable depends on every use site, and a node graph only sees its uses while it is being walked (inference on why the one-line fix failed; the page states no root cause). After the follow-up commits he wrote "It seems the performance issue is now gone." The page gives no frame-time numbers. CI bundle cost: Core and WebGL +0 B; WebGPU +706 B minified, +280 B gzipped.

**Why It Matters**

Anyone compiling a node graph to shader code inherits this class of bug: duplicate evaluation hides in generated code, where nobody reads it, and shows up only as a slow frame. For a node-based shader editor, a test that counts emitted declarations is cheaper than a profiler session. The general rule: variable placement must be decided with all uses known, and "same condition" blocks should share declarations.

**Go Deeper**

- [three.js PR #35067, TSL: Fix external variables being generated twice when used in a branch (primary source, includes the review thread)](https://github.com/mrdoob/three.js/pull/35067)
- [three.js repository](https://github.com/mrdoob/three.js)

---

## 2. vLLM: Silent MoE Weight Corruption in RL Weight Sync Came From the CUDA Caching Allocator Reusing a Buffer With a Read Still Pending on Another Stream

**Category:** AI Infra / Training-Inference Systems

**The Technical Why**

PR #62555 (author dharak-cohere, merged Oct 22 by aoshen02 as c723f6d, auto-merged after 96 checks, approved by aoshen02 and hao-aaron) fixes `packed_nccl_broadcast_consumer`, which receives new model weights in packs during RL updates. Each pack's post-unpack callback runs on one of two slot streams. Layerwise reload keeps views into the receive buffer and reads them from a later pack's callback, on the other stream, then drops the views. The caching allocator saw a free buffer with no record of the pending read, handed the block to the next receive, and NCCL overwrote it before the read ran. Result: silent corruption of fused MoE `w13_weight`. Reproducer: 10/10 corrupted reloads on main, 0/10 with the fix.

What is hard: nothing errors, and a second cause hides the first. The code also assumed NCCL completes receives in issue order, which masks the race in practice but is not a guarantee. The fix does two things. `record_stream` marks each buffer as used by every slot stream and the caller's stream, so the allocator will not reissue it early. A per-pack event chain makes each callback wait for the previous pack's receive, without delaying receives. Cost, per the author: latency within +/-1.5% (OLMoE 13 packs, NET: 4,945 ms to 4,928 ms), but peak reserved memory rises on slow transports, e.g. +134 MB in one served case and +364 MB in one repro, because buffers with pending reads cannot be reused early. The page says a small theoretical window remains: the ordering test still depends on a GPU sleep of about 0.1 s.

**Why It Matters**

RL post-training loops push fresh weights into the serving engine every step, so a corrupt sync quietly poisons rollouts and the training signal, with no crash to point at. The pattern to keep: when a tensor crosses streams, the allocator needs `record_stream` or an explicit event, and "it worked because the other thing is usually ordered" is not a contract.

**Go Deeper**

- [vLLM PR #62555, Fix cross-stream reuse of packed NCCL receive buffers (primary source, includes the benchmark and memory tables)](https://github.com/vllm-project/vllm/pull/62555)
- [PyTorch docs, CUDA memory management and `Tensor.record_stream`](https://pytorch.org/docs/stable/notes/cuda.html#memory-management)

---

## 3. llama.cpp: Intel XMX Matrix Engines Now Handle 2 to 80 Activation Columns, Aimed at Speculative Decoding Verification

**Category:** AI Inference / GPU Kernels

**The Technical Why**

PR #29864 (author budiony, first-time contributor; merged Oct 22 by arthw as d81d0d8; 29 commits) adds multi-column `mul_mat_vec_q` kernels to the SYCL backend that run on Intel XMX int8 DPAS units through ESIMD. It covers reordered K-quant and Q8_0 weights and targets verifying 2 to 8 draft tokens in speculative decoding, plus short prompt batches of 9 to 80 tokens. The old dp4a kernels repeat weight unpacking and per-column scale math for every column, so cost grows with batch size; the author measured a 4-token step 46% slower than a 1-token step on a Q6_K model. A fused gate/up kernel (up to 16 columns) computes the FFN's up x act(gate) in one launch with SWIGLU or GEGLU in the epilogue.

What is hard: decode is memory bound, so one column gains little (Q8_0 15.9 to 18.0 tok/s). The win appears at 2 to 32 columns, where unpacked weights are reused across columns and fed to matrix hardware. Author's numbers on an Arc Pro B70: Q4_K_S pp16 50.3 to 308 tok/s, Q2_K speculative decoding 30.2 to 60.0 tok/s average. Correctness had real bugs: the first version returned NaN when the row count was not a multiple of 4, and the existing `test-backend-ops` MUL_MAT cases use k below 1024, so they do not exercise the kernel (the kernel declines below 1024). Perplexity on Q3_K_S was 4.8360 off versus 4.8349 on. Weight reordering happens on the first call with 8 or fewer columns, so a larger first batch leaves weights unreordered. Third-party results are uneven: arthw saw Qwen3-8B Q4_K_M pp64 go from 340.32 to 1435.58 tok/s with fa=0 but about +0.01% to +0.19% with fa=1, and a -7.44% pp512 row on an Arc770.

**Why It Matters**

Speculative decoding only pays off if verifying k tokens costs close to one token. This makes that true on Intel GPUs, which widens where local inference is practical. Read the flash-attention caveat before trusting the headline gain. The author disclosed AI assistance for kernel optimization and testing, and the page shows ggerganov as a code owner still awaiting review at snapshot time.

**Go Deeper**

- [llama.cpp PR #29864, Multi-column on Intel XMX matrix engines refactored (primary source, includes all benchmark tables)](https://github.com/ggml-org/llama.cpp/pull/29864)
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

---

## 4. PostgreSQL: A Cancelled LISTEN/NOTIFY Read Left `isAdvancing` Set, Freezing the Notification Queue Tail (Backpatched Through 19)

**Category:** Databases / Concurrency

**The Technical Why**

Commit 871dee0 (author shihao zhong, reviewed by Tom Lane, committed by tglsfdc, 1 file `async.c`, +14/-3, backpatched through 19) fixes the NOTIFY queue. `asyncQueueReadAllNotifications()` sets an `isAdvancing` flag while a backend reads the queue and clears it when done. If an error interrupts it, for example a cancel while waiting for a safe snapshot, the flag stays set. The commit message says `SignalBackends()` then "never advances or wakes us again," so the queue tail stops moving. `AtAbort_Notify()` now clears the flag. It takes `NotifyQueueLock` in shared mode and only acts if the backend is a registered listener and the flag is set. A code comment says only the backend's own process changes this flag, so it can be checked without the lock. Subtransaction abort needs no change because the read happens only in top-level transactions.

What is hard: this is the classic "set flag, do work, clear flag" pattern with no cleanup path on error. A shared queue's reclaim point is the slowest listener, so one stuck backend can stall everyone. Whether the stalled tail causes unbounded queue growth is my inference from how a queue with a fixed tail behaves; the commit message does not say so.

**Why It Matters**

LISTEN/NOTIFY is a common cheap pub/sub inside Postgres for cache invalidation and job wakeups. A user cancel at the wrong moment could silently stop one connection from ever being woken again. The lesson: every flag set in a critical section needs an abort-path reset, and in-progress markers belong in the transaction's abort handler, not only at the happy-path exit.

**Go Deeper**

- [postgres commit 871dee0, Clear async.c's isAdvancing flag when a notify read is aborted (primary source)](https://github.com/postgres/postgres/commit/871dee0)
- [Mailing list discussion linked from the commit](https://postgr.es/m/CAGRkXqQ1bmVwBw2CXiyZeWWP=XkHPXLPcQ7BSg2Q5JKSvDAnTA@mail.gmail.com)

---

## Thread to Watch

Hidden ordering assumptions that only fail under load: vLLM's cross-stream buffer reuse (#62555), three.js declaration placement across same-condition branches (#35067) and Postgres's missing abort-path reset (871dee0). Tomorrow, watch whether llama.cpp #29864 draws follow-ups on the flash-attention regression and the k below 1024 test gap, and whether the vLLM RL weight sync path gets a deterministic test in place of the GPU sleep.
