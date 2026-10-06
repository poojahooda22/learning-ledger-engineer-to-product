# Daily Viral Tech Report | 2026-10-06

Every story below is checked against a primary GitHub page (commit, PR or release). Dates are as those pages print them. Where a page does not explain a mechanism, I label my explanation as inference. Two stories that surfaced in search (Google's Gemini 4 Argon launch and Siemens shutting down OpenRadioss) were dropped: I could only reach secondary coverage, not a primary source, so I did not verify them.

---

## 1. PostgreSQL: B-tree Deduplication Rescinded for bpchar and oidvector Because Index-Only Scans Could Return Wrong Results

**Category:** Databases / Systems

**The Technical Why**

On Oct 6 Peter Geoghegan committed "Rescind unsafe deduplication support", backpatched through version 14. nbtree deduplication stores one index tuple with a posting list of TIDs for many equal keys. That is only safe if "equal" means "byte-identical". For `bpchar` (blank-padded char with no length), equality ignores trailing spaces, so `'a'` and `'a '` are equal but have different bytes. For `oidvector`, equality ignores the array lower bound. The commit message says a posting list split could put a TID under an index tuple whose key was equal to, but not bytewise identical to, the table value, so index-only scans "could return subtly wrong results". The fix removes support function 4 (the dedup opt-in) from `bpchar_ops`, `bpchar_pattern_ops` and `oidvector_ops`. On branches 14 to 18 it hard-codes dedup off instead of changing the catalog. It also touches amcheck's `verify_nbtree.c`, and the message tells users to REINDEX any B-tree index on a bpchar column.

Why it is hard: an index-only scan returns the key stored in the index, not the heap value. So the stored bytes are part of the query result, not just a search aid. Any optimization that merges "equal" keys must prove equal means identical in storage (my inference on why the bug is silent: results look plausible, nothing crashes).

**Why It Matters**

Anyone running Postgres 14 or later with a bpchar index could get index-only results that differ in trailing spaces from the heap. Watch for the minor release notes and plan the REINDEX. The lesson for any engineer building a cache, dedup or compression layer: equality and identity are different contracts.

**Go Deeper**

- [Postgres commit a32b8dc, Rescind unsafe deduplication support (primary source)](https://github.com/postgres/postgres/commit/a32b8dc9865aa5189d20eca273700b10dfd92fc7)
- [Postgres docs, B-Tree deduplication](https://www.postgresql.org/docs/current/btree.html)

---

## 2. llama.cpp: Tensor Parallelism Over RPC (`-sm tensor`) Splits One Model Across Two Machines Using RDMA

**Category:** AI / ML Inference

**The Technical Why**

llama.cpp PR #26610 (merged Oct 6, by am17an with Georgi Gerganov as co-author) adds `-sm tensor` for RPC backends. Instead of giving each machine whole layers, it splits the compute graph at reduction boundaries so each node computes a slice of every layer. Nodes run subgraphs asynchronously, cache computed graphs by a unique ID (as the CUDA backend does), and exchange partial results with a custom all-reduce directly between RPC servers over dedicated ports. New 2D tensor get/set commands move data efficiently. The PR reports DeepSeek-v4 MXFP4 MoE on 2 DGX Sparks: 619.36 t/s prompt processing (pp2048) and 19.75 t/s generation (tg128).

Limits stated in the PR: only 2 RPC backends use the fast path (more fall back to the slower meta-backend all-reduce), it needs direct RDMA between nodes, it is poor over gigabit Ethernet, and the protocol bump means client and server versions cannot be mixed.

Why it is hard: tensor parallelism does an all-reduce every layer, so link latency sits on the critical path of every token. Layer splitting only passes activations once per boundary (standard comparison, not stated in the PR).

**Why It Matters**

It lets two small boxes serve a model neither can hold alone, which matters for local and on-prem inference. The honest read: generation at 19.75 t/s is usable, but only with RDMA-class networking.

**Go Deeper**

- [llama.cpp PR #26610, RPC -sm tensor (primary source)](https://github.com/ggml-org/llama.cpp/pull/26610)
- [llama.cpp RPC backend README](https://github.com/ggml-org/llama.cpp/tree/master/tools/rpc)

---

## 3. three.js: GTAOPass Drops the Combined Depth-Normal Texture Path and Gets Buffer Fixes Ahead of r187

**Category:** Web Graphics and GPU

**The Technical Why**

On Oct 6 three.js merged PR #34836 (bhouston, merged by Mugen87, milestone r187) removing the combined depth-and-normal texture input from `GTAOPass`. The PR says the format came from the older HBAOPass, "couples their storage precision", was inconsistently supported, and had no producer or example in the repo. The pass now relies on separate inputs or its own rendered buffers, and still supports depth-only use. A sibling PR, #34831 "GTAOPass: Fix external buffer handling", was merged the same day; I only confirmed its title, not its details.

GTAO (ground-truth ambient occlusion) samples the depth buffer around each pixel to estimate how much sky it can see. Why it is hard: packing depth and normals in one texture forces both to share one precision, but depth needs far more bits than a normal. Splitting them lets each use a suitable format (my inference from the PR's storage-precision wording).

**Why It Matters**

This is a small API removal, but it signals the post-processing stack being cleaned before a release. If you pass a combined texture to GTAOPass, expect it to break in r187.

**Go Deeper**

- [three.js PR #34836, GTAOPass remove combined depth and normal texture (primary source)](https://github.com/mrdoob/three.js/pull/34836)
- [three.js PR #34831, GTAOPass external buffer fix](https://github.com/mrdoob/three.js/pull/34831)

---

## 4. Rust 1.99.0: C-Variadic Function Definitions Stabilized, riscv64 musl Reaches Tier 2, rustdoc Up to 40% Faster

**Category:** Languages and Compilers

**The Technical Why**

Rust 1.99.0 (Oct 1) stabilizes defining C-variadic functions in Rust (PR #155697), so Rust code can now implement `printf`-style `...` functions, not only call them. PR #159746 stabilizes `#[unsafe(naked)]` for this case. The release also adds an allow-by-default `raw_borrows_via_references` lint, new `Box::into_non_null` and `Vec::into_parts` APIs, and promotes `riscv64-unknown-linux-musl` to Tier 2 with host tools (PR #158766). Rustdoc trait-impl filtering got five PRs of optimization, reported as 20% faster on average and up to 40% on some real crates.

Why it is hard: a variadic callee must read arguments off a platform-specific `va_list`, and each ABI lays that out differently, with no type information at the call boundary (general ABI fact, not stated on the release page).

**Why It Matters**

Rust can now replace C in libraries that must export variadic symbols, which removes one of the last reasons to keep a C shim. Tier 2 host tools means prebuilt toolchains for riscv64 musl, which helps embedded and container builds.

**Go Deeper**

- [Rust 1.99.0 release notes (primary source)](https://github.com/rust-lang/rust/releases/tag/1.99.0)
- [Rust PR #155697, stabilize C-variadic definitions](https://github.com/rust-lang/rust/pull/155697)

---

## Thread to Watch

Optimizations that silently assume a contract the data does not keep: Postgres dedup assumed equal means identical, llama.cpp RPC assumes RDMA-class links, three.js packing assumed shared precision. Tomorrow, watch the Postgres minor release announcement for the dedup REINDEX guidance, and whether the Gemini 4 Argon launch gets a primary technical write-up I can verify.
