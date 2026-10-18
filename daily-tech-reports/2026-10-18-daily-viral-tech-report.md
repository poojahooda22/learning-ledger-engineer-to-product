# Daily Viral Tech Report | 2026-10-18

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. Sources outside GitHub are blocked from this environment, so no story rests on a news site. Skipped as thin: llama.cpp #30602 (CUDA fusion of a Qwen4Exp chain; no description worth learning from) and three.js #35073 (orthographic GTAO view direction; small and local).

---

## 1. vLLM: Structured-Output Masks in Speculative Decoding Are Now Driven by GPU-Side Logit Counts, Not a CPU-Built Mapping

**Category:** AI Serving / Constrained Decoding

**The Technical Why**

PR #52477 (author LucasWilkinson, merged Oct 18 as 6540d5c, approved by sfeng33 and yzong-rh, 183 checks passed) changes how grammar masks reach the sampler when speculative decoding is on. Per the PR, the scheduler now emits a fixed-width block of `num_spec_tokens + 1` mask rows per structured-output request. The MRV2 kernel launches once per grammar request and applies only the positions that the device-side `cu_num_logits` says are active. That deletes `_build_grammar_mapping` and the worker's dependence on `cu_num_logits_np` and scheduled draft counts.

What is hard: with speculative decoding, each request verifies a variable number of draft tokens per step, and adaptive verification can set that budget to zero on some steps (a bonus-only step). The grammar mask for position k is only valid if the grammar state advanced through positions 0 to k-1. The old design had the CPU predict which mask row maps to which logit row, so any disagreement with what the GPU actually verified applied the wrong mask. The new design lets the GPU, which knows the truth, pick the rows. Cost, per a reviewer: padding short requests to fixed blocks sends a few extra bytes. Evidence: a smoke test with ngram_gpu and async scheduling failed 19 of 24 requests on main and 0 of 24 with the PR; an H100 run with a forced zero-budget every 3rd step matched main's greedy outputs for 96 of 96 requests. The PR reports no performance numbers.

**Why It Matters**

Agents depend on JSON-schema and tool-call outputs that must parse, and speculative decoding is how serving stacks get the latency down. Making the two features compose without a CPU/GPU bookkeeping handshake removes a class of silent corruption (inference: wrong-mask steps would produce grammar-invalid tokens, not crashes). Whoever serves structured outputs at speed wins; the general lesson is to let the component that holds the ground truth, here the GPU, own the index.

**Go Deeper**

- [vLLM PR #52477, Drive grammar masks from GPU logit counts (primary source)](https://github.com/vllm-project/vllm/pull/52477)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 2. llama.cpp: CUDA MMVQ Retuned for Low-Bit Quants, Up to 1.73x Token Generation on Older and Smaller GPUs, Almost Nothing on the Newest

**Category:** AI / ML Infrastructure / GPU Kernels

**The Technical Why**

PR #30517 (author am17an, merged Oct 17 as 2744e48, approved by pwilkin and IMbackK, reviewed by JohannesGaessler) changes the launch parameters of the CUDA MMVQ kernel (quantized matrix times vector, the core of token generation) for low-bit types: `calc_nwarps`, `calc_rows_per_block`, `ggml_cuda_should_use_mmvq` and the `MMVQ_PARAMETERS_GCN` table. A second commit forces `nwarps = 1` on AMD GCN for Q3_K at 2 or 3 columns and Q6_K at 4 columns, after testers found regressions.

What is hard: the kernel is memory-bound (a reviewer's description), so speed depends on how many warps and rows share each block's loads, and the best split differs per quant type, batch width and GPU generation. The author measured a chain of 60 distinct 4096x14336 matrices so weights are not in L2, and used q4_K and q8_0 (unchanged) as the noise floor. Results, base to PR, tg128: RTX 3090 Qwen3.5-9B Q3_K_S 81.04 to 126.22 t/s (1.56x); A16 Q3_K_S 14.43 to 24.95 (1.73x); RTX 4090 1.08x; RTX PRO 6000 1.02x; GB10 1.01x. On a Mi50 the first version made Q3_K at n=2 7.5% slower and Q6_K at n=4 11.6% slower, and the `nwarps = 1` rule turned them into +27.7% and +10.5%. Slowdowns under 5% were treated as noise.

**Why It Matters**

Low-bit quants (Q2_K, Q3_K) are how big models fit on consumer cards, and those cards are the older, smaller ones where this gain lands. Newer GPUs with more bandwidth and cache already hid the inefficiency. The lesson for anyone shipping GPU code: one set of launch constants is a bet on one hardware generation, so keep a per-architecture table and a benchmark matrix across vendors before merging.

**Go Deeper**

- [llama.cpp PR #30517, CUDA: faster MMVQ for low-bit types (primary source, includes the benchmark tables)](https://github.com/ggml-org/llama.cpp/pull/30517)
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

---

## 3. three.js: BatchedMesh Resize Left Velocity (Motion Vector) Data at the Old Size, Throwing a RangeError Every Frame

**Category:** Web Graphics / GPU

**The Technical Why**

PR #35075 (author HEOJUNFO, merged Oct 18 by Mugen87 as 0978e2b, milestone r187) fixes a stale cache. `BatchedMesh.setInstanceCount()` replaces `_matricesTexture` with one sized for the new count. `getPreviousNode()` caches the previous-frame matrices (used for velocity output) but never checked that the cache still matched. After the material was rebuilt with velocity enabled, the old-size texture survived, and when the count grew the per-frame copy threw "RangeError: offset is out of bounds". The fix stores the source `_matricesTexture` in the cache and, on mismatch, disposes the old previous texture and recreates it.

What is hard: velocity is the position this frame minus last frame, so it needs a second copy of every instance matrix that stays aligned with the live one across resizes. The author's test (3 instances, max 4, rendered through `mrt({ output, velocity })`, then `setInstanceCount(64)`) threw on frames 3 to 7 before the fix. On WebGL, static frames showed velocity of 245 px before and 0 after. Cost on the WebGPU build: +99 B minified, +26 B gzip; core and WebGL unchanged. Note the page says the description was produced with Claude Code, so the evidence is the author's own test page plus the e2e job (no diff for `webgpu_mesh_batch` and `webgpu_shadowmap_array`).

**Why It Matters**

Velocity buffers feed motion blur and temporal anti-aliasing, so a wrong one shows as smeared static objects (245 px of fake motion). Dynamic instancing, such as a particle or crowd system that grows, is exactly where this bites. The pattern repeats from earlier this week: any cache derived from a resizable resource needs the resource's identity in its key, or it goes stale silently.

**Go Deeper**

- [three.js PR #35075, Batch: Recreate the previous matrices texture when the instance count changes (primary source)](https://github.com/mrdoob/three.js/pull/35075)
- [three.js repository](https://github.com/mrdoob/three.js)

---

## 4. PostgreSQL: The Planner Treated Outer Joins as Swappable When Strictness Came Only Through a Lateral Reference (Backpatched to 16)

**Category:** Databases / Query Optimization

**The Technical Why**

Commit 7787aec (Richard Guo, reviewed by Tender Wang, 3 files, +54/-0, backpatch through 16) fixes `make_outerjoininfo()` in `initsplan.c`. The planner applies "outer join identity 3" (reordering two outer joins) only if the join clause is strict for the lower join's right side, meaning a NULL there makes the clause not true. Strictness was computed by `find_nonnullable_rels()`. When it flows through a PlaceHolderVar (PHV, a marker for an expression that must be evaluated at a specific join level), that function can name relations the clause only references laterally, while `pull_varnos()` reports only where the PHV is evaluated. The two disagreed, so joins were marked commutable when they were not. The fix intersects `strict_relids` with `clause_relids`, so the planner ignores relations the clause is strict for but does not reference.

What is hard: join reordering is only correct if every transformation preserves NULL-extension semantics, and PHVs and lateral references each carry their own bookkeeping that must agree. Per the commit message, the symptom is an assertion failure in `standard_join_search()` on assert builds and "variable not found in subplan target lists" errors on production builds. The regression test is an EXPLAIN of a left join, a lateral left join with a PHV on `t2.f1`, another left join on `s.x`, and an inner join with `s.y is not null`.

**Why It Matters**

This is a query that fails with a planner error, not a wrong answer, on real builds, which is the better failure mode but still an outage for the affected query shape (ORMs and views that stack LATERAL and LEFT JOIN produce these). Teams on 16 or newer get the fix in the next minor release (inference: the commit page gives no release date). The lesson: the optimizer is a pile of set-valued facts computed by different functions, and bugs live where two functions answer "which relations matter" differently.

**Go Deeper**

- [postgres commit 7787aec, Fix bogus outer-join commutation with lateral refs in PlaceHolderVars (primary source)](https://github.com/postgres/postgres/commit/7787aec)
- [Mailing list discussion linked from the commit](https://postgr.es/m/CAMbWs49xvh4-uPv3rxWkk_rtMT1ic+psHvEAxaFEXu2Rf-32Tw@mail.gmail.com)

---

## Thread to Watch

Fixed-shape, GPU-owned indexing: vLLM moved mask selection onto the device to stop CPU/GPU disagreement, and llama.cpp keeps retuning per-GPU launch tables. Tomorrow, watch whether vLLM #52477 gets follow-ups for #52620 and #49694 (which reviewers said it should fix), and whether three.js r187 gets a date.
