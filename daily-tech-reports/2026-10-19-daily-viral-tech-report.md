# Daily Viral Tech Report | 2026-10-19

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. I did not use news sites. Skipped as thin: llama.cpp #28277 (`--connect` p2p server access; a feature, little mechanism on the page I read) and vLLM #62547 (pre-commit check for `trust_remote_code=True`; small and local).

---

## 1. llama.cpp: The MoE Expert Cache Moves From LRU to a 50/50 Segmented LRU, With a Regression Report Already Attached

**Category:** AI / ML Infrastructure / Memory Hierarchy

**The Technical Why**

PR #30637 (author eapache, merged Oct 19 by CISC as 33d6135, approved by am17an) replaces the plain LRU policy of the MoE expert cache with a segmented LRU split 50/50: an older, stable half for long-lived experts and a younger, volatile half for recent ones. The author compares it to generational garbage collection. The cache keeps hot experts on the GPU while the full set lives in host RAM.

What is hard: in a mixture-of-experts model each token activates a different subset of experts, so a scan of one-off experts can evict the ones that every token needs. Plain LRU is exactly weak to that scan. The author says an AI agent tested LFU, CPU fallback for rare experts and hybrids on two GPUs (RTX 3060 on a slow PCIe link, RTX 4500 on a fast one) and two models, and that 50/50 SLRU won on gain and simplicity: about 1 to 5 percent over LRU, no exact figures on the page. The cost showed up fast. A user reported GLM 5.3 Flash dropping from about 15 to about 12.5 tokens/s with `--n-cpu-moe 40 --moe-cache-mib 4500`. The author's explanation, which they call plausible but unconfirmed: an expert is about 10 MiB, so 4500 MiB holds about 450, a token touches about 300, and halving the cache leaves neither half able to hold one token's working set, so it churns. The page does not confirm that this PR caused the drop.

**Why It Matters**

Running a big MoE on a consumer GPU is a caching problem, not a compute problem, and the policy decides tokens per second. The lesson for anyone building caches: a policy that wins on average can lose badly when the cache is small relative to the per-request working set. Expect a size threshold, a warning or a fallback to plain LRU (all three were floated on the page).

**Go Deeper**

- [llama.cpp PR #30637, Replace MoE LRU cache with a 50/50 SLRU cache (primary source, includes the regression thread)](https://github.com/ggml-org/llama.cpp/pull/30637)
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

---

## 2. three.js: WebGPU Instanced Meshes Now Share Shader Builds, Cutting One Project From 512 Node Builds to 215

**Category:** Web Graphics / GPU

**The Technical Why**

PR #35007 (author mrdoob, merged Oct 19 by sunag as ead726c, milestone r187) stops `InstancedMesh` from adding its uuid to the render-object cache key. Before, every instanced mesh forced its own node build and pipeline. Now `referenceAttribute()` binds `instanceMatrix`, `instanceColor` and the previous-frame matrices (used for motion vectors) by looking them up on the rendered object at draw time, so the compiled shader no longer embeds per-object data. Meshes are keyed by the kind of instance data they use.

What is hard: sharing a compiled pipeline means the object-specific state must come in from outside it, and every path that used to be "rebuild on change" must now be detected instead. A reviewer (Mugen87) found that replacing `instanceColor` was not noticed; the fix makes `RenderObject` detect replaced bound attributes. Builds that still embed object data are flagged `objectSpecific` and are not shared (geometries with no room for more vertex buffers, instanced morph targets). Author's numbers on their own project: node builds 512 to 215, pipelines 444 to 155, uploads 3.7 MB/frame to 480 KB/frame. Bundle cost: about +700 B gzip on the WebGPU builds. At merge, sunag listed open follow-ups, among them that the shared build retains the first mesh's arrays.

**Why It Matters**

Pipeline creation is the expensive, hitching part of WebGPU startup and of scenes that add meshes at runtime. Fewer pipelines means shorter load time and less GPU memory, and the upload drop matters on mobile. Anyone building an editor that spawns many similar objects (a node-graph shader tool included) faces the same trade: shared compiled code, per-object data bound outside it.

**Go Deeper**

- [three.js PR #35007, WebGPURenderer: Share builds between instanced meshes with referenceAttribute() (primary source)](https://github.com/mrdoob/three.js/pull/35007)
- [three.js repository](https://github.com/mrdoob/three.js)

---

## 3. vLLM: Freed Sliding-Window KV Blocks Were Still Read by Attention, and Masked NaN Poisoned the Output

**Category:** AI Serving / KV Cache

**The Technical Why**

PR #60767 (author stu-cao, merged Oct 19 by ZJY0516 as e1f3141, approved by TheEpicDolphin) fixes a stale-pointer bug. The sliding-window KV manager frees blocks that fall wholly outside a request's window, but the worker's persistent block table keeps the old block IDs. Attention kernels load KV in tiles, so with 64-token pages the tile at the window start can include a page wholly below the window. That page is masked, yet masked NaN still propagates (0 times NaN is NaN). Once a freed block is reused, its contents can decode as NaN and the request's attention turns NaN. The fix replaces every sliding-window entry wholly below the window with the null block, only in the forward-pass copy of the table, on the GPU, with no host sync.

What is hard: the masking looked correct, so the bug only appears when a freed block is reused and holds bytes that decode to NaN. It is data-dependent and shows up as a quality drop, not a crash. Per the PR, in issue #60560 a DFlash2 drafter on B300 at 32 concurrent sessions saw draft acceptance fall to 27 to 29 percent; with the PR it is 41.21 percent against 39.61 percent with the hybrid KV cache manager off. Caveats on the page: the V1 runner is not covered, context parallelism is excluded, the fix assumes the null block holds no NaN, and validation so far covers one drafter.

**Why It Matters**

Long-context and hybrid-attention models rely on freeing KV memory early to fit more concurrent users. A bug that corrupts only under reuse and load is the kind that passes unit tests and fails in production. Whoever serves these models at high concurrency gets quality back without giving up the memory saving.

**Go Deeper**

- [vLLM PR #60767, Null sliding-window block-table entries below the window (primary source)](https://github.com/vllm-project/vllm/pull/60767)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 4. PostgreSQL: The Planner Removed a Placeholder Join Relation That a Nested PlaceHolderVar Still Needed (Backpatched to 14)

**Category:** Databases / Query Optimization

**The Technical Why**

Commit 3138f24 (Richard Guo, reviewed by Ayush Tiwari, 3 files, +184/-2, backpatched through 14) fixes `remove_useless_result_rtes()` in `prepjointree.c`. An RTE_RESULT is a dummy relation the planner can drop, but PlaceHolderVars (PHVs, markers for expressions that must be evaluated at a specific join level) tied to it must then be evaluated elsewhere. For an inner join, they can move up to the join of all sibling relations, allowed only if no sibling needs them earlier. The existing check looked only at join quals and LATERAL RTEs. It missed an outer PHV, evaluated at some but not all siblings, that contains one of the moved PHVs; this can sit in the targetlist and arises when pulled-up expressions get wrapped in PHVs by outer joins or grouping sets.

What is hard: after the move, the outer PHV needs a value only available above its own level, so its relation becomes laterally dependent on its siblings. If one of those references the outer PHV, a relation depends on itself, and `create_lateral_join_info()` hits an assertion failure. The fix adds `find_nested_dependent_phvs_walker()` and refuses the removal in that case, but still allows it when the outer PHV covers all siblings. Per the commit message, the visible symptom is that assertion; it names no other user-facing effect, so how it surfaces on non-assert production builds is not stated.

**Why It Matters**

Queries stacking LEFT JOIN, LATERAL and GROUPING SETS (common from ORMs and generated reporting SQL) can hit this planner path. Backpatching to 14 means every supported branch gets it in its next minor release (inference: the page gives no release date). The pattern worth keeping: an optimization that removes a node is only safe if every consumer of that node's outputs is checked, including the ones hiding in the targetlist.

**Go Deeper**

- [postgres commit 3138f24, Don't remove an RTE_RESULT needed by a nested PlaceHolderVar (primary source)](https://github.com/postgres/postgres/commit/3138f24)
- [Mailing list discussion linked from the commit](https://postgr.es/m/CAMbWs4_hLOsW8_SePp_3EmYTLX8Oc4zEZEvypyPRszpoMhDdGQ@mail.gmail.com)

---

## Thread to Watch

Cache policy versus working-set size: llama.cpp's SLRU regression report (#30637) and vLLM's freed-block reuse bug (#60767) are the same shape, a memory-saving trick that is right on average and wrong at the edge. Tomorrow, watch whether llama.cpp adds a small-cache fallback or reverts, and whether the follow-ups sunag listed for three.js #35007 land before r187.
