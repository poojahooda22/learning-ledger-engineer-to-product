# Daily Viral Tech Report | 2026-10-02

Every story below is checked against a primary page on GitHub (the source family reachable in this run). The pages print month and day reliably; the year the page reader showed was inconsistent, so I give dates as month and day only. Where the notes do not explain a mechanism, I label my explanation as inference. Rust 1.99.0, PyTorch 2.14.1, vLLM v0.30.0 and three.js r186 were covered in earlier reports, so they are skipped. Stories 1 and 3 are from Sep 30 to Oct 2, and story 4 is a spec in progress from Sep 21 to 22, the best web-graphics item I could verify.

---

## 1. GitHub CLI v2.102.0: Four Security Fixes, Three of Them in Supply-Chain Verification and File Handling

**Category:** Developer Tooling (security and supply chain)

**The Technical Why**

gh v2.102.0 (Sep 30) fixes four advisories. Two are in `gh attestation verify`, the command that checks a build's signed provenance. `--source-ref` was compared case-insensitively, so an attestation from branch `Release` could satisfy a policy written for `release` (GHSA-4mq3-hpgx-9cx8). `--signer-workflow` was matched only against the start of the certificate identity, so a workflow whose path merely began with the pinned value passed (GHSA-wjmr-j3rp-mh2g). The other two: download commands (`gh release download`, `gh run download`, `gh attestation download`) could write remote content through a symbolic link to a file outside the intended folder (GHSA-39wj-f2f4-978v), and `gh skill search` passed a repo path from search results to `gh skill install` without an option separator, so a hostile result could inject installer flags (GHSA-qcwj-mr2r-2cx7).

Why it is hard: a verifier is a string comparison against a security policy, and "starts with" or "equals ignoring case" is almost the right check. Each bug is one missing exactness rule. The symlink bug is the classic check-then-write race with the filesystem: the path you validated is not the file you write. The skill bug is argument injection, fixed by the `--` convention that ends option parsing.

**Why It Matters**

Teams that gate deploys on attestations (SLSA-style provenance) were trusting a check looser than the policy they wrote. Anyone running `gh` in CI or on a laptop should update. Winners are teams that pin by exact ref and exact workflow identity and treat their verifier as code that needs tests too.

**Go Deeper**

- [GitHub CLI v2.102.0 release notes (primary source)](https://github.com/cli/cli/releases/tag/v2.102.0)
- [GitHub CLI repository](https://github.com/cli/cli)

---

## 2. PostgreSQL Master: Autovacuum Leaders Stop Using Stale Cost Limits, and VACUUM Reports Progress Per Index

**Category:** Systems and Databases

**The Technical Why**

Two commits landed Oct 1 on master. Commit 42e96cf (Masahiko Sawada) fixes parallel autovacuum: a previous change made the leader push cost-based delay settings (the throttle that keeps vacuum from saturating I/O) to workers only inside `vacuum_delay_point()`. But a leader blocked in `WaitForParallelWorkersToFinish()` never reaches that point, so config reloads and worker-count changes sat unseen until the workers finished. The fix refreshes cost parameters on every wakeup of that wait loop and sets latches on balanced workers when the count changes so a waiting leader wakes up. Commit 1378aa1 (Michael Paquier) adds `current_index_relid`, `index_blks_total` and `index_blks_done` to `pg_stat_progress_vacuum`. Only B-tree indexes report block counts today (others show 0), and parallel workers appear as their own rows with the same `relid`.

Why it is hard: the cost budget is shared across all autovacuum workers, so each worker's slice shrinks or grows when another starts or stops, and a blocked process cannot update its slice. Inference: this is why the fix uses latches, the wake-up primitive Postgres processes already sleep on, instead of polling.

**Why It Matters**

Vacuum on a table with many indexes can run for hours, and until now the progress view showed only heap activity during the index phase. Operators get a real answer to "which index is it stuck on and how far along," and big parallel-autovacuum setups get throttling that obeys the settings they changed. These are master-branch commits, not a released version.

**Go Deeper**

- [Commit 42e96cf, refresh autovacuum costs while waiting for parallel workers (primary source)](https://github.com/postgres/postgres/commit/42e96cf2fe095c822f76bc94669ea875cb1e3351)
- [Commit 1378aa1, per-index progress in pg_stat_progress_vacuum](https://github.com/postgres/postgres/commit/1378aa13430e264a990a18597e9d8fdae740927e)
- [PostgreSQL repository](https://github.com/postgres/postgres)

---

## 3. llama.cpp b11344 to b11352: A Buffer Allocation Interface for Multi-Device Splits, and Q2_K/Q3_K Quantization on Qualcomm Hexagon NPUs

**Category:** AI / ML (local inference)

**The Technical Why**

llama.cpp ships a numbered build per merged change, and b11342 to b11352 all landed Oct 2. b11351 (PR #23671) adds `alloc_buffer_n` and `get_alloc_size_n` to ggml's buffer type interface. Before, `ggml_backend_meta_alloc_ctx_tensors_from_buft` was described in the PR as a temporary workaround tied to the meta backend (the layer that splits one model across several devices, as in tensor parallelism). Now a backend receives a list of tensors and decides itself how they map to one or more sub-buffers; default handling covers the simple case, the meta type creates per-device sub-contexts, and CPU and Metal get NULL implementations. One reviewer cited Hexagon, which needs different tensors in different sub-buffers. In the same window, b11345 (PR #29717, co-authored by Qualcomm's Max Krasnyansky) adds `q2_k` and `q3_k` quantization support to the Hexagon backend, and b11344 fixes two broken Volta FlashAttention cases in CUDA.

Why it is hard: K-quants pack weights in blocks with per-block scales, and the dot-product kernel must unpack them in the NPU's own vector instructions. Inference: Hexagon likely needs separate buffers because the NPU can only address memory it has mapped, so allocation cannot be one flat malloc.

**Why It Matters**

Phone NPUs are the biggest installed base of inference hardware, and 2 and 3 bit weights are what make a multi-billion parameter model fit in phone RAM. A clean allocation interface is what lets new backends and multi-GPU splits stop living as special cases. Winners are on-device apps; the notes give no speed numbers, so treat the NPU gain as unmeasured.

**Go Deeper**

- [llama.cpp PR #23671, alloc_buffer_n (primary source)](https://github.com/ggml-org/llama.cpp/pull/23671)
- [llama.cpp b11345 release, Hexagon q2_k and q3_k](https://github.com/ggml-org/llama.cpp/releases/tag/b11345)
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

---

## 4. WebGPU Spec: Subgroup Matrix Gets a Direct3D 12 Mapping, and Uniformity Assertions Are Proposed for WGSL

**Category:** Web Graphics and GPU

**The Technical Why**

Two open gpuweb pull requests moved Sep 21 to 22. #10964 (alan-baker, approved by both reviewers) updates the subgroup-matrix proposal for D3D12: it adds the API requirements and mapping, plus `minSubgroupSize` and `maxSubgroupSize` on `GPUSubgroupMatrixConfig`, and validation that a chosen subgroup size is inside the config's range. #10959 proposes uniformity assertions for WGSL (it references issue #6955; its notes in the PR page do not describe the mechanism, so I do not claim one). Inference from the name: WGSL already analyzes whether control flow is uniform across a workgroup, because barriers and derivatives are only valid when all threads reach them, and an assertion would let a shader author state "this point must be uniform" and get a compile error otherwise.

Why it is hard: subgroup matrix is hardware matrix multiply (tensor-core style) exposed through a portable API. Vulkan, Metal and D3D12 each offer different shapes and subgroup sizes, so the spec must expose only what every backend can honor and still validate it on the CPU side.

**Why It Matters**

Matrix-multiply speed in the browser decides whether on-device ML and heavy compute shaders are practical on Windows, the largest desktop GPU platform. These are proposals, not shipping browser features, so nothing to adopt today.

**Go Deeper**

- [gpuweb PR #10964, subgroup-matrix for D3D12 (primary source)](https://github.com/gpuweb/gpuweb/pull/10964)
- [gpuweb PR #10959, uniformity assertions proposal](https://github.com/gpuweb/gpuweb/pull/10959)
- [gpuweb repository](https://github.com/gpuweb/gpuweb)

---

## Thread to Watch

Verifier strictness: gh's attestation bugs (case-insensitive ref, prefix-matched workflow) are the kind a policy test suite catches. Also on the radar, per a third-party aggregator I have not verified against a GitHub primary page: GitHub Enterprise Cloud with data residency stops accepting TLS clients that offer only X25519 on Oct 7 ([releasebot summary](https://releasebot.io/updates/github)). Check any old client or proxy of yours before then.
