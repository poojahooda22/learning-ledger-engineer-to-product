# Day 95: A 500-million-action build finishes in minutes because 94 to 98 percent of it is never re-run. How Bazel, Turborepo, Nx and Salsa (Google, Vercel, rust-analyzer) cache computation itself

**Date:** 2026-10-22
**Difficulty:** Advanced (caching the work, not the data)
**Topic:** Day 4 and Day 19 cached DATA (a page, a row). Today caches COMPUTATION: "I have already run this exact program on these exact inputs, so here is the answer." One idea powers Google's build system, your CI, your IDE's red squiggles and a node-graph shader compiler.
**Stack relevance:** Rare.lab compiles a node graph into shippable shader code and stores it content-addressed in Cloudflare R2. That is already half of a build cache. Section 9 shows the missing half (the action cache) and the per-node early cutoff that makes live editing fast.
**A note on the pasted transcript:** the LeetCode mock interview is already taught in [Day 74](074-leetcode-online-judge-end-to-end.md) (sandbox Day 50, leaderboard Day 54). I did not repeat it. I used its recipe in Section 0 and summarized it again in Section 10, as you asked for every reference to be explained.
**Honesty labels:** CONFIRMED = read in a source today (search excerpts only; I could not open full pages). INFERENCE = my reasoning, labeled.

---

## 0. The interview recipe applied to a build cache

1. **Functional.** Given a build step (compile, bundle, test), return its outputs. If the same step ran before anywhere on the team, reuse the result instead of running it.
2. **Non-functional.** Correct above all (a wrong cached answer is worse than a slow build). Fast lookup (milliseconds). Scales to thousands of engineers and CI runners. Survives cache loss (fall back to running it).
3. **Entities.** Action (command plus declared inputs), input digest, action key, action result (exit code plus output digests), blob (a file's bytes).
4. **API.** `GetActionResult(key)`, `FindMissingBlobs(digests)`, `Upload(digest, bytes)`, `Download(digest)`.
5. **Naive design.** Every developer and every CI job rebuilds everything from scratch on its own machine.
6. **Deep dives.** The breaking numbers, the two-store design, hermeticity, early cutoff, cache poisoning.

---

## 1. Named companies and the breaking numbers

- **Google, ~500 million build actions in a day (CONFIRMED, a 2010 Google slide deck, old but real).** On that day the action cache hit rate was 94 percent. So only about 30 million of 500 million actions actually ran.
- **Google today (CONFIRMED, a 2025 SREcon talk abstract on Google's planet-scale build and CI stack):** a 98 percent cache hit rate across diverse environments, using layered local, remote and peer-to-peer caches.
- **The naive cost (INFERENCE, simple arithmetic):** if a full build is 40 minutes and 300 engineers each trigger 10 builds a day, that is 3,000 builds x 40 min = 2,000 machine-hours a day for work that is mostly identical. At a 95 percent hit rate you pay for about 100 hours.
- **Turborepo and Nx (CONFIRMED):** both hash a task's declared inputs and check a local cache first, then a remote one. A monorepo with 40 packages where one changes should rebuild 1 to 3 packages, not 40.

**The breaking number for the naive design: thousands of identical builds a day, each 40 minutes, each throwing away an answer someone already computed.**

---

## 2. Why the naive design dies

**Failure 1: Redoing known work.**
- Analogy: 300 students each solving the same homework from scratch, because nobody shares the answer key.

**Failure 2: Slow feedback kills flow.**
- A 40 minute CI turns "fix a typo" into a half-day. Engineers batch changes, batches break more, fixes take longer. That is a feedback loop (Section 6).

**Failure 3: The lazy fix is wrong.**
- The first cache people build is "skip if the output folder exists" or "key it by branch name." It serves stale results. Real example from the Turborepo community (CONFIRMED, third-party source, treat as an example): an undeclared `API_URL` env var was not part of the hash, so a stale staging build was restored from the shared cache into production. A cache that is fast and wrong is a bug factory.

---

## 3. The architecture, top to bottom

```
Developer laptop / CI runner
   |  1. "I need to run action A"
   v
Build client (Bazel, Turbo, Nx)
   |  2. Compute the ACTION KEY = hash( command + hash of every declared input file
   |                                    + tool versions + declared env vars )
   |       job: a fingerprint of the question. (Analogy: a library call number.)
   v
Local disk cache  --hit--> use outputs                  (analogy: your own bookshelf)
   | miss
   v
Remote cache service (gRPC or HTTP)
   +-- ACTION CACHE (AC):  key -> {exit code, [output file digests]}
   |       job: a small lookup table. (Analogy: the card catalog.)
   +-- CAS (content-addressable store): digest -> bytes
           job: the files, named by hash of their content. (Analogy: the stacks, shelved by fingerprint.)
   |  3. AC hit: download the listed CAS blobs, skip execution
   | miss
   v
Execute the action (locally, or on a remote execution worker pool)
   |  4. Upload output blobs to CAS, then write AC entry LAST
   v
Eviction (LRU) runs inside the cache service, never in the client
```

Annotations:
- **Two stores, not one (CONFIRMED, Bazel docs and BuildBuddy).** The AC is tiny and says "what happened." The CAS holds the bytes. Because files are keyed by their own hash, identical outputs from different actions are stored once, and a corrupt blob is detectable.
- **Write order matters (INFERENCE, standard practice):** upload blobs first, AC entry last, same "pointer last" trick as Day 93. Otherwise a reader can see an AC entry that points at blobs not yet there.
- **Misses are never fatal (CONFIRMED):** Bazel falls back to local execution on a miss or error from either store. The cache is an optimization, never a dependency.
- **Remote execution is optional.** The same CAS lets you also run misses on a farm of workers, so only the changed 5 percent costs compute.

---

## 4. The transferable mechanisms (6)

1. **Content addressing.** Name data by the hash of its bytes. Identical content, identical name, stored once, verifiable. (Day 23 covered Merkle DAGs.) Where else: Docker layers, git objects, R2 scene JSON.
2. **Memoization with a correct key.** Cache `f(inputs) -> outputs`, keyed by a hash of EVERYTHING `f` reads. The whole difficulty is the word everything.
3. **Hermeticity (declared inputs only).** An action may read only what it declares. Not the clock, not the network, not an undeclared env var. If it cheats, the key lies. Bazel sandboxes actions to enforce this (CONFIRMED in concept: non-hermetic inputs cause unexpected misses or poisoned results).
4. **Early cutoff (a.k.a. backdating, "green").** If an upstream step re-ran but produced byte-identical output, downstream steps need NOT re-run. Salsa and rustc do this: a query is green if its result hash did not change after re-execution (CONFIRMED, rustc dev guide excerpt). Example: you edit a comment in `math.c`; it recompiles, the `.o` file is identical, so linking and tests are skipped.
5. **Layered caches.** Laptop disk, then team remote cache, then peer cache. Each layer is bigger and slower. Same idea as L1/L2 in Day 3 but for work.
6. **Revision counters and durability (Salsa).** A global revision number bumps on each input change. Each cached value remembers the revision it was verified at. Inputs that rarely change (like the standard library) are marked high durability so a change to a low-durability file (the one you are typing in) skips checking them at all (CONFIRMED, Salsa docs excerpt).

---

## 5. The trade-offs accepted

- **Correctness vs hit rate (the big one).** A broader key (hash more things) is safer but misses more. A narrower key hits more but risks stale answers. Nx lets lint and build narrow their inputs on purpose (CONFIRMED). Choice: be correct first, then narrow with evidence.
- **Consistency vs availability per data type.**
  - AC entries: eventually consistent is fine. A missed entry just means one extra build.
  - CAS blobs: must be exactly correct (hash verified on download) but may be missing (falls back).
  - Poisoned entries: no tolerance at all. Trust is the problem (Section 6).
- **Cost vs latency.** Storing every output costs money (terabytes); evicting loses hits. Download time can exceed compute time for tiny actions, so some tools skip remote caching for cheap steps (INFERENCE, common practice).
- **Shared cache vs security.** One team-wide cache is the biggest win and the biggest attack surface (the jmmv.dev write-up is about accidental and malicious poisoning, CONFIRMED).

---

## 6. The systems-thinking lens

**The loop that actually causes failure: the slow-build death spiral.**
- Builds are slow, so engineers batch bigger changes. Bigger changes touch more inputs, so more cache misses. More misses make builds slower. Slower builds mean bigger batches. (A reinforcing loop.)
- It has a second, nastier twin: **cache poisoning.** One non-hermetic action writes a wrong result under a key. Everyone downloads it. Everyone's build is now wrong in the same way, and the "fast" cache hides it.

**The senior fix BREAKS the loop, it does not buy more machines:**
- Enforce hermeticity (sandbox actions) so the key cannot lie.
- Make AC writes trusted only from CI, while laptops read-only. (INFERENCE, a common policy; Bazel supports separate read and write access.)
- Early cutoff so a small change cannot cascade.
- Watch hit rate as a first-class metric and alert when it drops. A sudden drop from 95 to 60 percent usually means something non-deterministic (a timestamp, an absolute path) crept into an input.

---

## 7. Scale story at three tiers

- **1,000 actions (one small app):** a local disk cache is enough. Break: a new teammate or a fresh CI runner has an empty cache, so every CI run is a cold build.
- **100,000 actions (a monorepo, 50 engineers):** add a shared remote cache (Turborepo, Nx Cloud, bazel-remote). Break: network download beats compute for small actions, and the CAS fills up with old blobs. Fix: LRU eviction, skip tiny actions, compress.
- **10 million+ actions a day (Google):** layered caches (local, remote, peer-to-peer), sharded AC and CAS, and remote execution so misses run on a worker farm. Hit rate 94 percent (2010) to 98 percent (2025), per the sources above. Break at this tier: the hot blobs (a popular library) become hot keys (Day 16); fix by replicating and caching close to the runners.

---

## 8. A worked example, end to end

Monorepo with `ui`, `core`, `app`. You change one line of a comment in `core/math.ts`.

1. Turbo/Bazel hashes `core`'s inputs. The hash changed, so AC miss for `core:build`.
2. `core:build` runs (8 seconds). Output `core.js` has the SAME bytes as before (comments are stripped), so same CAS digest.
3. `app:build` depends on the digest of `core.js`, which did not change. Its key is unchanged, AC hit, download, done. With early cutoff (Bazel does this because it keys on input digests) `app` and its 4 minute tests are skipped.
4. Your teammate pulls the branch. Her `core:build` key equals yours. AC hit, she downloads `core.js` in 80 ms instead of building for 8 seconds.
5. Total compute: 8 seconds instead of 40 minutes.

Now the failure: the build script writes `Built at 14:02` into `core.js`. Every run has new bytes, so early cutoff never fires and `app` always rebuilds. Hit rate collapses. The fix is a deterministic build, not a bigger cache.

---

## 9. Map to Rare.lab's stack

**Already used:**
- **Content-addressed immutable scene JSON in R2.** That IS a CAS. A scene's hash is its name, so it caches forever at the edge with no invalidation. The manifest is the mutable pointer, updated last.

**The ceiling (INFERENCE):** a node-based editor recompiles the whole graph to shader code on every edit. Two upgrades from today's patterns:
1. **Per-node action cache.** Treat each node's compile as an action: key = hash(node type + params + hashes of upstream node outputs + compiler version). Store {generated GLSL/WGSL snippet digest} in a small table (a Supabase table or R2 object keyed by the key) and the snippet in R2 by digest. Dragging a slider on one node then recompiles that node and its downstream chain, not the graph.
2. **Early cutoff.** If a changed parameter produces an identical snippet (clamping, a constant folded away), stop. Do not recompile downstream or re-link the shader program.
3. **Share the cache across users.** Two users using the same "Noise" node with the same params get a hit. Include the compiler version in every key, or an upgrade serves stale code. Never let browsers write to the shared cache without server validation (poisoning).
4. **Browser side:** the one shared WebGL context compiles shader programs, which can take tens of milliseconds to seconds on mobile. Cache linked programs by source digest in memory so identical shaders are never compiled twice (see Day 60, PSO precaching).

Concrete first step: log the cache key hit rate of the compile step for a week. If it is under 50 percent on repeated edits, the keys are leaking nondeterminism.

---

## 10. References, summarized in plain language

1. **BuildBuddy, "Bazel's Remote Caching and Remote Execution Explained."** Walks through the two stores: the action cache (key to result) and the CAS (hash to bytes). On a hit Bazel downloads outputs and skips execution; on a miss it runs locally or remotely and uploads. Lesson: separate the tiny index from the big blobs.
2. **Bazel docs, remote caching and distributed builds (bazel.googlesource.com).** States the requirement that matters most: for remote caching to work, builds must be completely reproducible. Describes `--disk_cache`, `--remote_cache`, and that eviction is the cache service's job, not Bazel's.
3. **jmmv.dev, "bazel remote caching" (2025) and the blogsystem5 post.** The skeptical read: all the ways a shared cache can be poisoned, accidentally or on purpose. Lesson: trust boundaries (who may write) matter as much as the hash.
4. **SREcon25 EMEA talk, "The Bitter and the Sweet of Running a Planet-Scale Build & CI Stack at Google."** Reports a 98 percent hit rate with layered local, remote and P2P caches. I saw only the abstract; internals of Google's system are not public beyond this, so treat Sections 3 and 7 for Google as inference.
5. **Google 2010 slide deck (Ashish Kumar, QCon SF).** 94 percent action cache hit rate on ~500M actions in a day. Old, but the only hard number I found from Google on this.
6. **Turborepo caching docs and a community post on cache misses from env vars.** Global hash plus per-task hash, and the "undeclared env var" trap. Lesson: anything not declared is not hashed.
7. **Nx, "How Caching Works."** The computation hash covers project sources, dependencies, global config, external dependency versions, runtime values like the Node version and CLI flags. Local first, then remote.
8. **Salsa book (red-green algorithm) and rustc dev guide on incremental compilation.** Revision counter, per-query verification, durability, and green nodes whose result hash did not change. This is early cutoff in a compiler. I did not find a source on Build Systems a la Carte in my searches; it is the classic paper (Mokhov, Mitchell, Peyton Jones) and worth reading, but I have not read it for this lesson.
9. **The pasted LeetCode mock interview (YouTube).** A Google engineer designs LeetCode: functional requirements (list problems, view and code in any language, submit with instant verdict, live contest leaderboard), non-functional (availability over consistency, low latency, isolation of untrusted code, 100k contest users, no single point of failure), entities (problem, user, submission, leaderboard), REST APIs with pagination, DynamoDB for problems, language-specific containers (the interviewer noted VMs isolate better but cost more and suggested serverless), a queue between API and runners with exponential retry, Redis cache for the leaderboard, serialized language-neutral test cases, and primary-replica databases. Full teaching is in [Day 74](074-leetcode-online-judge-end-to-end.md). Link to today: a judge that sees the same submission or the same test run twice can return the cached verdict, keyed by hash(code + language + problem version + test set version), which is exactly an action cache. Cheat prevention note: include the judge version in the key.

---

## 11. One-line takeaway

**Key the work by a hash of everything it reads, store results by the hash of what they contain, stop propagating when the output did not change, and make cheating (non-determinism) impossible instead of unlikely.**

## Sources

- [BuildBuddy, Bazel's Remote Caching and Remote Execution Explained](https://blog.buildbuddy.io/blog/bazels-remote-caching-and-remote-execution-explained)
- [Bazel docs, remote caching](https://bazel.googlesource.com/bazel/+/master/docs/versions/8.7.0/remote/caching.mdx)
- [Bazel docs, distributed builds](https://bazel.googlesource.com/bazel/+/master/docs/versions/8.7.0/basics/distributed-builds.mdx)
- [jmmv.dev, bazel remote caching](https://jmmv.dev/2025/09/bazel-remote-caching.html)
- [Blog System/5, Understanding Bazel remote caching](https://blogsystem5.substack.com/p/bazel-remote-caching)
- [USENIX SREcon25 EMEA, Planet-Scale Build & CI Stack at Google](https://www.usenix.org/conference/srecon25emea/presentation/koczorowski)
- [Ashish Kumar, Developing Products at the Speed and Scale of Google (QCon SF 2010)](https://qconsf.com/sf2010/dl/qcon-sanfran-2010/slides/AshishKumar_DevelopingProductsattheSpeedandScaleofGoogle.pdf)
- [Turborepo, Caching](https://turborepo.dev/docs/core-concepts/caching)
- [ComputingForGeeks, Turborepo cache misses and environment variables](https://computingforgeeks.com/turborepo-cache-misses-environment-variables/)
- [Nx, How Caching Works](https://nx.dev/concepts/how-caching-works)
- [Salsa, the red-green algorithm](https://git.joshthomas.dev/mirrors/salsa/raw/branch/gh-pages/reference/algorithm.html)
- [rustc dev guide, incremental compilation](https://rust.googlesource.com/rust-lang/rustc-dev-guide/+show/refs/heads/main/src/queries/incremental-compilation.md)
- [rustc dev guide, How Salsa works](https://rustc-dev-guide.rust-lang.org/salsa.html)
