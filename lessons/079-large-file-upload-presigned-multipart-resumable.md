# Day 79: How do 10,000 people upload 500 MB files at once without melting your servers? Presigned URLs, multipart and resumable uploads (S3, R2, Dropbox, tus)

**Date:** 2026-10-03
**Difficulty:** Intermediate to Advanced (control plane vs data plane, part math, resumability, and a cleanup trap)
**Topic:** Large file ingestion. Move the bytes around your servers, not through them, and make a dropped connection cost seconds instead of the whole file.
**Stack relevance:** Rare.lab stores scenes and assets in Cloudflare R2 and uses Supabase Storage. Section 7 shows what you already do right and the next ceiling.

---

## 0. The framework (same six steps as Day 74)

1. Functional: a user picks a big file (video, texture pack, scene bundle) and it ends up durably stored and linked to their account.
2. Non-functional: survive flaky mobile networks, never route gigabytes through app servers, keep the database consistent with what is in storage, do not pay for abandoned junk.
3. Entities: Client, Upload session, Part (chunk), Object (the final file), Metadata row.
4. API: `POST /uploads` (start), `PUT <presigned part url>` (send bytes), `POST /uploads/:id/complete` (finish).
5. Naive design: the browser POSTs the whole file to your API, the API writes it to storage.
6. Deep dives: control plane vs data plane, presigned URLs, multipart math, resumability, block dedup, the orphan problem.

---

## 1. The company and the breaking number

**Dropbox, S3, Cloudflare R2, and one number: 10,000 users x 500 MB = 5 TB flowing through your app tier in a few minutes.**

- That is plain arithmetic, not a measured figure. A server with a 1 Gbps network card moves about 125 MB/s at best. 50 app servers move about 6 GB/s, so 5 TB takes roughly **14 minutes** if nothing else happens, and every upload holds a worker open the whole time.
- A phone on weak signal drops the connection around the 80 percent mark. A single-shot upload restarts from byte zero. Real example: a 2 GB video on a 10 Mbps uplink takes about 27 minutes. One drop at minute 25 throws away 25 minutes.
- Storage limits shape the design. Per Cloudflare R2's published limits (seen via search excerpts, see Section 9): a single-part upload tops out at **35 GiB**, a multipart upload at **4.995 TiB**, parts must be **at least 5 MiB** (except the last), at most **5 GiB**, and there can be at most **10,000 parts**.

Analogy: you run a warehouse. Naive design makes every customer hand parcels to your receptionist, who carries each one to the shelves. Real design: the receptionist hands out a gate pass, customers walk to the loading dock themselves, and you only check that the delivery arrived.

---

## 2. Why the naive design dies

**Naive version: browser to API server to storage.**
- **Bandwidth is paid twice.** Bytes enter your server, then leave it for storage. Your servers become a very expensive pipe.
- **Workers are held hostage.** A request lasting 20 minutes pins a thread or connection. 2,000 slow uploads can use up the pool that should serve the 50 ms API calls. This is the same shape as Day 78 (connections), with bytes instead of SQL.
- **Proxy limits bite.** Many gateways cap request bodies. Cloudflare's proxy rejects bodies over 100 MB on its free and pro plans (a known issue pattern, for example a GitHub issue titled "Uploads over 100 MB fail ... Cloudflare 413" in the references).
- **No resume.** One dropped packet restarts everything. At 1% failure per 100 MB, a 2 GB upload fails often enough that users give up.
- **Memory.** If the server buffers the file before writing, 100 concurrent 500 MB uploads is 50 GB of RAM.

---

## 3. The architecture

```
Client (browser, phone, desktop app)
   |  1. "I want to upload scene.zip, 480 MB, sha256 = ab12..."
   v
Edge / load balancer                job: TLS, rate limit, route. Small JSON only.
   |
   v
Stateless API tier (CONTROL PLANE)  job: auth, quota check, create upload session,
   |                                    sign URLs. Never touches file bytes.
   |  2. returns: uploadId + N presigned part URLs (or one URL)
   v
Client  ----- 3. PUT part 1..N, in parallel, retry each part alone ----->
                                                                         v
                                              Object storage (DATA PLANE)
                                              S3 / R2 / Supabase Storage
                                              job: accept bytes, store parts,
                                              verify, assemble
   |  4. client calls /complete with the list of part ETags
   v
API tier verifies, tells storage to assemble, writes metadata row
   |
   v
Postgres (metadata only)            job: "user X owns object Y, size, hash, state"
   |
   v
Queue + workers (async)             job: virus scan, thumbnails, transcode, index
```

Per layer:
- **Edge/LB:** the front gate. Only sees small requests.
- **API tier (control plane):** the receptionist who writes gate passes. Cheap, stateless, scales by adding instances.
- **Object storage (data plane):** the loading dock. Built to take enormous parallel byte streams.
- **Postgres:** the ledger. Stores facts about files, never the files.
- **Queue:** the back office. Slow work (scanning, transcoding) happens after the user sees "uploaded".

---

## 4. The transferable mechanisms

### 4.1 Control plane vs data plane
Split "decide and authorize" (small, needs your logic) from "move bytes" (huge, needs no logic). Your code stays on the small path. This is the single most reusable idea in the lesson and it applies to video (Day 5), CDNs (Day 4) and WebRTC media (Day 49).

### 4.2 Presigned URL (capability token)
The API signs a URL with its secret key, including the object key, method (`PUT`), an expiry (minutes to hours) and optionally size and content type. Whoever holds the URL can do exactly that one thing until it expires. Storage verifies the signature without calling your API. Analogy: a valet ticket that only opens one specific car for one hour. Real example: S3 and R2 both support this through the S3-compatible API.

### 4.3 Multipart upload with part math
Split a file into parts, upload them in parallel, then ask storage to assemble. Each part gets an **ETag** (a checksum-like receipt) that the client returns at completion.
- Part count math (R2 limits): the file must fit in 10,000 parts. 480 MB at 8 MiB parts is about 60 parts. A 4.995 TiB file needs parts of about **500 MiB** (4.995 TiB / 10,000).
- Rule of thumb (my guidance): 8 to 16 MiB parts for ordinary files. Smaller parts mean cheaper retries but more requests, and storage bills per request. Larger parts mean fewer requests but more wasted work per failure.
- R2 quirk from its docs: all parts except the last must be the **same size**, stricter than S3.
- Parallelism: 4 to 6 parts in flight saturates most home uplinks. More just causes congestion.

### 4.4 Resumability (state lives somewhere)
Two common models:
- **Multipart with a part list:** after a drop, the client asks storage which parts it has (`ListParts`), then uploads only the missing ones. Resume cost is at most one part.
- **tus protocol (open standard):** the client sends `HEAD` to the upload URL, the server answers with an `Upload-Offset` header ("I have the first 312,000,000 bytes"), and the client continues with `PATCH` from that offset. **Supabase Storage implements tus** for resumable uploads (its docs recommend it for files above about 6 MB, from my memory of the docs, verify before citing). Uppy and tus-js-client are the usual browser libraries.

### 4.5 Content hashing and block-level dedup (Dropbox)
Dropbox splits each file into **4 MB blocks**, hashes each with SHA-256, and sends the list of hashes first. The server replies with the hashes it has never seen. The client uploads only those. Edit one slide in a 300 MB deck and you upload one or two blocks, not 300 MB. Magic Pocket, Dropbox's storage system, is an **immutable** block store, and blocks are grouped into roughly 1 GB logical containers for efficient disk replacement and erasure coding (Day 32). This is the same content-addressed idea as Day 23, now applied at upload time.

### 4.6 Verify, then commit (the two-phase finish)
Bytes landing in storage is not the same as "the upload is done". Finish with a `complete` call that checks size and hash, then writes the metadata row and flips state from `uploading` to `ready`. Only `ready` rows are visible to other users. Analogy: a package is not "delivered" until someone signs for it.

### 4.7 Garbage collection of abandoned uploads
Every aborted multipart upload leaves orphan parts that are **billed storage**. Set a lifecycle rule to abort incomplete multipart uploads after, say, 7 days (S3 has this as a lifecycle action, R2 supports similar cleanup). Also sweep metadata rows stuck in `uploading` for over 24 hours.

---

## 5. The trade-offs

**CAP per data type.**
- **File bytes:** durability first. Object stores replicate or erasure-code before acknowledging (Day 32). Availability of reads after write is strong on S3 and R2 today.
- **Metadata row vs object (two systems, no shared transaction):** you cannot make storage and Postgres commit atomically. Pick an order. Write the row as `uploading`, upload, then flip to `ready`. If the flip fails, a sweeper repairs it. This is a small saga (Day 20).
- **Quota counters:** consistency matters (do not allow 10x quota), but a few percent overshoot for one minute is usually acceptable. Prefer a guarded atomic increment at session start.

**Cost vs latency.**
- Direct-to-storage removes your bandwidth bill and CPU, but you lose the chance to inspect bytes inline. Scanning moves to async, so there is a window where a file exists but is not yet safe. Solve with the `ready` state gate.
- More parts means faster, more resilient uploads but more billable requests.
- R2 has no egress fees, which makes serving the file back cheap (Day 4). Ingress is free on most providers.

**Choice made by most mature systems:** presigned direct upload, multipart above roughly 100 MB (multipart is allowed down to 5 MiB), async post-processing, lifecycle cleanup.

---

## 6. The systems-thinking lens

**The loop: a slow or lossy upload gets retried from scratch, the retries add load, the load slows everything, more uploads fail and retry. A retry death spiral, with bytes.**

```
Network blip or slow storage node
   -> whole-file uploads fail at 90%
   -> clients restart from byte 0 (retry amplification: each retry costs 100% again)
   -> aggregate bandwidth demand doubles
   -> congestion, more failures
   -> self-sustaining even after the blip ends
```

Worked numbers: 1,000 clients each uploading 1 GB, 5% fail near the end and restart. That is 50 GB of repeat traffic. If failures rise to 20% because of the congestion, repeat traffic is 200 GB, then it compounds.

The senior fix breaks the loop at the unit of retry, it does not buy bandwidth:
- **Shrink the retry unit.** Per-part retry means a failure wastes 8 MiB, not 1 GB. Waste per failure drops by about 100x.
- **Resume from offset** (tus or `ListParts`) so progress is never discarded.
- **Exponential backoff with jitter** on part retries, and a cap on parallel parts per client.
- **Idempotent completion:** `complete` called twice must not create two objects (Day 12). Key the session by upload ID.
- **Admission control** on `POST /uploads` (Day 13): when storage is struggling, refuse new sessions quickly with `429` and `Retry-After`, but let in-flight sessions finish.
- **Do not put your API on the byte path**, so the spiral cannot take the login and read APIs down with it.

---

## 7. Map to Rare.lab's stack

Rare.lab: Supabase Postgres with RLS, Cloudflare R2 (content addressed immutable scene JSON plus manifest), an embeddable runtime with one shared WebGL context.

| Pattern | Rare.lab touchpoint | Status | Action |
|---|---|---|---|
| Control plane vs data plane | API mints, R2 stores | Likely already, verify | No scene asset should stream through a Worker or API function body. Presign from the API, upload straight to R2. |
| Presigned URLs | R2 S3-compatible API | Available | Scope each URL to one key and a short expiry (minutes). Because keys are content hashes, the key can be `sha256/<hash>` and the signer can bind to it. |
| Content addressing / dedup | Immutable scene JSON | Already doing this | Free dedup: before issuing an upload URL, `HEAD` the hash key. If it exists, skip the upload entirely (Dropbox's "send hashes first" trick, one level up). |
| Multipart + resumable | Large textures, video/LUT bundles, exported scene packs | Not needed for small JSON, will be for assets | Use multipart above about 100 MB, 8 to 16 MiB parts (equal sized for R2). Supabase Storage's tus endpoint is the no-code option. |
| Verify then commit | Manifest | Natural fit | The manifest write is the commit. Write the manifest only after the object hash is verified, never before. Readers then never see a manifest pointing to a missing blob. |
| Orphan cleanup | Aborted uploads | Check | Add a lifecycle rule to abort incomplete multipart uploads after 7 days, and a sweeper for `uploading` rows. |
| RLS | Upload permission | Check | Gate session creation with an RLS-checked insert into an `uploads` table, so signing is authorized by the same policy as everything else. |

**Where the next ceiling is (inference, not measured):** not storage, R2 absorbs this easily. The ceiling is the **metadata path**: a burst of `complete` calls each writing a Postgres row and a manifest update. A launch day with thousands of simultaneous exports hits the connection and RLS limits from Day 78 before it hits R2. Order of fixes: (1) presign plus direct upload, (2) skip uploads that already exist by hash, (3) batch manifest updates through a queue (Day 9), (4) lifecycle cleanup, (5) pooled DB connections (Day 78).

**One-line lesson for Rare.lab:** make the hash the upload key, check "do we already have it?" before signing anything, and keep every byte off your API, because the cheapest upload is the one you skip and the safest server is the one that never touched the file.

---

## 8. What is inside the video you shared (recap, already covered)

The "Design LeetCode" mock interview (Google engineer Vanika Agarwal, with Hello Interview and Shreyansh Jain's channel recommended at the end) is fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md) and recapped in Day 78, so I did not teach it a third time. Plain-language spine for your notes, in the order she worked:
1. **Functional requirements:** list problems (paginated), view a problem and code in any language, submit and get instant pass/fail, live contest leaderboard. **Out of scope:** auth, payments, analytics.
2. **Non-functional:** availability over consistency (a user seeing 990 vs 1,000 problems is fine), low latency (about 2 to 5 seconds for results), isolation for untrusted code, 100k concurrent contest users, no single point of failure.
3. **Core entities:** Problem, User, Submission, Leaderboard.
4. **REST API:** `GET /problems?page&limit`, `GET /problems/:id?language`, `POST /problems/:id/submission`, `GET /leaderboard/:competitionId?page&limit`.
5. **Fixes in layers:** do not run user code on the API server (security, CPU hogging, no fault tolerance), VMs isolate better but cost more, so containers per language, with the interviewer noting serverless as an alternative. Add a queue between API and runners with exponential retry, cache the leaderboard (polling the DB every 5 seconds does not scale), replicas with failover for the submissions DB, serialize test inputs as JSON and deserialize per language.
6. **Link to today:** a code submission is a tiny upload. If LeetCode allowed attaching a 50 MB input file, the pattern above (presigned URL, verify, then enqueue the judge job) would be the right shape.

---

## 9. References and what is actually in them

**Honest note on access:** developers.cloudflare.com, docs.aws.amazon.com and tus.io were blocked by the network proxy in this session. Everything below was identified through web search excerpts, not read in full. Verify numbers before quoting them.

- [Cloudflare R2 limits](https://developers.cloudflare.com/r2/platform/limits) (appeared in search results). Source for 5 MiB minimum part, 5 GiB maximum part, 10,000 parts, 35 GiB single-part and 4.995 TiB multipart ceilings.
- [R2 docs PR on multipart part size in the rclone example](https://github.com/cloudflare/cloudflare-docs/pull/7396) and [Apache Arrow issue 41506](https://github.com/apache/arrow/issues/41506). Real-world proof of the R2 equal-sized-parts quirk and how clients tripped on it.
- [Multipart uploads to Cloudflare R2 + Workers](https://notjoemartinez.com/blog/cloudflare_r2_multipart_upload_s3sdk/) (Joe Martinez) and [Multipart Uploads on alos.no](https://alos.no/cfnet/articles/r2/multipart.html). Practitioner walkthroughs of start, upload part, complete with the S3 SDK.
- [Screendrop issue 46](https://github.com/fayazara/Screendrop/issues/46). A real project hitting the Cloudflare 413 limit on large bodies and proposing multipart through a Worker. Shows the naive design failing in the wild.
- [Inside the Magic Pocket](https://dropbox.tech/infrastructure/inside-the-magic-pocket) (Dropbox engineering) and the [Dropbox content hash reference](https://www.dropbox.com/developers/reference/content-hash). 4 MB blocks, SHA-256 per block, immutable block store, hash-of-hashes for a whole file. Summary-level sources in search results: [Coffee Codex on Magic Pocket](https://meshan.dev/blog/coffee-codex-magic-pocket/).
- [tus resumable upload protocol 1.0.x](https://tus.io/protocols/resumable-upload) and [Understanding tus (golangbot)](https://golangbot.com/understanding-tus/). HEAD returns `Upload-Offset`, PATCH with `application/offset+octet-stream` continues from there.
- [Supabase resumable uploads docs](https://supabase.com/docs/guides/storage/uploads/resumable-uploads) and [Supabase Storage upload processing (DeepWiki)](https://deepwiki.com/supabase/storage/3.3-upload-processing). Supabase Storage speaks tus, usable with Uppy or tus-js-client.
- [Build resumable video uploads with tus, Node, and tus-js-client](https://dev.to/masonwritescode/build-resumable-video-uploads-with-tus-node-and-tus-js-client-2c62) and [Resumable Large File Uploads With Tus (Buildo)](https://www.buildo.com/blog-posts/resumable-large-file-uploads-with-tus). Hands-on, lower authority, good for seeing real code.

**Inference, labeled:** the 5 TB and 14 minute arithmetic, the 27 minute 2 GB example, part size rules of thumb (8 to 16 MiB, 4 to 6 in flight), the retry-amplification numbers, and everything in Section 7 about Rare.lab. The Supabase 6 MB recommendation is from memory.

**Related ledger lessons:** Day 4 (CDN, zero egress), Day 5 (video storage), Day 9 (queue), Day 12 (idempotency), Day 13 (load shedding), Day 20 (sagas), Day 23 (content addressed storage), Day 32 (erasure coding), Day 49 (WebRTC SFU), Day 74 (LeetCode), Day 78 (connection pooling).
