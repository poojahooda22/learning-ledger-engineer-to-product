# Day 90: A user taps "Delete my account". How do you really delete their data from 40 systems, replicas, caches, logs, backups and an immutable store, and prove it? Data deletion and the right to erasure at scale

**Date:** 2026-10-16
**Difficulty:** Advanced (deletion is the one write that every storage engine is built to avoid, and every derived copy of the data fights it)
**Topic:** Earlier lessons built systems that keep data safe: WAL and CDC (Day 17), LSM trees (Day 21), Merkle DAGs and content addressing (Day 23), Postgres MVCC (Day 88). Today is the mirror image: all of those designs make data hard to delete. One "delete" has to chase every copy.
**Stack relevance:** Rare.lab stores scene JSON immutably and content-addressed in Cloudflare R2, with Postgres rows in Supabase. Immutable plus content-addressed is the hardest shape to delete from.

---

## 0. The framework (same recipe as Days 74, 86 to 89)

The LeetCode mock interview in your prompt follows one recipe; the full walk-through is [Day 74](074-leetcode-online-judge-end-to-end.md). Short version: functional requirements plus out of scope, non-functional requirements, core entities, API routes, simplest design per requirement, then break it and optimize while saying each trade-off out loud. The interviewee's key move was "running user code on the API server is bad, so isolate it and put a queue in front". Today's equivalent: "deleting a row in the primary database is bad enough as a design, so isolate the delete as its own tracked job and fan it out".

Today's recipe:
- **Functional:** user requests deletion; every copy of their personal data is removed within a deadline; user can see status; data that must be kept (invoices, fraud records) is kept and minimized, not silently dropped.
- **Non-functional:** complete (no forgotten copy), provable (an audit trail), bounded in time (GDPR says "without undue delay", and in practice within one month, extendable by two), safe (never delete the wrong user's data), cheap (no full-database rewrite per request).
- **Out of scope:** legal advice, consent UI.
- **Core entities:** Subject (the person), Data copy (a location holding their data), Deletion request, Deletion task (one per copy), Deletion ledger entry.
- **API:** `POST /deletion-requests` (idempotent), `GET /deletion-requests/{id}`, an internal webhook or event `subject.deleted`.
- **Naive design:** `DELETE FROM users WHERE id = 42;` and call it done.

---

## 1. The company and the breaking number

**The company: Meta (Facebook).** Meta's Data @Scale talk "Deleting Data @ Scale" (Boston, 2018) presented a Deletion Framework whose stated job is to manage deletion across distributed systems, detect gaps, and confirm completeness. The talk exists because a single account's data lives in many separate systems, and nobody can hand-check each one. (Confirmed from the conference page summary; I could not find a public engineering post with the framework's internals, so everything below the abstract is my synthesis.)

**The breaking number (my worked example, not a company figure).** Take a product like Rare.lab at 1,000,000 users. Count the places one user's data lives:

- 1 primary Postgres row set (profile, projects, comments)
- 2 read replicas
- 1 search index (Elasticsearch/OpenSearch or similar)
- 1 cache tier (Redis, CDN edge copies)
- 1 analytics warehouse (events, plus derived tables)
- 1 ML feature store or training set
- 1 object store with their uploaded files and scene JSON
- 1 email provider (their address, event history)
- Application logs and traces kept for 30 days
- Backups kept for 35 days
- A Kafka-style event log kept for 7 days

That is about **11 kinds of place, and 25 to 40 actual systems** at a mid-sized company. Now the breaking part: if each of the 40 systems is 99.9 percent reliable for one delete call, the chance all 40 succeed on the first try is 0.999^40 = **96 percent**. At 1,000 deletion requests a day, about 40 per day have at least one copy left behind, silently. Over a year that is roughly **14,600 people whose data you told the regulator you deleted but did not**. (0.04 x 1,000 x 365 = 14,600. Arithmetic is mine; reliability numbers are illustrative.)

Analogy: you photocopied a letter, mailed copies to 40 offices, and now the author asks you to burn "the letter". Burning the original does nothing. You need the mailing list, a receipt from each office, and a plan for the copies that sit in a locked archive for 35 days.

---

## 2. Why the naive design dies

Naive design: one `DELETE` in the primary database.

- **Derived copies survive.** Search index, cache, warehouse and ML sets are built from the primary by pipelines (Day 17, CDC). Deleting the source does not delete what was derived from it. Worse, a rebuild from a stale snapshot can bring the person back ("resurrection").
- **Databases do not physically erase on delete.** In Postgres a `DELETE` just marks the row version dead; the bytes stay on disk until `VACUUM` reclaims the space (Day 88). In an LSM tree (Day 21) a delete writes a tombstone, and the old value remains in older files until compaction merges them. Kafka is the same: the KIP-354 proposal on the Apache wiki notes a compacted log segment can remain uncompacted for an unbounded time, because eligibility depends on dirty ratio and compaction lag settings. So "deleted" can mean "invisible but still on disk for days".
- **Backups are write-once.** A 35-day backup contains the user. You cannot edit a snapshot without breaking its checksum, and rewriting 50 TB of backups per request is impossible at 1,000 requests a day.
- **Immutable and content-addressed stores refuse to forget.** If two users upload the identical file, content addressing stores it once under its hash (Day 23). Delete user A's reference and the blob must stay because user B still points to it. Delete the blob and B loses data.
- **Partial failure.** Step 7 of 12 fails, the job crashes, nothing records which steps completed. Nobody knows the state, so nobody retries.
- **Wrong target.** Deleting by email when the email was recycled, or by a user id that was merged, wipes someone else.

---

## 3. The architecture, drawn top to bottom

```
User taps "Delete account"
        |
        v
Edge API (stateless)
   - re-authenticates the user (recent login or email confirm)
   - idempotency key, writes DeletionRequest row, returns 202 + request id
        |
        v
Grace period (e.g. 14 to 30 days) = SOFT delete
   - account hidden, sessions revoked, billing stopped
   - user can cancel (this is the "oops" window; also lets legal holds be checked)
        |
        v   (timer fires: Day 87 timers / Day 57 durable scheduling)
Deletion orchestrator (durable workflow, Day 37)
   - reads the DATA MAP: a registry of every system that holds subject data
   - fans out one idempotent task per system, each with retries and a deadline
        |
        +--> Primary DB task        : hard delete rows, anonymize rows that must stay (invoices)
        +--> Search index task      : delete-by-query on subject id
        +--> Cache / CDN task       : purge keys and URLs
        +--> Warehouse task         : delete or null-out rows, rebuild affected partitions
        +--> Object store task      : decrement references, delete blobs at refcount 0
        +--> Vendor tasks           : email provider, analytics SaaS, support desk APIs
        +--> Key vault task         : DESTROY the subject's encryption key (crypto-shredding)
        |
        v
Deletion ledger (append-only, tiny, kept on purpose)
   - stores only a salted hash of the subject id + timestamp + per-system result
   - replayed after any backup restore so deleted people stay deleted
        |
        v
Verifier (the auditor)
   - after N days, scans every registered system for the subject id, expects zero hits
   - failures page a human; successes close the request and email the user
```

| Layer | Single job | Analogy |
|---|---|---|
| **Edge API + idempotency key** | Accept the request once and prove who asked | The signed form at the records office |
| **Grace period** | Make deletion reversible for a short window | The "recently deleted" bin |
| **Data map (registry)** | Know every place data lives | The mailing list of the 40 offices |
| **Orchestrator (durable workflow)** | Run each step to completion, even across crashes | A clerk with a checklist who never loses it |
| **Per-system delete tasks** | Remove or anonymize one copy, idempotently | One courier per office |
| **Key vault (crypto-shredding)** | Make unreachable copies unreadable | Burning the only key to the locked archive |
| **Deletion ledger** | Remember who was deleted, without remembering who they were | A list of shredded names stored as fingerprints |
| **Verifier** | Prove nothing is left | The inspector who walks every office afterwards |

Vocabulary:
- **Right to erasure (GDPR Article 17):** a person can ask a company to delete their personal data without undue delay, with exceptions such as legal obligations.
- **Soft delete:** mark hidden, keep data. **Hard delete:** physically remove.
- **Tombstone:** a marker record meaning "this key is deleted", used by LSM trees and Kafka.
- **Compaction:** background merge that finally drops data covered by a tombstone.
- **Crypto-shredding:** encrypt each person's data with that person's own key; to erase every copy including backups, destroy the key.
- **Data map / data inventory:** the registry of where personal data lives. Without it you cannot be complete.
- **Anonymization:** removing identity so the row can stay (for example keep the invoice total, replace name and email with a hash or null).

---

## 4. The transferable mechanisms

### 4.1 Soft delete first, hard delete on a timer

Two phases: instantly hide (sessions revoked, profile 404), then physically erase after a grace window. The first phase makes the product feel done in 100 ms and protects against "I clicked by mistake". The second phase is a scheduled job (Day 87: timers at scale), not a request-path cost. Trade-off: the data is still held for the window, which your privacy policy must state.

### 4.2 Data map as code: registry-driven fan-out

Every new table, topic, bucket or vendor integration must register how it deletes (a function plus an owner). A CI check fails the build if a table has a column tagged `personal` and no registered deleter. This is what Meta's framework abstract describes at the goal level: detect gaps automatically and ensure completeness, because humans forget systems. The mechanism: **make forgetting a build error**, not an audit finding. (The CI check is my suggestion; Meta's internals are not public.)

### 4.3 Idempotent per-system tasks under a durable workflow

Each delete task must be safe to run twice: `DELETE ... WHERE subject_id = ?` returns zero rows the second time and that is success. Orchestrate with a durable execution engine (Day 37) so a crash at step 7 resumes at step 7. Retries use exponential backoff with a cap (Day 13). Same lesson as the LeetCode mock: queue the work, retry with growing delay, but here the retried action is naturally safe to repeat.

### 4.4 Tombstones plus compaction, and knowing when "gone" really is gone

Know your engine's meaning of delete:

| Store | What delete does | When bytes really go |
|---|---|---|
| Postgres | Marks row version dead | After `VACUUM` (autovacuum, Day 88); `VACUUM FULL` or a rewrite is the hard guarantee |
| LSM tree (RocksDB, Cassandra) | Writes tombstone | After compaction reaches the old files; Cassandra also keeps tombstones for `gc_grace_seconds` so replicas can sync first (Day 22) |
| Kafka compacted topic | Null-value record for the key | After compaction, which is not time-guaranteed; consumers offline longer than `delete.retention.ms` can miss the tombstone (a source suggests 7 days) |
| Search index | Marks doc deleted | After segment merge |
| Object store | Deletes object, or a version | Immediately for the key; versioned buckets keep old versions until expired |

The rule: **write down, per store, how long "deleted" takes to become "unrecoverable", and make that number part of your stated retention.** Do not claim instant erasure if compaction takes days.

### 4.5 Crypto-shredding: delete a key, not a thousand copies

Encrypt each subject's sensitive fields (or files) with a per-subject data key, itself wrapped by a master key (envelope encryption). To erase: destroy the per-subject key. Every copy anywhere, in backups, replicas, Kafka segments, warehouse files, turns into noise. This solves the two problems rewriting cannot: immutable backups and append-only logs. Costs: key management becomes critical infrastructure (lose the key store and you lose everyone's data), reads need a key lookup (cache the data keys in memory for a short time), and you must keep identifiers and metadata you want to query outside the encrypted fields. Sources on Kafka and general cloud systems recommend it for immutable storage; one is sponsored content and promotional, and I found no regulator guidance in the search results on whether key destruction legally counts as erasure under Article 17, so treat that as an open legal question and ask counsel.

### 4.6 The deletion ledger: replay after restore

If you restore a 20-day-old backup after an outage, everyone deleted in the last 20 days comes back. Fix: keep a tiny append-only ledger of deletions (store a salted hash of the subject id, not the id), and make "restore" a two-step procedure: restore the data, then replay the ledger to delete again before serving traffic. The ledger itself holds no personal data, so it is safe to keep forever. This is the same shape as WAL replay (Day 17): log the intent, replay to reach the right state.

### 4.7 Reference counting for shared immutable objects

For content-addressed blobs (Day 23): keep a table `blob_hash -> set of owner ids` (or a count). Deleting a user removes their references. When the set is empty, a garbage collector deletes the blob after a delay (to avoid racing a concurrent upload of the same hash). Edge: if the user's own file is identical to a public template, it was never "theirs alone", so dropping the reference is enough.

### 4.8 Verify, do not trust

A verifier job searches every registered store for the subject id after the deadline. Zero hits closes the request. Any hit opens an incident. Audit trails win regulator conversations; "we ran the script" does not.

---

## 5. The trade-offs

**Consistency vs availability, per data type:**

| Data | Choice | Why |
|---|---|---|
| Deletion request record | Strongly consistent, durable before returning 202 | Losing a request is a legal failure |
| "Is this account visible?" | Strong on the primary, short TTL on caches (seconds) | A deleted profile showing for an hour is a trust and legal issue |
| Replicas, caches, search index | Eventually consistent deletion with a stated upper bound (for example 24 hours) | Chasing every replica synchronously would make deletes slow and fragile |
| Backups | Not edited; covered by crypto-shredding or by expiry (35 days) plus ledger replay | Rewriting immutable backups per request is infeasible |
| Deletion ledger | Append-only, replicated, never deleted | It is the memory that keeps deleted people deleted |

**Cost vs latency:**
- Crypto-shredding costs a key service and a lookup per read, and saves rewriting every copy. It is cheap per deletion and expensive to build.
- Rewrite-based deletion (rebuild warehouse partitions without the subject) costs compute proportional to partition size, not to the number of deleted rows. Batch deletions weekly per partition: 1,000 requests in one rewrite costs about the same as 1.
- Short retention (logs 30 days, Kafka 7 days, backups 35 days) is the cheapest erasure there is: the data deletes itself. Shorter retention costs debugging power and recovery depth.
- Fast, complete deletion fights audit logs and fraud analysis, which want to remember. The compromise is minimization: keep the invoice amount, drop the name.

**The choice this system makes:** reversible and fast on the surface (soft delete), eventually complete underneath (bounded hours to days), provable via a ledger and a verifier, with backups handled by expiry or key destruction rather than edits.

---

## 6. The systems-thinking lens

**The feedback loop: copy proliferation and resurrection.**

1. A new feature needs user data, so an engineer adds a pipeline that copies it (a dashboard table, a search index, a training set). Nobody updates the delete job.
2. Each copy is another place a delete must reach. The registry drifts from reality. Completeness silently falls from 100 percent.
3. A delete runs, misses the new copy. Later a rebuild job (reindex, backfill, restore) reads from that copy and writes the person back into the primary or into other stores.
4. Now the "deleted" person exists in more places than before, and each resurrection spawns fresh derived copies. The loop reinforces itself.
5. Audit finds it months later, the cleanup is a multi-team project, and trust (and fines) are lost.

The senior fix **breaks the loop** rather than adding more delete workers:
- **Make the data map load-bearing.** Registration is required to create a personal-data column, topic or bucket (CI gate). New copy without a deleter cannot ship.
- **Never write back from derived to primary.** Rebuilds read from the system of record only, so a stale copy cannot resurrect anyone.
- **Gate rebuilds and restores with the ledger.** Any restore or backfill replays the deletion ledger before data is used.
- **Reduce the number of copies, not the speed of deleting them.** Retention limits, pseudonymous ids in analytics (join key held in one table, delete that row and the analytics become anonymous), and crypto-shredding for the long tail.
- **Verify continuously.** A nightly sample check that deleted ids return zero hits turns drift into an alert in days, not an audit finding in months.

---

## 7. Mapping to Rare.lab

Rare.lab: Supabase Postgres with RLS, Cloudflare R2 (immutable content-addressed scene JSON plus manifest), embeddable runtime with one shared WebGL context.

**Already doing, whether named or not:**
- **RLS and per-owner rows.** Every row carries an owner id, which makes "find everything this user owns in Postgres" a well-defined query. That is the hardest first step of a data map, already done for the database.
- **Immutable manifests over immutable blobs.** Because scenes are content-addressed and a manifest points to them, deleting a user's project can be done by deleting the manifest entry and decrementing references, exactly mechanism 4.7. You never edit a blob.
- **Postgres cleanup behavior.** Supabase runs Postgres, so Day 88 applies: deleted rows linger until vacuum. Fine for ordinary requests; not an instant physical erase.

**Where this bites next (my inference from your stack description, not from reading your code):**

| Risk | Why it matters | Action |
|---|---|---|
| **Shared content-addressed blobs** | Same scene JSON uploaded by two users is one object; deleting it for A breaks B, and never deleting it keeps A's data forever | Add an `object_refs(hash, owner_id)` table; a GC job deletes a blob when its refs hit zero, after a delay to avoid racing a new upload |
| **Public share links and CDN caches** | A deleted project can still be served from the Cloudflare edge until its TTL ends | Purge by URL (or cache tag) as the first step of the delete task; keep share-link TTLs short; the immutable URL cannot be "updated", only purged |
| **Embeddable runtime in customer sites** | Customers may have embedded a scene that the author deleted; the embed now 404s | Decide the policy: deleting an account anonymizes and keeps scenes that others embed (documented), or gives a tombstone scene; write it into your terms |
| **Backups** | Supabase backups hold deleted users for the backup window | State the window in your policy; for sensitive fields consider per-user envelope encryption so you can crypto-shred |
| **Analytics and logs** | Product events keyed by user id | Use a pseudonymous id and a single mapping table; deleting that row anonymizes the history |
| **No data map yet** | The first third-party tool (email, analytics, error tracker) adds a copy nobody tracks | Start a `data_map.md` plus a CI check on a `personal` tag in migrations; one row per store with its delete method |

**Next ceiling:** not storage or throughput, but **completeness**. At 1,000 users it is a script; at 100,000 users and ten vendors it is a workflow with a ledger and a verifier.

**One-line lesson for Rare.lab:** write the data map before you need it, give every shared immutable object a reference count, purge the CDN as step one of deletion, and delete derived copies from the system of record, never the other way round.

---

## 8. Sources and what is actually in them

**Honest note on access:** this session I used web search summaries and did not open the full pages. Meta's Deletion Framework internals are not public beyond the talk abstract. The registry, ledger, verifier and reference-count designs are my synthesis, labeled as inference. Section 1 numbers are my own arithmetic with illustrative reliability. Nothing here is legal advice; Article 17 interpretation (especially for backups and crypto-shredding) needs counsel.

- [Meta Engineering: Data @Scale Boston, "Deleting Data @ Scale"](https://engineering.fb.com/2018/11/13/core-infra/data-scale-boston/). Plain language: the conference recap listing Ben Strahs's talk on Facebook's Deletion Framework: the problem is deleting across many distributed systems, with mechanisms to detect gaps, confirm completeness and delete the right data. Only an abstract is public.
- [Meta Engineering: Automating data removal (SCARF), Oct 2023](https://engineering.fb.com/2023/10/31/data-infrastructure/automating-data-removal/). Plain language: how Meta safely finds and removes unused data types in a tangled codebase, using static analysis (Glean) plus runtime read counts, using the shut-down Moments app as the example. The lesson: you cannot delete safely if you do not know what depends on the data, which is why the data map matters.
- [USENIX PEPR '24: Lingwei Meng and Yan Li, Meta](https://usenix.org/conference/pepr24/presentation/meng). Plain language: how the ads system honors users deleting their historical ad-interaction events, which required changes to ingestion, storage, feature representation and even the model architecture. Shows deletion reaching into ML systems, the hardest derived copy.
- [Conduktor: GDPR and Kafka, Right to Erasure](https://conduktor.io/blog/gdpr-kafka-right-to-erasure). Plain language: vendor article on erasing from an append-only log: short retention, tombstones on compacted topics (with `delete.retention.ms` around 7 days so consumers see them), and crypto-shredding. Notes that physical deletion in Kafka means rewriting segments, which Kafka does not directly support.
- [Conduktor: Crypto-shredding for Kafka](https://www.conduktor.io/glossary/crypto-shredding-for-kafka.md). Plain language: per-subject keys, destroy the key to make all copies unreadable.
- [Opencredo: Kafka, navigating GDPR compliance](https://www.opencredo.com/blogs/kafka-navigating-gdpr-compliance). Plain language: practical rules: key messages by a per-user id, enable compaction on every topic that holds personal data, send a null-value tombstone on deletion, and use short retention such as 28 days.
- [Apache Kafka wiki: KIP-354 (time-based log compaction policy)](https://cwiki-test.apache.org/confluence/x/7o2zBQ). Plain language: a proposal explaining why compaction cannot guarantee deletion timing (segments can stay uncompacted for an unbounded time; merged segments take the newest timestamp so old records can outlive retention) and motivating a deadline-based policy for deleting PII within a set number of days. I opened only the search summary; exact page may differ among the several wiki URLs returned.
- [DesignGurus: soft vs hard TTL for GDPR deletion](https://www.designgurus.io/answers/detail/how-would-you-implement-soft-vs-hard-ttl-for-gdpr-deletion). Plain language: tutorial-level answer; pairs crypto-shredding with a deletion ledger so a restored backup can replay erasures.
- [Verdict: crypto-shredding for GDPR in cloud systems](https://www.verdict.co.uk/crypto-shredding-gdpr-cloud-systems/). Plain language: sponsored content calling crypto-shredding the best option; promotional, does not dig into key-management risk. Use for the idea, not the endorsement.
- [Gravitee: data erasure](https://gravitee.io/corpus/gen-1808/data-erasure/index.html). Plain language: warns tombstones are not full erasure: earlier values stay recoverable until compaction and old segments age out, and consumers must not rebuild deleted state from historical replays.
- Video context: the LeetCode mock interview in your prompt (Day 74 has the full summary). Reuse for today: "isolate the risky work, put a queue in front, retry with growing delay, and say the trade-off out loud". Deletion is that pattern with a twist: the retried job must be naturally idempotent and must finish for every store, not just most.

**Confirmed (from search summaries):** Meta's Deletion Framework exists and targets completeness across distributed systems; Meta's SCARF uses static analysis plus runtime reads; Kafka compaction timing is not guaranteed and tombstones need a long enough `delete.retention.ms`; crypto-shredding is widely recommended for immutable stores.
**Inference (mine):** the data-map CI gate, the ledger replay procedure, the verifier, the reference-count table, all numbers in section 1, and every Rare.lab recommendation.

**Related lessons:** Day 12 (idempotency), Day 17 (WAL and CDC), Day 21 (LSM trees), Day 23 (content-addressed storage), Day 37 (durable execution), Day 58 (online migrations), Day 87 (timers), Day 88 (Postgres vacuum and bloat).
