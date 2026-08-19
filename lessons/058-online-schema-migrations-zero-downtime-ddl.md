# Day 58 — How does GitHub change the schema of a MySQL table with a billion live rows, when the same `ALTER TABLE` the naive way locks it for over an hour?

**Date:** 2026-08-19
**Difficulty:** Expert
**Topic:** Online, zero-downtime schema migrations on a table too large and too live to ever take offline. Why a plain `ALTER TABLE` on a table with a billion rows is not a slow query, it is a full-table rewrite behind an exclusive lock that blocks every read and write against that table for as long as the rewrite takes, and why "just run it at 3am" stops working the moment the table, and the traffic against it, is large enough that there is no longer a quiet hour. How the two real solutions to this, Percona's trigger-based `pt-online-schema-change` and GitHub's own triggerless `gh-ost`, both solve it the same fundamental way: build a second, empty copy of the table next to the live one, keep it converging on the live table's current state without ever blocking the live table's own writes, and only take a lock for the fraction of a second it takes to swap the two tables' names. And why even that fraction of a second is not risk-free at extreme scale, using GitHub's own November 2021 incident, where the final rename step of a schema migration put a significant portion of their MySQL read replicas into a deadlock, as the proof that solving 99.9% of a hard problem still leaves a sharp edge in the remaining 0.1%.
**Stack relevance:** Rare.lab's Cloudflare R2 scene storage already gets this pattern for free: scene JSON is content-addressed and immutable, so nothing is ever changed in place, a new version is a new object at a new address, and the manifest pointer that says "this is the current version" is swapped atomically. The exposure is on the other side of the stack, in Supabase Postgres, specifically the `projects`, `nodes`, and account tables that every live editor session and every embedded runtime instance reads through Row Level Security policies on nearly every query. Those tables are small today. The day one of them needs a schema change that Postgres can't do as a fast, metadata-only operation, a plain `ALTER TABLE` run the naive way doesn't slow one query down, it locks the table every RLS-gated session depends on, for every tenant, all at once.

---

## 1. The company and the breaking number

**GitHub, 2016.** By this point GitHub's core MySQL tables, the ones backing pull requests, issues, and commits across every repository on the platform, had grown into the billions of rows. A completely ordinary, completely necessary operation, adding a column to the `pull_requests` table, a table with billions of rows in it, run the way `ALTER TABLE` is taught in every SQL tutorial, would lock that table for over an hour. Not "run slowly." Locked. Every read, every write, every one of the many services that touch a pull request on any repository, on hold, for over sixty minutes, waiting on one operation that a tutorial presents as a single, forgettable line of DDL.

This is the breaking number this whole lesson turns on: at small scale, `ALTER TABLE some_table ADD COLUMN x` is a statement you run and move on from. At GitHub's scale, in 2016, it was an outage-shaped operation, and the existing open-source fix for it, Percona's `pt-online-schema-change`, came with its own scaling problem: it used triggers, and triggers under GitHub's write volume were serializing enough extra work onto the replication stream to cause replication lag of their own. GitHub needed a schema-change tool that could survive their scale, and the tool that didn't exist yet, they built: `gh-ost`, GitHub's Online Schema Transmogrifier, authored by Shlomi Noach and open-sourced in 2016.

The concrete, later example of exactly why this problem never fully goes away, even once you've built the right tool: on **November 27, 2021**, GitHub had a platform-wide incident, degrading GitHub Actions, API requests, Codespaces, Git operations, Issues, Packages, Pages, Pull Requests, and Webhooks, that traced back to the final step of a routine schema migration on a large MySQL table. Not the naive, hour-long lock this lesson opens with, a genuinely new failure mode inside the very tool built to prevent that one. Writes stayed healthy the entire time and no data was corrupted, but a significant portion of GitHub's MySQL read replicas entered a deadlock during that migration's rename step, and because so many of GitHub's core services all read through that same database layer, one migration's final, supposedly-safe moment became a company-wide degradation. That is the number this lesson keeps coming back to: not "over an hour," but "under a second, and still not zero risk at large enough scale."

---

## 2. Why the naive (demo) design dies

**Version one: run the `ALTER TABLE` directly against the live table.** This is what every SQL course teaches, because at demo scale it's completely correct and instant. The failure isn't in the syntax, it's in what MySQL (and, differently, Postgres, covered in Section 5 and the Rare.lab mapping) actually does underneath most schema changes: for many operations, adding a column with a computed default, changing a column's type, adding certain kinds of constraints, the database doesn't edit the table's rows in place, it builds an entirely new copy of the table with the new schema and copies every existing row into it, then swaps the new copy in. That rewrite takes a lock for its full duration, an exclusive metadata lock in MySQL's case, that blocks every other statement, read or write, against that table until the rewrite finishes. On a thousand-row table that's milliseconds, invisible. On a table with a billion rows, it's the "over an hour" figure from Section 1, and for that entire hour, the table isn't slow, it's unavailable.

**Version two, the part that turns a slow operation into an outage: connection exhaustion behind the lock.** A blocking DDL statement doesn't just make the table wait, it makes every application thread that tries to touch that table wait, and each of those waiting threads is holding open a database connection while it waits. A service under real production load doesn't send one query and patiently wait, it sends hundreds or thousands of concurrent queries against a hot table, and every single one of them queues up behind the same metadata lock. Connection pools have a maximum size for a reason, and a queue of waiting connections that never drains hits that maximum fast. The result: even parts of the application that have nothing to do with the table being altered start failing, because the connection pool serving them is entirely consumed by threads stuck waiting on a lock for an unrelated table. The DDL statement would have succeeded eventually. The outage it caused along the way is the actual problem, and it's a problem the DDL statement itself has no way to know it's causing.

**Version three, the naive fix that only postpones the failure: "just run it in a maintenance window."** This genuinely works, until it doesn't. It works as long as there's a period of the day with low enough traffic that an hour-long lock is tolerable, and as long as the table is small enough that the lock only takes an hour rather than four. Neither of those stays true as a product grows. A platform serving a genuinely global user base has no quiet hour, every hour is peak somewhere. And table size only moves in one direction, the same migration that took twenty minutes at a hundred million rows takes two hours at a billion. The maintenance-window fix isn't a fix, it's a countdown to the day the table is too big and the traffic too constant for any window to be safe, and by the time that day arrives, so many other systems depend on this table staying up that even scheduling downtime for it, let alone taking it, has become organizationally difficult in its own right.

---

## 3. The architecture

```
[Source table, on the primary, serving all live production traffic]
   the original table, completely untouched by the migration for its
   entire duration, still taking every real read and write while
   everything below happens beside it, not to it
   job: never stop serving, not for one row of the migration
   analogy: a two-lane highway that stays fully open in both
   directions while a parallel road is built next to it, not instead
   of it

   |
   v

[Ghost table: an empty table with the new, target schema]
   created once, up front, alongside the source table, taking zero
   production traffic, existing purely as a destination
   job: give the migration somewhere to build the new shape of the
   data before any of it is real yet
   analogy: pouring the concrete deck of a new bridge right next to
   the one everyone's still driving across

   |
   v

[Chunked backfill: copy existing rows in small primary-key ranges]
   reads the source table's existing rows in ordered chunks (a
   default chunk size, tunable, small enough that copying one chunk
   is a short transaction, not a long one) and INSERTs each chunk
   into the ghost table
   job: move a billion existing rows without ever holding one lock,
   or one transaction, big enough to matter
   analogy: moving a warehouse's inventory by forklift, one pallet
   at a time, never tipping the whole building on its side to empty
   it in one motion

   |
   v

[Binlog reader, connected to the cluster as if it were a replica]
   while the backfill above is still running, gh-ost's triggerless
   design connects to an actual MySQL replica and pulls the binary
   replication log for the source table, capturing every INSERT,
   UPDATE, and DELETE that real production traffic is making to the
   live table in real time
   job: see every change happening to the moving target, without
   adding one synchronous instruction to the live table's own write
   path, this is the specific design choice that makes gh-ost
   different from trigger-based tools (Section 4)
   analogy: a court stenographer transcribing everything said in the
   live courtroom from off to the side, never once interrupting the
   trial to ask a question

   |
   v

[Async apply: replay captured changes onto the ghost table]
   every change the binlog reader captured gets replayed onto the
   ghost table, so a row updated on the source five seconds ago gets
   the same update applied to its ghost-table counterpart shortly
   after
   job: keep the ghost table converging on the source table's actual
   current state, continuously, for as long as the migration runs

   |
   v

[Throttle control: watch a replica's replication lag, pause on
 threshold]
   a background check continuously polls the replication lag on a
   designated replica; gh-ost's default ceiling is 1,500 milliseconds
   of lag before it pauses, and GitHub runs its own production
   migrations with tighter thresholds, in the 300 to 1,000
   millisecond range; crossing the threshold stops BOTH the backfill
   and the change-apply completely, no row copies, no event
   processing, until lag recovers
   job: make absolutely sure the migration's own background load is
   never the thing that degrades the replicas serving real reads
   analogy: a site foreman who halts every incoming delivery truck
   the moment the road out front starts backing up, instead of
   adding to a jam that's already forming

   |
   v

[Cut-over: sentry table + atomic RENAME]
   once the ghost table has fully caught up to the source, a "Lock
   Connection" acquires a lock via a temporary sentry table, and a
   second "Rename Connection" issues an atomic multi-table RENAME
   (source becomes source_old, ghost becomes source); the rename
   stays blocked by the sentry lock until gh-ost is certain
   everything is caught up and ready, then the lock releases and the
   rename executes, typically in under one second
   job: shrink the one moment that genuinely cannot avoid an
   exclusive lock down to the smallest possible window, instead of
   pretending that moment doesn't have to exist at all
   analogy: swapping a scoreboard's display panel in the two-second
   gap between innings, not stopping the game to do it, but also not
   pretending a swap that fast has zero chance of a bad bounce
```

---

## 4. The transferable mechanisms

- **Shadow copy plus atomic swap, instead of mutating the live thing in place.** Both `pt-online-schema-change` and `gh-ost` refuse to touch the source table's structure directly. They build a second, complete copy in the target shape, keep it converging on the source, and only ever "commit" to the new shape with one atomic operation at the very end. This is the same pattern as a blue-green deployment, or, closer to home for this ledger, Day 23's content-addressed storage: build the new artifact fully, somewhere that doesn't affect anyone yet, and only flip the pointer once it's completely ready. Never edit the thing everyone is currently depending on.

- **Change-data-capture via a durable log, not a synchronous hook in the write path.** This is the specific mechanical difference between the two tools, and it's the reason `gh-ost` exists at all: `pt-online-schema-change` uses triggers, `AFTER INSERT`, `AFTER UPDATE`, `AFTER DELETE`, installed directly on the source table, so every single production write pays the cost of that trigger firing, synchronously, before the write is even considered done. `gh-ost` instead reads the binary replication log, the same durable, ordered record of every change that MySQL's own replication already relies on, from a position outside the live write path entirely. The general lesson, echoing Day 17's write-ahead-log lesson: if you need to observe every change to a piece of data, read it from the log the system already produces, don't bolt a new synchronous cost onto every write to manufacture your own feed.

- **Bound the unit of bulk work by size, not by hoping it finishes fast.** The backfill copies rows in small, fixed-size, primary-key-bounded chunks, each its own short transaction, rather than one giant `INSERT ... SELECT` across the whole table. A giant single-transaction copy would itself hold locks and generate a huge undo/redo footprint for as long as it ran, recreating the exact problem this whole design exists to avoid. Chunking turns one dangerous, unbounded operation into thousands of small, safe, individually cheap ones.

- **Feedback-driven throttling, keyed to a real downstream health signal.** The migration doesn't run at a fixed rate and hope for the best, it watches replication lag, an actual measurement of whether its own background work is starting to hurt production, and pauses completely the instant that signal crosses a threshold. This is the same shape as Day 13's backpressure lesson: the thing generating load must be able to sense the health of the thing it's loading, and slow itself down (or stop entirely) based on that signal, rather than a fixed schedule that has no way to know it's already causing damage.

- **Rehearse the irreversible step before you take it.** `gh-ost` can run its entire flow, backfill, binlog capture, convergence, against a replica without ever executing the cut-over, letting an operator verify the migration behaves correctly against real production data volume and shape before the one step that actually matters gets attempted for real. The transferable version: whenever a step in a process is genuinely hard to undo, build a way to run everything up to that step, safely, as many times as needed, and only ever cross the irreversible line once you've already seen this exact migration succeed short of that line.

- **Minimize the unavoidable lock instead of denying it exists.** Neither tool eliminates the exclusive lock at cut-over, they can't, both MySQL and Postgres genuinely require some exclusive moment to swap catalog entries for a table. What they do is engineer every other part of the system so that this is the only moment requiring one, and make that moment as short as physically possible, under a second for a table with a billion rows, instead of the hour-plus a naive `ALTER TABLE` would have held.

---

## 5. The trade-offs

**Consistency vs. availability, and the window is deliberate, bounded, and entirely internal.** For the full duration of the migration, the ghost table is, by design, behind the source table, converging on it but never quite caught up until the very last moment before cut-over. That's a real, temporary inconsistency, but it's a safe one, because nothing reads from the ghost table until the rename makes it the source of truth. The trade being made is explicit: accept an internal staleness window that no user or query can ever observe, in exchange for the source table never once being unavailable. Compare that to the naive design's trade, zero staleness anywhere, because there's only ever one copy of the table, in exchange for that one copy being completely locked for the length of the rewrite. Neither trade is free, the online-migration design just moves the cost somewhere nobody can see it happen.

**Cost vs. speed, paid in duplicated I/O and wall-clock time.** Every row that exists gets written twice, once by the backfill, once by the async replay of whatever change touched it during the migration, and every new production write during the migration is captured and replayed a second time onto the ghost table on top of its normal cost against the source. That doubles the write I/O against the cluster for the migration's entire duration, and a migration on a table with a billion rows, throttled deliberately to protect replication lag, can take hours to days, where the naive, unsafe version would have finished, disastrously, in the time the lock alone took. The entire design is spending extra time and extra I/O specifically to buy zero downtime, and that exchange rate is exactly the trade-off, not a side effect of it.

**The trade the design explicitly does not make: pretending the residual lock can be zero.** GitHub's November 2021 incident is the clearest evidence that "minimize the lock to under a second" and "eliminate the lock entirely" are different engineering claims, and only the first one is actually true. At extreme enough concurrency, even a sub-second exclusive lock has to wait for every in-flight, lock-compatible operation ahead of it to clear, and MySQL's metadata lock queue is first-come-first-served, meaning ordinary reads that arrive after the exclusive rename request has already started waiting get stuck behind it too, not just around it (this queueing behavior is covered in Section 6). The tool's designers made the residual risk as small as they reasonably could. They did not, and structurally could not, make it exactly zero, and treating "very small" as "zero" is precisely the gap that turned one migration's cut-over into a platform-wide incident.

---

## 6. The systems-thinking lens

The feedback loop that turned GitHub's November 2021 cut-over into a platform-wide incident is a **lock convoy**, and it's worth being precise about why a sub-second lock produced an outage that lasted far longer than a second. MySQL's metadata lock queue doesn't let a newly arriving, lock-compatible query cut in line ahead of an already-waiting exclusive request, that rule exists specifically to stop an endless stream of small reads from starving out a DDL statement forever. But on a table hot enough, that fairness rule has a nasty second-order effect: the instant the cut-over's exclusive rename request starts waiting on some long-running query that got there a moment earlier, every ordinary read that arrives after it, reads that would otherwise have been served instantly and harmlessly, gets forced to queue up behind the rename too, not around it. The line doesn't grow at the rate the rename is slow, it grows at the rate production traffic normally arrives, which on GitHub's read replicas at that scale is a lot faster than any queue like that can drain. That's the shape of a **metastable failure**: the triggering event, one rename waiting slightly longer than expected, is small and brief, but it pushes the system into a state (a rapidly growing lock-wait queue on read replicas serving core production traffic) that the system cannot recover from on its own even after the original trigger clears, because by the time the rename finally completes, the queue behind it has grown far larger than the moment that caused it.

The senior fix here isn't "hold the lock more carefully" or "add more replica capacity so queues drain faster," both of those reduce how often this happens without changing what happens when it does. GitHub's actual response was to shrink the blast radius structurally: prioritizing functional partitioning, splitting a monolithic database cluster into shards organized around functional boundaries, so that any single migration's cut-over lock only ever has to contend with the traffic of one functional shard, never the platform's entire read-replica fleet at once, and running migrations in canary mode on a single shard first, so this exact class of failure would show up small and contained before it was ever attempted everywhere simultaneously. This is the same move Day 34's cell-based architecture lesson makes in a different context: the fix for a shared-resource convoy is not "manage the resource more precisely," it's "shrink how much traffic can ever end up queued behind the same lock, on the same resource, at the same time," so that even the worst case is a contained incident on one shard instead of a company-wide one.

---

## Map to Rare.lab's stack

Rare.lab already has one half of this pattern for free, and is exposed on the other half. On the R2 side, scene JSON is content-addressed and immutable: a save never edits an existing object, it writes a new object at a new content address and then updates the manifest to point there. That manifest update is the exact functional equivalent of `gh-ost`'s cut-over rename, a tiny, atomic, near-instant pointer flip that happens only after the new version is fully built and ready, never a rewrite of something already in flight. Rare.lab gets this specific class of problem solved structurally, for the same underlying reason `gh-ost` gets it, nothing is ever mutated in place.

The exposure sits in Supabase Postgres, specifically in whatever tables hold project, node-graph, and account metadata that the live editor and the embeddable runtime read through Row Level Security on close to every request. Postgres itself already has a version of this lesson built in that's worth using deliberately rather than by accident: since Postgres 11, `ALTER TABLE ... ADD COLUMN ... DEFAULT` is a fast, metadata-only change that doesn't rewrite the table, but a wide range of other common changes, a column type change, a `NOT NULL` constraint added the naive way, certain new constraints, still take the strongest lock Postgres has, `ACCESS EXCLUSIVE`, and block every read and write on that table for the whole operation, exactly the MySQL failure mode this lesson opened with, just under a different database engine. Because every one of those reads is also running through an RLS policy check, a blocking `ALTER TABLE` on a shared metadata table doesn't stall one tenant's queries, it stalls the row-level-security check that gates every tenant's session touching that table at once.

Three concrete, specific moves, worth adopting now, before any of these tables are large enough for the difference to be an incident instead of a line in this lesson: default to `CREATE INDEX CONCURRENTLY` instead of a plain `CREATE INDEX` for any index added to a live table, since it takes the much weaker `SHARE UPDATE EXCLUSIVE` lock that blocks other writers briefly rather than blocking everything; for any change that does need a genuine rewrite (a type change, a backfilled constraint), use the expand-and-contract pattern by hand, add the new column nullable, backfill it in small chunked batches exactly the way `gh-ost` chunks its row copy, add the constraint in a fast, separate transaction only once the backfill is verified complete, then drop the old column later, on its own schedule; and if a full-table rewrite genuinely can't be avoided, reach for `pg_repack`'s concurrent mode, which uses logical decoding to capture live changes the same way `gh-ost` reads the binlog, and only takes its own brief `ACCESS EXCLUSIVE` lock to swap the underlying table file at the very end, the same "minimize the one unavoidable lock to milliseconds" move as `gh-ost`'s sentry-table cut-over. The right time to put these habits in place is now, while the tables are still small enough that it's a choice, not the week it becomes the only option left.

---

## Sources

- [github/gh-ost, GitHub repository](https://github.com/github/gh-ost): fetched directly. Primary source for gh-ost's core architecture (triggerless design, binlog-based change capture from a connected replica, ghost table creation, chunked row copy, and the atomic cut-over), and its "testing on replicas" capability for rehearsing a migration before it touches production.
- [gh-ost: GitHub's online schema migration tool for MySQL, The GitHub Blog](https://github.blog/news-insights/company-news/gh-ost-github-s-online-migration-tool-for-mysql/): primary source for gh-ost's 2016 origin story, including the motivating fact that GitHub's MySQL tables had grown into the billions of rows and that a naive `ALTER TABLE` on the `pull_requests` table would lock it for over an hour. Direct fetch of github.blog was blocked by this session's network egress policy; this fact was relayed through a search-indexed summary rather than a first-hand read, and is worth re-verifying against the primary post before citing elsewhere with confidence.
- [GitHub Availability Report: November 2021, The GitHub Blog](https://github.blog/news-insights/company-news/github-availability-report-november-2021/): primary source for the November 27, 2021 incident, a schema migration's final rename step causing a significant portion of MySQL read replicas to enter a semaphore deadlock, degrading Actions, API requests, Codespaces, Git operations, Issues, Packages, Pages, Pull Requests, and Webhooks, with writes remaining healthy and no data corruption, and GitHub's stated remediation of prioritizing functional partitioning and pausing schema migrations until the failure mode was better understood. Direct fetch was blocked by this session's network egress policy; relayed through a search-indexed summary of the primary report.
- [gh-ost/doc/cut-over.md, github/gh-ost repository](https://github.com/github/gh-ost/blob/master/doc/cut-over.md): source for the sentry-table cut-over mechanism, the Lock Connection and Rename Connection two-connection design, and the atomic, typically sub-one-second rename that swaps the ghost table into place. Relayed through a search-indexed summary of the document.
- [gh-ost/doc/throttle.md, github/gh-ost repository](https://github.com/github/gh-ost/blob/master/doc/throttle.md): source for the replication-lag-based throttling mechanism, the default `--max-lag-millis` ceiling, and GitHub's own tighter production thresholds. Relayed through a search-indexed summary of the document.
- [pt-online-schema-change, Percona Toolkit documentation](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html): source for `pt-online-schema-change`'s trigger-based design (`AFTER INSERT`/`AFTER UPDATE`/`AFTER DELETE` triggers replicating writes to a shadow table alongside a throttled batch row copy) and its interaction with MySQL metadata locks. Relayed through a search-indexed summary rather than a first-hand read.
- [PostgreSQL Documentation: ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html) and [Which ALTER TABLE Operations Lock Your PostgreSQL Table?, DEV Community](https://dev.to/mickelsamuel/which-alter-table-operations-lock-your-postgresql-table-1082): sources for Postgres's `ACCESS EXCLUSIVE` lock behavior on most `ALTER TABLE` operations, the PostgreSQL 11 change making `ADD COLUMN ... DEFAULT` a fast, metadata-only operation, and `CREATE INDEX CONCURRENTLY`'s weaker `SHARE UPDATE EXCLUSIVE` locking. Relayed through a search-indexed summary rather than a first-hand read.
- [pg_repack documentation, PostgreSQL Extension Network / pg_repack project](https://reorg.github.io/pg_repack/): source for `pg_repack`'s concurrent mode, its use of logical decoding to capture live changes during a table rebuild, and the brief `ACCESS EXCLUSIVE` lock it takes only to swap the rebuilt table's underlying file at the end. Relayed through a search-indexed summary rather than a first-hand read.

---

*Inference vs. fact, stated plainly: gh-ost's architecture (triggerless binlog-based change capture, chunked backfill, replication-lag throttling, and the sentry-table cut-over) is documented directly in gh-ost's own repository and its `doc/` files, and the repository's top-level README was read directly in this session rather than only relayed through search; the specific default and production threshold numbers for throttling, and the exact wording of the cut-over mechanism, came through search-indexed summaries of the linked `doc/` files rather than a first-hand read of those specific files, and are worth re-verifying directly. The 2016 origin story, including the "over an hour" lock on the `pull_requests` table, and the full account of the November 2021 incident, both come from GitHub's own official blog, but direct fetch of github.blog was blocked by this session's network egress policy throughout, so both are relayed through search-indexed summaries rather than a first-hand read of the primary posts, and should be treated as very likely accurate but not independently verified against the original source in this session. The Postgres lock behavior and `pg_repack` mechanics are documented, stable facts from their own respective official sources, also relayed via search rather than direct fetch, since docs.percona.com, postgresql.org via the specific dev.to explainer, and the pg_repack site were reached only through search summaries in this session. The architecture diagram's specific seven-layer framing, the "lock convoy" and "metastable failure" framing applied to the November 2021 incident in Section 6, the fire-extinguisher-adjacent analogies throughout, and the Rare.lab mapping are this lesson's own synthesis built on top of the documented mechanics above, not a claim that GitHub, Percona, or the PostgreSQL project describe their own systems in exactly these terms.*
