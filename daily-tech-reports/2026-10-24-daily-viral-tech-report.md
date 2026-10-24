# Daily Viral Tech Report | 2026-10-24

Every story below is checked against a primary GitHub page (PR, issue or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. Skipped: three.js #35164 (per-sub-camera ArrayCamera culling; a close follow-up to the 10-20 story, and the final opt-in property name was not confirmed on the page), wgpu #10383 (StagingBelt trimming; the page has no measurements) and the Linux 7.4 merge window (I only saw merge titles, not contents).

---

## 1. Redis: Large Sorted Sets Drop Skiplist Plus Dict for a Packed B+ Tree, Cutting Memory by 60.6%

**Category:** Databases / Data Structures

**The Technical Why**

PR #15635 (author antirez, opened Aug 13, merged Oct 23 by sundb into `unstable`) replaces the skiplist plus hash table that backs large sorted sets with a B+ tree ordered by (score, member). Small sets stay listpack. Leaves are single allocations of at most 4095 bytes and 96 elements, holding bit-packed scores (a base value plus small differences), u16 record offsets, one hash-tag byte per member, and member records growing from the end. Inner nodes keep child pointers, the max score below each child and per-child counts, so rank queries are a tree descent. Members of 320 bytes or more get their own allocation so a score change does not copy them. Member lookup uses a separate open-addressing index with 8-slot buckets, each slot an 8-bit tag plus a 16-bit leaf ID (32 bits when needed), resized incrementally.

What is hard: a skiplist spends a pointer array per node and the dict spends another entry per member, so overhead dominates for short members. Packing many members per leaf removes that, but now every insert shifts bytes inside a leaf, ZSCAN needs a new cursor scheme (a reverse-binary counter over bucket groups, as `dictScan()` does), and the memory win has to survive without a latency regression. Author's numbers on 200,000 random 8 to 40 byte members: MEMORY USAGE down 60.59%, RSS growth down 45.57%, and after 200,000 random score changes the object is still 54.23% smaller. Random lookup is "even"; missing lookup, score updates, rank and large range deletions are faster; random construction is 4.28% slower. Insertion-order lookup and equal-score ordered builds regress more, documented in a report file I could not read in full. Two optimizations were dropped because they lost on measurement: a ZSCAN leaf-decode cache (-25.7% x86, -32.1% ARM) and a memchr-based scan (-9.68% ARM). The page was truncated at 100,000 of 140,368 characters in my read, so I did not see every commit.

**Why It Matters**

Leaderboards, rate limiters and time-ordered queues are sorted sets, and memory per member is what sets the instance size you pay for. RDB and AOF formats are unchanged (saved as RDB_TYPE_ZSET_2), so mixed-version replication works without migration. Trade-off per the author: much more complex code, and ZSCAN order and cursor values differ from the skiplist encoding, with possible duplicates even without a resize. Antirez notes Valkey did something similar and promises a density comparison later.

**Go Deeper**

- [Redis PR #15635, Reduce large sorted set memory with a packed B+ tree (primary source, design notes and benchmarks)](https://github.com/redis/redis/pull/15635)
- [Redis repository](https://github.com/redis/redis)

---

## 2. Go Compiler: arm64 Loops Stop Rebuilding 64-Bit Constants Every Iteration, 20% Faster on a Hash Loop

**Category:** Compilers / Performance

**The Technical Why**

Commit addee8c (author lizthegrey, reviewers Keith Randall and David Chase, committed Oct 22, Gerrit CL 840405, closes issue #81755) makes the register allocator preload multi-instruction constants before a loop on arm64. Per the commit message, the allocator never preloaded rematerializable values before a loop, so a 64-bit constant needing a `MOVD` plus up to three `MOVK` instructions was rebuilt on every iteration. The change adds a `costlyRemat` helper in `regalloc.go` and preloads such constants when a register is free, as it already does for other values live into the loop. Single-instruction constants are untouched. The issue's suspected cause (the reporter's, not confirmed by the commit): `MOVDconst` is marked rematerializable by opcode alone, so a 4-instruction constant looked as cheap as a 1-instruction one.

What is hard: rematerialization is a register-pressure trade. Recomputing a value avoids holding a register, which is right when it costs one instruction and wrong when it costs four inside a hot loop. The allocator needs a cost model per architecture without spilling something more valuable. The fix is guarded by "when a register is free." Repro on a Neoverse N1: `Hash` (constant `0xcf1bbcdcb7a56463`, `uint32((v * prime) >> 45)`) went from 5.165µs to 4.128µs, -20.09% (p=0.000, n=10); `HashVar`, which loads the constant from a variable, did not change. On amd64 the constant is one `MOVQ`, and the issue measured only about 4% there. The issue also notes riscv64, ppc64, mips64 and loong64 carry the same flag; whether they gain from this commit is not on the page.

**Why It Matters**

The bug surfaced in klauspost/compress's zstd encoder, where a workaround (loading hash primes from a package variable on arm64 only) gave 6 to 8% encode throughput on Graviton. Hash functions, PRNGs and checksums all use big multiplicative constants, and arm64 servers are now a normal deployment target. Recompiling with the new toolchain recovers that speed without source hacks. A hand-written workaround stops being needed.

**Go Deeper**

- [Go commit addee8c, cmd/compile: preload multi-instruction constants before loops on arm64 (primary source, benchmarks)](https://github.com/golang/go/commit/addee8c2130be905557bc68a5237eea54c8fe194)
- [Go issue #81755, arm64 rematerializes multi-instruction 64-bit constants inside loops (repro and assembly)](https://github.com/golang/go/issues/81755)

---

## 3. wgpu: Multi-Planar (NV12, P010) Textures Were Marked Initialized Without Ever Being Zeroed

**Category:** Web Graphics / GPU

**The Technical Why**

PR #9700 (author inner-daemons, approved by andyleiserson, merged Oct 22 as 0f0262a via merge queue, closes issue #9688) fixes zero-initialization for multi-planar texture formats. Per the issue, `clear_texture_via_buffer_copies` returned early for NV12 and P010 because `COPY_DST` is unsupported for them (a TODO noted this), yet the texture was still marked initialized, so the skipped clear went unnoticed. The fix in `wgpu-core/src/command/clear.rs` computes each plane's size with the format's subsampling factors, so chroma planes are cleared at half resolution for NV12 and P010. The test (`zero_init.rs`) ran under llvmpipe with `LVP_POISON_MEMORY=true`, which fills fresh allocations with a poison pattern so a missed clear shows up as wrong pixels.

What is hard: zero-init is lazy bookkeeping. wgpu tracks which regions are initialized and clears only when a read could observe them, so a wrong "initialized" flag is silent. The issue itself says it is "not security-relevant" because multi-planar textures are a native extension, not in the WebGPU spec. Sampling uninitialized GPU memory is exactly what zero-init exists to prevent, so the extension status limits who is exposed, not the nature of the bug. Open work per the PR: copy validation to and from subsampled planes, mip-level sizing for multi-planar textures (#8491), correct clearing through render passes (#10710, open), and Metal NV12/P010 support (#9624, open). The `expect_fail` test annotations were removed because the plane tests now pass. At merge 73 of 77 checks had passed; the page does not say what happened to the other four.

**Why It Matters**

NV12 and P010 are the native output formats of hardware video decoders, so this path matters to anyone importing decoded video frames into a renderer without a copy. The wgpu work is tracked on the "WebGPU for Firefox" board, so correctness fixes here affect the engine behind Firefox's WebGPU. For a shader or effects runtime that ingests video, the lesson is that "never read before write" needs a test with poisoned memory, not trust.

**Go Deeper**

- [wgpu PR #9700, Correctly zero init multiplanar textures (primary source, review thread)](https://github.com/gfx-rs/wgpu/pull/9700)
- [wgpu issue #9688, Zero-initialization is silently skipped for multi-planar textures (the bug report)](https://github.com/gfx-rs/wgpu/issues/9688)

---

## 4. SGLang: Diffusion Image Models Lose Per-Step Host Syncs, So the CPU Can Queue Work While the GPU Denoises

**Category:** AI Inference / Serving

**The Technical Why**

PR #46046 (author mickqian, opened Oct 23, merged Oct 24 as c9f4055; stacked on #46045) removes per-step host reads and pageable uploads across diffusion image models. The PR body's premise: a host read of a device value, or a pageable upload, blocks the host until the stream drains, so the host cannot queue the next step while the GPU runs the current one. Fixes are per model and mostly "do it once per request": FLUX.3 Action copies scheduler ticks to the host once instead of `float(tick)` each step; GLM-Image swaps `prior_embedding[prior_token_drop] *= 0.0` for `torch.where`, saving two syncs per forward; Ideogram-4 computes the LLM conditioning projection once per branch on step one and reuses it; Qwen-Image-Layered shares one device tensor between its two CFG branches so the modulation cache hits on the negative branch; SANA casts its prompt mask once per forward, not once per block.

What is hard: the sync is invisible. `tensor.item()`, `float(tensor)`, boolean indexing and pageable `.to(device)` all look like ordinary Python but force a stream drain (the first two are standard PyTorch behavior, my inference for why the listed patterns stall). Each is cheap once and costly 20 to 40 times in a denoising loop. Reported gains are small and honest: Ideogram-4 FP8 at 1024x1024, 20 steps, TP2, median client latency 3.673 s to 3.588 s (-2.3%), denoising stage 3.38 s to 3.29 s; Qwen-Image-2.1 edit 5.300 s to 5.295 s (-0.1%, noise, because it had no per-step syncs on main). Decoded pixels were identical to main in 2 of 2 runs for each of 12 model configurations. Host syncs per request for Cosmos3-Nano are still 42; #45967 is said to remove the rest. Several models (GLM-Image, HunyuanImage-3, Krea-2, SenseNova-U1, SANA and others) were covered by unit tests only, not GPU benchmarks. At merge 145 of 153 checks had passed; the page showed failing "Extra" and AMD ROCm runs.

**Why It Matters**

At serving scale a 2% latency cut is a fleet-size cut, and sync removal is free of accuracy risk when output is bit-identical. It also shows where diffusion serving sits compared with LLM serving: the loop is short and fixed, so CPU launch overhead, not memory bandwidth, is the lever. Who wins: operators running many image models on one engine, and anyone running a GPU effects pipeline, where the same rule holds (never read a GPU value back mid-frame).

**Go Deeper**

- [SGLang PR #46046, [Perf] diffusion: Remove per-step host syncs and uploads in image models (primary source, per-model changes and benchmarks)](https://github.com/sgl-project/sglang/pull/46046)
- [SGLang repository](https://github.com/sgl-project/sglang)

---

## Thread to Watch

Cost models hiding behind a flag: Go's "rematerializable" opcode (addee8c) and Redis's skiplist overhead (#15635) were both cheap by default and wrong at scale. Tomorrow, watch whether Redis publishes the promised density comparison against Valkey and the ZSCAN regression numbers, and whether SGLang lands #45967 to clear Cosmos3's remaining 42 host syncs.
