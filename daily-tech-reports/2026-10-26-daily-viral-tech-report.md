# Daily Viral Tech Report | 2026-10-26

Every story below is checked against a primary GitHub page (PR or commit). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Numbers are the authors' own, on their own machines, not independent benchmarks. Skipped: llama.cpp #29887 (GPU cache for MoE experts in host memory; solid, but it merged Oct 7, so it is not from the last day), three.js #35007 (instanced build sharing; merged Oct 19 and already covered in an earlier report), and the Postgres commit "Avoid waiting for a safe snapshot in notify-reading transactions" (6e9e568; I saw only the title).

---

## 1. vLLM: MXFP4 MoE Activation Quantization Stops Searching for Each Row's Expert, Up to 3.39x on the Kernel

**Category:** AI Serving / GPU Kernels

**The Technical Why**

PR #63101 (author ilmarkov, opened Oct 22, approved by LopezCastroRoberto Oct 23, merged by mgoin Oct 26, merge commit f3bdb0e) speeds up the step that quantizes activations to MXFP4 before the grouped expert matmuls in a mixture-of-experts layer. After routing, token rows are sorted so each expert's rows sit together. The old kernel found each row's expert by searching the expert offset table (`offset_search`). The PR adds an optional `sorted_expert_ids` input to `mxfp4_experts_quant`, `silu_and_mul_mxfp4_experts_quant` and `get_cutlass_moe_mm_data`, so each sorted row already carries its expert index and the search is skipped. If the argument is omitted, the old path runs. The author says the new path is bitwise identical to `offset_search` on the microbenchmark cases.

What is hard: the search cost grows with expert count, and modern MoE models have many experts. On an NVIDIA B300, the plain quant kernel went from 7.33 us to 2.16 us at 256 experts (3.39x), while the new kernel stays flat around 2.2 us from 32 to 256 experts. At 16 experts nothing changed (1.94 to 1.98 us). The win is also diluted: `get_cutlass_moe_mm_data` plus the shuffle costs about 14 to 15 us at 256 experts on both paths, so the full path gains about 1.35x (22.48 to 16.78 us). End to end on `RedHatAI/GLM-5.3-MXFP4` (TP=4, concurrency 1, 32 prompts, 256 input and 1024 output tokens), output throughput went from 85.55 to 91.51 tok/s and median TPOT from 11.43 to 10.67 ms. The PR text says about +5.7%; the table in the same PR works out to about +7.0%. GSM8K moved from 0.9052 to 0.9060 (flexible-extract), which is noise.

**Why It Matters**

Decode on large MoE models is latency bound at small batch sizes, where a few microseconds per layer, repeated across many layers and every token, shows up in TPOT. The lesson is general: if a producer already knows the answer (here, routing sorted the rows by expert), pass it down instead of making the consumer rediscover it with a search. Caveats on the page: the microbenchmark covers decode-sized batches only, the prefill-sized kernel variant was not benchmarked, the end-to-end hardware is not stated, and the gain is one model at one concurrency level. A reviewer (yewentao256) asked for raw logs and a smaller diff before merge. Who wins: operators serving MXFP4 MoE models on Blackwell-class GPUs with many experts.

**Go Deeper**

- [vLLM PR #63101, [Perf][MXFP4] Optimize activation quant for mxfp4 (primary source, kernel tables, end-to-end run, review thread)](https://github.com/vllm-project/vllm/pull/63101)
- [vLLM repository](https://github.com/vllm-project/vllm)

---

## 2. llama.cpp: Vulkan RMS Norm Uses Subgroup Reductions Instead of One Workgroup Tree, Up to 79% Faster on Wide Rows but Not Everywhere

**Category:** GPU Kernels / Cross-Vendor Inference

**The Technical Why**

PR #29882 (author virajwad, opened Oct 2, approved by 0cc4m, merged by ggerganov Oct 26, commit 70c5f26) replaces the single-workgroup reduction in the Vulkan RMS norm shader with a two-stage subgroup reduction. RMS norm needs the sum of squares of a row, then scales every element by the inverse root of the mean. A shared-memory tree reduction across a whole workgroup needs several barrier rounds. A subgroup reduction first sums inside each subgroup (a hardware-sized group of lanes, a warp on NVIDIA) with one built-in instruction, then combines the few partial sums across subgroups. The PR ships variants for block sizes 128, 256 and 512 chosen by row length, the fused forms (RMS norm with multiply-add, and with set-rows to f32 or f16) from the same shader source, and a fast path that skips the second stage when one subgroup covers the block. The host code picks the subgroup or fallback binary by device.

What is hard: the right strategy depends on the shape and the vendor. 0cc4m's per-op microbenchmarks (base to head, us): RTX 3090 RMS_NORM [128,16384,1,1] 141.95 to 29.54 (-79.2%), Intel A770 309.06 to 67.65 (-78.1%), Nvidia GB10 156.88 to 35.08 (-77.6%). But Nvidia GB10 [4096,512,1,1] went 20.55 to 27.18, a 32.3% regression in that run, and A770 [4096,1,1,1] and [4096,512,1,1] were slightly slower (+4.9% and +2.3%). 0cc4m later could not reproduce the GB10 regression and called it a fluke; the page has no dedicated retest. AMD was first excluded by a vendor gate, then enabled after Peterc3-dev measured a Radeon 890M (RADV): the q-norm op RMS_NORM_MUL (128,32,512,1) went 720 to 186 us. Apple stays excluded because of an existing MoltenVK workaround that clears subgroup arithmetic support.

**Why It Matters**

Whole-model gains are small: RTX 4060 Ti with Qwen3-0.6B Q4_K_M at pp1024 went 24067.88 to 28280.84 tok/s with flash attention on, but with Qwen3.5-9B Q6_K pp512 only 2320.02 to 2430.82, and decode (tg128) is flat within noise. Small models and many per-head norms benefit most, because the norm is a bigger slice of their step. Takeaway: a portable backend (Vulkan covers NVIDIA, AMD, Intel) cannot assume one reduction shape is best everywhere, so the real work is the gate: which devices, which row widths. Who wins: users on Intel Arc and AMD GPUs, who depend on the Vulkan backend rather than CUDA.

**Go Deeper**

- [llama.cpp PR #29882, vulkan: rms norm optimization using subgroup reductions (primary source, per-device tables, review thread)](https://github.com/ggml-org/llama.cpp/pull/29882)
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

---

## 3. PostgreSQL: A Rare nbtree Page Deletion Bug Logged Stale Sibling Links to the WAL, Corrupting Standbys

**Category:** Databases / Replication

**The Technical Why**

Commit d42c545 ("Fix nbtree multi-level page deletion WAL record bug", author Peter Geoghegan, committed to `master` Oct 26, backpatched through version 14) fixes an oversight in commit efada2b, which had fixed a race in nbtree page deletion. Deleting a B-tree page that has a deletable subtree is a two-phase job. Phase one marks the leaf "half-dead". Phase two unlinks it and marks the top parent page fully deleted. In phase two, per the commit message, the code releases the leaf's lock before examining the target page. In that gap, the leaf's left sibling can split, which changes the leaf's left sibling link. The code did not re-copy the sibling links after relocking, so a stale value could be written into the WAL record. The buffer is registered with `REGBUF_WILL_INIT`, meaning the standby rebuilds the page from the record instead of reading the old page, so the stale link lands in the standby's half-dead leaf page as corruption. The fix, in `_bt_unlink_halfdead_page()` in `nbtpage.c` (+11 -3), reloads `leafleftsib` and `leafrightsib` from the leaf's `btpo_prev` and `btpo_next` right after relocking.

What is hard: this needs a concurrent split at a precise moment between two lock acquisitions, so it was found through `wal_consistency_checking` failures, a debug mode that compares the page a replayed WAL record produces against the primary's page. The message says the bug was very hard to hit and that the standby damage usually ended when the page was marked fully deleted shortly afterward. The page lists no reviewers, and I did not see the mailing list thread.

**Why It Matters**

Replication correctness lives in the WAL record, not in the primary's memory. Any value logged after a lock gap must be re-read under the lock, or the replica replays a state that never existed. A full-page-init record makes this worse, because it overwrites instead of merging. Who wins: anyone running Postgres 14 or later with physical standbys, once the minor releases pick up the fix; I did not verify the release schedule. The takeaway for system design: "copy values, drop lock, retake lock, log" is a bug pattern, and a consistency checker that replays and compares is how you find it.

**Go Deeper**

- [PostgreSQL commit d42c545, Fix nbtree multi-level page deletion WAL record bug (primary source, full message and diff)](https://github.com/postgres/postgres/commit/d42c545)
- [PostgreSQL repository](https://github.com/postgres/postgres)

---

## 4. three.js: GLTFLoader Parses Embedded KTX2 Textures Straight From the Buffer, Skipping a Blob, an Object URL and a fetch

**Category:** Web Graphics / Asset Loading

**The Technical Why**

PR #35193 (author Construkted-Reality, opened Oct 23, merged by Mugen87 into `dev` Oct 26, milestone r187) changes how `GLTFParser.loadImageSource()` handles an embedded image. Before: for an image stored in a `bufferView`, the parser copied the bytes into a `Blob`, made an object URL, and `KTX2Loader.load()` then `fetch`ed that URL and called `arrayBuffer()` to get the same bytes back, all on the main thread. After: when the image has a `bufferView` and the loader is a KTX2 loader (a new `isKTX2Loader = true` flag, mirroring `isImageBitmapLoader`), the parser hands the `bufferView` directly to `loader.parse()`. Other images keep the old path.

What is hard: the cost is a round trip through browser APIs that exist for a different purpose (URLs), and the byte copies and fetch machinery run on the thread that must also render. The author's measurements on Chromium (50 glTF binaries from a 3D Tiles mesh, 50 embedded KTX2 images, 3.0 MB total, 5 repeats): the Blob round trip cost 18.6 ms of main-thread time per MB (59.4 ms with 4x CPU throttle), made of 0.64 ms per image for `new Blob()` plus `createObjectURL()` and 5.5 ms per MB for `fetch()` plus `arrayBuffer()`. In a later comment the author gave 5 to 14 ms per MB for the Blob path, so treat the headline as a range, not a constant. Tradeoff found in review: `parse()` transfers its buffer to a worker, and `KTX2Loader` caches tasks by buffer, so two images pointing at one `bufferView` would share a texture. The author tried a per-image copy, then reverted it because it added main-thread work to every load; Mugen87 called the shared case a cloning problem to fix only if a real file shows it. donmccurdy, who maintains the glTF loader, raised the detachment concern as non-blocking and said he had never seen such a file.

**Why It Matters**

Tiled 3D scenes load dozens of small glTF files at once, so per-file main-thread stalls add up to dropped frames while streaming. Verification was output equivalence: same texture formats, counts and mip bytes at 20 views, with pixel differences of at most 11 pixels on WebGL and 1 pixel (by 2 of 255) on WebGPU, within the author's baseline noise of up to 15. Not tested: Safari and Firefox, and the repo's own test suites were left to CI. Who wins: apps streaming many glTF files with KTX2 textures through three.js, such as map and digital twin viewers.

**Go Deeper**

- [three.js PR #35193, GLTFLoader: Parse embedded KTX2 images without a Blob (primary source, timings, equivalence evidence, review thread)](https://github.com/mrdoob/three.js/pull/35193)
- [three.js repository](https://github.com/mrdoob/three.js)

---

## Thread to Watch

Specialization by shape and by device: vLLM skips a search because routing already knows the expert (#63101), llama.cpp's Vulkan norm wins on wide rows and is gated per vendor (#29882). Tomorrow, watch whether the vLLM follow-up for prefill-sized MXFP4 batches appears, and whether anyone reports on the GB10 RMS norm case in #29882 after merge.
