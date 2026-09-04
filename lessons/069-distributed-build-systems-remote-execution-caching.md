# Day 69 — How does Google compile a 2-billion-line codebase for 40,000 commits a day, without every engineer's laptop recompiling the whole world from scratch?

**Date:** 2026-09-04
**Difficulty:** Expert
**Topic:** Distributed build systems and remote execution caching: the infrastructure that decides, on every single `bazel build` or `buck2 build` a developer or a CI job runs, which pieces of a codebase actually need to be recompiled right now, and which pieces someone else, somewhere in the world, already compiled from the exact same inputs a minute ago. This ledger has already used content-addressed hashing twice at two very different layers: Day 23 hashed immutable data blobs into a Merkle DAG so storage could dedupe and verify by content instead of by name, and Day 60 hashed a GPU shader's exact render-state permutation so a driver could skip recompiling a pipeline it had already compiled once. This lesson applies the same underlying idea, hash the inputs, cache by that hash, to a third layer: not data, not a shader, but an entire compile-and-test action inside a build graph with millions of files. Day 9's queue-as-shock-absorber supplies the scheduling piece (a queue of cache-missed actions in front of a worker fleet), and Day 13 and Day 68's systems-thinking lesson, that a senior fix breaks a feedback loop structurally instead of adding a checklist step, reappears here applied to correctness rather than load.
**Stack relevance:** Rare.lab's node-based editor is, mechanically, a build system: a graph of nodes is the source, and "compile to shippable code" is the build. Today, every compile of a shader graph almost certainly starts from nothing, no memory that a near-identical blur node, noise node, or standard PBR stack was already compiled somewhere else in the product, by a different artist, in a different project, five minutes ago. That is fine at the scale of a demo graph with a dozen nodes. It stops being fine at two different ceilings: inside the editor, as node graphs grow into the hundreds of nodes and a full recompile-on-every-edit starts costing the "instant preview" feel the same way a from-scratch laptop rebuild costs Google's engineers their edit-compile-test loop; and across the product, as more studios build on shared, common subgraphs and the exact same compiled output gets produced, and paid for in compute, over and over across completely different users who happen to have built the same thing. The next ceiling is content-addressing the compiled output of a node subgraph by a hash of its topology, its parameter values, and the target compiler/runtime version, and sharing that cache across the whole organization and its embedded runtimes, not just within one open editor session, so that "compile this 500-node graph" becomes "look up the 480 nodes the fleet has already compiled, and only run the compiler on the 20 that actually changed."

---

## 1. The company and the breaking number

**Google's Piper, and the number underneath "billions of lines."** Rachel Potvin and Josh Levenberg's 2016 Communications of the ACM paper, "Why Google Stores Billions of Lines of Code in a Single Repository," discloses the concrete numbers: as of that writing, Google's single monolithic repository held more than 2 billion lines of code across roughly 9 million source files, more than 10,000 engineers committed to it, and on a typical workday it absorbed roughly 40,000 commits, about 16,000 made directly by humans and another 24,000 made by automated systems. The repository itself served an average of roughly 500,000 queries per second on a normal workday, spiking to roughly 800,000 queries per second at peak. Every one of those 40,000 daily commits can, in principle, invalidate the cached build result for anyone downstream who depends on the files it touched, which is the number that actually matters for a build system: not how big the codebase is, but how fast it keeps changing underneath every engineer at once.

**What sits on top of that repository: Bazel's own account of the scale it was built for.** Bazel's official documentation on distributed builds states plainly that Google runs millions of builds a day, executing millions of test cases and producing petabytes of build output, from a codebase of billions of lines of source code, every single day. That is not a peak number, it is the routine, daily figure the internal predecessor to Bazel, Blaze, was built to survive. "Software Engineering at Google," the book Google's own engineers wrote about this infrastructure, names the internal remote-execution system behind it Forge, and describes its three working parts: a Distributor that sends each unit of build work, called an action, to a Scheduler; the Scheduler holds a cache of action results and can return an already-computed answer immediately if anyone, anywhere, has already produced that exact result before; and a large pool of Executor jobs that actually run the actions that are not already cached, storing their results in a content-addressed store (ObjFS, backed by Bigtable) that every future action, and every other engineer, can read from.

**Meta's independent, second data point: Buck2.** Facing the same shape of problem in a different monorepo (fbsource), Meta rebuilt its build system from scratch as Buck2, disclosed in an April 2023 engineering blog post as a full rewrite in Rust, chosen specifically because garbage collection is a poor fit for a build system holding a dependency graph with millions of nodes in memory. Meta's own reporting states that thousands of developers now run millions of Buck2 builds a day, completing roughly twice as fast as the equivalent build under Buck2's predecessor, Buck1, and that Buck2 was deliberately built around a single incremental dependency graph with no separate phases, specifically to avoid the class of bugs a phased build produces when different phases disagree about what has actually changed. Two organizations, two completely different codebases and languages, arrived at the same architecture independently, which is itself evidence this is a load-bearing pattern and not one company's idiosyncratic choice.

**The disaster that shows what happens when the cache itself cannot be trusted: CVE-2025-36852, "CREEP," disclosed June 27, 2025.** Security researchers at Nx publicly disclosed a critical vulnerability, severity score 9.4, in remote build caching systems built on shared object storage (S3, GCS, and similar), affecting Nx's own remote cache and, structurally, any build system using the same "first write wins" caching model. The flaw: when a pull-request branch and a protected branch (like `main`) contain identical source files and both try to build and write to the same cache key at close to the same time, whichever finishes first becomes the accepted result for everyone, including the trusted, protected branch. Nx and independent researchers documented this as allowing any contributor with ordinary pull-request access, no special privileges, to inject a compromised build artifact into what a production deployment would treat as a verified, cached result, and Nx's own disclosure described the vulnerability as affecting thousands of organizations worldwide. It bypasses encryption, access controls, and checksum validation entirely, because the poisoning happens during the race to create the cache entry, before any of those checks ever run. A related, independently documented case from security researcher Adnan Khan's May 2024 GitHub Actions cache-poisoning research describes a real four-step chain against Angular's own `dev-infra` repository: script injection, deliberately filling and evicting cache slots to force a specific replacement, poisoning the resulting `node_modules` cache entry, and having a later trusted workflow restore that poisoned cache and expose credentials. Neither incident is a story about a build system being too slow. Both are stories about what happens when the one property the entire cache depends on, that a given key can only ever mean one specific, verified result, quietly stops being true.

---

## 2. Why the naive (demo) design dies

**The obvious version:** every developer, and every CI job, runs the exact same command any small project runs: check out the source, and compile everything from scratch, using only whatever the local machine already has sitting in its own build cache from previous local runs. This is exactly how build tooling starts everywhere, `make`, a language's own default build tool, a from-scratch Docker build, because for a codebase of a few thousand files touched by a handful of engineers, it is completely adequate: a clean build takes seconds to minutes, and the local disk cache from your last build usually covers most of what you did not just change.

**Death one: the same compile happens millions of times, once per machine, for identical inputs.** At Google or Meta's scale, thousands of engineers regularly depend on the same shared libraries, and any of them touching an unrelated file elsewhere in a 2-billion-line repository still, in a naive local-only design, forces their own machine to recompile everything downstream of whatever changed, even when a hundred other engineers already compiled that exact same object file, from the exact same inputs, minutes earlier on their own machines. The waste does not scale with the size of the codebase alone, it scales with the number of engineers multiplied by the rate of change, and at 40,000 commits a day into one shared repository, that product is enormous. A local-only cache has no way to know that the answer it is about to spend ten minutes computing already exists, verified, somewhere else.

**Death two: a laptop's cores and memory are a fixed, small number, and a dependency graph with millions of nodes is not.** Buck2's own design rationale is explicit about this: a build graph at Meta's scale has to hold millions of nodes in memory and schedule work across it with real concurrency, which is precisely why garbage-collected memory management was rejected for Buck2's core. A single developer machine, however capable, has a fixed core count and a fixed amount of RAM. A build that could, in principle, be parallelized across a thousand machines is capped, on a laptop, at however many cores that one laptop has, and a codebase whose dependency graph keeps growing eventually produces a from-scratch build time measured in hours on a machine that was supposed to give an engineer a sub-minute edit-compile-test loop.

**Death three: without a strict contract on what an action is allowed to depend on, nothing is safe to cache, and nothing is safe to trust once it is.** A build step that silently reads the local system clock, an unpinned environment variable, or whatever version of a system library happens to be installed on that particular machine (Bazel's own documentation names exactly this failure mode for C and C++ toolchains picking up glibc versions invisibly) produces a result that looks identical to a correct one but is not reproducible: run the same command on two different machines and get two different binaries, both silently labeled as "the same build." A naive local build tolerates this because nobody else ever sees the result. The instant you try to share that result as a cache entry for someone else to reuse, an unhermetic action becomes actively dangerous rather than merely wasteful: it does not just fail to save time, it can silently hand every future consumer of that cache key a wrong answer they have no way to detect, which is precisely the mechanism CVE-2025-36852 and the Angular cache-poisoning chain both exploited.

---

## 3. The architecture

```
Developer / CI client (bazel build ..., buck2 build ...)
  - job: parse the build files and construct the action graph, a DAG where
    every node is one unit of work (compile this file, link this binary,
    run this test) with an explicit, declared list of inputs and outputs
  - analogy: a foreman reading the full blueprint before assigning a
    single hammer swing, so every task's exact dependencies are known
    up front, not discovered mid-build

        |
        v
Content-addressed cache key derivation (hash of every declared input:
source bytes, compiler flags, toolchain version, not file names or
timestamps)
  - job: turn "did anyone already do this exact unit of work" into a
    single hash lookup, the same content-hashing idea Day 23 used for
    data blobs and Day 60 used for a compiled GPU shader permutation
  - analogy: a fingerprint taken of the ingredients and the recipe
    together, not the dish's name, so two identical dishes made by two
    different cooks in two different kitchens produce the same print

        |
        v
Local action cache (L1, on the developer's own machine)
  - job: skip re-running anything the same machine already computed on
    a previous build with the same inputs, at zero network cost
  - analogy: a cook's own fridge of leftovers from yesterday

        |   (cache miss locally)
        v
Remote cache / Action Cache lookup (Google's Scheduler, Buck2 and
Bazel's remote cache over the Remote Execution API)
  - job: check whether ANY machine, anywhere in the fleet, across any
    engineer, has already produced this exact result, and return it
    immediately if so, this is where the real savings live, because one
    engineer's build warms the cache for every other engineer touching
    the same code
  - analogy: a company-wide commissary kitchen's inventory list, checked
    before anyone fires up a stove

        |   (cache miss everywhere: genuinely new work)
        v
Scheduler / work queue in front of the remote worker fleet
  - job: absorb bursts of cache-missed actions (a big merge, a mass
    dependency bump) without every engineer's build stalling on
    contention for the same handful of workers, the same shock-absorber
    role Day 9's queue plays in front of any bursty backend
  - analogy: a ticket queue at a busy kitchen, not everyone shouting
    their order at the same three cooks at once

        |
        v
Remote execution worker fleet (Google's Executor pool, Buck2's Remote
Execution API workers), stateless, sandboxed, horizontally scalable
  - job: actually run the cache-missed action, in a sandbox that denies
    access to anything not explicitly declared as an input, so the
    result is reproducible and safe to cache for everyone else
  - analogy: a commissary kitchen with a thousand identical stoves, any
    of which can cook any ticket, because every ticket lists its exact
    ingredients and nothing is assumed to already be sitting on a
    counter

        |
        v
Content-addressed result store (Google's ObjFS on Bigtable, Buck2 and
Bazel's Content Addressable Storage / CAS)
  - job: durably store every action's output, keyed by the same input
    hash, deduplicated automatically because identical inputs produce
    identical keys, available for the next cache lookup from anyone
  - analogy: the commissary's own walk-in pantry, labeled by recipe
    fingerprint, not by which cook happened to make it
```

---

## 4. The transferable mechanisms

- **Hash the inputs, not the identity or the timestamp, and cache by that hash.** A cache key built from the content of every declared input, source bytes, flags, toolchain version, means two engineers who happen to produce the exact same unit of work get the exact same key, and the second one to ask gets a free, instant answer. This is the same mechanism Day 23 used to deduplicate immutable blobs and Day 60 used to skip recompiling an already-seen GPU shader permutation; here it is applied one layer up, to an entire build-and-test action.

- **Make hermeticity a sandbox rule, not a convention.** An action's declared inputs have to be the true and complete list of everything it is allowed to read, enforced by denying filesystem and network access to anything else at execution time, not merely hoped for by the engineer who wrote the build rule. This is what makes a cache hit trustworthy: if an action genuinely cannot see anything outside its declared inputs, then identical inputs are structurally guaranteed to produce identical outputs, and a cache key can be trusted without re-verifying the work it stands for.

- **Separate "did the inputs change" from "did the file's timestamp change."** A build graph reasons about staleness through the input hash, not the file's modification time, because a file can be touched, reformatted, or checked out fresh without its actual content changing, and a timestamp-based system either wastefully rebuilds things that did not really change or, worse, wrongly trusts a stale result whose timestamp happens to look fine.

- **Let the graph, not the developer, decide what needs to re-run.** Because every action explicitly declares its inputs and outputs, the build system can compute the exact transitive set of actions actually affected by a given change and skip everything else entirely, not merely cache it, the same DAG-based incremental-recomputation idea that shows up anywhere a system needs to redo the minimum necessary work after a change.

- **Fan work out to a stateless, sandboxed worker fleet instead of one machine's fixed core count.** Because a hermetic action can run correctly on any worker with no assumptions about what is already on that machine, the same graph that would take hours on one laptop's cores can be spread across a thousand interchangeable remote workers, turning wall-clock build time into a function of fleet size rather than any one engineer's hardware.

- **Queue the cache misses, don't let them stampede the workers.** A scheduler sitting between "here is the work that was not already cached" and "here is a fleet of workers" absorbs bursty demand, a big merge, a compiler upgrade that invalidates a huge slice of the cache at once, the same shock-absorber role a queue plays in front of any backend that cannot instantly scale to match a traffic spike.

---

## 5. The trade-offs

**A wrong cache hit is worse than a cache miss, and every design here is built around that asymmetry.** A miss costs time: the action runs, correctly, just later than it could have. A wrong hit costs correctness silently: it hands back a result that looks verified and is not, and because the entire point of a cache is to let consumers skip re-verifying it, nobody downstream has a reason to notice. CVE-2025-36852 exploited exactly this asymmetry: the race condition did not slow anyone's build down, it made a compromised result indistinguishable from a legitimate one. Every real system in this lesson pays extra cost, sandboxing overhead, hash computation, strict input declarations, specifically to keep misses cheap and hits provably trustworthy, rather than the reverse.

**Cost vs. latency, paid as fleet size.** A remote execution and caching layer is not free: it means running and maintaining a fleet of worker machines and a large, durable content-addressed store, storage and compute that exist purely to make other engineers' builds faster. Google and Meta both judged that cost worth paying at their scale because the alternative, tens of thousands of engineers each redoing the same compiles independently, costs vastly more in aggregate compute and, more expensively, in engineer time spent waiting. A five-person team would rightly make the opposite trade and just build locally.

**Consistency of the cache vs. openness of who can write to it.** The safest design only lets trusted, verified pipelines (a protected CI job, not an arbitrary pull request branch) write results that other trusted consumers will read, exactly the boundary CVE-2025-36852 showed getting blurred when an untrusted pull-request build and a trusted main-branch build were allowed to race for the same cache slot. Restricting write access to the shared cache to trusted contexts only closes this hole, but it also means a developer's own local, unverified experiment never gets to warm the shared cache for anyone else, a real loss of potential savings traded away for a real security guarantee.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is **a cache turns one bad, unverified write into an instantly replicated failure, because the entire value of a cache is that nobody downstream re-checks it.** A single non-hermetic action, one that reads an environment variable it never declared, or one write that wins a race it should have lost, produces a wrong result under a key that every future identical request will now match. That wrong result does not stay local to whoever produced it. The next engineer who touches the same inputs gets the same poisoned answer, instantly, without running anything themselves, and any action further downstream that consumes that result as one of its own inputs inherits the corruption silently, because its own hash was computed trusting an input it never independently verified. This is the same shape as the retry storms and thundering herds this ledger keeps returning to, a small local event amplified by a mechanism built for speed, except here the thing that spreads is silent incorrectness rather than load, which makes it strictly harder to notice: a thundering herd shows up immediately as a graph spike, a poisoned cache entry shows up, if it shows up at all, as a mysteriously broken build weeks later that nobody can reproduce because the "broken" machine is just faithfully returning the cached answer it was told to trust.

The naive fix, telling engineers to write careful, hermetic build rules and review them closely, does not break this loop, because it relies on every single action, forever, being written correctly by hand, and CVE-2025-36852 demonstrates that even a fully automated, well-funded pipeline can have this exact property violated by a race condition nobody was specifically looking for. The senior fix is structural, the same move Day 13's backpressure lesson and Day 68's feature-flag lesson both make for load and rollout risk respectively, applied here to correctness: enforce hermeticity by construction, a sandbox that makes it physically impossible for an action to read anything outside its declared inputs, so there is no such thing as an accidentally non-hermetic action to begin with; and restrict who is allowed to write a first-class, trusted cache entry at all, so an untrusted pull-request build can never even enter the race that CVE-2025-36852 depended on winning. Neither fix is "add more review," both are "remove the class of state where the bad outcome was ever reachable," which is the difference between a system that occasionally gets lucky and one that cannot fail that particular way at all.

---

## Sources

- [Why Google Stores Billions of Lines of Code in a Single Repository, Communications of the ACM (Potvin & Levenberg, 2016)](https://cacm.acm.org/research/why-google-stores-billions-of-lines-of-code-in-a-single-repository/): primary source for Google's Piper monorepo scale, more than 2 billion lines of code, roughly 9 million files, more than 10,000 engineers, roughly 40,000 daily commits (about 16,000 human, 24,000 automated), and the roughly 500,000 average / 800,000 peak queries-per-second figures; accessed via search-indexed excerpts and secondary summaries, direct fetch of cacm.acm.org was blocked by this session's network egress policy.
- [Distributed Builds, bazel.build official documentation](https://bazel.build/basics/distributed-builds): source for the disclosed figure that Google runs millions of builds a day, executing millions of test cases and producing petabytes of build output, from a codebase of billions of lines of code; direct fetch was blocked by this session's egress policy, details drawn from search-indexed excerpts of the official Bazel documentation.
- ["Software Engineering at Google," Chapter 18 (Winters, Manshreck, Wright; O'Reilly / abseil.io)](https://abseil.io/resources/swe-book/html/ch18.html): source for the Forge remote execution architecture, the Distributor, Scheduler (action-result cache), and Executor pool roles, and the ObjFS-on-Bigtable content-addressable result store accessed via objfsd; direct fetch of abseil.io was blocked by this session's egress policy, details drawn from search-indexed excerpts.
- [Build faster with Buck2: Our open source build system, engineering.fb.com (April 6, 2023)](https://engineering.fb.com/2023/04/06/open-source/buck2-open-source-large-scale-build-system/): source for Meta's Buck2 scale (thousands of developers, millions of builds a day), the roughly 2x speed improvement over Buck1, the Rust-for-memory-control-over-large-graphs rationale, the single incremental dependency graph with no separate phases, and Buck2's use of the Bazel-originated Remote Execution API for remote caching and execution; direct fetch of engineering.fb.com was blocked by this session's egress policy, details drawn from search-indexed excerpts.
- [CVE-2025-36852: Critical Cache Poisoning Vulnerability Affects Multiple Build Systems, nx.dev blog](https://nx.dev/blog/cve-2025-36852-critical-cache-poisoning-vulnerability-creep): primary disclosure source for the "CREEP" vulnerability, disclosed June 27, 2025, severity 9.4, the first-write-wins race condition between untrusted pull-request builds and protected-branch builds writing to the same bucket-based remote cache key, and its description as affecting remote-cache plugins broadly, not only Nx; accessed via search-indexed excerpts, direct fetch blocked by this session's egress policy.
- [Nx Identifies Critical Security Vulnerability in Build Cache Systems, Businesswire (June 26-27, 2025)](https://www.businesswire.com/news/home/20250626857363/en/Nx-Identifies-Critical-Security-Vulnerability-in-Build-Cache-Systems-Affects-Thousands-of-Organizations-Worldwide): corroborating source for the disclosure timing and the "affects thousands of organizations worldwide" scope claim.
- [The Monsters in Your Build Cache - GitHub Actions Cache Poisoning, Adnan Khan, Security Research (May 6, 2024)](https://adnanthekhan.com/2024/05/06/the-monsters-in-your-build-cache-github-actions-cache-poisoning/): source for the general GitHub Actions cache-poisoning technique (cache-smashing to force eviction and replacement) and, via a companion post on the same site, the real four-step chain against Angular's own dev-infra repository (script injection, cache filling, node_modules poisoning, credential exposure on cache restore); accessed via search-indexed excerpts, direct fetch blocked by this session's egress policy.
- Day 9 (this ledger, queue as shock absorber), Day 13 (backpressure and load shedding), Day 23 (content-addressed storage and Merkle DAGs), Day 60 (shader compilation and PSO caching), Day 68 (feature flags and progressive rollout): the ledger's own prior lessons this one builds directly on, for the content-hashing mechanism, the queue-in-front-of-a-worker-fleet pattern, and the "break the loop structurally" systems-thinking frame reused here for cache correctness instead of load or rollout risk.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of every cacm.acm.org, bazel.build, abseil.io, engineering.fb.com, nx.dev, and adnanthekhan.com page consulted, so the figures above are drawn from search-indexed excerpts of those pages rather than a full read of the original text. The core scale figures this lesson leans on hardest, Google's 2-billion-line / 40,000-commits-a-day repository numbers and the "millions of builds, millions of tests, petabytes of output" Bazel figure, come from a peer-reviewed CACM paper and Bazel's own official documentation respectively, and are treated as solid; the Forge architecture description and the Buck2 figures rest on a smaller number of secondary and official-blog sources and are treated as accurate but less independently cross-checked; the CVE-2025-36852 and Angular incident details come from the vulnerability's own primary disclosure and a named security researcher's published, dated research, both treated as solid, dated, real-world accounts rather than speculative examples.

---
