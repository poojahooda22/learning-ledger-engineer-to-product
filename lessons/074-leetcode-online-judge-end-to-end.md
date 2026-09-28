# Day 74: How does LeetCode judge 100,000 strangers' code in a weekly contest, in seconds, without one bad submission taking the site down?

**Date:** 2026-09-28
**Difficulty:** Advanced (capstone: a full interview-style design, start to finish)
**Topic:** The complete "design LeetCode" blueprint, taught as a reusable FRAMEWORK: requirements, entities, API, a naive high-level design, then deep dives. Earlier lessons covered two slices of this in depth (Day 50: sandboxing untrusted code with Firecracker; Day 54: Dream11's precomputed leaderboard). This lesson is the whole map that connects them, plus the two parts neither covered: fair job scheduling on the judge fleet, and the contest-start stampede.
**Stack relevance:** Rare.lab compiles a user's node graph into shippable code and runs it in an embeddable runtime. That is a "run a stranger's program, get a verdict" pipeline in miniature. Section 6 maps every box below onto Rare.lab's Supabase, Cloudflare R2 and shared-WebGL-context stack.

---

## 0. The framework (learn this shape once, reuse it for any product)

The mock interview you shared follows the same order every good system design answer follows. Memorize the order, not LeetCode.

1. **Functional requirements.** What can a user DO? Say what is out of scope out loud (login, payments, analytics), so nobody assumes it.
2. **Non-functional requirements.** How must it BEHAVE? Availability vs consistency, latency, scale, isolation and security, fault tolerance.
3. **Core entities.** The nouns: Problem, User, Submission, Leaderboard.
4. **API.** One REST route per requirement (`GET /problems?page=1&limit=100`, `GET /problems/{id}?language=python`, `POST /problems/{id}/submission`, `GET /leaderboard/{competitionId}`).
5. **Naive high-level design.** Client, API server, database. Get a working version on the board first.
6. **Deep dives.** Attack the naive version where it breaks: sandbox, queue, cache, scaling, replicas.

The rest of this lesson follows the six numbered sections of the ledger format, with this framework as the spine.

---

## 1. The company and the breaking number

**LeetCode, and one Sunday-morning number.** LeetCode has roughly 4,000 to 5,000 problems (the interviewee's working figure) and about a million daily active users. The catalog is tiny and boring: 5,000 rows fit in a laptop's memory. The number that breaks a naive design is the weekly contest: **about 100,000 people start the same 4 problems in the same 90 minutes**, and each person submits code many times.

**Worked arithmetic (labeled estimate, not a LeetCode figure).**
- 100,000 contestants x roughly 6 submissions each over 90 minutes = about 600,000 submissions.
- Submissions bunch up at the start (everyone hits problem 1 at minute 0) and in the last 10 minutes. Assume the peak is 5x the average. Average = 600,000 / 5,400 sec = 111 submissions/sec. Peak = about 550 submissions/sec.
- Each submission runs against maybe 50 to 100 test cases and takes 0.5 to 3 seconds of real CPU. So the fleet needs roughly 550 x 2 sec = **1,100 CPU-seconds of judging every second**, i.e. about 1,100 busy cores at peak, just for judging.
- Meanwhile 100,000 browsers refresh a live leaderboard. At one refresh per 5 seconds that is **20,000 leaderboard reads/sec**.

The two breaking numbers: **~550 heavy jobs/sec that must never run on the web server, and ~20,000 reads/sec that must never touch the database.**

---

## 2. Why the naive (demo) design dies

**The naive version:** one API server, one database. User posts code, the server runs it (`python solution.py`), compares output, writes the verdict, returns it. The leaderboard is `SELECT user, SUM(points) ... GROUP BY user ORDER BY ...` on every page load.

It collapses in four ways:

- **Security: the server runs a stranger's code.** Someone submits `import os; os.system("rm -rf /")` or reads your database password from the environment. The interviewee named this correctly: a malicious submission can delete data or take the whole service down. Analogy: letting every visitor cook in your home kitchen, unsupervised.
- **Resource hogging: one loop eats the machine.** `while True: pass` pins a CPU forever. A fork bomb (`os.fork()` in a loop) exhausts the process table. `x = [0] * 10**10` eats all RAM. With one shared server, one bad submission starves every other user. Analogy: one person leaves every tap running and the whole building loses water pressure.
- **Synchronous waiting kills the web tier.** If each web request blocks for 3 seconds while code runs, 550 requests/sec needs about 1,650 web workers held open doing nothing. The API's job (accept, validate, answer) gets buried under the judge's job (compute).
- **The leaderboard query melts the database.** `GROUP BY ... ORDER BY` over 600,000 submission rows, executed 20,000 times/sec, is a full sort each time. Analogy: recounting every ballot in the country every time someone asks "who is winning?"

---

## 3. The architecture

```
Clients (browser: editor + problem page + live leaderboard)
  - job: render the problem, send code as a plain string plus a language tag,
    then WAIT FOR A VERDICT (poll or listen), never a blocking request
  - analogy: you hand your exam paper to a desk and get a ticket number

        |
        v
Edge / CDN (Cloudflare, CloudFront)
  - job: serve the problem statement, sample tests and the JS bundle from
    cache. At contest start, 100,000 people fetch the SAME 4 problem pages.
    Those bytes are identical for everyone, so the origin should see ~1 request per region, not 100,000
  - analogy: photocopies of the exam paper stacked at every entrance

        |
        v
Load balancer
  - job: spread requests across identical API servers, drop dead ones from rotation
  - analogy: the host at a restaurant who seats people at whichever table is free

        |
        v
Stateless API tier (many small servers)
  - job: validate, write a Submission row with status QUEUED, put a job on the
    queue, return 202 Accepted with a submissionId in a few milliseconds.
    Never runs user code.
  - analogy: the counter clerk who stamps your paper and hands it to the back room

        |                       |
        v                       v
Cache + Primary DB            Message queue (Kafka / SQS / Redis Streams)
  - Postgres (or DynamoDB)      - job: hold the judging backlog durably.
    is the system of record       Absorbs the spike: the queue can grow to 50,000
    for problems, users,          waiting jobs and nobody gets an error, they get
    submissions. Read replicas    a slower verdict.
    serve reads. A failed         - analogy: a ticket rail in a kitchen; orders
    primary is replaced by          pile up on the rail, cooks take them at their pace
    promoting a replica.
        |                       |
        |                       v
        |            Judge worker fleet (autoscaled on queue depth)
        |              - job: pull ONE job, run it inside a sandbox with hard
        |                limits, run all test cases, write the verdict
        |              - one pool per language (Python, Java, C++), because each
        |                needs a different compiler and runtime
        |              - analogy: separate stoves for baking, grilling and frying
        |                                |
        |                                v
        |            Sandbox (per submission, thrown away afterward)
        |              - job: make a hostile program harmless. No network, read-only
        |                filesystem, CPU-time cap, wall-clock cap, memory cap, process cap,
        |                syscall allowlist
        |              - analogy: a blast chamber. Whatever happens inside stays inside
        |                                |
        v                                v
Leaderboard store: Redis Sorted Set, one per contest
  - job: on every ACCEPTED verdict, worker updates the contestant's score.
    Reads are O(log n) rank lookups from memory, not SQL.
  - analogy: a scoreboard bolted to the wall that the referee updates once,
    instead of everyone re-adding the scores themselves
```

Three details the interviewee raised that are worth keeping:

- **Test data is language-neutral.** Store each test case as JSON (input and expected output). Each language pool has a small deserializer that turns the JSON into that language's native shape (a Python dict, a C++ `unordered_map`). One test set, many runtimes.
- **The `language` on the problem route is not decoration.** It selects the right starter template AND the right worker pool for the judge queue.
- **Two-tier storage choice.** DynamoDB was the interviewee's pick because a problem is one nested document (statement, tests inside it) with no joins. Many production write-ups pick Postgres instead for ACID on submission status. Either is defensible; the reason must be stated (Section 5).

---

## 4. The transferable mechanisms

Each one is a reusable primitive, not a LeetCode trick.

- **Queue as shock absorber (async job + 202 Accepted).**
  - Split "accept the work" (milliseconds, cheap) from "do the work" (seconds, expensive). Put a durable queue between them.
  - The queue converts a spike in ARRIVAL RATE into a wait in LATENCY, which users tolerate, instead of errors, which they do not.
  - Autoscale the workers on QUEUE DEPTH, not CPU. Depth is the direct signal of "work waiting".
- **Isolation ladder: process, container, microVM.**
  - Plain process: no isolation. Never for hostile code.
  - Container (Docker; namespaces + cgroups): lightweight, starts in milliseconds, shares the host kernel, so a kernel bug is an escape. The interviewer's point in the video is correct: a VM isolates better; a container packs better.
  - Judge-specific sandboxes such as **isolate** (used by the IOI and the CMS contest system): namespaces + cgroups plus flags for CPU time (`-t`), wall time (`-w`), memory (`-m`), process count (`-p`, default 1, so fork bombs die instantly), and per-sandbox box IDs so parallel runs never share a directory.
  - MicroVM (Firecracker, Day 50): hardware-level isolation at container-like start cost.
  - gVisor: a user-space kernel that intercepts syscalls, in between.
  - Pick by threat: contest of your own users = isolate/nsjail-class plus seccomp; a public "run any code" product = microVM.
- **Hard limits as the timeout ("TLE").**
  - CPU-time limit catches `while True`. Wall-clock limit catches a program that sleeps or blocks. Memory limit catches the giant allocation. Process limit catches the fork bomb. "Time Limit Exceeded" is the product name for the CPU/wall cap firing.
- **Precomputed leaderboard (Redis sorted set, or the Day 54 batch pattern).**
  - A sorted set is a hash map plus a skip list. `ZADD` and `ZRANK` are O(log n); `ZREVRANGE 0 99` (top 100) is O(log n + 100).
  - Encode the tiebreak into the score: score = points, then subtract a time penalty, so "faster wins" is one number, no second sort.
  - Write on the verdict (rare, ~550/sec), read from memory (common, ~20,000/sec). The `ORDER BY` never runs.
- **Retry with exponential backoff plus an idempotency key.**
  - If a worker dies mid-judge, the job returns to the queue after a visibility timeout. Retry after 1s, 2s, 4s (with random jitter).
  - The job carries the `submissionId`. Judging it twice must write the same verdict once (Day 12), or a retry could double-count a contestant's points.
- **Cache immutable, shared, hot data at the edge.**
  - The four contest problems are read-only during the contest and identical for everyone. Serve them from the CDN with a long TTL. Reserve the origin for the writes.

---

## 5. The trade-offs

**Consistency vs availability, per data type (the interviewee chose availability):**

| Data | Choice | Why |
|---|---|---|
| Problem list, statements | Eventual (AP) | If one user sees 990 problems and another sees 1,000 for a minute, nobody is harmed. A new problem appearing a bit late is fine. |
| Submission verdict | Strong per submission (CP-ish) | A submission is one row with one state machine (QUEUED, RUNNING, ACCEPTED/WRONG/TLE). The user must see the true verdict for THEIR row. Reads on one key are cheap to make consistent. |
| Live leaderboard | Eventual, seconds stale | Top-100 being 2 seconds behind is invisible. Final standings are recomputed authoritatively from the submission table when the contest closes. |
| Points and rank finalization | Strong, once, offline | Prizes and rating changes depend on it. Run the exact recompute after the contest. |

**Cost vs latency:**
- VM per submission: best isolation, slowest and priciest. Container: cheap, fast, weaker isolation. The senior answer is a container-class sandbox plus seccomp for first-party contests, and a microVM only when the code source is fully anonymous.
- Warm pool of pre-booted sandboxes costs idle money but removes cold-start latency from the p99.
- Serverless (Lambda) was suggested in the video. It gives free scaling, but cold starts and per-invocation overhead hurt a p99 of "under 2 seconds", and a per-language pool is easier to tune. A legitimate trade, not a wrong answer.
- Availability over consistency for the site, but never over ISOLATION. Isolation is a hard constraint, not a dial.

---

## 6. The systems-thinking lens

**The feedback loop: the retry death spiral plus the thundering herd.**

The contest starts. 100,000 browsers load the problem pages within seconds (a thundering herd on the origin). Everyone starts submitting. Judging slows because the fleet is saturated. Contestants see "Pending" and hit Submit AGAIN, or mash refresh. Now each person has 2 or 3 jobs in the queue, the queue grows, verdicts get slower, more people retry. A system that was 20% over capacity becomes 300% over capacity, purely from its own users' retries. This is a metastable failure: even after the original spike ends, the retry backlog keeps it down.

**The senior fix breaks the loop; it does not just add servers.**
- **Idempotency at the door:** one `(user, problem, code hash)` in flight at a time. A second identical submit returns the existing `submissionId` instead of queuing new work.
- **Per-user concurrency caps and fair dispatch:** round-robin across users, not strict FIFO, so one user (or a script) with 200 submissions cannot starve everyone else.
- **Load shedding with an honest signal:** when the queue is over a threshold, reject NEW submissions with a fast "busy, retry in 10s" (HTTP 429 plus Retry-After) rather than letting them wait forever. A visible short wait beats an invisible long one.
- **Push, not poll, for the verdict:** hold a WebSocket or Server-Sent Events channel and push the verdict when done. Stops 100,000 clients polling every second (Day 13 backpressure).
- **Jittered client backoff** so retries do not synchronize into waves (Day 8, Day 71).
- **Pre-warm before the contest:** scale the worker fleet up and pre-push the problem pages to the CDN at T minus 5 minutes. The spike is scheduled, so treat it as a known event.

---

## 7. Map to Rare.lab's own stack

| Judge concept | Rare.lab equivalent | Already have? | Next ceiling |
|---|---|---|---|
| Stateless API + system-of-record DB | Supabase Postgres with RLS | Yes | Connection limits: Supabase pooler (PgBouncer) caps concurrent connections. Use the pooler URL for anything server-side and keep long jobs off the DB connection. |
| Immutable, cacheable public content on a CDN | Cloudflare R2, content-addressed scene JSON plus a manifest | Yes, and it is the strongest fit. Content-addressed = infinitely cacheable, like the contest problem pages. | Manifest is the one mutable pointer. Keep its TTL short and everything it points to immortal. |
| Untrusted code execution + limits | Compiled shader graph running in the embeddable runtime | Partly. WebGL runs in the browser's own sandbox. | A shared WebGL context means one bad shader can hang the GPU for every effect on the page. Add a compile-time complexity budget (max loop bounds, max texture reads) as the equivalent of the CPU/wall cap. |
| Queue plus worker pool | Server-side compile or thumbnail render jobs (if/when added) | Not yet needed | The first time compiling or rendering takes over ~1 second per request, do not do it in the request. Enqueue it (Cloudflare Queues + Workers, or Supabase Queues/pgmq), return 202, push the result. |
| Idempotency key | Content hash of the graph | Natural fit | Use the graph hash as the job key: identical graphs compile once, ever, and the result lives in R2 forever. |
| Leaderboard | Any "trending shaders" or usage ranking | No | Do not `ORDER BY` on read. Precompute on write (or on a schedule) into a small cached list. |

**One-line lesson for Rare.lab:** treat "compile and run a user's graph" as an async job keyed by the graph's content hash, with hard resource budgets and a cached immutable result. That single move gives you idempotency, edge caching and abuse limits at once.

---

## 8. What is inside the video transcript you shared (plain-language summary)

**Source:** a mock system design interview, "Design LeetCode", where a Google software engineer answers and the interviewer probes. The candidate recommends the Hello Interview site and a system design YouTube channel; the interviewer mentions the GKCS channel.

- **0:00 to 4:00, functional requirements.** View list of problems; view one problem and code in any language; submit and get instant feedback; live contest leaderboard. Auth, profiles, payments and analytics explicitly out of scope.
- **4:00 to 11:00, non-functional.** Availability over consistency (a user seeing 990 vs 1,000 problems is fine); isolation and security of user code; low latency (verdict under a few seconds, exact cap left open); scale to 100k concurrent contestants and millions of monthly users; fault tolerance, no single point of failure.
- **11:00 to 12:30, entities.** Problem, User, Solution/Submission, Leaderboard.
- **12:30 to 20:30, API.** REST. `GET /problems?page&limit` (paginated even at 4,000 problems); `GET /problems/{id}?language=` (language picks templates and the compiler pool); `POST` a submission with code and language as strings; `GET` leaderboard by competition id, paginated.
- **21:00 to 25:00, first design.** Client, API server, database. DynamoDB chosen: nested document (tests inside the problem), no joins, room to grow.
- **25:00 to 32:30, the sandbox debate.** Run on the API server: rejected (malware, DDoS, CPU hogging, no fault isolation). VMs: better isolation, but heavy and costly. Containers: lightweight, one runtime per language, many jobs per container. The interviewer corrects one point: VMs isolate better than containers, containers win on utilization and elasticity (ECS, EKS/Kubernetes). Serverless suggested; candidate stays with containers.
- **32:30 to 38:30, leaderboard.** Submission schema: id, user id, competition id, timestamp, result, runtime, error. Competition id as the partition key so a contest query reads only its own rows. Interviewer: one API, one grouped query, optimize later.
- **38:30 to 44:30, scaling.** Timeouts against infinite loops; temp directories cleaned up; polling the DB rejected (load, latency); a cache for the leaderboard; horizontal scaling; a queue between API and runtime workers to smooth spikes; exponential-backoff retries.
- **44:30 to 46:50, closing.** Same test cases in every language via JSON plus per-language deserializers; primary/replica databases with promotion on failure, needed for submissions, not for the small problems table.

**What the interview leaves for you to add** (this lesson fills them in): the leaderboard as a Redis sorted set instead of a cached SQL query; fair scheduling and idempotency to stop the retry spiral; push instead of poll; CDN for the identical problem pages; and a specific isolation mechanism with named limits.

---

## 9. References and what is actually in them

**Read directly this session:**
- [isolate, GitHub (ioi/isolate)](https://github.com/ioi/isolate): the sandbox built for programming contests, used by the IOI and the CMS contest system. It uses Linux namespaces and cgroups to limit CPU time, wall-clock time, memory and process count, with box IDs so parallel sandboxes stay separate. Its man page confirms the flags: `-t` CPU seconds, `-w` wall seconds, `-m` memory, `-p` processes (default: one), `--cg` for group accounting. Plain meaning: this is the actual off-the-shelf "blast chamber" many judges use.

**Seen only as search-result excerpts (the network proxy blocked opening the pages, so I have NOT read the full text; open them yourself for the detail):**
- [System Design: Leetcode style online judge, Puneet Patwari, Medium](https://medium.com/@patwaripuneet15/system-design-leetcode-style-online-judge-3375a2d2e8b9), [LeetCode Contest: System Design, Sanjiv Singh, Medium](https://medium.com/@singh.sanjiv/leetcode-contest-system-design-5df2cd1c0277), [Design a Global-Scale Online Judge, Dilip Kumar, Medium](https://dilipkumar.medium.com/design-an-online-judge-like-leetcode-30ff9e73b248), [Design Leetcode, Kshitij Agrawal, Medium](https://medium.com/back-2-basecs/system-design-series-design-leetcode-ba6d4e55f630), [Designing LeetCode, DEV Community](https://dev.to/matt_frank_usa/designing-leetcode-online-code-judge-system-4372): community write-ups. The search excerpts agree on the shape used here: queue between API and workers with 202 Accepted, sandboxed workers with namespaces and seccomp, a Redis sorted set per contest, Postgres as the record. Treat as secondary and educational.
- [LeetCode System Design, System Design School](https://systemdesignschool.io/problems/leetcode/solution) and [Online Judge System Design guide, intervu.dev](https://intervu.dev/blog/online-judge-leetcode-system-design/): excerpts mention autoscaling workers on queue backlog and fair dispatch (per-user concurrency caps, round-robin across users), which is where Section 6's fair-scheduling fix comes from.
- [Design Code Judging System like LeetCode, DesignGurus](https://www.designgurus.io/course-play/grokking-system-design-interview-ii/doc/design-code-judging-system-like-leetcode) and [Hello Interview: LeetCode breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/leetcode) (blocked; the interviewee recommends this site by name): interview-prep breakdowns in the same six-step framework.
- [Redis sorted sets docs](https://redis.io/docs/latest/develop/data-types/sorted-sets/) and [gVisor docs](https://gvisor.dev/docs/) (blocked): the O(log n) sorted set complexity and gVisor's user-space-kernel role are stated from general knowledge and should be confirmed there.

**Inference, labeled:** the capacity math in Section 1 (600,000 submissions, 5x peak, 1,100 cores, 20,000 reads/sec) is my back-of-envelope estimate, not a published LeetCode number. LeetCode's real internals are not public.

**Related ledger lessons:** Day 8 (rate limiting), Day 9 (queue as shock absorber), Day 12 (idempotency), Day 13 (backpressure and load shedding), Day 50 (sandboxed execution, Firecracker), Day 54 (real-time leaderboard), Day 71 (distributed rate limiting).
