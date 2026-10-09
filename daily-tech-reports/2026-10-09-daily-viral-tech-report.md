# Daily Viral Tech Report | 2026-10-09

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. I did not include Claude Haiku 5.5 (listed on Anthropic's news page for Oct 7) because I could not open a page with its pricing, context window or benchmarks, and the search-result figures disagreed with each other. Postgres FK fast-path commits were skipped because the 10-08 report covered them.

---

## 1. llama.cpp: One 65,535 Limit Breaks Norm Kernels on CUDA, Vulkan and WebGPU, and All Three Backends Fixed It This Week

**Category:** GPU Kernels / AI Inference

**The Technical Why**

llama.cpp PR #28175 (merged by ggerganov Oct 8, approved by JohannesGaessler) fixes the CUDA kernels `norm_f32`, `rms_norm_f32`, `l2_norm_f32` and `rms_norm_mul_rope_f32`, which crashed with "invalid argument" (issue #27901) when a tensor had more than 65,535 channels or samples. The kernels launched with a grid of `(nrows, nchannels, nsamples)`, and CUDA caps `gridDim.y` and `gridDim.z` at 65,535 (only `x` goes higher). The fix clamps y and z to 65,535 and loops inside the kernel over the rest. Because `gridDim.y` no longer equals the channel count, the counts are passed in explicitly. The multi-warp variants gained a trailing `__syncthreads()` because `block_reduce` reuses one shared buffer across loop iterations. Vulkan PR #30145 (0cc4m, Oct 9) copied the strategy ("loop over the excess"), and WebGPU PR #30219 (yomaytk, merged by ggerganov Oct 9) moved every op that used a 1D workgroup dispatch to a 2D grid via `compute_2d_workgroups()`, so norms over 65,536 or more rows work.

What is hard: the loop is not free to add. Without the barrier, iteration 2 can overwrite the shared buffer while a slow warp is still reading iteration 1. The author's numbers: normal shapes flat or slightly faster (`RMS_NORM 4096x512` 14.8 to 14.3 us), degenerate shapes slightly slower (`RMS_NORM [4,1,65535,1]` 136 to 159 us). jeffbolznv approved the Vulkan change but wrote he was "slightly concerned this might interact badly with some fused shader." The WebGPU page names no exact cap (inference: same class of limit, `maxComputeWorkgroupsPerDimension` defaults to 65,535 in the WebGPU spec; I did not verify this on the PR).

**Why It Matters**

A hardware dispatch limit stays invisible until a new model shape (many heads or very long batches) crosses it. Then a kernel that passed every test fails at launch. The seven new test cases at 65,536 only existed because someone hit the crash. If you write GPU code, test at limit plus one, on every backend you ship.

**Go Deeper**

- [llama.cpp PR #28175, CUDA norm with more than 65535 channels (primary source)](https://github.com/ggml-org/llama.cpp/pull/28175)
- [llama.cpp PR #30145, Vulkan rms_norm workgroup overflow](https://github.com/ggml-org/llama.cpp/pull/30145)
- [llama.cpp PR #30219, WebGPU 2D workgroup dispatch](https://github.com/ggml-org/llama.cpp/pull/30219)

---

## 2. llama.cpp: Backend Sampling Graph Made Static So Cached Compute Graphs Stop Aborting Under Load

**Category:** AI / ML Inference

**The Technical Why**

PR #30223 (ServeurpersoCom, approved and merged by ggerganov Oct 9, same day it opened) fixes a crash in backend sampling, where sampler steps run as nodes in the compute graph on the device instead of on the CPU (enabled with `LLAMA_ARG_BACKEND_SAMPLING=1`). The reserve pass built `n_outputs_max_per_seq` sampling chains per sampler, but a real decode built one chain per output row. So the graph shape at reserve differed from the shape at decode. In builds that forbid reallocation (`GGML_SCHED_NO_REALLOC`, used by server-cuda and server-metal), a later decode with the same node count but one more output row needed a bigger `out_ids` buffer and aborted with "unexpected graph reallocation." The fix always builds `n_outputs_max_per_seq` chains. Chains for real rows come first, the unused ones sit on a padding row, and `ggml_build_forward_select` skips them. Measured: a draft test graph had 549 nodes at reserve versus 417 at decode. A stress run with 4 parallel slots, a draft model and 32 concurrent requests aborted on master and served fully with the PR. Default `n_outputs_max_per_seq` is 1, so the usual graph is unchanged.

What is hard: to reuse a compiled graph (avoiding a rebuild and re-plan per step), every buffer must be sized once at reserve time, so the graph must have the same topology on every call. Variable work (how many rows are live) has to become fixed structure plus masking. The cost is wasted nodes on padding rows.

**Why It Matters**

This is the same trade every static-shape system makes: pad and mask to keep the graph fixed, or pay for dynamic shapes. It shows up in CUDA graphs, XLA, and any inference server that batches requests of unequal length. Servers with draft models and concurrent users hit it first.

**Go Deeper**

- [llama.cpp PR #30223, static backend sampling graph (primary source)](https://github.com/ggml-org/llama.cpp/pull/30223)

---

## 3. PostgreSQL: Use-After-Free in Run-Time Partition Pruning During EvalPlanQual Rechecks, Backpatched to 18

**Category:** Databases / Systems

**The Technical Why**

Commit 1b5dd3a by David Rowley (reported by Vladimir Savin, reviewed by Andrey Rachitskiy, backpatched through 18) fixes memory corruption in `EvalPlanQualStart()`. EvalPlanQual (EPQ) is how Postgres re-checks a row under READ COMMITTED when another transaction changed it while yours waited on its lock. The recheck builds a mini executor state that reuses the parent's `es_part_prune_states`. `InitExecPartitionPruneContexts()` then reallocated the pruning fields inside the recheck's own memory context, overwriting the parent's pointers. When the recheck finished, `FreeExecutorState()` deleted that memory context, and the parent could later prune through freed memory. The fix adds an `initialized` flag to `PartitionPruneState` and returns early if it is already set. The isolation test: session 1 locks row `id = 1` in `lk`, session 2 runs a `WITH ... FOR UPDATE` query whose CTE updates partitioned table `lp` and blocks, session 1 commits, and session 2 must run the EPQ recheck with run-time pruning.

What is hard: the commit says debug builds clobber freed memory and catch it, but production builds may not, depending on whether the allocator really `free()`d the chunk or kept it for reuse. So the bug can pass tests and sit silently. It needs two sessions, a lock wait, a partitioned table and a CTE. That is why it needs an isolation test and not a unit test. The commit also notes an earlier fix (8741e48) was backpatched to v18 as 9a82a64 but bb3ec16 omitted it.

**Why It Matters**

Partitioned tables with `UPDATE ... FROM` or CTEs under concurrent writers are common in billing and ledger schemas. The failure mode is wrong or crashing behavior only under lock contention, the hardest kind to reproduce in production. Anyone on Postgres 18 should read the backpatch note.

**Go Deeper**

- [Postgres commit 1b5dd3a, EPQ run-time pruning fix (primary source)](https://github.com/postgres/postgres/commit/1b5dd3a)
- [Postgres commit ce3387b, reuse zstd decompression contexts in WAL replay (same day: up to 4% faster recovery per Michael Paquier's measurement)](https://github.com/postgres/postgres/commit/ce3387b)

---

## 4. three.js: `range()` Randomness Stops Depending on Node Creation Order, and a std140 Uniform Array Layout Bug Is Fixed

**Category:** Web Graphics / GPU

**The Technical Why**

Yesterday's hashed `range()` (PR #34897) shifted every value when any node was added, because the hash salt was `this.id` from a global node counter. PR #34920 (sunag, merged Oct 9, milestone r187) gives `RangeNode` its own counter ("only range() calls in user code affect it"), with one per-object seed uniform shared by all range nodes. Cost: +51 B raw (+28 B gzipped) in the WebGPU bundle. By inference from the page, the salt still depends on the order of `range()` calls in user code, so reordering those calls still changes values. Separately, PR #34890 (mrdoob, merged Oct 9) fixes WebGL uniform buffer layout: arrays of `float`, `vec2` and `vec3` did not use the std140 array stride, so `float data[3]` built from `new Uniform(1), new Uniform(2), new Uniform(3)` got wrong offsets. Arrays of `vec4`, `mat3` and `mat4` were fine.

What is hard: std140 rounds every array element up to a 16-byte stride, so a `float[3]` takes 48 bytes, not 12. A CPU-side writer that packs tightly reads garbage on the GPU for elements after the first, with no error. Deterministic "random" values need a stable seed, and a global counter makes the seed depend on unrelated code.

**Why It Matters**

For a shader editor that compiles node graphs to code, both are rules to copy: derive salts from the user's graph, never from library internals, and run a layout test for every uniform type against the GPU's real packing. Reproducible visuals across versions is a product promise, and tests that compare screenshots caught it here (`webgpu_particles_soft`, `webgpu_tsl_galaxy`).

**Go Deeper**

- [three.js PR #34920, RangeNode order-independent values (primary source)](https://github.com/mrdoob/three.js/pull/34920)
- [three.js PR #34890, std140 layout of uniform arrays](https://github.com/mrdoob/three.js/pull/34890)

---

## Thread to Watch

Fixed limits that only bite at new shapes: the 65,535 grid cap this week, static graph shapes for sampling, and std140 strides. Tomorrow, watch whether more llama.cpp backends (SYCL, Metal, OpenCL) get 65,536-row norm tests, and for the r187 three.js milestone merging more node-system fixes.
