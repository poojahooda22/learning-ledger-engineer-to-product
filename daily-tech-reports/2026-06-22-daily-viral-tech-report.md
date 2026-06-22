# Daily Viral Tech Report | 2026-06-22

---

## 1. TypeScript 7.0 RC Ships: The Compiler Is Now Written in Go and Builds VS Code in 7.5 Seconds

**Category:** Developer Tooling

**The Technical Why**
The TypeScript compiler (tsc) has always been a bootstrapped JavaScript program: TypeScript compiles itself to JS, and Node runs that JS on every build. JS is single-threaded, so tsc was always single-threaded. The TypeScript 7 team ported the entire codebase to Go over about 12 months, preserving the same type-checking semantics method by method rather than rewriting from scratch. Go gives two things JS cannot: native code execution speed, and shared-memory parallelism via goroutines. The new compiler runs parsing, type inference, and emit concurrently across files. Parsing one file has no dependency on another file's parse result, so every file fans out in parallel. Type-checking requires some dependency resolution, but large subgraphs of the module tree are also independent and run in parallel. The result on the VS Code codebase (1.5 million lines of TypeScript) is a build that drops from 77 seconds to 7.5 seconds. The RC shipped June 18, 2026; stable is expected around mid-July. Install with `npm install -D typescript@rc`.

**Why It Matters**
Every TypeScript project on the planet gets this speedup for free on the stable release. CI pipelines that currently block on tsc drop dramatically. The hard engineering achievement is behavioral parity: the Go compiler must produce identical type errors to the JS compiler on every real-world codebase, and the team validated this across the TypeScript test suite and the largest open-source TS repos before shipping.

**Go Deeper**
- [Announcing TypeScript 7.0 RC (Microsoft DevBlogs)](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/)
- [TypeScript 7.0 RC Ships: VS Code build time drops from 77s to 7s (TechTimes)](https://www.techtimes.com/articles/318666/20260618/typescript-70-rc-ships-go-compiler-cuts-vs-code-build-time-77-seconds-seven.htm)
- [tsgo: Install guide and 10x speedup verification (DEV Community)](https://dev.to/tonkotsuboy_com/tsgo-released-typescript-7s-new-compiler-installation-guide-10x-speedup-verification-536g)

---

## 2. Inception Mercury 2: Diffusion Beats Autoregressive at 1,009 Tokens Per Second

**Category:** AI / ML

**The Technical Why**
Every autoregressive LLM generates exactly one token per forward pass: embed input, run N transformer layers, sample from the output distribution, append the token, repeat. Throughput is bounded by the number of forward passes you can run per second times one token each. Mercury 2 uses masked diffusion instead. It starts with a sequence of all-masked (unknown) positions and runs a fixed number of denoising steps, typically 8 to 16, each of which unmasks and refines all token positions simultaneously. One forward pass touches the entire output sequence, not just the next token. Total forward passes needed for a 512-token output: 8 to 16 with diffusion vs 512 with autoregression. On NVIDIA Blackwell GPUs, Mercury 2 reaches 1,009 tokens per second with 1.7 seconds end-to-end latency, five times faster than leading autoregressive speed-optimized models. The quality challenge with diffusion LMs is that each denoising step sees a partial future (it is trained to unmask a randomly corrupted subset of tokens), which makes precise conditional generation harder to learn than autoregressive next-token prediction. Mercury 2's improvement over Mercury 1 is closing that quality gap: it now benchmarks on par with Claude 4.5 Haiku and GPT 5.2 Mini on standard reasoning tasks. The June 2026 arxiv paper (2506.17298) documents the training objective and scaling behavior.

**Why It Matters**
Agentic pipelines chain 10 to 30 LLM calls per user request. At 1,000 tokens per second, each reasoning hop takes milliseconds instead of seconds, which makes real-time voice AI and multi-hop search agents actually interactive. This is the only credible architectural alternative to transformers with a hard performance argument, not just a tradeoff of quality for speed.

**Go Deeper**
- [Mercury: Ultra-Fast Language Models Based on Diffusion (arxiv 2506.17298)](https://arxiv.org/pdf/2506.17298)
- [Inception Launches Mercury 2: 5x Faster Than Leading Speed-Optimized LLMs (Business Wire)](https://www.businesswire.com/news/home/20260224034496/en/Inception-Launches-Mercury-2-the-Fastest-Reasoning-LLM-5x-Faster-Than-Leading-Speed-Optimized-LLMs-with-Dramatically-Lower-Inference-Cost)
- [What Is Mercury 2? The Diffusion LLM 5x Faster Than Claude Haiku (MindStudio)](https://www.mindstudio.ai/blog/mercury-2-diffusion-based-language-model-5x-faster)

---

## 3. Three.js r184 Kills GC Pauses and Ships Stable Compute Shader Support

**Category:** Web Graphics and GPU

**The Technical Why**
Three.js r184 (March 2026) fixed a structural performance bug in the render loop. At 1,000 meshes running at 60fps, the renderer was allocating 240,000 to 500,000 temporary JavaScript objects per second as it built per-draw-call descriptor objects inside the render loop. The GC had to collect all of these on the next cycle, causing periodic frame drops. r184 restructures the render pipeline to pre-allocate and reuse descriptor objects across frames, removing the GC from the hot path entirely. The second major change is that TSL (Three Shader Language) compute shaders are now stable. TSL lets you write shaders in JavaScript-flavored syntax that compiles at runtime to either WGSL (for the WebGPU path) or GLSL (for the WebGL fallback path). Compute shaders running through TSL get access to storage buffers, workgroup shared memory, and GPU dispatch groups with the same syntax for both APIs. A particle system of one million particles can run its entire integration loop on GPU, reading and writing to a storage buffer, with no CPU readback. WebGPU hit Baseline in January 2026 and now has roughly 82% global browser support, so the WGSL path is the default and the GLSL fallback exists for the remaining 18%.

**Why It Matters**
The GC fix matters immediately: any Three.js scene with high mesh counts was previously paying a latency tax on every garbage collection cycle. The TSL compute story matters structurally: it is now practical to run physics, fluid simulation, and particle systems entirely on GPU in the browser with one codebase that works across WebGPU and WebGL. For any browser-based visual tool, this is the moment the GPU becomes a first-class compute target, not just a rasterizer.

**Go Deeper**
- [What's New in Three.js 2026: WebGPU, TSL, New Workflows (Utsubo)](https://www.utsubo.com/blog/threejs-2026-what-changed)
- [GPU-Side Physics: A Three.js WebGPU Compute Shader Demo (webgpu.com)](https://www.webgpu.com/showcase/threejs-webgpu-compute-physics/)
- [Three.js r171 release notes: WebGPURenderer as default (GitHub)](https://github.com/mrdoob/three.js/releases/tag/r171)

---

## 4. Google Splits Its TPU Into Two Chips for the First Time: 8t for Training, 8i for Inference

**Category:** Systems and Engineering

**The Technical Why**
Eight TPU generations used one chip design that had to serve both training and inference. TPU v8 is the first time Google ships two distinct dies. The reason is a fundamental memory access pattern mismatch. Training is throughput-bound: the GPU must stream large gradient tensors and optimizer states (Adam maintains two FP32 momentum buffers per parameter) sequentially through HBM. TPU 8t is tuned for this: 216 GB HBM3e running at 6,528 GB/s, 12.6 petaFLOPS FP4. Inference, especially long-context multi-turn chat with modern 128k-token models, is latency-bound by KV cache lookups: every generated token must read every prior token's key and value from memory for every attention head. On-chip SRAM is 10 to 100x faster than HBM for random-access patterns. TPU 8i triples the on-chip SRAM to 384 MB versus the previous Ironwood generation, and adds a dedicated Collectives Acceleration Engine (CAE) that offloads all-reduce and all-gather operations from the main tensor cores, reducing collective latency by up to 5x. The Virgo Network switches from a 3-layer fat-tree topology to a flat 2-layer non-blocking fabric using high-radix switches (more ports per switch reduces switch hops). TPUDirect RDMA lets the NIC write data directly to HBM without staging through host DRAM or CPU, cutting the data pipeline from storage to tensor core by two copies. The result: 134,000 TPU 8t chips can form a single fabric in one data center, and over 1 million TPUs can join one training cluster across sites.

**Why It Matters**
This is the hardware acknowledgment of a workload shift the industry has observed since 2024: inference now accounts for more than 80% of AI GPU cycles at production scale, and a chip optimized for training is the wrong tool. Every hyperscaler is now building or buying separate inference silicon; custom AI chips are differentiating on memory hierarchy design (SRAM vs HBM tradeoffs) not raw FLOPS. Engineers designing inference serving systems should understand this bifurcation because it changes how you partition models, schedule batches, and set SLA targets.

**Go Deeper**
- [TPU 8t and TPU 8i Technical Deep Dive (Google Cloud Blog)](https://cloud.google.com/blog/products/compute/tpu-8t-and-tpu-8i-technical-deep-dive)
- [Google Bolsters AI Hypercomputer with New TPU Chips and Virgo Interconnect (HPCwire)](https://www.hpcwire.com/2026/04/22/google-bolsters-ai-hypercomputer-with-new-tpu-chips-virgo-interconnect-speedier-lustre/)
- [Inside Google's TPU v8 Strategy: 1 Million TPUs Per Cluster (Tom's Hardware)](https://www.tomshardware.com/tech-industry/semiconductors/google-splits-its-tpu-into-two-chips-for-the-first-time-with-training-and-inference-variants)

---

## Thread to Watch

Diffusion LMs (Mercury 2, DiffusionGemma) are the only architectural alternative to autoregressive transformers with a hard speed argument. The open question is whether they can close the quality gap on instruction following and tool use, the two tasks agentic systems depend on most. If they do, the entire autoregressive inference serving stack (KV cache, speculative decoding, continuous batching, PagedAttention) becomes irrelevant overnight, and the hardware implications for SRAM vs HBM tradeoffs flip completely.
