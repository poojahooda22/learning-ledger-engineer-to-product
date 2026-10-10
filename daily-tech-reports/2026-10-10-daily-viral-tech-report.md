# Daily Viral Tech Report | 2026-10-10

Every story below is checked against a primary GitHub page (PR, issue or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. I skipped a second Postgres item (the NOT NULL planner fixes by Richard Guo, same day) to keep the spread wide, and the llama.cpp 65,535 grid-limit work because the 10-09 report covered it.

---

## 1. PostgreSQL: Hash Index Scans That Start During a Bucket Split Silently Missed Rows, Backpatched to 14

**Category:** Databases / Systems

**The Technical Why**

Commit 173cf85 (Peter Geoghegan and Amit Kapila, reviewed by Vignesh C, backpatch through 14) fixes wrong query answers from hash indexes. A hash index grows by splitting a bucket into two. A scan whose target bucket is still being populated by a split must read both buckets: forward scans read the populated bucket then the one being split, backward scans the reverse. `_hash_readnext` and `_hash_readprev` made that hop, but `_hash_next` did not, so at the end of the first bucket it ended the scan and matching tuples in the second bucket were never returned. The bug came from commit 7c75ef5 (page-at-a-time hash scans). The fix gives `_hash_next` the same hop through two shared helpers, `_hash_step_to_split_bucket` and `_hash_step_to_populated_bucket`, and adds an Assert that such a scan ends in the bucket its direction visits last. The commit message credits Claude Code with finding the problem, and the committed test is a simplified version of the test it wrote.

What is hard: it needs several conditions at once. The last page `_hash_next` read in the first bucket must have a match (otherwise the loop in `_hash_readpage` makes the hop), and a matching row must have been inserted into the populated bucket after the split began. An incomplete split can linger "until an inserter finished the split at some indeterminate later time," so the window is open for a long time, but wrong answers did not require one. The test forces it with two new injection points (`hash-split-before-relocation` to fail a split on purpose, `hash-finish-incomplete-split` to observe the repair): 10 rows with `v = 1`, about 5,000 inserts to fail the split, one more `v = 1` row, then forward count and backward cursor must both return 11. The message admits the real flaw is that page stepping logic is scattered, and "really belongs in one routine, modeled on nbtree's `_bt_steppage`," but that is "much too invasive for a backpatched fix."

**Why It Matters**

An index returning too few rows with no error is the worst failure class for a database, because nothing alerts and results look plausible. Anyone running hash indexes on PG 14 to 18 under concurrent inserts should watch for the minor release. The pattern to learn: when the same traversal rule is copied into three functions, one copy will drift.

**Go Deeper**

- [postgres commit 173cf85, Fix hash index scans that start during a bucket split (primary source)](https://github.com/postgres/postgres/commit/173cf85)
- [PostgreSQL docs, hash indexes](https://www.postgresql.org/docs/current/hash-index.html)

---

## 2. Rust: WebAssembly `wide-arithmetic` Target Feature Stabilized for 1.101.0

**Category:** Languages / Compilers

**The Technical Why**

rust-lang/rust PR #160877 (alexcrichton, opened Aug 10, merged Oct 10 via rollup #164099, milestone 1.101.0) stabilizes the `wide-arithmetic` target feature for Rust's WebAssembly targets. It maps to LLVM's `wide-arithmetic` feature and gates four new instructions from the WebAssembly proposal, centered on 128-bit add and subtract and a widening 64x64 to 128-bit multiply. The PR says the proposal reached phase 4 of the WebAssembly CG process, so engines are expected to ship it. There are "no library APIs, no impact on the Rust language, nor any impact on ABIs." It stays off by default and needs `-Ctarget-feature=...`. The visible effect is better codegen for 128-bit multiplication; reviewer scottmcm noted improved WASM output for `u64::carrying_mul_add`.

What is hard: without these instructions, a 64x64 to 128-bit multiply on Wasm (which has only 32 and 64-bit integer ops) must be built from several 32-bit partial products plus carry handling. That is the inner loop of big-integer math in crypto, so a single instruction matters. The thread is also a lesson in process: RalfJung asked whether stabilizing a target feature needs t-lang, not just t-compiler, since the `target_features.rs` header says "Stabilizing a target feature requires t-lang approval." The FCP was reopened with lang added, and jieyouxu opened rust-forge#1122 to fix the docs. Inference: the exact instruction names are not on the PR page, so check the proposal repo.

**Why It Matters**

Wasm is where crypto, zero-knowledge proofs and bignum libraries run in browsers and edge runtimes, and these are CPU-bound loops where the instruction count dominates. Shipping the feature as opt-in means library authors can build a fast variant and a fallback, and users get speed once their engine supports it.

**Go Deeper**

- [rust-lang/rust PR #160877, Stabilize wide-arithmetic (primary source)](https://github.com/rust-lang/rust/pull/160877)
- [WebAssembly wide-arithmetic proposal](https://github.com/WebAssembly/wide-arithmetic)

---

## 3. llama.cpp: A Missing `__syncthreads()` Caused a Flaky Flash Attention Race on AMD, Found After Three Weeks

**Category:** GPU Kernels / AI Inference

**The Technical Why**

PR #30103 (IMbackK, approved by JohannesGaessler, merged by ggerganov Oct 10) fixes a race in the CUDA/HIP kernel `flash_attn_ext_f16_process_tile`, reported in issue #29314 on Sep 23 as a flaky `test-backend-ops` failure on AMD gfx1201 ("I can repo, sometimes, its flaky"). Per the issue, the cross-warp reduction reuses the shared-memory buffer that loads Q, and "without synchronization there is a race condition with the next tile." The barrier was skipped when `np == 1`, which is safe only then. Gaessler wrote he had forgotten that `nbatch_combine` can differ from `DKQ/2`, which also creates the race. The agreed fix is a narrower guard: `if (np > 1 || nbatch_combine != DKQ/2)` instead of always syncing.

What is hard: a race on shared memory depends on warp scheduling, so it passes most runs and fails some, and a different GPU can hide it. A reviewer (am17an) reported `compute-sanitizer --tool=racecheck ./test-backend-ops -o FLASH_ATTN_EXT` "fails for me quite reliably," on a build without the PR, and said he had not checked with it. So the page shows no passing re-run, and maintainers merged on the logic ("even if this is not the only issue, its def. an issue"). The narrow guard exists because a barrier per tile costs throughput in the hottest kernel in inference. No performance numbers are given.

**Why It Matters**

Flash attention is the main cost center of LLM serving, and local inference runs on AMD as well as NVIDIA. A rare wrong-result race becomes a rare wrong token, which is nearly impossible to debug from model output. If you write kernels, a race detector in CI is cheaper than three weeks of flaky tests.

**Go Deeper**

- [llama.cpp PR #30103, fix race in flash_attn_ext_f16_process_tile (primary source)](https://github.com/ggml-org/llama.cpp/pull/30103)
- [llama.cpp issue #29314, fattn failure on gfx1201](https://github.com/ggml-org/llama.cpp/issues/29314)

---

## 4. three.js: WebGPU Storage and Uniform Buffer Nodes Stopped Retaining Replaced Attributes and CPU Arrays

**Category:** Web Graphics / GPU

**The Technical Why**

PR #34939 (WilliamLiu-1997, fixes issue #34938, merged by Mugen87 Oct 9, milestone r187) and its follow-up #34941 (same change for `NodeUniformBuffer`, merged Oct 9) fix a leak. The issue title is "NodeStorageBuffer: Retains replaced attributes and CPU arrays." The author's description: "Don't store the initial attribute in `NodeStorageBuffer`'s base fields. Its public getters already read the current node value. Read `Buffer.byteLength` through the buffer getter so it follows replacements too." Validation used `WeakRef` plus forced GC to prove the old attribute and array become collectable, and 1530 unit tests passed. By inference (the pages show no diff), the base class kept a field pointing at the first attribute, so swapping in a new one left the old one, and its CPU-side typed array, reachable.

What is hard: GPU resources have two copies of the data, a JS typed array and a GPU buffer, and a stale reference keeps the large JS copy alive even after the GPU one is replaced. It is invisible in a short demo and shows up as slowly growing memory in long sessions. The fix also corrected `byteLength`, which was read from the old reference, so it did not follow replacements. Bundle effect: WebGPU builds shrank by 16 B on #34939 and 85 B on #34941.

**Why It Matters**

Compute-heavy WebGPU apps (particles, simulation, GPGPU) resize or swap storage buffers while running. Leaked arrays hit mobile Safari and Chrome tabs first, where memory limits kill the page. For a node-based shader editor, any node that caches "the first value" instead of reading the current one will do the same.

**Go Deeper**

- [three.js PR #34939, NodeStorageBuffer stale attribute references (primary source)](https://github.com/mrdoob/three.js/pull/34939)
- [three.js PR #34941, NodeUniformBuffer follow-up](https://github.com/mrdoob/three.js/pull/34941)

---

## Thread to Watch

Silent wrong answers in infrastructure: a hash index that drops rows (Postgres), a shared-memory race that flips a token (llama.cpp), and leaked buffers (three.js). Tomorrow, watch for the PG minor-release announcement that carries 173cf85, and whether the AMD flash attention fix gets a clean racecheck run.
