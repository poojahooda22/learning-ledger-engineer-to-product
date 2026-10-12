# Daily Viral Tech Report | 2026-10-12

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. All four items merged on Oct 12. Numbers are the authors' own, on their own machines, not independent benchmarks. I skipped the other llama.cpp builds of the day (Metal, SYCL, Hexagon fixes) and Postgres docs commits to keep the spread wide.

---

## 1. three.js: TSL Shader Function Layouts Deduplicated, 682 Code Generations Down to 33

**Category:** Web Graphics / GPU

**The Technical Why**

PR #34848 (sunag, merged Oct 12, milestone r187, fixes issue #34843) changes how TSL (three.js Shading Language) function layouts are pulled into a generated shader. Before, an include was added once per call site, so a nested function was walked once per call path on every build. Now each include is added once and each function generates its code once per builder. The author says the generated shaders are byte-identical to before. The PR also adds default parameters: `scale( uv().x )` compiles to `fn0( nodeVarying0.x, 2.0 )`, and if a default depends on another parameter the call is expanded inline. Before, a missing input logged an error and passed `float(0)`. `FunctionOverloadingNode` now matches by vector type, so `color()` matches a `vec3` overload.

What is hard: a node graph compiles to one shader string, and shared helpers (noise, color math) can be reached by many paths. Without a visited set, work grows with the number of paths, not the number of distinct functions, which can blow up on deep nesting. The author's numbers: `FunctionNode.generate()` calls per build fell from 682 to 33 with all MaterialX noises in one shader, and a nested layout at depth 12 went from 3.87 ms to 0.40 ms per build. Cost: +822 B gzipped on the WebGPU build; core and WebGL unchanged. A benchmark page, `examples/webgpu_tsl_layout_benchmark.html`, compares build time, generate calls, shader hashes and pixel output between `build/` and `src/`.

**Why It Matters**

Build time is paid every time a user edits a node graph or a material is created at runtime, so a compiler that revisits shared subgraphs makes live editing feel slow. For any node-based shader tool, the transferable rule is: memoize per builder, and prove the optimization with a byte-identical output check, not just a timer.

**Go Deeper**

- [three.js PR #34848, TSL: Optimize function layouts with include deduplication and default parameters (primary source)](https://github.com/mrdoob/three.js/pull/34848)
- [three.js releases](https://github.com/mrdoob/three.js/releases)

---

## 2. llama.cpp: Padding Attention Keys and Values to 256 Keeps Flash Attention on for Embedding Models (363 ms to 58 ms)

**Category:** AI / ML Infrastructure

**The Technical Why**

PR #30416 (ServeurpersoCom, approved by CISC and ggerganov, merged Oct 12 as commit 358a420, fixes issue #30398) pads the KV dimension of the no-cache attention path to a multiple of 256. Embedding and encoder models run attention without a KV cache, so keys and values were passed at their true length, equal to the token count. Some CUDA flash attention kernels for head size 512, such as the one EmbeddingGemma 2 needs, require a padded length. For most input lengths the op was therefore unsupported on the GPU and fell back to the CPU. The `-fa auto` startup probe runs with one token, saw the unsupported op, and disabled flash attention for the whole model. The fix pads the mask the way `llama_kv_cache::get_n_kv` does and pads K and V with zeros to match. The page does not state how padded positions are kept out of the softmax; presumably the padded mask entries are masked out (inference), and the author says embeddings are unchanged.

What is hard: GPU kernels assume tile-aligned shapes, so a model that works at one length can silently take a slow path at another, and an auto-detect probe run at one token can decide for every length. The author's measurement on CUDA with EmbeddingGemma 2 at 3841 tokens: 363 ms before, 58 ms after, about 6x.

**Why It Matters**

Local and server embedding workloads (search, RAG indexing) are throughput bound, and a silent CPU fallback is invisible in output quality. Anyone shipping inference kernels learns that alignment is a performance contract, and that capability probes must test representative shapes, not the smallest one. Note the PR disclosed AI assistance.

**Go Deeper**

- [llama.cpp PR #30416, graph: pad the KV dimension of the no-cache attention to 256 (primary source)](https://github.com/ggml-org/llama.cpp/pull/30416)
- [llama.cpp releases](https://github.com/ggml-org/llama.cpp/releases)

---

## 3. Rust: New Trait Solver Optimizations Cut the Slowdown vs the Old Solver from 1.49x to 1.31x (diesel)

**Category:** Languages / Compilers

**The Technical Why**

rust-lang/rust PR #163981 (nnethercote, approved by jdonszelmann, opened Oct 8, merged Oct 12 via rollup #164193 as commit ebe0c47, milestone 1.101.0) is a set of small speedups to the next-generation trait solver (`-Znext-solver`, tracked by the Trait System Refactor initiative), aimed at the `diesel` crate. The changes: an early return in `unify_query_var_values` when the two values are equal; `clone_from` instead of `extend` in `canonicalize_param_env`; moving an early-return check ahead of its single call site in `probe_and_consider_param_env_candidate` and skipping default allocations on `Err(NoSolution)`; passing `goal_lang_item` into `fast_reject_assumption` so it is fetched once per loop, not per element; and preinterning more common types (type variables, anonymous bound types, some root-universe placeholders).

What is hard: a trait solver answers goals like "does `T: Iterator`?" thousands of times per crate, and `diesel` is trait-heavy, so constant factors in the hot loop dominate. Local solver instruction counts: 37.9B with the old solver, 56.4B with the original new solver (1.49x), 49.5B with this PR (1.31x). Trade-off, stated by the author: preinterning costs a little at startup, so very short benchmarks like `helloworld` regressed slightly. The post-rollup perf run showed a mean primary regression of 0.3% across 4 benchmarks and 0.25% artifact size growth, with the `perf-regression` label added and no triage on the page. The author also says some ideas came from LLM analysis of profiles, while he wrote the code and text himself.

**Why It Matters**

The new solver fixes long-standing soundness and coherence bugs, but it cannot become the default while it is much slower on real crates. Closing the gap in small measured steps is what makes the switch plausible, and Rust users with heavy generic code benefit on compile time.

**Go Deeper**

- [rust-lang/rust PR #163981, New solver opts for `diesel` (primary source)](https://github.com/rust-lang/rust/pull/163981)
- [Rust compiler team, next-solver tracking via the Trait System Refactor initiative](https://github.com/rust-lang/trait-system-refactor-initiative)

---

## 4. PostgreSQL: EXPLAIN ANALYZE Now Reports Material Node Storage Per Parallel Worker

**Category:** Databases / Systems

**The Technical Why**

Two commits by David Rowley (reviewed by Tatsuya Kawata and Chao Li, Oct 12, master only, no backpatch). First, c6b8af4 changes `tuplestore_get_stats()` from returning a `char *` pointing at the constant "Disk" or "Memory" to filling a `bool *used_disk`. Reason in the message: a pointer to a string constant is meaningless in another process, because the constant lives at a different address in the parallel leader. Then a241bc4 (7 files, +189/-5) lets each parallel worker write its tuplestore stats (peak `maxSpaceUsed`, `usedDisk`) into its own slot of a shared-memory array (`SharedMaterialInfo` in dynamic shared memory), which the leader reads after the workers finish. It adds `T_MaterialState` cases to the estimate, init-DSM, init-worker and retrieve-instrumentation hooks in `execParallel.c`. At node end each worker takes the max of space used and ORs the spill flag, so a rescanned Gather does not overwrite earlier peaks.

What is hard: parallel workers are separate processes, so every statistic must be explicitly placed in shared memory in a form that survives the process boundary: no pointers, fixed layout, sized before workers start. Earlier commit 1eff827 reported Material storage for the leader only, so a Material node under Gather or Gather Merge was missing the worker numbers. The commit page shows no sample EXPLAIN output, so I cannot show the exact new lines.

**Why It Matters**

Material nodes can spill to disk, and a spill inside a worker was invisible when tuning `work_mem` for parallel plans. Rowley chose not to backpatch because new output is feature-like, which is the usual Postgres line between fix and feature.

**Go Deeper**

- [postgres commit a241bc4, Show Material node Maximum Storage for parallel workers (primary source)](https://github.com/postgres/postgres/commit/a241bc4)
- [postgres commit c6b8af4, Change tuplestore_get_stats() API](https://github.com/postgres/postgres/commit/c6b8af4)
- [Discussion thread on the mailing list](https://postgr.es/m/CAApHDvoUWCms-b27MpDKd5_RUFtAnVrxPgN9u3QoQbBihk+S6g@mail.gmail.com)

---

## Thread to Watch

Silent slow paths and invisible costs: a CPU fallback hidden behind a one-token probe (llama.cpp), per-worker spills missing from EXPLAIN (Postgres), and repeated subgraph walks (three.js). Tomorrow, watch whether the Rust next-solver perf run gets triaged and what the next instruction-count numbers say, and whether three.js r187 gets a release date.
