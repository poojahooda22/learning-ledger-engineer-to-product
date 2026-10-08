# Daily Viral Tech Report | 2026-10-08

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. General web search returned nothing useful for Oct 8, so I worked from merged-PR and commit lists. Items from the vLLM v0.31.0 release (Oct 5) and Postgres REPACK (Oct 7) were skipped because the 10-07 report already covered them.

---

## 1. llama.cpp: The MoE Expert Cache Now Spans Multiple GPUs, Decode Goes From 34.8 to 60.4 t/s on Two RTX 4090s

**Category:** AI / ML Inference

**The Technical Why**

llama.cpp PR #30112 (am17an, approved by ggerganov and pwilkin, merged Oct 8) extends yesterday's single-device MoE expert cache so each GPU holds its own cache of expert weights that live in host RAM. Cache size is now one global `--moe-cache-mib` value split by `--tensor-split` (commit 5b68503). Slots are shared across layers that have the same expert type, which the author says raises hit rate by about 5%. On 2x RTX 4090 with Qwen3.8-Flash-Next Q4_0 (65 GiB of experts): decode 34.78 t/s on master, 60.38 t/s (1.74x, 85.9% hit rate) with a 6,544 MiB cache, 64.39 t/s (1.85x, 95.8%) with `-cmoe` and 15,000 MiB.

What is hard: the cache only pays when the hit rate beats PCIe cost. Prompt processing got slower in the same table (440.7 to 402.0 and 321.5 t/s). A user reported GLM 5.3 Flash dropping from 33 to 16 t/s with 43.5% hits for small batches and 6.9% for large ones, with no maintainer reply on the page. Another user noted that `-ts 1,0` gives the second GPU a 0 MB cache even with free VRAM. The page does not describe the eviction policy.

**Why It Matters**

Big MoE models on two consumer GPUs plus system RAM is the cheapest local setup, and this makes it about 1.7x to 1.85x faster for decode. The lesson for anyone building a cache tier: a gain on one model and 11 prompts is a hypothesis, and other models can lose.

**Go Deeper**

- [llama.cpp PR #30112, multi-GPU MoE cache (primary source)](https://github.com/ggml-org/llama.cpp/pull/30112)
- [llama.cpp PR #29887, the single-GPU cache it extends](https://github.com/ggml-org/llama.cpp/pull/29887)

---

## 2. three.js: `range()` Stops Uploading Random Arrays and Hashes `instanceIndex` in the Shader Instead

**Category:** Web Graphics / GPU

**The Technical Why**

three.js PR #34897 (mrdoob, approved by Mugen87, merged by sunag Oct 8, milestone r187) rewrites TSL's `RangeNode`. It used to allocate `count x 4` `Math.random()` floats on the CPU and upload them as a uniform buffer or a hidden `__range` vertex attribute. Now the shader hashes `instanceIndex` with a per-object or per-node `seed` uniform. The PR lists what the old design broke: programs baked in `object.count` (raising `mesh.count` later could read out of bounds), instanced meshes with different counts could not share a compiled program, rebuilding a material re-rolled the values, `range()` threw in compute shaders where `builder.object` is null, and the hidden attribute mutated shared geometry. The diff is 30 insertions and 66 deletions, and the bundle bot shows WebGPU builds 238 B smaller raw.

What is hard: a hash is deterministic, so the output now depends on the seed. After merge, bhouston reported that the seed uses a global node counter, so adding any module-level node shifts every `range()` value and broke the `webgpu_particles_soft` and `webgpu_tsl_galaxy` screenshots. sunag opened draft PR #34920 to make values independent of node creation order. A related fix, PR #34919 (MikeFernandez-Pro), makes WebGPURenderer clear `texture.updateRanges` after upload; the description says it previously ignored them and re-uploaded whole textures while the array grew without bound.

**Why It Matters**

Moving per-instance data from CPU buffers into shader math makes the compiled program independent of instance count, which is what lets shader programs be shared and cached. For a node-based shader editor that compiles to code, this is the rule to copy: generated programs should not bake in runtime data sizes. Also expect a run-to-run reproducibility trade-off: seeds must not depend on build order.

**Go Deeper**

- [three.js PR #34897, RangeNode hash (primary source)](https://github.com/mrdoob/three.js/pull/34897)
- [three.js PR #34919, clear texture update ranges](https://github.com/mrdoob/three.js/pull/34919)

---

## 3. PostgreSQL 19: Foreign-Key Fast Path Gets Two Correctness Fixes, Read-Only Transactions and Command IDs

**Category:** Databases / Systems

**The Technical Why**

Two commits by Amit Langote, both backpatched through 19. Postgres checks a foreign key by running `SELECT ... FOR KEY SHARE` on the referenced row through SPI, the internal query interface. Version 19 adds a fast path (`ri_FastPathCheck` in `ri_triggers.c`) that locks the referenced row directly. c993d9f ("Refuse RI fast-path row locks in read-only transactions") fixes a gap: the SPI path is refused in a read-only transaction unless the table is temporary, but the fast path skipped that check, so a deferred check firing at COMMIT after `SET TRANSACTION READ ONLY` could succeed. The fix calls `PreventCommandIfReadOnly("SELECT FOR KEY SHARE")` and adds regression tests for both permanent and temporary tables. 061065e ("Lock RI fast-path rows as of the scan snapshot's command ID") passes `snap->curcid` instead of the current command ID. If a user-defined equality function advances the command counter during an index scan and updates the referenced row, the fast path hit "attempted to lock invisible tuple" where the SPI path correctly reported a foreign key violation. The commit itself calls this mostly hardening.

What is hard: a fast path must reproduce every observable behavior of the slow path it replaces, including error cases. Each shortcut that skips the executor also skips checks the executor did implicitly.

**Why It Matters**

Foreign-key checks run on every insert into a referencing table, so a faster path helps write-heavy schemas. These commits show the price: the speedup is only safe once every executor-level guard is rebuilt by hand. Whether the fast path is also faster in numbers is not stated on these pages.

**Go Deeper**

- [Postgres commit c993d9f, read-only refusal (primary source)](https://github.com/postgres/postgres/commit/c993d9f)
- [Postgres commit 061065e, snapshot command ID (primary source)](https://github.com/postgres/postgres/commit/061065e)

---

## 4. llama.cpp CUDA: Top-k Gets a Four-Algorithm Dispatcher, Prefill Up 1.6x on GB300

**Category:** GPU Kernels / AI Inference

**The Technical Why**

llama.cpp PR #28713 (praneshgo, merged by ORippler Oct 8, approved by am17an, gaugarg-nv and ORippler) rewrites how the CUDA backend picks a top-k algorithm. Bitonic sort handles short rows (now up to a padded 1024 columns while rows fit in one wave of SMs). A grid-over-rows radix select handles several long rows and replaces CUB's per-row kernel. CCCL's `DeviceTopK` (CCCL 3.4.3 or later) handles single rows. CUB argsort is the fallback for a single long row. Radix and bitonic work in chunks to bound scratch memory. Measured geomean speedups over 1027 shapes: 295% on RTX 5090 with default CCCL, 3109% with CCCL 3.4.3; an earlier radix-only version on a 34,816-token run cut 1,671,253 kernel launches to 2,329 (5,761.8 ms to 941.8 ms). End to end on Qwen3.8-Flash-Next, GB300 prefill goes 1755.8 to 2852.8 t/s (1.625x).

What is hard: no single top-k algorithm wins across row counts and widths, so the work is the dispatch boundary, and review caught a regression where bitonic replaced `DeviceTopK` for single short rows. Decode is unchanged (100.08 vs 99.89 t/s) because the author notes top-k is not on that model's batch-1 decode path.

**Why It Matters**

Prefill speed sets time to first token for long prompts. The pattern generalizes to any GPU library: ship several kernels and benchmark the selector as carefully as the kernels. Gains are model and hardware specific, and the "3109%" figure is a geomean over synthetic shapes, not a user-visible speedup.

**Go Deeper**

- [llama.cpp PR #28713, CUDA top-k algorithm selection (primary source)](https://github.com/ggml-org/llama.cpp/pull/28713)

---

## Thread to Watch

Fast paths and caches that must prove they match the slow path: Postgres's FK fast path, llama.cpp's MoE cache, three.js's hashed `range()`. Tomorrow, watch three.js PR #34920 (order-independent `range()` seeds) and whether the llama.cpp GLM 5.3 Flash slowdown on PR #30112 gets a maintainer reply.
