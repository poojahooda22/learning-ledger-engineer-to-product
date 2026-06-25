# Daily Viral Tech Report | 2026-06-25

---

## 1. OpenAI Jalapeño: First Custom LLM Inference ASIC, Built in 9 Months with Broadcom

**Category:** AI / ML Infrastructure

**The Technical Why**

LLM inference is not the same problem as LLM training. Training is compute-bound: you do enormous matrix multiplications across huge batches, and a GPU's thousands of CUDA cores stay busy. Autoregressive inference is memory-bandwidth bound: to generate each token you must load the entire model's weights from HBM into compute cores, one token at a time. A 70B-parameter model at 16-bit precision needs 140 GB just for the weights. A general-purpose H100 carries CUDA cores that are mostly idle during each token step, wasting power on capability the workload never uses. Jalapeño is designed from scratch for the inference case: a reticle-limited ASIC co-designed with OpenAI's own inference workloads across ChatGPT, Codex, and the API, so the memory hierarchy, the precision modes, and the on-chip interconnects are tuned for transformer forward passes specifically. Broadcom handled silicon implementation (they already build custom ASICs for Google TPUs and Meta MTIA); OpenAI supplied the workload specification and reportedly used its own AI models to accelerate parts of chip design and verification. The 9-month tape-out timeline is roughly three times faster than a typical HPC ASIC cycle, which normally runs 2 to 3 years.

Early testing shows Jalapeño delivers better performance per watt than the current state of the art. Initial deployment targets gigawatt-scale data centers with Microsoft beginning in 2026. The chip is described as designed for current and future LLMs across the industry, not locked to OpenAI's weights, which hints at a foundry or licensing play alongside internal use.

**Why It Matters**

Every dollar OpenAI saves on inference cost widens its margin advantage over competitors running on leased Nvidia hardware. Custom silicon is how Google made the TPU investment pay off at scale; OpenAI is now on the same path. The broader signal: the hyperscale AI players are all vertically integrating into silicon because inference cost is now the dominant constraint on profitability, not model quality.

**Go Deeper**

- [OpenAI and Broadcom Unveil Jalapeño (openai.com)](https://openai.com/index/openai-broadcom-jalapeno-inference-chip/)
- [Broadcom and OpenAI unveil custom-built Jalapeño inference processor (Tom's Hardware)](https://www.tomshardware.com/tech-industry/artificial-intelligence/broadcom-and-openai-unveil-custom-built-jalapeno-inference-processor-openais-first-chip-is-a-massive-reticle-sized-asic-built-in-an-ultra-fast-nine-month-development-cycle)
- [OpenAI unveils its first custom chip, built by Broadcom (TechCrunch)](https://techcrunch.com/2026/06/24/openai-unveils-its-first-custom-chip-built-by-broadcom/)

---

## 2. Figma Config 2026: AI-Generated WebGPU Shader Fills Ship Live, Code Layers Enter Beta

**Category:** Web Graphics and GPU

**The Technical Why**

Figma started migrating its rendering backend from WebGL to WebGPU in 2023 and has been writing its internal shaders in WGSL (WebGPU Shading Language) since then. The migration is not a line-for-line port. WebGL's state machine, a global context with implicit state validated per draw call on the CPU, is replaced in WebGPU by Pipeline State Objects compiled once at creation time and reused with zero per-draw-call CPU overhead. The practical payoff is that GPU compute, previously a shader hack in WebGL (write to a texture, read it back, no workgroup memory, no synchronization primitives), becomes a first-class citizen: Figma can now run procedural effects, noise generators, and filter passes entirely on the GPU without CPU round-trips.

The new shader fills feature layers an AI code-gen step on top of that infrastructure. A designer describes an effect such as "frosted glass," "liquid metal," or "fractal noise," and the Figma agent generates a parameterized WGSL program with controls (sliders, color pickers) that map directly to uniform variables in the shader source. The generated code runs on the GPU in the browser tab. No shader programming knowledge is required. Effects include dithering, pixelation, blur, moiré patterns, and animation, and the generated WGSL is exposed if the designer wants to read or edit it. This shipped live for Full seat users on paid plans as of Config 2026.

Code Layers, also announced today, links any canvas layer to a GitHub repository. Teams can clone a repo, let the Figma agent generate or edit code layers on the canvas, and sync changes back to the branch. This puts Figma's canvas in direct competition with AI coding tools that start from code and move toward design.

**Why It Matters**

GPU shader fills powered by WebGPU are now a commercial product feature, not a demo. Any team building a visual or node-based tool in the browser has a concrete production reference for what AI-generated WGSL shaders look like at Figma's scale. The Code Layers announcement is the larger strategic bet: Figma is repositioning the canvas as a development surface, not just a handoff tool.

**Go Deeper**

- [What's new from Config 2026 (Figma Help Center)](https://help.figma.com/hc/en-us/articles/39582753756695-What-s-new-from-Config-2026)
- [Figma Config 2026: Code Layers Challenge Cursor as GPU Shaders Hit Paid Plans (TechTimes)](https://www.techtimes.com/articles/319041/20260625/figma-config-2026-code-layers-challenge-cursor-gpu-shaders-hit-paid-plans.htm)
- [Figma Config 2026: Motion, Code, Shaders and AI recap (explainx.ai)](https://explainx.ai/blog/figma-config-2026-complete-recap-motion-code-shaders-ai-2026)

---

## 3. Qualcomm Bets $14B on RISC-V to Break Nvidia's Compiler Moat

**Category:** Platform and Market Move

**The Technical Why**

Nvidia's dominance in AI hardware is not only the H100 GPU. It is CUDA: a compiler toolchain, a runtime, and a library ecosystem (cuDNN, cuBLAS, NCCL) built up over more than a decade. An open instruction set architecture like RISC-V gives any chip designer a royalty-free base, but it does not by itself crack CUDA's moat because the moat is software, not silicon.

Qualcomm's two acquisitions attack both sides simultaneously. Tenstorrent, the RISC-V AI accelerator startup led by Jim Keller (the architect behind modern x86 Athlon 64, Apple A4-A5, and AMD Zen), brings custom inference silicon built on the open ISA at a reported $8 to 10 billion. Modular, acquired earlier in 2026 for $3.9 billion, brings the MAX platform: the MLIR-based compiler infrastructure, the Mojo language, and the MAX Engine inference runtime. The compiler is the hardest part. To make a transformer workload run as efficiently on Tenstorrent silicon as on an H100, you need a compiler that understands how to tile matrix operations across that chip's specific memory hierarchy, schedule inter-core communication, and fuse kernels, without requiring every developer to rewrite their PyTorch or JAX model. Modular's MAX Engine is designed to ingest standard ML graphs and emit optimized code for any target backend. Together, the RISC-V chip plus the open compiler stack is Qualcomm's complete answer to why a cloud provider would swap even one GPU server for non-Nvidia hardware.

**Why It Matters**

This is the most technically complete challenge to Nvidia's software moat assembled by any single company outside the hyperscalers. Intel tried with Gaudi; Google's TPUs stay internal; Meta's MTIA is narrow in scope. A combined $14 billion commitment signals that Qualcomm believes the CUDA lock-in is breakable with a full stack, and cloud providers negotiating multi-billion GPU contracts have every incentive to want that to be true.

**Go Deeper**

- [Qualcomm Bets $14B on Cracking Nvidia's AI Monopoly With RISC-V and an Open Compiler (TechTimes)](https://www.techtimes.com/articles/319017/20260624/qualcomm-bets-14-billion-cracking-nvidias-ai-monopoly-risc-v-open-compiler.htm)
- [Qualcomm said to be circling Tenstorrent in $10B RISC-V power play (The Register)](https://www.theregister.com/systems/2026/06/16/qualcomm-said-to-be-circling-ai-chip-biz-tenstorrent-in-10b-risc-v-power-play/5256084)
- [Qualcomm in Talks to Acquire Tenstorrent for $8-10 Billion (AI Unfiltered)](https://www.arturmarkus.com/qualcomm-in-talks-to-acquire-tenstorrent-for-8-10-billion-jim-kellers-risc-v-ai-chip-startup-valuation-triples-in-one-year/)

---

## 4. Airbnb Mussel v2: Replacing the Storage Backend Under 100TB of Live Traffic

**Category:** Systems and Engineering

**The Technical Why**

Mussel is Airbnb's key-value store for derived data: search rankings, user feature vectors, recommendation scores, and other read-heavy outputs that are expensive to recompute but cannot live in a plain cache. Mussel v1 used a custom RocksDB-backed engine maintained by Airbnb's own infrastructure team. Compaction tuning, bloom filter configuration, and manual re-sharding whenever a table outgrew its partition required continuous operator attention. V2 replaced the custom backend with a NewSQL distributed store (TiKV-based) that handles sharding, replication, and compaction automatically, freeing the team to ship features instead of tuning storage internals.

The core design choice is range sharding with presplitting instead of hash partitioning. Hash partitioning distributes keys evenly across shards but destroys sort order, so any range scan must hit every shard in the cluster regardless of how many rows it actually needs. Range sharding keeps sorted key ranges collocated on the same shard so a range scan only reads the shards whose key range overlaps the query. The challenge with range sharding is split hotspots: when a range grows large enough to split, the split operation itself creates a brief write bottleneck on the shard being split. Presplitting solves this by creating the correct number of splits ahead of a bulk load, based on the expected key distribution, so the system never splits reactively under production load. The result is 100TB+ tables at p99 read latency under 25ms and over 100K sustained writes per second, numbers that v1's custom engine could not hold under bursty write patterns.

The migration strategy kept production live throughout. V2 ran in dual-write shadow mode for months: v1 served all reads while both stores received writes, and a consistency checker compared outputs continuously. Reads cut over in a rolling deployment only after consistency was confirmed. No table needed a maintenance window.

**Why It Matters**

Airbnb's path from a hand-rolled storage engine to a managed NewSQL backend is the playbook for any team that built custom infrastructure during the early scaling phase and is now spending more engineering time on infra maintenance than on product work. The specific lesson on presplitting range shards applies directly to any high-write table where hash-partitioned storage is starting to show hot-key or range-scan pain at scale.

**Go Deeper**

- [Building a Next-Generation Key-Value Store at Airbnb (Airbnb Tech Blog)](https://airbnb.tech/infrastructure/building-a-next-generation-key-value-store-at-airbnb/)
- [Mussel: Airbnb's Key-Value Store for Derived Data (Airbnb Tech Blog original)](https://airbnb.tech/data/mussel-airbnbs-key-value-store-for-derived-data/)
- [Airbnb's Mussel V2: Next-Gen Key Value Storage to Unify Streaming and Bulk Ingestion (InfoQ)](https://www.infoq.com/news/2025/10/airbnb-nextgen-kv-storage-mussel/)

---

## Thread to Watch

Custom inference silicon just went from a Google-and-Amazon story to a five-player race: Google (TPU), Amazon (Trainium), Apple (M-series plus PCC), Groq (LPU), and now OpenAI (Jalapeño). Every major AI lab and cloud provider is now asking whether their next GPU procurement cycle can include non-Nvidia hardware. The answer depends entirely on whether the compiler layer (Modular MAX, OpenXLA, or a successor) can close the performance gap without requiring developers to leave PyTorch. Watch for any major cloud provider announcing a pilot of non-Nvidia AI compute in 2026 H2; that would signal the compiler problem is solved enough to matter.
