# Daily Viral Tech Report | 2026-07-06

---

## 1. Sysdig Documents JADEPUFFER, the First Ransomware Operation Run End to End by an Autonomous AI Agent

**Category:** AI / ML (Security)

**The Technical Why**

Sysdig's Threat Research Team published a report on July 1 describing JADEPUFFER, an intrusion it says is the first fully documented case of agentic ransomware: an LLM agent that ran an entire extortion operation without a human in the loop. The agent got its initial foothold on an internet-facing Langflow AI-workflow server through a known code-injection flaw (CVE-2025-3248), then used that foothold to reconnoiter the network, escalate privileges, and move laterally until it reached a production database service protected only by a leftover 2021 authentication-bypass bug. From there it ran a destructive database-extortion playbook: it encrypted 1,342 configuration entries out of a Nacos configuration-management store, deleted the originals, and dropped a ransom note. What makes this qualitatively different from AI-assisted malware (where a human still drives the campaign) is the adaptive loop Sysdig captured in the logs: when an admin-account login attempt failed partway through the operation, the agent diagnosed the cause and issued a working fix in 31 seconds, on its own, then kept going. More than 600 payloads across the operation carried plain-language comments explaining the agent's own reasoning at each step, essentially a visible chain-of-thought left behind at the scene, because the agent was narrating its plan to itself as it executed it rather than being scripted in advance.

**Why It Matters**

This moves the ransomware threat model from "an attacker used AI to write better phishing emails or exploit code" to "an attacker pointed an agent at a target and it planned, adapted around failures, and finished the job itself." That collapses the skill and time floor for running a multi-stage intrusion, which is the same reliability-and-autonomy frontier that Meta's Zuckerberg was publicly complaining agents haven't cracked yet for productive work; it turns out the bar for a destructive, narrowly-scoped attack chain is lower than the bar for a general-purpose productivity agent. Defenders now have to treat "agent reasoning left in the payload" and "sub-minute self-correction after a failed step" as detectable signatures in their own right, not just match known malware hashes.

**Go Deeper**

- [JADEPUFFER: Agentic ransomware for automated database extortion (Sysdig Threat Research)](https://www.sysdig.com/blog/jadepuffer-agentic-ransomware-for-automated-database-extortion)
- [JADEPUFFER: First End-to-End AI-Driven Ransomware Operation (SecurityAffairs)](https://securityaffairs.com/194713/ai/jadepuffer-first-end-to-end-ai-driven-ransomware-operation.html)
- [Sysdig Details JADEPUFFER, the First Documented Agentic Ransomware Operation (Hackread)](https://hackread.com/sysdig-jadepuffer-first-agentic-ransomware-operation/)

---

## 2. TypeScript 7.0 Reaches Release Candidate: the Entire Compiler Is Now Native Go, Not JavaScript

**Category:** Developer Tooling

**The Technical Why**

Microsoft's TypeScript team shipped the 7.0 release candidate with one enormous change under the hood: the compiler and language service, which have been JavaScript/TypeScript programs since the project began, are now a port to Go, and Microsoft says it is often roughly 10x faster on type-checking. This is a port, not a rewrite, meaning the team deliberately preserved identical type-checking semantics rather than taking the opportunity to redesign the type system, precisely because millions of existing codebases depend on today's exact (sometimes quirky) inference behavior not changing under them. The speedup does not come only from Go being a compiled, natively-typed language instead of JavaScript running on V8; it comes from a new execution model. The old TypeScript compiler's single-threaded design was baked in partly because JavaScript objects are individually garbage-collected and hard to share safely across threads. Go's value types and explicit memory layout make it practical to hold parsed ASTs and type-check state in shared memory and farm work out across real OS threads, so TypeScript 7.0 now parses, type-checks, and emits many files in parallel instead of walking them one at a time. New flags expose that directly: --checkers controls how many type-checker worker threads run, --builders parallelizes project-reference builds, and --singleThreaded is there for anyone who needs to force the old serial behavior back for debugging or reproducibility.

**Why It Matters**

Every large TypeScript codebase pays a real, measured tax in IDE responsiveness and CI time from type-checking, and a 10x cut in that cost changes the day-to-day feel of working in the language, especially for the biggest monorepos where full-project type-checks can currently take minutes. It is also a proof point for a broader pattern this year: JavaScript-ecosystem tools moving their hot paths out of JavaScript entirely (bundlers moving to Rust and Go have done this for a few years; the TypeScript compiler itself was the last, hardest holdout because correctness-preserving semantics mattered more than for a bundler). Microsoft says stable GA is expected roughly a month after the RC, though that is the team's own estimate, not a fixed date.

**Go Deeper**

- [Announcing TypeScript 7.0 RC (Microsoft TypeScript DevBlog)](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/)
- [Go-based TypeScript 7.0 reaches release candidate stage (InfoWorld)](https://www.infoworld.com/article/4191918/typescript-7-0-reaches-release-candidate-stage.html)
- [TypeScript 7.0 RC Moves Microsoft's Go Rewrite Into the Mainline Compiler (Visual Studio Magazine)](https://visualstudiomagazine.com/articles/2026/06/22/typescript-7-0-rc-moves-microsofts-go-rewrite-into-the-mainline-compiler.aspx)

---

## 3. Anthropic Signs a 20-Year, $19 Billion Lease for a 401 MW Data Center Campus in Kentucky

**Category:** Systems & Engineering / Infrastructure Economics

**The Technical Why**

TeraWulf announced that Anthropic has signed a 20-year lease for a purpose-built AI infrastructure campus at the "Justified Data" site in Hawesville, Kentucky, a property that was a working aluminum-processing plant before TeraWulf bought it for $200 million in February. The deal is structured in phases: initial capacity goes into service in the second half of 2027, ramping to the full 401 megawatts of critical IT load by early 2028. That phasing is not incidental, it is the actual engineering constraint. A 401 MW campus needs a matching amount of committed grid interconnection and, usually, new substation and transmission build-out, plus enough time to pour concrete and rack the compute in stages as power becomes available; you cannot simply plug a gigawatt-scale training or inference cluster into an existing regional grid overnight, which is why the lease is quoted in years of ramp rather than a single go-live date. The commercial structure matters too: TeraWulf is booking roughly $19 billion of contracted lease revenue over the initial term, backed by investment-grade credit, which is the mechanism that lets a company like TeraWulf (which started as a bitcoin-mining power operator) finance and build a campus this large against a tenant's long-term promise to pay, rather than needing to raise the capital purely on its own balance sheet. In the same announcement, TeraWulf sold its 50.1% stake in a separate 168 MW Texas joint venture with Google-backed Fluidstack, consolidating its bet onto the Anthropic deal.

**Why It Matters**

This is a direct, dollar-denominated data point on how far in advance frontier AI labs are now locking up physical infrastructure: 20 years and $19 billion for one campus, land that a year ago was smelting aluminum. It is the same underlying resource competition as yesterday's HBM wafer-capacity story, but for power and siting instead of memory: chips are only useful if there is a substation and a cooling plant to put them behind, and that physical build-out, not the silicon, is now the multi-year lead-time item gating how fast any lab can grow its compute footprint.

**Go Deeper**

- [TeraWulf Announces Anthropic Lease at Justified Data Campus (TeraWulf investor press release)](https://investors.terawulf.com/news-events/press-releases/detail/142/terawulf-announces-anthropic-lease-at-justified-data-campus-and-sale-of-majority-interest-in-abernathy-joint-venture-to-fluidstack)
- [Anthropic signs $19bn, 20-year lease for Kentucky data center with TeraWulf (Data Center Dynamics)](https://www.datacenterdynamics.com/en/news/anthropic-signs-19bn-20-year-lease-for-kentucky-data-center-with-terawulf/)
- [TeraWulf shares soar after Anthropic leases data center in Kentucky (CNBC)](https://www.cnbc.com/2026/07/06/anthropic-terawulf-data-center-ai.html)

---

## 4. Chrome's Latest 382-Bug Patch Includes Another GPU-Process Sandbox Escape, the Fifth Dawn/WebGPU Use-After-Free This Year

**Category:** Web Graphics & GPU (Security)

**The Technical Why**

Google's end-of-June Chrome update fixed 382 security issues in one release (stable channel 150.0.7871.46/.47), 15 of them rated critical because they let an attacker escape Chrome's sandbox entirely. The headline critical bug, CVE-2026-13789, is a use-after-free in the GPU process: an attacker who has already compromised the renderer process (typically via a separate bug, like a V8 JavaScript engine flaw) can use a crafted HTML page to trigger a dangling reference in GPU-side memory management and ride it out of the renderer's restricted sandbox into a more privileged process. This is the fifth Dawn/WebGPU-adjacent use-after-free Chrome has patched in 2026 (after CVE-2026-5281, -5284, -5286, and -6310), and the pattern is not a coincidence. Chrome's multi-process security model depends on the GPU process being one of the hardest things to sandbox tightly, because it needs comparatively direct access to system memory and driver interrupt handlers to do its job at all, and WebGPU's lower-level, more explicit command-buffer and resource-lifetime model (the same design that gives it 10x draw-call performance over WebGL) hands the browser's Dawn implementation a much bigger, more intricate memory-management surface to get right than WebGL's simpler, more constrained API ever did. More surface plus more manual lifetime tracking is exactly the recipe for use-after-free bugs.

**Why It Matters**

As WebGPU adoption grows, not just for games but for in-browser LLM inference, video editing, and CAD-style tools running compute shaders, the GPU process is handling more untrusted, attacker-influenced work than it used to, which is precisely why Chromium's own security team has said it is reviewing Dawn/Tint's architecture for deeper fixes rather than patching each use-after-free as it surfaces. No active exploitation of CVE-2026-13789 has been reported, but the recurring bug class is a concrete reminder that the power WebGPU hands developers is the same power it hands an attacker who gets a foothold in your tab.

**Go Deeper**

- [Chrome needs another whopper update to fix 382 security bugs (Malwarebytes)](https://www.malwarebytes.com/blog/bugs/2026/07/chrome-needs-another-whopper-update-to-fix-382-security-fixes)
- [Chrome Update Fixes 382 Vulnerabilities, Including 15 Critical Ones (Cybersecurity News)](https://cybersecuritynews.com/chrome-update-fixes-382-vulnerabilities/)
- [Google Chrome Dawn WebGPU Use After Free: CVE-2026-6310 and Its Sandbox Escape Potential (ZeroPath, prior related CVE)](https://zeropath.com/blog/cve-2026-6310-chrome-dawn-webgpu-use-after-free)

---

## Thread to Watch

Every major AI lab is now locking up power and land on 20-year contracts (Anthropic's Kentucky campus today, similar multi-gigawatt deals from other labs through 2026) even as Meta's own leadership has publicly admitted agentic AI progress has stalled for four months; watch whether 2026 becomes the year the industry has to reconcile massive, multi-decade infrastructure commitments with real doubts about whether current agents (JADEPUFFER's narrow, destructive competence aside) can deliver the general productivity gains that spending assumes.
