# Day 86: How does a file you saved on your laptop show up, intact, on your phone seconds later, when 700 million people are editing, moving and deleting files at the same time? File sync, chunking, dedup and the three trees

**Date:** 2026-10-10
**Difficulty:** Advanced (sync correctness, content-defined chunking, content-addressed block storage, deterministic testing)
**Topic:** Day 79 covered getting one big file UP to storage. This lesson is the harder problem next door: keeping a whole folder tree identical on many devices while every device can change it at any time.
**Stack relevance:** Rare.lab scenes are content-addressed JSON in R2 plus a manifest. That is already half of a sync engine. Section 7 shows the other half.

---

## 0. The framework (same recipe as Day 74)

1. Functional: put files in a folder, see them on every linked device. Edit, rename, move, delete, share. Work offline and merge later.
2. Non-functional: never lose a file, never corrupt one, upload only what changed, converge to the same state everywhere, stay fast on a 2 GB file.
3. Entities: File, Block (a chunk of bytes named by its hash), Namespace (a user's tree), Device, Journal (ordered list of changes).
4. API: `list_folder(cursor)`, `upload_blocks(hashes)`, `commit(file, block_list)`, `longpoll(cursor)`.
5. Naive design: the client zips the folder, uploads it, and the server overwrites.
6. Deep dives: chunking and dedup, the metadata journal, the three trees, atomic moves, conflict copies, and how you test it.

If you watched the LeetCode mock interview in your prompt: it ends with "distributed database, replicas, no single point of failure". File sync is the opposite exercise. There the hard part was running untrusted code. Here the hard part is that the data is trusted but changes arrive from many places in no order.

---

## 1. The company and the breaking number

**Dropbox.** Public figures I can state with confidence from search results: the desktop sync engine runs on **hundreds of millions of user machines** (Dropbox engineering, 2020), and file content is stored as immutable blocks of at most **4 MB**, named by a **SHA-256** hash (Magic Pocket write-ups and Dropbox's own content-hash API docs, see sources).

The breaking number I want you to remember is not a traffic number. It is this one (my own experiment, labelled as mine):

- Take an 8 MB file. Cut it into fixed 400 KB blocks, 20 blocks. Insert **10 bytes** near the start. Result: **20 of 20 blocks change**, because every later byte shifted.
- Cut the same file by content (boundaries decided by the bytes themselves, average 64 KB). Same 10-byte insert. Result: **1 of 99 blocks changes**.

So one tiny edit can mean "upload the whole file again" or "upload 1% of it". At 100 million edited documents a day that difference is the whole bandwidth bill. The second breaking number is on the metadata side: renaming a folder with 50,000 files must be ONE operation, not 50,000 deletes and 50,000 adds.

Analogy: a library that reprints a whole book when you fix one typo, versus one that swaps a single page.

---

## 2. Why the naive design dies

**Naive: client uploads each changed file in full; server stores files by path; last write wins.**

- **Re-upload everything.** A 2 GB video project with one changed frame costs 2 GB of upload. On a home uplink of 20 Mbps that is about 13 minutes, and the user sits waiting. Multiply by every device that must then download it.
- **Path as identity breaks on rename.** If a file is identified by its path, then moving `/Photos` to `/Archive/Photos` looks like "delete 50,000 files, create 50,000 files". Slow, not atomic, and a crash halfway leaves half the tree in each place. Dropbox says its legacy engine had exactly this weakness and that the new protocol gives atomic moves independent of subtree size (Dropbox, 2020).
- **Two devices, one edit each, offline.** Laptop edits `report.docx`. Phone deletes it. Both reconnect. Without a record of what the common starting point was, a remote create and a local delete look identical, and you cannot tell which side changed (this is the "merge base" problem in section 4.3).
- **Duplicates waste storage.** 10,000 people save the same 80 MB conference deck. Storing it 10,000 times costs 800 GB for 80 MB of information.
- **Polling.** Asking the server "anything new?" every second from hundreds of millions of devices is a self-inflicted denial of service.

Analogy: five people editing a shared paper notebook by photocopying it. Nobody knows whose copy is newest or what the original looked like.

---

## 3. The architecture, drawn top to bottom

```
Device A (laptop)                                  Device B (phone)
  filesystem watcher                                  |
  chunker + hasher                                    |
  sync engine (3 trees, planner)                      |
        |                                             ^
        | 1. "which of these block hashes do you lack?"
        v                                             |
Edge / load balancer  (Day 85)                        |
        |                                             |
        v                                             |
Metadata service (sharded SQL)  <-- journal: ordered change log per namespace
        |   2. commit(file = [hash1, hash2, ...])     |
        |   3. append to journal, bump cursor         | 5. longpoll wakes: "cursor moved"
        v                                             |
Notification service (long poll / push) --------------+
        |
Block service  -->  Block store (Magic Pocket: immutable blocks, erasure coded, Day 32)
        4. client uploads ONLY the missing blocks
```

| Layer | Single job | Analogy |
|---|---|---|
| **Filesystem watcher** (client) | Notice a file changed without scanning the disk | A doorbell on each file |
| **Chunker + hasher** (client) | Cut the file into blocks, hash each | Tearing a book into pages and stamping each page with a fingerprint |
| **Block service + store** | Keep each unique block once, forever immutable, replicated or erasure coded | A warehouse where each item has one shelf, found by fingerprint |
| **Metadata service** | Map "file X at version 7 = these blocks" and keep the ordered journal | The card catalogue |
| **Journal + cursor** | Give every namespace a total order of changes; each device remembers "I have seen up to entry 41,902" | A numbered ledger and a bookmark |
| **Notification service** | Tell sleeping devices "your bookmark is stale" without polling | A text message saying "new mail" |
| **Sync engine** (client) | Compare three trees and plan the operations that make them agree | The accountant who reconciles two statements against last month's balance |

Key vocabulary:

- **Block / chunk.** A piece of a file, stored and addressed by the hash of its bytes.
- **Content-addressed.** The name IS the hash. Same bytes, same name, stored once. This is Day 23.
- **Dedup (deduplication).** Before uploading a block, ask the server if it already has that hash. If yes, skip the upload.
- **Journal.** An append-only ordered list of metadata changes. Devices read it from their cursor forward. This is the commit log idea from Day 42 and the change stream idea from Day 17, applied to a folder.
- **Long poll.** The client opens a request; the server holds it open until something changes or a timeout hits. One cheap idle connection instead of constant polling.
- **Merge base.** The last state both sides agreed on. Needed to tell who changed what.

---

## 4. The transferable mechanisms

### 4.1 Content-defined chunking: let the bytes choose the cut points

Fixed-size blocks (cut every 4 MB) are simple and are what Dropbox's public hash scheme uses: the API docs describe splitting a file into 4 MB blocks, hashing each with SHA-256, concatenating the hashes and hashing that (confirmed through third-party code samples that reproduce the official method). Fixed blocks have the shift problem from section 1.

Content-defined chunking (CDC) fixes it. Slide a small window (say 48 bytes) along the file, compute a rolling hash, and declare a boundary wherever the hash matches a pattern (for example its low 16 bits are zero, which happens about once per 65,536 bytes). Boundaries depend only on nearby bytes, so inserting 10 bytes at the front disturbs only the block it landed in. Everything after resynchronises.

Walked end to end, from my run (random 8 MB buffer, my own toy rolling hash, min block 16 KB, max 400 KB):

| Strategy | Blocks before | Blocks changed by a 10-byte insert near the start |
|---|---|---|
| Fixed 400 KB | 20 | **20** |
| Content-defined, avg ~64 KB | 99 | **1** |

Honest note: search results disagreed on whether Dropbox itself uses fixed or content-defined blocks internally. One secondary article claimed variable-size chunking, with no primary source, and the public hash API clearly documents 4 MB blocks. I treat "Dropbox uses fixed 4 MB blocks at the API boundary" as confirmed and "Dropbox uses CDC internally" as unverified. CDC itself is proven elsewhere (rsync's rolling checksum, the LBFS paper, restic, borg).

Trade-off: smaller average block = better delta savings but more metadata rows and more round trips. Bigger block = less overhead, worse delta. Typical systems pick an average between 64 KB and 4 MB.

### 4.2 Hash before you send: dedup as a question, not a transfer

Protocol sketch (inference of the standard shape, consistent with the 4 MB block and hash facts above):

1. Client chunks and hashes: `[h1, h2, h3, h4]`.
2. Client sends only the list of hashes: 4 x 32 bytes = 128 bytes.
3. Server answers: "I lack h3."
4. Client uploads block 3 only.
5. Client commits: `file report.docx at base_version 6 = [h1, h2, h3, h4]`.

Real example: a 50 MB PDF that three colleagues also have. The hash list is about 13 blocks x 32 bytes = 416 bytes. If all blocks exist, the upload is **416 bytes, not 50 MB**. This is also why "uploading" a file that someone else already stored can finish instantly. Security note: asking "do you have this hash" leaks existence, so real systems must authorize the commit, not just the hash (a known weakness in naive cross-user dedup).

### 4.3 The three trees: keep the merge base

This is the core of Dropbox's rewritten engine, "Nucleus", as described in Dropbox's 2020 engineering posts (confirmed from search summaries; I could not open the pages directly, see the access note). The engine persists three trees:

- **Remote Tree:** latest known state in the cloud.
- **Local Tree:** last observed state on disk.
- **Synced Tree:** the last state both agreed on. It plays the role of a merge base in version control.

Why three, not two? Take the folder `/foo` containing `fum`:

- Remote has `fum`. Local does not. Synced has `fum`. Meaning: the user deleted it locally. Plan: delete it remotely.
- Remote has `fum`. Local does not. Synced does NOT have `fum`. Meaning: someone added it on the website, the laptop has not downloaded it yet. Plan: download it.

The same two-tree picture, different truth. The third tree tells you the direction of the change. Dropbox's post makes this exact point.

Design consequence (stated in the testing post per the search summary): Nucleus stores trees, not a queue of pending work. The goal is simply "make the three trees converge". A crash cannot lose a to-do item because there are none; on restart the planner diffs the trees again and recomputes. Same idea as Kubernetes controllers (Day 38) and the autoscaling loop (Day 84): store desired state, not commands.

### 4.4 Identity by node ID, not path

Dropbox says nodes in Nucleus are keyed by unique identifiers, so a move is an update to one node's parent attribute, not a delete and recreate of every descendant (confirmed). Result: moving a folder with 50,000 files is one metadata write, atomic, instant, and a crash cannot leave half of it moved. Names live on the edge between parent and child, not in the child's identity.

Real example: dragging `Wedding 2024` (3,200 photos) into `Archive`. With path identity: 3,200 deletes and 3,200 creates, and any other device sees a storm. With node identity: set `parent(WeddingNode) = ArchiveNode`. One row.

### 4.5 The journal and cursor: pull with a bookmark, wake with a push

Every namespace has an ordered journal. `list_folder` returns entries after your cursor plus a new cursor. Because the cursor is just a position in a log, a device that was offline for 3 weeks asks for "everything after 41,902" and gets exactly what it missed, no more. A longpoll call tells it when to ask again. Dropbox's public API exposes this very shape (`list_folder/continue` with a cursor, `list_folder/longpoll`), which I am stating from my own knowledge of the public API, not from this session's searches.

### 4.6 Immutable blocks make everything else safe

A block never changes after it is written. So: no locks on data, caches are always valid, replication is a copy, repair is a re-copy, and a half-finished upload cannot corrupt anything because the file only becomes visible at the final `commit`. All mutability is concentrated in the small metadata layer. This is the same split as Day 23 and Day 32: big dumb immutable bytes, small smart mutable index.

### 4.7 Conflicts: detect with versions, resolve by keeping both

`commit` carries the base version the client edited from. If the server's current version is not that base, someone else wrote in between. The server refuses (a compare-and-set, Day 6). The client then keeps both: the loser is saved as `report (Pooja's conflicted copy 2026-10-10).docx`. Nothing is silently overwritten. For opaque binary files there is no safe automatic merge, so "keep both and tell the human" is the only honest policy. (Contrast Day 15 and Day 73, where the data model is mergeable text.)

### 4.8 Test the engine like a database: deterministic simulation

From Dropbox's testing post (search summary, confirmed): the core of the engine is deterministic for fixed inputs, which enables two tools.

- **CanopyCheck** tests the planner. It generates random trees, runs the planner, checks invariants about the converged result, and shrinks any failure to a minimal case (property-based testing in the QuickCheck family).
- **Trinity** attacks concurrency. It injects async events, changes local or remote state mid-sync, simulates IO failures and controls timing, and can also run on the real filesystem and network to reproduce platform quirks.

The lineage: the 2015 paper "Mysteries of Dropbox" (Hughes, Pierce and colleagues) used property-based testing against the old client's observed behaviour. Lesson: when a program runs on hundreds of millions of machines you cannot reproduce bugs by hand. You make the logic deterministic and let a machine search for the one input that breaks it.

---

## 5. The trade-offs

**Consistency vs availability, per data type:**

| Data | Choice | Why |
|---|---|---|
| File metadata (the tree, versions) | **Strongly consistent per namespace**, serialised through the journal | Two devices must never see two different truths about a folder. Dropbox says the rewrite redesigned the protocol for strong consistency (2020). A single ordered log per namespace is cheap because one user's folder is small. |
| File content (blocks) | **Immutable, so consistency is trivial** | Same hash always means same bytes. Replicate lazily. |
| Local edits while offline | **Availability first** | You can always edit offline. Reconcile on reconnect, keep both on conflict. |
| Notifications | **Best effort** | A missed push only delays; the cursor catches up on the next poll. |

**Cost vs latency:**

- Dedup and delta save bandwidth and storage, but cost CPU on the client (hashing a 2 GB file) and metadata rows on the server. Cheap trade at scale, because storage grows with every user and CPU is on the user's machine.
- Small blocks save more bytes, big blocks save more metadata. The per-namespace journal is the thing that must not outgrow one shard.
- Strong consistency on metadata costs a cross-region round trip for writers far from the primary. Accepted: metadata writes are tiny and infrequent compared to reads.

**The choice this system makes:** correctness of the tree over speed of any one operation. Losing or corrupting a user's file is the one unforgivable failure, so metadata is strict, data is immutable, and conflicts are never resolved by deletion.

---

## 6. The systems-thinking lens

**The feedback loop that hurts sync systems: the sync echo (a ping-pong storm).**

1. Device A downloads a change from the cloud and writes it to disk.
2. A's filesystem watcher sees a file change and thinks the USER made it.
3. A uploads it as a new local edit. The journal grows by one entry.
4. Device B sees a new entry, downloads it, writes it, its watcher fires, B uploads it.
5. Back to step 3, forever, with every device in the account.

It gets worse with a second trigger: two devices fighting over the same file (an antivirus tool or a cloud-backed app rewriting a timestamp on every download), which generates endless "conflicted copies".

How the senior fix **breaks** the loop instead of adding capacity:

- **Remember the merge base.** The Synced Tree records "this write came from the cloud". When the watcher reports the same content, the planner sees Local == Synced == Remote and plans nothing. The echo dies at step 2, not at the bandwidth meter.
- **Content hashes, not timestamps.** If the bytes are identical, there is no change, whatever the modified time says.
- **Idempotent operations.** Applying "set parent of node N to M" twice is the same as once (Day 12).
- **Backoff and coalescing.** Collapse a burst of watcher events on one file into one hash and one commit; slow down per-file when it flaps (Day 13).
- **Determinism so you can find it.** A loop like this is exactly what a Trinity-style randomized simulation finds before users do.

---

## 7. Mapping to Rare.lab

Rare.lab: Supabase Postgres with RLS, Cloudflare R2 with content-addressed immutable scene JSON plus a manifest, an embeddable runtime with one shared WebGL context.

**What we already do right:**

- **Immutable blocks + small mutable index.** Scene JSON in R2 is named by content hash, the manifest is the only mutable pointer. That is sections 4.1, 4.2 and 4.6 already.
- **Dedup for free.** Two users saving the same preset write the same key. The existence check is a cheap R2 `HEAD`.

**Where the ideas apply next:**

| Idea from today | Rare.lab application | Action |
|---|---|---|
| Chunk so a small edit changes a small part | A scene is one JSON blob. Tweaking one node rewrites the whole blob and a new hash | Split a scene into per-node (or per-graph-section) objects named by hash, with the scene as a list of hashes. Editing one shader node then uploads one small object |
| Journal + cursor | Editor autosave and the embeddable runtime checking for updates | Keep an append-only `scene_versions` table (scene_id, seq, manifest_hash). Clients ask for "after seq N" instead of refetching the manifest blindly |
| Base version on commit | Two tabs or two collaborators saving | `update scenes set manifest = $new, version = version + 1 where id = $id and version = $base`. Zero rows updated means conflict, so keep both as a fork |
| Node IDs, not paths | Node graph in the editor | Give every graph node a stable UUID. Reparenting a group is one edit, and later CRDT or OT merging (Days 15 and 73) becomes possible |
| Merge base to kill the echo | Realtime sync of an editor state that the local app also writes to | Track "last applied from server" so a remote update never re-triggers an upload |
| Deterministic core + property tests | The graph compiler (node graph to shader code) | Make compile pure, then generate random graphs and check "compiles, output deterministic, same graph hash gives same shader hash". Same spirit as CanopyCheck |

**Next ceiling:** the manifest row. Every publish and every embed check touches one hot mutable pointer per scene. Cache it at the Cloudflare edge with a short TTL and a version in the URL for the immutable parts (Day 19, Day 16), and keep Postgres writes to genuine edits.

**One-line lesson for Rare.lab:** keep bytes immutable and named by hash, break big objects into content-sized pieces so small edits upload small deltas, and make every commit carry the base version it was edited from so conflicts are detected, never overwritten.

---

## 8. Sources and what is actually in them

**Honest note on access:** direct page fetches were blocked by the network proxy this session (dropbox.tech did not resolve). Everything attributed to Dropbox below was read through search result summaries, not the original pages, so quoted detail is limited to what those summaries stated. The CDC experiment and all numbers in section 1 and 4.1 are my own run and labelled as such. The sync echo loop in section 6 is my own framing, not a Dropbox claim. The upload protocol sketch in 4.2 and the API details in 4.5 are inference or prior knowledge, labelled where used.

- [Dropbox Tech: Rewriting the heart of our sync engine (2020)](https://dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine). Plain language: the team explains why they rebuilt the desktop sync engine from scratch in Rust over about four years. The old engine had weak guarantees and path-based nodes. The new one has stable node IDs, a protocol built for strong consistency, atomic moves of any size, and three trees (Remote, Local, Synced). Read it for the "merge base" explanation and the reasons incremental fixes ran out.
- [Dropbox Tech: Testing sync at Dropbox (Isaac Goldberg, April 2020)](https://dropbox.tech/infrastructure/-testing-our-new-sync-engine). Plain language: how they test a sync engine that replaces the old one on hundreds of millions of machines. CanopyCheck generates random trees and checks the planner's invariants, shrinking failures. Trinity injects races, IO errors and timing. The key idea is a deterministic core that can be replayed.
- [InfoQ: Dropbox testing sync engine summary](https://www.infoq.com/news/2020/04/dropbox-testing-sync-engine). Plain language: a short, readable restatement of the testing post for people who do not want to read the long original.
- [Dropbox Tech: Broccoli, syncing faster by syncing less](https://dropbox.tech/infrastructure/-broccoli--syncing-faster-by-syncing-less). Plain language: Dropbox's compression work to send fewer bytes during sync (appeared in search results; I did not read it). Worth reading for the "cheapest byte is the one you do not send" idea.
- [Mysteries of Dropbox, Hughes, Pierce et al., ICST 2015](https://www.cis.upenn.edu/~bcpierce/papers/mysteriesofdropbox.pdf). Plain language: academics wrote a formal model of what the older Dropbox client does and used property-based testing to find surprising behaviours. Shows that sync correctness is hard enough to need formal methods.
- [Gigazine: Dropbox has completely improved its sync engine](https://gigazine.net/gsc_news/en/20200425-dropbox-nucleus/) and [Michael Tsai, Rewriting Dropbox's Sync Engine in Rust](https://mjtsai.com/blog/2020/03/17/rewriting-dropboxs-sync-engine-in-rust). Plain language: secondary summaries with quotations from the original posts.
- [Dropbox content hash reference implementations (example-code.com)](https://example-code.com/cpp/dropbox_content_hash.asp) and [Dev.to, Dropbox content hash in Elixir](https://dev.to/dak425/how-to-create-the-content-hash-for-a-dropbox-file-upload-in-elixir-2cp0). Plain language: third-party code that reproduces the official content hash: split into 4 MB blocks, SHA-256 each, concatenate, SHA-256 again. This is the best evidence in this session for the 4 MB block and SHA-256 facts.
- [How did Magic Pocket help Dropbox save millions? (Level Up Coding)](https://levelup.gitconnected.com/how-did-magic-pocket-help-dropbox-save-millions-2f744c643194). Plain language: a secondary write-up of Dropbox's own block storage system: immutable 4 MB blocks, a sharded MySQL metadata layer, buckets replicated in 1 GB units. Treat as a single author's account.
- [Design Dropbox / Google Drive (techinterview.org)](https://www.techinterview.org/post/3233460004/design-dropbox-google-drive/). Plain language: an interview-style walkthrough of chunking, dedup, delta sync and conflicts. Useful for the interview framing; it makes unverified claims (a 70 percent dedup saving, variable-size chunking inside Dropbox), which I did not repeat as fact.

**Confirmed vs inference.** Confirmed from search summaries: Nucleus rewrite in Rust over about four years; three trees; merge-base role of the Synced Tree; node-ID identity and O(1) atomic moves; strongly consistent protocol; CanopyCheck and Trinity; 4 MB SHA-256 block content hash. Unverified: Dropbox using content-defined chunking internally; the 70 percent dedup saving. Mine: the chunking experiment, the commit protocol sketch, the sync echo loop, the Rare.lab mapping.

**About the video in your prompt:** the LeetCode mock interview is taught in [Day 74](074-leetcode-online-judge-end-to-end.md). The structure you saw there (requirements, entities, API, naive design, deep dives, scaling) is exactly the recipe sections 0 to 6 follow here.

**Related lessons:** Day 17 (CDC and write-ahead logs), Day 23 (content-addressed storage), Day 32 (erasure coding), Day 42 (commit log), Day 73 (OT), Day 79 (large uploads), Day 84 (control loops).
