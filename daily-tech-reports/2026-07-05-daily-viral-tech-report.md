# Daily Viral Tech Report | 2026-07-05

---

## 1. Zuckerberg Tells Meta Staff Agentic AI Hasn't Accelerated in Four Months, and the Reason Is a Known Engineering Problem

**Category:** AI / ML

**The Technical Why**

At an internal town hall on July 2, Mark Zuckerberg told employees that "the trajectory of the agentic development over at least the last four months hasn't really accelerated in the way that we expected," and that Meta's AI reorg (roughly 10% of staff cut in May, about 7,000 people reassigned onto AI teams) "hasn't come to fruition yet." The admission lands on top of up to $145 billion in planned 2026 AI infrastructure spend. The technical reason this keeps happening across the industry, not just at Meta, is a compounding-reliability problem: a demo agent runs one clean task with a tidy prompt and no adversarial input, but a production agent has to chain many tool calls (read a file, call an API, parse the result, decide the next step, write output) end to end without a human catching mistakes mid-flight. If each step in that chain succeeds 95% of the time, a 20-step plan only completes successfully 36% of the time (0.95^20), and a 40-step plan drops to about 13%. Nobody has shipped a general, cheap way to push per-step reliability high enough, or to make failures self-correcting mid-run, so the gap between "the agent nailed the demo" and "the agent replaces a human workflow at 99.9% uptime" stays wide. That is the specific, nameable engineering wall Zuckerberg is describing without naming it.

**Why It Matters**

Meta is the clearest public data point yet that throwing headcount and capital at agent products does not reliably compress the timeline, because the bottleneck is a reliability-engineering problem (verification, error recovery, long-horizon planning) rather than a scale problem you fix by moving more engineers onto it. Zuckerberg still told staff to expect "more significant benefits" in three to six months; anyone building an agent product should read this as confirmation that the failure mode is structural across the field, and budget real engineering time for step-level verification and recovery loops instead of assuming a bigger model closes the gap.

**Go Deeper**

- [Mark Zuckerberg tells staff that AI agents haven't progressed as quickly as he'd hoped (TechCrunch)](https://techcrunch.com/2026/07/02/mark-zuckerberg-tells-staff-that-ai-agents-havent-progressed-as-quickly-as-hed-hoped/)
- [Mark Zuckerberg says Meta's agentic AI efforts aren't progressing as fast as he had hoped (SiliconANGLE)](https://siliconangle.com/2026/07/02/mark-zuckerberg-says-metas-agentic-ai-efforts-arent-progressing-fast-hed-hoped/)
- [Meta's Zuckerberg Admits AI Agents Are Behind Schedule, While AWS and Microsoft Bet Billions They're Not (FourWeekMBA)](https://fourweekmba.com/ai-meta-zuckerberg-ai-agents-behind-schedule-aws-microsoft/)

---

## 2. Three.js r185 Ships WebXR on WebGPU and Forward+ Clustered Lighting, Bringing Many-Light Scenes to the Open Web

**Category:** Web Graphics & GPU

**The Technical Why**

Three.js r185 (released July 1) adds two things that matter more than a typical point release: WebXR support running on the WebGPURenderer, and Forward+ (clustered) lighting. The WebXR-on-WebGPU piece is hard because an XR compositor has strict, real-time framing requirements, it needs a stereo frame (one image per eye) submitted just in time for the headset's display refresh, or the user gets visible judder that causes motion sickness. WebGL's rendering model was simple and constrained enough that browsers could paper over timing gaps easily; WebGPU hands the app much more explicit, lower-level control over command buffer submission and GPU queues, which is more powerful but means the browser and the library both have to get frame pacing right themselves instead of relying on the old API's guardrails. That is why WebXR-on-WebGPU support lagged well behind basic WebGPU rendering. Forward+ clustered lighting solves a different old problem: classic forward rendering's shading cost scales with (number of lights times number of pixels touched), so a scene with a few hundred dynamic lights becomes unrenderable in real time. Clustered forward+ instead divides the camera's view frustum into a 3D grid of cells (typically tens of cells across, depth-sliced logarithmically), computes which lights overlap which cell in a compute shader pass, and then each pixel only tests the small list of lights assigned to its own cell rather than every light in the scene. That turns an O(lights x pixels) problem into something closer to O(pixels x lights-per-cluster), the same trick that let real-time games move past a few dozen lights years ago, now available as one renderer setting in a browser-only 3D library.

**Why It Matters**

A single Three.js codebase can now target desktop, mobile, and VR/AR headsets with hundreds of dynamic lights, all through WebGPU, with no native engine or app-store install required. For anyone building GPU-heavy interactive tools that ship straight into a browser tab, this closes two of the last gaps (headset support, many-light scenes) that used to force a choice between "ship on the open web" and "get real-time graphics fidelity."

**Go Deeper**

- [Release r185 (mrdoob/three.js, GitHub Releases)](https://github.com/mrdoob/three.js/releases/tag/r185)
- [WebGPURenderer (three.js official docs)](https://threejs.org/docs/pages/WebGPURenderer.html)
- [Field Guide to TSL and WebGPU (Maxime Heckel, independent technical explainer)](https://blog.maximeheckel.com/posts/field-guide-to-tsl-and-webgpu/)

---

## 3. An AI Coding Agent Rewrote Postgres in Rust, and It Now Passes 100% of Postgres's Own Regression Suite

**Category:** Developer Tooling / Systems

**The Technical Why**

pgrust is a from-scratch reimplementation of PostgreSQL in Rust that, as of its latest milestone, matches real Postgres's output on all 46,000+ queries in Postgres's own regression test suite, and can boot directly from an existing Postgres 18.3 on-disk data directory. The reason this is a hard problem, not just a big one, is that Postgres's true behavior (locking semantics, MVCC visibility rules, planner cost-model quirks, decades of undocumented edge cases) exists only in the C source and in what the regression suite happens to check, there is no separate spec to port from. The pgrust team's approach was to treat Postgres's own test suite as the oracle: keep the new implementation "Postgres-shaped," run every change against the same 46,000 queries, and let mismatches define the remaining work, rather than reasoning from documentation that does not fully capture real behavior. The project leaned on AI coding agents (Codex, per the maintainer) as the primary engine writing the roughly 250,000 lines of Rust, with a human steering architecture and reviewing against the oracle. It is explicitly not production-ready yet: it has not been performance-tuned, and Postgres extensions that rely on C's dynamic-loading model (PL/Python, PL/Perl, PL/Tcl) do not port cleanly into Rust without a compatibility shim, since Rust has no equivalent of C's loose ABI for loadable modules.

**Why It Matters**

This is a concrete, checkable data point on what current AI coding agents can do unsupervised on a large systems codebase, and Postgres is a brutal test case precisely because "mostly correct" fails the moment a query's output differs by one row or one lock ordering from 25 years of accreted production behavior. If the pattern holds past this one project, it is a template for memory-safe rewrites of other C systems whose real specification is "match what the existing reference implementation's test suite says," with AI doing the bulk of the line-by-line work and humans steering against the oracle.

**Go Deeper**

- [pgrust: Postgres rewritten in Rust, now passing 100% of the Postgres regression tests (GitHub)](https://github.com/malisper/pgrust)
- [pgrust: Rebuilding Postgres in Rust with AI (malisper.me, project maintainer's blog)](https://malisper.me/pgrust-rebuilding-postgres-in-rust-with-ai/)
- [pgrust update: at 67% Postgres compatibility, and accelerating (malisper.me)](https://malisper.me/pgrust-update-at-67-postgres-compatibility-and-accelerating/)

---

## 4. HBM's Land-Grab on DRAM Wafer Capacity Is Now the Reason Ordinary RAM and SSDs Cost More

**Category:** Systems & Engineering / Infrastructure Economics

**The Technical Why**

AI accelerators hit a "memory wall": a modern training or inference chip can execute far more floating-point operations per second than a standard DRAM package can feed it data, so High Bandwidth Memory exists specifically to maximize bytes-per-second delivered to the compute die, not to store data more cheaply. HBM does this by stacking multiple DRAM dies vertically and connecting them to the logic die with through-silicon vias (TSVs), a process that is physically more complex per gigabyte than printing a standard planar DDR5 die. That complexity shows up directly in fab economics: HBM consumes roughly three times the wafer capacity of DDR5 per gigabyte produced, and the TSV-based share of total global DRAM wafer capacity is on track to rise from about 19% at the end of 2025 to about 23% by the end of 2026 as Samsung, SK Hynix, and Micron all convert lines toward HBM. Because HBM3E and HBM4 output is already reported sold out through 2027 on orders from Nvidia and AMD, none of the three major DRAM makers has a financial reason to reverse that reallocation. Every wafer redirected into HBM is roughly three wafers' worth of ordinary DRAM that does not get made, and that shortfall is now showing up as tightening supply and rising prices for the DDR5 and NAND flash that go into ordinary phones, laptops, and SSDs, hardware that has no HBM in it at all.

**Why It Matters**

This is the memory wall, an architecture concept usually confined to systems papers, showing up as a real global pricing event: Micron posted record fiscal Q3 revenue of $41.46 billion (up 346% year over year) even as its stock swung on questions about whether the HBM reallocation is a durable moat or an unsustainable capex bet. For engineers, the takeaway is concrete: component costs for anything with RAM or flash storage are being set right now by how many wafers Nvidia's supply chain can pull toward HBM, a direct, traceable link between AI accelerator architecture and the price of a laptop.

**Go Deeper**

- [Here's why HBM is coming for your PC's RAM (Tom's Hardware)](https://www.tomshardware.com/pc-components/ram/hbm-is-eating-your-ram)
- [AI Reportedly to Consume 20% of Global DRAM Wafer Capacity in 2026, HBM and GDDR7 Lead Demand (TrendForce)](https://www.trendforce.com/news/2025/12/26/news-ai-reportedly-to-consume-20-of-global-dram-wafer-capacity-in-2026-hbm-gddr7-lead-demand/)
- [Memory Chip Shortage 2026: HBM Takes 23% of DRAM Wafers (Tech Insider)](https://tech-insider.org/memory-chip-shortage-2026-ai-consumer-electronics/)

---

## Thread to Watch

pgrust is one hobby-scale proof that an AI coding agent can drive a from-scratch, behavior-matching rewrite of a systems codebase most people assumed was un-rewritable without years of manual labor; watch whether anyone points the same playbook (real implementation's test suite as oracle, AI agent as primary author, human as architect and reviewer) at another C system whose spec is "whatever the reference implementation does" (SQLite, Redis, a JS engine), and whether pgrust itself gets far enough on performance to attempt a real, non-toy deployment.
