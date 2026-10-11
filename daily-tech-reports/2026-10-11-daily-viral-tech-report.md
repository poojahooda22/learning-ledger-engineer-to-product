# Daily Viral Tech Report | 2026-10-11

Every story below is checked against a primary GitHub page (PR, release or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. The vLLM release is from Oct 5, not the last 24 hours; I kept it because nothing newer in AI infrastructure had a primary page with engineering depth. I skipped the Rust `BTreeMap::Iter::next` fast path (#163521) to keep the spread wide.

---

## 1. three.js: ArrayCamera Uniforms Isolated per Renderer and View Count (WebGPU), 200x Fewer Uploads

**Category:** Web Graphics / GPU

**The Technical Why**

PR #34916 (Ben Houston, approved and merged by Mugen87 Oct 11, milestone r187) fixes a shared-state bug in the node system. The arrays behind `cameraProjectionMatrices`, `cameraViewMatrices` and `cameraPositions` in `Camera.js` were module-level singletons. A second `ArrayCamera` with a different sub-camera count replaced the shared array, so shaders already compiled for the first camera uploaded the wrong matrices and the array length no longer matched what they expected. The PR's example: a 2-view XR camera and a 48-view capture camera could not render in the same app. The final design caches array uniforms in a `WeakMap` keyed by renderer, then sub-camera count, then name, so different counts get separate buffers and shaders with the same count share one. Each cached array has an `onRenderUpdate()` callback that rebinds it to the current camera's sub-cameras, and a last commit stops those callbacks from retaining the `ArrayCamera`.

What is hard: compiled shaders bake in the array length, so one global uniform cannot serve two shapes, but one uniform per shader (the first attempt) multiplies upload work. The author's stress test, 200 distinct shaders through a 48-view camera on an Apple M-series GPU in Chrome: uniform uploads per frame fell from 600 (1.8 MB) to 3 (9 KB), 200x fewer. Frame rate went from about 179 to 214 at 48 views and from about 950 to 1895 at 2 views. These are the author's numbers on one machine, not an independent benchmark. Cost: about +66 B gzipped on the WebGPU build.

**Why It Matters**

Batching many views into one draw (the PR was extracted from #34909, LightProbeGrid baking with ArrayCamera, claimed about 4x faster bakes) only pays off if per-view data is uploaded once, not once per shader. For a node-based shader tool, the lesson is to key shared GPU state by the shape that compiled code depends on, not by a global name.

**Go Deeper**

- [three.js PR #34916, Camera: Isolate ArrayCamera uniforms across shader builders (primary source)](https://github.com/mrdoob/three.js/pull/34916)
- [three.js PR #34909, LightProbeGrid batch capture with ArrayCamera (the motivating change)](https://github.com/mrdoob/three.js/pull/34909)

---

## 2. Rust: `Box` Stops Using `Unique`, First Step Toward Removing It (1.101.0)

**Category:** Languages / Compilers

**The Technical Why**

rust-lang/rust PR #162849 (maxdexh, opened Sep 16, approved by hanna-kruppe and merged Oct 10 via rollup #164117, milestone 1.101.0) replaces the internal `Unique` pointer wrapper inside `Box` with a `(NonNull, PhantomData)` wrapper called `BoxRaw`. It is mostly a mechanical swap; the author calls it only the first step. Planned follow-ups: flatten `Box`'s fields to drop `BoxRaw`, then remove `Unique` entirely. `Box` layout is deliberately unchanged for now, tracked in issue #162850. The page does not say why `Unique` should go, and I will not guess beyond noting that it is a std-internal type.

What is hard: `Box` is a compiler-special type, so a small std change can shift codegen. Miri and mir-opt tests had to be re-blessed. The perf run showed about a 0.4% instruction-count regression on 3 primary benchmarks at first, with max RSS about 2.9% lower; after the rollup, only tiny doc-build regressions on hyper and syn remained, which Kobzol triaged as acceptable, noting `Box` changes often cause small shifts. Binary artifact size dropped from 407.22 MiB to 406.59 MiB. A side bug surfaced: adding a `From<BoxRaw<T>> for NonNull<T>` impl hit a "missing stability attribute" error, and `#[unstable]` then tripped the `ineffective_unstable_trait_impl` lint, worked around with `#[expect(...)]` and filed as #164109.

**Why It Matters**

Language runtimes are paid for in small, reviewed, benchmarked steps. This one shows the process: split a risky refactor so layout stays fixed, measure instructions and memory on a perf server, then land it in a rollup. Rust users see nothing today; compiler maintainers get a simpler `Box`.

**Go Deeper**

- [rust-lang/rust PR #162849, Replace `Unique` in `Box` with a `(NonNull, PhantomData)` wrapper (primary source)](https://github.com/rust-lang/rust/pull/162849)
- [rust-lang/rust issue #162850, Box layout](https://github.com/rust-lang/rust/issues/162850)

---

## 3. vLLM v0.31.0: `vllm preload` Keeps Weights in GPU Memory Across Restarts

**Category:** AI / ML Infrastructure

**The Technical Why**

vLLM v0.31.0 (Oct 5, marked Latest on the releases page) adds `vllm preload` (#56680), which starts a weight-cache daemon that keeps post-quantized weights resident in GPU memory across engine restarts. Later PRs extended it to data parallelism (#57386), MTP draft models (#57312), a `/health` endpoint (#58552) and a readiness wait (#58370). Also in the release: draft-model speculative decoding on Model Runner V2 (#43091) plus custom logits processors there (#56497); FlashMLA attention with the DeepSeek-V4.1 NVFP4 compressed KV cache made the default on SM100 GPUs (#56935); `--max-num-active-seqs` (#56758), which caps how many requests enter RUNNING separately from `max_num_seqs`; and `--trust-request-mm-kwargs` (#58830), which rejects per-request multimodal processor kwargs unless the operator opts in.

What is hard: a restart normally means reloading weights and re-running quantization, which dominates cold-start for large models. A daemon that owns the GPU copy moves that cost out of the engine process. By inference (the release notes do not describe the mechanism), the engine must then attach to memory it did not allocate, so ownership, versioning and cleanup are the tricky parts. Model Runner V2 became the default for all models in v0.29.0, with V1 deprecated and removal targeted for v0.32.

**Why It Matters**

Faster restarts shorten deploys, crash recovery and autoscaling reaction time on expensive GPUs, where idle seconds cost money. The security flag is the same lesson as any server: per-request knobs that reach a media or processor pipeline are an attack surface and should default to off.

**Go Deeper**

- [vLLM v0.31.0 release notes (primary source)](https://github.com/vllm-project/vllm/releases/tag/v0.31.0)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 4. PostgreSQL: `IS DISTINCT FROM` Wrongly Treated as Never NULL by the Planner, Backpatched Through 19

**Category:** Databases / Systems

**The Technical Why**

Commit f5635c2 (Richard Guo, reported by Robert Haas, Oct 10, backpatch through version 19) removes the `DistinctExpr` case from `expr_is_nonnullable()`. Earlier commit c8d2f68 had taught the planner that `a IS DISTINCT FROM b` never returns NULL. The commit message says that is wrong: when neither input is NULL, the result can still be NULL if the underlying equality operator returns NULL, and nothing forbids an operator from doing that. The message warns it "could lead to wrong query results." The same commit reorders a check in `var_is_nonnullable()` so relkind is tested before `has_subclass()`, avoiding a wasted syscache lookup for partitioned tables.

What is hard: planner shortcuts are proofs, and a proof is only as strong as its weakest assumption. By inference from the removed regression test (the page shows no symptom), the affected optimization folds `IS NULL` tests on expressions believed never NULL, for example `WHERE (a IS DISTINCT FROM b) IS NULL`, into a constant-false One-Time Filter. If the belief is false, the planner drops a filter that could pass rows. SQL lets user-defined operators return NULL, which is what breaks the "built-in semantics" assumption.

**Why It Matters**

Wrong answers from an optimizer are silent. A fix that lands in the same cycle as the optimization, with a backpatch, shows the review loop working, but it is a reminder that extensible type systems and aggressive rewrites collide.

**Go Deeper**

- [postgres commit f5635c2, Fix expr_is_nonnullable() for IS DISTINCT FROM (primary source)](https://github.com/postgres/postgres/commit/f5635c2)
- [PostgreSQL docs, comparison functions and operators](https://www.postgresql.org/docs/current/functions-comparison.html)

---

## Thread to Watch

Shared state keyed too coarsely: a global camera array in three.js, an optimizer assumption in Postgres, and a daemon-owned weight cache in vLLM. Tomorrow, watch whether three.js #34909 (the 4x faster LightProbeGrid bakes) lands now that #34916 is in, and the Rust follow-up PRs that flatten `Box` and remove `Unique`.
