# Day 50 — How do you run millions of strangers' untrusted code on the same machines without one program reading another's secrets?

**Date:** 2026-08-07
**Difficulty:** Expert
**Topic:** Sandboxed multi-tenant execution: why running someone else's code in your own process is a one-line security disaster, why a full virtual machine per job is too slow to feel instant, and how AWS built Firecracker, a stripped-down microVM, to get hardware-grade isolation down to a 125 millisecond boot, the same problem an online judge like LeetCode solves at a smaller scale with Linux namespaces, cgroups, and seccomp instead of a hypervisor.
**Stack relevance:** Rare.lab's node-based editor compiles a user's graph into shippable code, then runs it in an embeddable runtime that shares one WebGL context. The moment that compiled output runs anywhere Rare.lab owns, on a shared render worker, in a server-side compile step, in a session another user can observe, it is the same problem this lesson is about: someone else's logic, running on infrastructure you are responsible for, that has to be both fast and unable to hurt anyone else using it.

---

## 1. The company and the breaking number

**AWS Lambda**, and the microVM it runs on, **Firecracker**. By late 2025, AWS's own re:Invent session on Lambda's internals (CNS423, delivered by Lambda's Principal Serverless Developer Advocate and a Principal Software Engineer on the Lambda team) put Lambda's monthly traffic north of **15 trillion invocations a month**, with **1.7 trillion invocations on Prime Day alone**, while holding **99.99% availability**. Those are AWS's own reported figures from their own conference talk, not a third-party estimate.

The number that breaks a naive design is not the trillions, it is a much smaller pair of numbers that have to be true at the same time: a function has to start running in well under a second (ideally closer to 100 milliseconds, since users perceive anything slower as lag), and AWS has to pack **thousands of different customers' functions onto the same physical host**, none of whom trust each other and none of whom AWS can afford to give a dedicated machine. The Firecracker NSDI '20 paper (Agache et al., the canonical source for this lesson) states the design target directly: a microVM has to boot and start accepting API calls in **under 125 milliseconds**, and it has to add **under 5 MiB of memory overhead** per instance, specifically so that **thousands of microVMs fit on one host** and the host can spin up new ones at a rate of **up to 150 per second**. A traditional full virtual machine boot, BIOS or firmware init, full OS boot, service startup, commonly costs seconds, sometimes five to ten, and reserves hundreds of megabytes to gigabytes of fixed memory per instance whether or not the workload uses it. Both of those numbers are roughly ten to a hundred times too slow, and too fat, for what Lambda needed.

The same underlying problem shows up at a completely different scale on a coding platform like LeetCode. A 2026 mock system-design interview on this exact question (a Google software engineer interviewing for the "design LeetCode" prompt, video: youtube.com/watch?v=QBHTbtWSECg) walks through the same realization the interviewer pushes the candidate toward directly: running a user's submitted code inside the API server process is "a very huge security risk," because a malicious or simply buggy submission, an infinite loop, a fork bomb, a program that reads environment variables it should never see, does not stay contained to that one request. LeetCode does not publish submission-per-second numbers, but the shape of the spike is well understood from the product itself: LeetCode's weekly and biweekly contests compress what is normally a steady trickle of submissions into a burst of tens of thousands of people clicking Submit inside the same 90-minute window, each one expecting an answer in a couple of seconds.

---

## 2. Why the naive (demo) design dies

**Version one: execute the submitted code directly inside the API server's own process.** This is the version every judge starts as a prototype, and every serious engineering discussion of the problem, including the mock interview above, immediately rejects it. Three concrete failures, not one:

**No isolation between tenants at all.** The submitted code runs with the same process privileges as the server handling everyone else's requests. It can read environment variables, open files the server has access to, or exhaust a shared resource, memory, open file descriptors, CPU, that every other in-flight request also depends on. One bad submission does not fail privately; it can take the whole server down for every other user currently being served by that process.

**CPU and memory hogging with no ceiling.** An accidental `while True:` loop or an unbounded recursive call pins a CPU core indefinitely and, without an externally enforced ceiling, keeps running until something else notices, usually a user complaining that the whole site is slow.

**No fault isolation.** Because there is no boundary between "code under test" and "code serving everyone else," a crash in the submitted code, a segfault, an out-of-memory kill, an unhandled kernel-level signal, can take the serving process down with it, turning one submission's bug into an outage.

**Version two: give every submission its own full virtual machine.** This fixes the isolation problem completely, VMs get hardware-enforced separation from the host and from each other, but it fails on the two numbers from section 1. A general-purpose VM boot is measured in seconds, not milliseconds, which is unacceptable when the product promise is "instant feedback." And a full VM's fixed overhead, its own kernel, its own device drivers, its own reserved memory, means a host can only fit a small number of them at once. During a contest spike where submission volume can jump by an order of magnitude in minutes, a fleet sized for full-VM-per-submission would need to provision that same order of magnitude more hardware, most of which sits idle the rest of the week. This is the AWS Lambda version of the exact same wall: hardware isolation was never in question, the speed and density were the wall.

**Version three: a plain container, chroot or a light namespace jail with no further hardening.** This is fast, boots in milliseconds because there is no kernel to boot, just a new set of namespaces around an existing kernel, and it is dense, thousands of containers can share one kernel with almost no per-container memory tax. But it shares the one thing version two isolated away: the host kernel itself. A container escape, a kernel bug reachable through a syscall the sandbox forgot to block, is a full breakout to the host and to every other tenant's container running on it, not a contained failure. For genuinely adversarial, untrusted code, and both a public cloud FaaS and a coding judge have to assume some fraction of submitters are actively trying to break out, container isolation alone is judged insufficient by everyone who has shipped this at meaningful scale: it is exactly why Firecracker exists instead of AWS simply running Lambda functions in bare Docker containers.

---

## 3. The architecture

The shape that both a hyperscale FaaS platform and a well-built online judge converge on, drawn as one pipeline:

```
[Client: submits code as a string + declared language]
   analogy: dropping a sealed letter in a mail slot, not
   handing it directly to whoever sorts the mail
   |
   v
[API server: validates the submission (size limits, declared
 language is supported), writes a job row, does NOT execute
 anything itself]
   analogy: the front desk that takes your form and gives
   you a ticket number, never the one who processes it
   |
   v
[Queue: absorbs bursts, decouples "accept rate" from
 "execute rate"; this is where a 10x contest spike gets
 smoothed instead of falling straight onto the execution
 fleet]
   analogy: a deli counter's ticket dispenser, everyone gets
   a number the instant they walk in, nobody's request is
   dropped just because the counter is currently backed up
   |
   v
[Worker pool, partitioned by language: a Python job goes to
 a Python-sized worker pool, a C++ job (which needs a compile
 step first) goes to a differently-provisioned pool]
   analogy: separate specialist stations at a hospital
   instead of one generalist queue for every kind of visit
   |
   v
[Sandbox creation, one fresh, disposable environment per job:
 either a Firecracker-style microVM (hardware-virtualized,
 its own tiny kernel, boots in around 125ms) or a
 namespace + cgroup + seccomp jail (isolate/Judge0 style,
 boots in low single-digit milliseconds because there is no
 kernel to start)]
   analogy: an individual soundproofed practice room, freshly
   reset before every musician, nobody can hear or touch
   what happens in the room next door
   |
   v
[Resource governor, enforced from OUTSIDE the sandboxed
 process, not by the process trusting itself: hard wall-clock
 timeout, memory ceiling, CPU quota, no network access,
 read-only filesystem except one scratch directory]
   analogy: a hotel minibar with a spending cap built into
   the lock, not an honor system
   |
   v
[Execution: run the compiled or interpreted program against
 the test cases, capture stdout, stderr, exit code, wall
 time]
   |
   v
[Teardown: destroy the microVM or jail completely, wipe the
 scratch directory, never reuse this exact environment for a
 different tenant's job]
   analogy: housekeeping resets the hotel room to zero before
   the next guest, nothing carries over
   |
   v
[Result writer: persist pass/fail per test case, runtime, and
 any error to the submissions store; update a cache the
 leaderboard reads from]
   |
   v
[Client: polls or gets pushed the result, sees pass/fail and
 timing within a couple of seconds]
```

The structural decision underneath this, stated once so it does not have to be repeated per layer: **untrusted execution never touches the box that is also serving requests for everyone else, and the environment it runs in is single-use.** Firecracker's own design document (firecracker-microvm/firecracker, `docs/design.md`) states the threat model plainly: a Firecracker vCPU thread is treated as running malicious code from the moment it starts, and every layer around it, seccomp filters restricting which syscalls the guest can even attempt, a companion process called the **jailer** that drops privileges and sets up cgroups and namespaces as a second line of defense before the VM even starts, a minimal device model (only VirtIO net and block, a serial console, nothing else emulated) so there is less surface to attack in the first place, exists because no single layer is trusted to be the only thing standing between the guest and the host.

---

## 4. The transferable mechanisms

**a. Defense in depth, stacked isolation layers, not one strong layer.** Firecracker does not rely on hardware virtualization alone; it adds seccomp syscall filtering and a jailer process on top of the VM boundary. Judge0's execution engine, `isolate` (originally built by Martin Mareš for the International Olympiad in Informatics and still the sandbox underneath Judge0 and CMS today), stacks Linux namespaces for process isolation, cgroups for hard resource limits, and seccomp for syscall filtering, three independent mechanisms, not one. The transferable rule: when the thing you're isolating is explicitly adversarial, assume any single layer will eventually have a bug, and make the layers independent enough that one failing does not collapse the rest.

**b. Shrink what you're virtualizing to buy back the speed hardware isolation normally costs you.** Firecracker's entire reason for existing is that a general-purpose VM (think QEMU booting a full guest OS with dozens of emulated devices) is too slow to use per-invocation, and a container alone is not isolated enough. Firecracker's answer is neither: strip the device model down to almost nothing, no full BIOS, no graphics, no USB, no legacy device emulation, so there is barely anything to initialize, and boot time drops from seconds to under 125 milliseconds while memory overhead drops to under 5 MiB. The generalizable lesson is not "use Firecracker specifically," it is that security and speed are not always a straight-line trade-off; sometimes the actual lever is reducing the surface area of what needs to start up in the first place.

**c. Per-tenant resource governance enforced from outside the thing being limited.** cgroups cap a process's CPU and memory from the kernel, not from inside the process; Firecracker's rate limiters are token buckets applied per microVM for network and disk I/O, the same token-bucket primitive this ledger covered for API rate limiting on Day 8, just applied at the tenant-isolation layer instead of the request layer. A timeout the sandboxed process is expected to honor itself is not a real limit, because a wedged or malicious process is exactly the thing that will not honor it; the limit has to live in something the sandboxed code cannot touch.

**d. Disposable, single-use execution environments.** Neither Firecracker microVMs handling a fresh invocation of an unfamiliar function, nor `isolate`'s per-submission boxes, are meant to be shared across tenants. Reuse across two different users' code is exactly the crack a state-leakage bug would live in; destroy-and-recreate is cheap precisely because mechanism (b) made creation cheap.

**e. Queue as the shock absorber between accept-rate and execute-rate.** Exactly the Day 9 pattern, applied here to a workload whose per-job cost is unusually unpredictable (a job might take 10 milliseconds or, if it's an accidental infinite loop, pin a worker until the wall-clock timeout kills it). The queue is what lets the API tier keep accepting a contest's submission burst at whatever rate it arrives, while the execution fleet works through it at its own, separately-scaled rate.

**f. Partition workers by workload shape, not just by load.** A Python submission and a C++ submission are not the same job: one needs a compile step first, the other doesn't; their typical execution times differ by an order of magnitude. Routing by language to differently-sized pools (the mock interview above lands on exactly this: "for each of the languages that we are using we can have like one container for that particular language") avoids a slow compiled-language job starving a pool sized for fast interpreted-language jobs.

---

## 5. The trade-offs

**Isolation strength versus density and cost, and different products land in different places on purpose.** A Firecracker microVM gets hardware-enforced, kernel-level isolation, a guest kernel exploit still has to escape the hypervisor boundary to touch the host, at the cost of running an actual (if minimal) VMM process and a real, if tiny, boot. `isolate`'s namespace-and-cgroup jail shares the host kernel with every other job on the box; a kernel-level bug is a full breakout, not a contained one. AWS accepts Firecracker's marginally higher cost per instance because Lambda's threat model is "any AWS customer, running genuinely arbitrary code, against every other AWS customer," the highest-stakes version of this problem that exists. A programming-contest judge accepts `isolate`'s thinner isolation because its threat model is narrower, competitive programmers trying to read another contestant's test cases or get free compute, not organized attackers probing for a kernel zero-day to pivot into a bank's cloud account next door. This is the same lesson this ledger keeps naming for consistency and availability, generalized to security posture: the right isolation strength is a property of who you're isolating from and what they stand to gain, not a single fixed policy applied everywhere.

**Cold-start latency versus idle cost.** Keeping a warm pool of pre-booted, idle sandboxes ready per language cuts the user-visible latency on the first submission of a burst down near zero, at the cost of paying for capacity that may sit unused most of the day. Booting a sandbox fresh on every submission is cheaper at rest but adds the full 125-millisecond-class boot penalty (or, for a namespace jail, a smaller but nonzero setup cost) to the very first job in each burst, exactly when a contest's traffic just spiked and users are watching a timer. Neither choice is free; it is a knob tuned to how spiky the traffic actually is.

**Warm reuse within a single tenant versus never reusing across tenants.** Lambda does reuse a warm execution environment across back-to-back invocations of the *same* function (the mechanism that makes most Lambda calls after the first one fast), which trades a small, well-scoped risk, state leaking between two calls belonging to the same customer, for a large latency win. That reuse never crosses a tenant boundary; the isolation this whole lesson is about is preserved exactly at the point where it matters, between strangers, while being relaxed exactly where the risk is contained, between two calls from the same owner.

---

## 6. The systems-thinking lens

**The feedback loop: a submission spike turning into a queue-backup death spiral.** Picture a contest's opening minute: thousands of users submit within the same short window. If job creation and sandbox execution are not cleanly decoupled by a queue, per-job latency starts climbing the moment the worker fleet saturates. Users watching a spinner past the 2-to-5-second mark they were promised do the human equivalent of a network retry: they hit Submit again, assuming the first attempt failed. Each retry is a brand new job landing on the same already-backed-up queue, which pushes latency up further, which triggers more retries. This is a thundering herd built entirely out of impatient humans instead of client libraries, and it is made structurally worse here than in an ordinary API-overload scenario, because a submission's execution cost is not fixed: a single accidental infinite loop, with no hard external timeout, can pin a worker indefinitely, silently shrinking the effective size of the pool draining the queue while the queue keeps growing.

**The senior fix breaks the loop at its two actual sources, not by adding more workers.** First, the wall-clock timeout has to be enforced from outside the sandboxed process, at the cgroup or hypervisor boundary, exactly mechanism (c), so a wedged submission is killed and its worker returned to the pool on a fixed schedule no matter how badly it misbehaves; an application-level timeout inside code that might itself be the thing that's hung is not a real backstop. Second, the client-facing side of the loop gets broken by giving the user something better to look at than a silent spinner past their expected wait: a visible queue position or an explicit "your submission is queued" state removes the ambiguity that turns into a reflexive resubmit, and a per-user idempotency check, don't enqueue a second job if an identical submission from the same user is already pending, closes off the retry path structurally instead of relying on the user's patience. Neither fix adds a single extra machine; both change what happens when the system is already under load, which is the same move this ledger named for retry storms on Day 13 and for thundering herds on Day 19, applied here to a workload where the "bad request" isn't just high-volume traffic, it can be one line of someone's buggy code.

---

## Map to Rare.lab's stack

**Where the same shape shows up, stripped of the judge-platform wrapper.** Rare.lab's node-based editor compiles a user's authored graph into shippable code, and the embeddable runtime executes that output inside one shared WebGL context per session. The instant any part of that compiled output runs somewhere Rare.lab owns and other users' sessions can be affected by, a server-side compile or validation step, a thumbnail or preview render, a shared render worker handling more than one customer's job, this is structurally the same problem as running a LeetCode submission: it is someone else's logic, and the naive move, run it in the same process, trust it not to misbehave, is precisely the version-one design that section 2 shows failing first, an infinite loop or an unbounded allocation in one user's shader graph does not stay contained to that one job, it takes down or stalls the shared worker serving everyone else's requests at the same time.

**The concrete first move, before any multi-tenant server-side execution ships.** Put a hard timeout and a hard memory ceiling on any server-side compile or render step at the process or container boundary, cgroups or an equivalent OS-level limit, not a `setTimeout` inside the JavaScript or shader-compile code itself, because a wedged or runaway compile is exactly the case where an in-process timer cannot be trusted to fire. Pair that with mechanism (d): treat each server-side compile/render invocation as disposable, a fresh worker or container per job rather than a long-lived compiler process reused across different users' graphs, so a bad graph's damage is bounded to the one job that triggered it and cleanup is a teardown, not a hope that the process's internal state is still clean. Rare.lab does not need Firecracker-grade hardware virtualization on day one, `isolate`-style namespace and cgroup limits are the right-sized version of this lesson's answer for a first version, exactly the way a contest judge chose that layer over a full microVM, but it does need the external, kernel-enforced ceiling before it needs anything else, because that is the one piece missing from every version of this story that failed first.

---

## References and summaries

**Firecracker design documentation (official): `firecracker-microvm/firecracker`, `docs/design.md`**
https://github.com/firecracker-microvm/firecracker/blob/main/docs/design.md
The primary source for this lesson's threat model and layered-isolation details: Firecracker treats every guest vCPU thread as running malicious code from the moment it starts, and stacks seccomp syscall filtering, a jailer process (privilege dropping, cgroups, namespaces, chroot) as a second line of defense, and a deliberately minimal device model (VirtIO net/block, a serial console, nothing else) so there is as little attack surface as possible even before the VM boundary itself is considered.

**Firecracker project site and GitHub README (official)**
https://firecracker-microvm.github.io/ and https://github.com/firecracker-microvm/firecracker
Source for this lesson's headline density and speed numbers: boot and start accepting API calls in under 125 milliseconds, under 5 MiB of memory overhead per microVM, up to 150 microVM creations per second on a single host, and token-bucket rate limiters applied per microVM for network and storage I/O so thousands of microVMs can share one host's resources fairly.

**AWS re:Invent 2025, session CNS423, "From Trigger to Execution: The Journey of Events in AWS Lambda," recapped at**
https://repost.aws/articles/AR3aVdHAAURayFAhfVyTzkKg
AWS's own current scale figures for Lambda, delivered by Lambda's Principal Serverless Developer Advocate and a Principal Software Engineer on the Lambda team: over 15 trillion invocations processed monthly, 1.7 trillion invocations on Prime Day alone, 99.99% availability, and internal queueing patterns including shuffle sharding to prevent one noisy tenant from creating a hot partition that degrades everyone else, and dedicated high-traffic "express lane" queues.

**`ioi/isolate` (official repository) and the original isolate design writeup**
https://github.com/ioi/isolate and https://mj.ucw.cz/papers/isolate.pdf
Primary source for the contest-judge side of this lesson: `isolate`, built by Martin Mareš (with Bernard Blackham) originally for the International Olympiad in Informatics and still maintained today, sandboxes untrusted submissions using Linux namespaces for process isolation, cgroups (v2) for hard CPU and memory limits, and seccomp for syscall filtering, the three-layer stack this lesson calls out as the lighter-weight, kernel-shared alternative to Firecracker's hardware virtualization.

**Judge0 (official repository)**
https://github.com/judge0/judge0
Secondary source used for this lesson's execution pipeline: a Rails API accepts a submission, writes it to Postgres, and enqueues a Resque job; the job runs inside an `isolate`-sandboxed process with per-job CPU, memory, and wall-clock limits, then writes results back for the API to return, the same accept-then-queue-then-sandbox shape this lesson generalizes in section 3.

**System Design Mock Interview: Design LeetCode, ft. a Google software engineer**
https://www.youtube.com/watch?v=QBHTbtWSECg
The motivating example for this lesson's naive-design walkthrough: the interviewee explicitly rejects running submitted code inside the API server (calling it a major security risk and a resource-hogging risk), works through a full-VM-per-submission option and its cost problems, and lands on a container-per-language-runtime pool fed by a queue, arriving independently at close to the same shape this lesson derives from Firecracker and `isolate`, useful as a worked example of the reasoning process, not as a primary engineering source with audited production numbers.

**Northflank engineering blog: "Firecracker vs gVisor: Which isolation technology should you use?"**
https://northflank.com/blog/firecracker-vs-gvisor
Secondary source for this lesson's isolation-strength trade-off in section 5: gVisor implements a userspace application kernel that intercepts and re-implements guest syscalls without a hypervisor boundary, faster to start than a microVM but without hardware-enforced kernel separation, while Firecracker's guest kernel runs inside real hardware virtualization, a genuine security boundary at the cost of running an actual (if minimal) VMM per instance.
