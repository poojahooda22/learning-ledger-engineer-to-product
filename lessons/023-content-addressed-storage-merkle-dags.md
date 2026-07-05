# Day 23 — How did Microsoft keep 4,000 engineers committing to one Git repo when a single `checkout` of the Windows codebase took 3 hours?

**Date:** 2026-07-06
**Difficulty:** Expert
**Topic:** Content-addressed storage: hashing content to derive its own address, Merkle DAGs, structural sharing and dedup, garbage collection by reachability, via Git's object model (git-scm.com's Pro Git internals docs) and Microsoft's GVFS / VFS for Git rescue of the Windows repo, generalized by the IPFS whitepaper's split of an immutable object layer from a mutable naming layer
**Stack relevance:** Cloudflare R2 storing content-addressed immutable scene JSON blobs, a Supabase Postgres manifest as the one mutable pointer, an embeddable runtime sharing one WebGL context. Day 22 already touched the R2 manifest pointer, but only for sibling reconciliation during offline multi-device edits; this lesson goes one layer down, into why the scene JSON blobs themselves are content-addressed rather than just versioned rows

---

## 1. The company and the breaking number

**Microsoft, Windows engineering, 2017. About 4,000 engineers, one shared codebase, one Git repository.** When the Windows team finished checking their entire source tree into Git, the repository held **roughly 3.5 million files and came to about 300GB**. That is not a hypothetical stress test, it is the actual production repository a real engineering org had to work in every day.

Microsoft's own Azure DevOps engineering blog, announcing the fix in 2017, states the failure in three blunt numbers: before the fix, running `git checkout` on this repo took **up to roughly 2 to 3 hours**, a plain `git status` took **almost 10 minutes**, and `git clone` took **12-plus hours**. Read that again: a developer opening their laptop for a normal day of work could lose a workday's morning just materializing the repository, and every single `status` check, something engineers run dozens of times an hour out of habit, cost 10 minutes. Brian Harry, the Microsoft engineer who wrote up the effort, described the team as trying to keep roughly **1,760 daily builds running across 440 branches**, plus thousands of pull-request validation builds, on top of that same repo. A later Azure DevOps follow-up post reported the repository's activity had grown further still, to around **4,400 active branches, 8,500 code pushes a day, and 6,600 code reviews a day**. None of that throughput is possible if every developer's basic Git command takes hours.

Here is the actual **breaking number**, and it is not a storage-capacity number, Git was never at risk of running out of disk: it is a *materialization* number. Git's default operating assumption is that every command that touches your working tree (clone, checkout, status) has to reconcile the **entire object graph reachable from HEAD** against your local disk, hashing and comparing file by file. At a few thousand files that assumption is invisible. At **3.5 million files**, it means every `git status` walks and hashes 3.5 million files to tell you that you changed one of them, even though Microsoft's own data showed **a typical developer only ever needs to touch about 50,000 to 100,000 of those 3.5 million files** in their day-to-day work. The naive design was paying a cost proportional to the whole repository's size, for an operation whose real cost should have been proportional to the tiny slice of the repository the developer actually cares about.

Hold onto that number: **3.5 million files, ~300GB, and a design that forces every local Git command to reconcile against all of it, when the actual working set for any one engineer is 50 to 100 thousand files.** Everything in this lesson is the machinery that makes it safe to only ever fetch, verify, and store the tiny slice you actually need, while still guaranteeing, with mathematical certainty and no central authority to ask, that the slice you have is exactly correct.

---

## 2. Why the naive (demo) design dies

The demo version of "store every version of every file" looks like this:

```sql
CREATE TABLE file_versions (
  repo_id      BIGINT,
  path         TEXT,
  version      INT,
  content      BYTEA,
  committed_at TIMESTAMP,
  PRIMARY KEY (repo_id, path, version)
);
```

One row per path, per version, keyed by where the file lives and which save this was. This works fine for a demo with a handful of files and a handful of edits. It fails in three concrete ways once real version history and real collaboration enter the picture.

**a. Duplicate bytes, everywhere, across versions and across branches.** If a 50,000-line configuration file gets one line changed, this design writes the *entire* 50,000-line file again as a new row, because "version" is just an incrementing integer with no relationship to what actually changed. Two branches that both start from the same file and each make an unrelated one-line edit store two full duplicate copies of everything they share, because the key is `(path, version)`, not `(what the bytes actually are)`. There is no way for the storage layer to notice "these two rows are 99.998% identical" without an expensive, explicit diff step bolted on afterward.

**b. Cache invalidation becomes a permanent, load-bearing problem, not an edge case.** A cache or CDN keyed on `path` alone has no way to know when the meaning of that key changed, because the same path legitimately points at different bytes over time. Every commit to a file is, from a cache's point of view, a silent correctness hazard: serve `path` a millisecond after a new version lands, and you serve stale content with no error raised anywhere. This is Day 19's caching-invalidation problem, except here it is not a performance optimization gone wrong, it is baked into the storage model's own addressing scheme from day one, because the address (`path`) and the identity of the content it names are two different things that can drift apart.

**c. No cheap way to verify integrity or diff two versions.** To compare version 12 and version 47 of the same file, or to check that a copy replicated to a second data center actually matches the original bit for bit, this design has exactly one tool: read both full BYTEA blobs and compare them byte by byte, or trust whatever diff the committer's tooling happened to generate at write time. There is no independent, cheap way to ask "did this file's bytes change between these two rows" without reading the whole file, and there is no way at all to detect silent corruption (a bit flipped on disk, a partial write from a crashed replica) unless you separately maintain a checksum column and remember to check it, which the naive schema above does not even have.

The naive design's real mistake is using **location** (`path`, `version` number) as if it were the same thing as **identity** (what the bytes actually are). Content-addressed storage fixes this by making them the same thing: the address of a piece of content **is** a cryptographic hash of the content itself, computed independently by anyone, verifiable by anyone, with no coordination required.

---

## 3. The architecture

Top to bottom, the shape Git (and, at larger scale, GVFS/VFS for Git wrapped around it), and independently, IPFS's generalized version of the same idea, both converge on:

```
[Client: git commit / R2 scene save / IPFS add]
   local content changes: one edited file, one edited node in a shader graph
   |
   v
[Hash the content: SHA-1 (Git today) / SHA-256 (R2, IPFS, and Git's own
 in-progress SHA-256 mode), a Merkle-DAG-aware hash of chunks or a whole file]
   the address IS the output of hashing the exact bytes, nothing else; two
   byte-for-byte identical blobs ALWAYS hash to the same address, on any
   machine, computed independently, with zero coordination between them
   analogy: a fingerprint scanner that doesn't care whose finger it is, only
   whether these exact ridges have been recorded before
   |
   v
[Dedup check against the object store]
   "does an object with this exact hash already exist?" if yes, write
   nothing new at all, just point at what is already there
   analogy: a library that never re-shelves a book it already owns a copy
   of, it just adds another catalog card pointing at the same shelf slot
   |
   v
[Merkle DAG: blobs -> trees -> a root]
   a tree object is a short list of (name, mode, hash-of-child) entries; a
   commit (or a manifest) points at one root tree. Change ONE file: its blob
   gets a new hash, which changes the one tree entry that names it, which
   changes every tree above it up to the root, but every OTHER blob and
   tree anywhere else in the graph, untouched, keeps its old hash and is
   reused byte-for-byte, unchanged
   analogy: a family photo album where only your own portrait gets retaken;
   every cousin's photo on the facing pages stays exactly the one on file
   |
   v
[Immutable blob store: .git/objects/, an R2 bucket, an IPFS block store]
   once written, the object living at a given hash is NEVER edited in
   place; a changed file is a NEW object at a NEW address, never an
   overwrite of the old one
   analogy: a numbered, sealed safety-deposit box, once box #4F9A is
   filled and locked, nobody edits its contents; a change means renting a
   different, new box
   |
   v
[Manifest / ref: the ONE mutable pointer. Git's refs/heads/main, an R2
 manifest.json, IPFS's IPNS mutable name record]
   this is the only place "what does main / current point to right now" is
   allowed to change; everything it points to, transitively, all the way
   down the Merkle DAG, is frozen
   analogy: the sign at the front desk reading "start here." The sign can
   be moved to point at a different room tomorrow, but no room behind it
   ever gets silently rebuilt
   |
   v
[CDN / cache layer]
   because address == hash(content), a cached blob is valid FOREVER: the
   only way its content could ever change is if its address changed too,
   which contradicts the premise, so cache with an effectively infinite
   TTL and no invalidation logic anywhere in the system
   analogy: a notarized, sealed archival copy. You never phone ahead to
   ask if the copy in your hand is still "the current one," because an old
   copy is never silently rewritten underneath you
   |
   v
[Garbage collection: sweep outward from the live manifests / refs]
   walk every object reachable from every current ref or manifest; anything
   the walk never touches is unreachable and safe to delete, once past a
   short grace window
   analogy: a librarian who keeps everything still listed on a current
   catalog card and only pulps the stock nobody's card points to anymore
```

The request-path work (hash, dedup check, write the immutable object, move the one mutable pointer) is what made GVFS's lazy version of this diagram fetch **only the 50 to 100 thousand files** an engineer actually touches, instead of all 3.5 million: because every object's address is self-verifying, a client can fetch any single object on demand, the first time it is opened, and know with certainty it got the right bytes, without ever needing to reconcile the whole tree up front.

---

## 4. The transferable mechanisms

**a. Content hash as address: the address is a mathematical function of the bytes, not a name someone assigned.** Git's own internals documentation describes exactly this: `git hash-object` takes data, stores it under `.git/objects`, and hands back the SHA-1 checksum of that data plus a small header as its permanent key, with the object filed under a subdirectory named by the first two characters of that hash and a filename of the remaining 38. Any two machines, given the identical bytes, independently compute the identical address, with no lookup table, no central registrar, and no network call required to agree on what to call it. Juan Benet's IPFS whitepaper (2014, arXiv:1407.3561) generalizes this exact idea past a single repository into a peer-to-peer network: content addressing, in IPFS's own framing, means all content is uniquely identified by its own checksum, so any two peers who fetch the "same" object from anywhere in the world are provably fetching the same bytes.

**b. Structural sharing via a Merkle DAG: a one-line change reuses almost everything else, for free.** Because a Git tree object is just a list of child hashes, changing one file only forces a new hash for that file's blob and for every tree on the path from that file up to the root, a cost proportional to tree depth, not repository size. Every sibling subtree, every untouched directory, every other file in a 3.5-million-file repository, keeps its old hash and its old, already-stored bytes. IPFS's paper describes this same object layer as forming "a generalized Merkle DAG," explicitly designed so that versioned filesystems, and anything else built on top, get structural deduplication as a direct consequence of the addressing scheme, not as a bolted-on feature.

**c. Immutability plus content addressing means cache-forever, with no invalidation logic at all.** This is the mechanism that fixes section 2b's cache-invalidation nightmare outright, rather than just managing it better: if the address of an object is a hash of its own bytes, then "the object at this address changed" is a logical contradiction, so any cache, anywhere, at any layer, can hold a content-addressed object with an unbounded TTL and never be wrong. This is the same principle earlier lessons in this ledger (Day 4, Day 19) applied to CDN edge caching of immutable assets; this lesson's contribution is naming *why* it is safe: it is not merely "we promise not to change this file," it is "the address is derived from the content, so an old copy being wrong is mathematically impossible, not just operationally unlikely."

**d. The manifest or ref is the one mutable pointer; everything it points to is frozen.** Git's `refs/heads/main` is a tiny mutable file holding one commit hash; moving it (a `git push`, a fast-forward) is the only mutation Git's model ever performs on a name, and it never touches the immutable object graph underneath. IPFS makes this split explicit as two separate protocol layers: an **Objects layer**, described as a Merkle DAG of content-addressed immutable objects, and a **Naming layer**, "a self-certifying mutable name system" (IPNS) built specifically because a pure content-addressed system has no way to say "give me whatever is current" without some mutable pointer sitting on top of the immutable graph. Every design in this lesson needs exactly one such pointer, and exactly one, per logical "current state": that pointer is where all of the system's mutability, and all of its conflict risk, concentrates.

**e. Garbage collection by reachability: mark-and-sweep from live refs, not "delete what looks old."** Git's own `git-gc` documentation describes the mechanism precisely: `git gc` gathers loose objects into packfiles and then, via `git prune`, removes any object that is not reachable from any branch, tag, the index, remote-tracking refs, or the reflog, defaulting to a **two-week grace window** (`--expire 2.weeks.ago`) before an unreachable object is actually deleted, specifically so an object that briefly looks orphaned (mid-rebase, a branch about to be re-pushed) is not destroyed before anyone notices. The reflog itself defaults to expiring entries after **90 days**, giving a much longer safety net for "restore a branch I thought I deleted." Newer Git versions add **cruft packs**, which store per-object modification times alongside a single pack of otherwise-unreachable objects, so the grace-window bookkeeping itself doesn't require keeping millions of tiny loose files around. The core idea, independent of Git specifically: nothing is ever deleted because it "looks unused," it is deleted only because a graph walk, starting from every live pointer, provably never reached it.

**f. Integrity verification comes for free, as a side effect of the addressing scheme itself.** In a path-keyed system, detecting silent corruption requires a separate, deliberately maintained checksum column, checked deliberately. In a content-addressed system, corruption is self-evident: if you fetch the object claimed to live at hash `H` and it does not actually hash to `H`, you know immediately, with no separate verification step, that something is wrong. IPFS's whitepaper lists this explicitly as a property of Merkle Links: content is verified against its own checksum, and tampering or corruption is detected automatically. Git's own `fsck` command relies on the identical property to find corrupted or truncated objects in a local repository.

---

## 5. The trade-offs

**Storage cost vs dedup savings depends entirely on the granularity you hash at.** Git hashes whole files as single blobs; change one byte anywhere inside a large file and the entire blob gets a brand-new hash, with zero byte-level sharing between the old and new versions at the object-addressing layer itself. GitHub's own engineering blog on Git's packed object store explains the fix Git actually ships: a **separate, secondary compression pass**, delta-compressing similar objects against each other inside packfiles, recovers much of the space a naive whole-blob scheme would waste, but this delta compression is layered *on top of* content addressing, not a property of the addressing scheme itself. This is exactly why Git famously struggles with large binary assets (video, design files, compiled bundles) and why Git LFS exists as a workaround: whole-file content addressing gives you perfect dedup when a file is unchanged, and close to zero dedup when a large file changes even slightly.

**Fixed-size chunking vs content-defined chunking is the sharper version of the same problem.** If you chunk a file into fixed-size blocks and hash each block independently (rather than hashing the whole file), a single byte inserted near the start of the file shifts every subsequent byte's chunk boundary, so every chunk after the insertion point gets a new hash even though its actual content, just shifted, is unchanged, destroying dedup for the entire rest of the file. Andrew Tridgell and Paul Mackerras's 1996 rsync algorithm paper (ANU technical report TR-CS-96-05) describes the fix that most modern content-defined chunking schemes still use: a **rolling checksum** scanned continuously across the byte stream, with a chunk boundary declared wherever the rolling checksum satisfies some pattern, rather than at a fixed byte offset. Because the boundary condition is a property of the *content* passing under the rolling window, not a fixed offset counted from the start of the file, an insertion shifts only the chunks immediately around it; every chunk before and after keeps its old boundaries and its old hash, and dedup survives the edit. The cost of this approach is real: computing a rolling hash over every byte of every write is more CPU than a fixed-size chunker, a direct write-amplification tax paid specifically to buy back dedup ratio under real-world edits.

**Garbage collection cost and complexity vs the simplicity of "just delete the file."** A naive system deletes a row and it is gone, one cheap operation, no graph to walk. A content-addressed system cannot do this safely, because the same object might be referenced by several different trees, several different manifests, or several different branches simultaneously; deleting it the moment one reference disappears risks deleting something still reachable from somewhere else. The correct operation, a full reachability walk from every live ref before anything is deleted, is strictly more expensive, and gets more expensive as the object graph grows, which is exactly why Git bounds it with the two-week and 90-day grace windows above rather than running a full sweep on every single commit. The trade is real and it does not go away: you buy structural sharing and cheap-to-fetch history, and you pay for it with a garbage collector that has to be run, correctly, on a schedule, forever.

---

## 6. The systems-thinking lens

**The feedback loop: unbounded storage growth is the natural resting state of an immutable store, and it is a slow-motion version of Day 21's compaction-falling-behind loop.** Because nothing in a content-addressed system is ever overwritten, every single write only ever adds bytes; the store's size is monotonically non-decreasing unless something actively runs a reachability sweep and deletes what it finds unreachable. Walk the loop through: normal operation, GC runs on schedule, unreachable objects (abandoned branches, superseded scene versions, orphaned uploads) get reclaimed within the grace window, storage stays roughly proportional to *live* content, not *all content ever written* → under load, or under organizational pressure to avoid GC's own CPU and I/O cost competing with live traffic (the same tension Day 21 named for compaction vs. reads), GC gets deferred "just this once" → deferred GC means the object graph GC has to walk next time is bigger, so the next GC pass costs more CPU and I/O, which makes it a more attractive target to defer *again* → each deferral both grows the backlog and raises the cost of clearing it, so the system does not drift back to a healthy state on its own once it starts falling behind, it accelerates away from one. That is the same metastable shape as Day 21's compaction backlog and Day 22's sibling explosion: a system with two stable operating points (GC keeping pace, GC perpetually behind and storage perpetually growing) that, once nudged into the second one, stays there.

**The senior fix is the same move as those two prior lessons: schedule the reclamation, do not add disk to a system that is actively growing unboundedly.** Adding storage capacity treats the symptom and leaves the loop's structure untouched; the object graph keeps growing exactly as fast either way. The structural fix is what Git ships by default: **`auto gc`**, triggered automatically once the count of loose objects crosses a threshold, rather than left to a human to remember, paired with bounded, non-negotiable grace windows (two weeks for unreachable objects, 90 days for reflog entries) so the sweep is cheap and predictable instead of an unbounded, dreaded, all-at-once operation someone has to schedule a maintenance window for. The general lesson, consistent with Day 21's compaction and Day 22's sibling reconciliation: an append-only or immutable design does not remove the need for cleanup, it just moves the cleanup from "part of every write" to "a separate, schedulable, reachability-based process," and that process still has to actually run, on a schedule tight enough that it never has to catch up on more than it can walk in one pass.

---

## Map to Rare.lab's stack

**Where Rare.lab already has this right, and it is not a coincidence:** Cloudflare R2 storing content-addressed, immutable scene JSON blobs is exactly section 3's diagram, correctly applied. Every scene save writes a new object keyed by the hash of its own bytes; nothing is ever overwritten in place; two devices that each save a different edit never corrupt each other's bytes, they simply produce two different hashes, both preserved. The infinite-TTL CDN caching this ledger's earlier lessons (Day 4, Day 16, Day 19) already credited to R2's content addressing is one direct consequence of this lesson's mechanism. A second consequence, not yet named in this ledger until now: **integrity verification is free.** If a scene JSON blob is ever fetched and its bytes do not hash to the key that was requested, that is detectable corruption, not silently wrong data served to a user's shader graph, the same property Git's `fsck` and IPFS's Merkle Links rely on.

**Where this lesson goes deeper than Day 22:** Day 22 covered the R2 manifest pointer, the single mutable value naming "which scene JSON is current," and the sibling-conflict problem that arises when two offline devices each move that pointer differently. This lesson is about the layer underneath the pointer: why the scene JSON blobs themselves are content-addressed at all, rather than rows in a `scene_versions` table keyed by `(project_id, version_number)`, section 2's naive design. The answer is everything sections 2 through 4 argue: dedup across near-identical scene saves (most edits touch a handful of nodes in a shader graph, not the whole scene), cache-forever semantics with zero invalidation code, and free corruption detection, none of which section 2's naive schema gets without a pile of bolted-on machinery.

**Two concrete, actionable next moves.** First, Rare.lab has no documented garbage-collection policy for R2 scene JSON blobs; every save is currently additive, forever, which is section 6's slow-motion spiral in miniature, just not yet painful because volume is still low. The fix: a scheduled job that walks every live manifest (current version, plus any explicitly retained history depth or named checkpoints), marks every blob hash reachable from that walk, and deletes anything unreached past a grace window, mirroring Git's own two-week default and Day 21's tombstone-reclaim pattern almost exactly. Second, and more directly tied to Rare.lab's stated bias toward scalability and performance: model a scene's JSON itself as a small Merkle DAG, one hash per node subtree, rather than one flat hashed blob for the whole scene. A small edit to one shader node would then only change that node's subtree hash and the hashes on the path back to the scene root, section 4b's exact mechanism, and the embeddable runtime could diff two scene versions by comparing subtree hashes and re-fetching, re-parsing, and re-compiling only the changed subtree, instead of redownloading and recompiling the entire scene on every save. For a runtime sharing one WebGL context across possibly many embedded instances, that is the difference between a re-save costing work proportional to the edit, versus work proportional to the whole scene, every single time.

---

## References and summaries

**Microsoft Azure DevOps Blog: "Announcing GVFS (Git Virtual File System)"** (2017)
https://devblogs.microsoft.com/devops/announcing-gvfs-git-virtual-file-system/
The primary source for this lesson's breaking numbers: the Windows repository's ~3.5 million files and ~300GB size, the pre-GVFS `clone` (12+ hours), `checkout` (roughly 2-3 hours), and `status` (almost 10 minutes) times, the post-GVFS improvements (clone in minutes, checkout in ~30 seconds, status in 4-5 seconds), and the description of how GVFS virtualizes the filesystem to download a file only the first time it's opened.

**Brian Harry's Blog: "The largest Git repo on the planet"**
https://devblogs.microsoft.com/bharry/the-largest-git-repo-on-the-planet/
Microsoft engineer Brian Harry's own account of the Windows-to-Git migration: the roughly 4,000 engineers working in the one repository, the daily build volume across hundreds of branches, and the reasoning for why a typical developer only needs 50,000 to 100,000 of the repo's 3.5 million files, the source for section 1's key contrast between total repo size and actual working-set size.

**Microsoft Azure DevOps Blog: "Beyond GVFS: more details on optimizing Git for large repositories"**
https://devblogs.microsoft.com/devops/optimizing-git-beyond-gvfs/
A later follow-up documenting the repository's continued growth: roughly 4,400 active branches, about 8,500 code pushes and 6,600 code reviews per day, used in section 1 to show the scale kept climbing well past the numbers at GVFS's original announcement.

**Microsoft: VFSForGit (GitHub repository)**
https://github.com/microsoft/VFSForGit
The open-source implementation of GVFS itself, documenting the virtualized-filesystem architecture (a file-system driver that makes every file appear present, but only materializes it on first access) referenced throughout section 3's lazy-fetch diagram.

**Microsoft Azure DevOps Blog: "Introducing Scalar: Git at scale for everyone"**
https://devblogs.microsoft.com/devops/introducing-scalar/
Microsoft's own later acknowledgment that GVFS was superseded by Scalar, a tool combining the lessons of running GVFS at scale with newer, native Git features (partial clone, sparse checkout), the same kind of honest "we later moved past our own solution" note Day 22's lesson gave for DynamoDB versus the original Dynamo paper.

**Pro Git (git-scm.com): "10.2 Git Internals - Git Objects"**
https://git-scm.com/book/en/v2/Git-Internals-Git-Objects
The authoritative reference for Git as a content-addressable filesystem: how `git hash-object` derives a SHA-1 key from data plus a header, how objects are filed under `.git/objects` by that hash, and how blob, tree, and commit objects form the Merkle DAG this lesson's section 3 and 4 are built on.

**Git Documentation: `git-gc`**
https://git-scm.com/docs/git-gc
The primary source for section 4e and section 6: how `git gc` gathers loose objects into packfiles and prunes unreachable objects, the objects it deliberately keeps alive (anything referenced by branches, tags, the index, remote-tracking refs, or the reflog), and the existence of cruft packs for tracking per-object mtimes without keeping millions of loose files around.

**Git Documentation: `git-prune`**
https://git-scm.com/docs/git-prune
The specific reference for the default two-week (`2.weeks.ago`) grace window before an unreachable object is actually deleted, and the 90-day default reflog expiry, the concrete numbers behind section 4e's "mark-and-sweep from live refs, not delete-on-sight" mechanism.

**GitHub Blog: "Git's database internals I: packed object store"**
https://github.blog/open-source/git/gits-database-internals-i-packed-object-store/
The source for section 5's point that Git's real-world dedup comes from a secondary delta-compression pass over packfiles, layered on top of whole-file content addressing, not from the addressing scheme itself, explaining why large binaries dedup poorly under Git's default object model.

**Juan Benet: "IPFS - Content Addressed, Versioned, P2P File System"** (2014)
https://arxiv.org/abs/1407.3561
The primary source generalizing this lesson's mechanism beyond a single repository into a distributed network: content addressing via a multihash checksum, a "generalized Merkle DAG" as the immutable objects layer, and IPNS as the explicitly separate, self-certifying mutable naming layer sitting on top of it, the source for section 3 and section 4a/d.

**Tridgell, Mackerras: "The rsync algorithm"** (ANU Technical Report TR-CS-96-05, 1996)
https://www.andrew.cmu.edu/course/15-749/READINGS/required/cas/tridgell96.pdf
The original rsync paper, the source for section 5's rolling-checksum, content-defined-chunking trade-off: why a boundary declared by a property of the content itself (rather than a fixed byte offset) survives insertions and deletions without destroying dedup for the rest of the file, at the cost of computing a rolling hash over every byte.
