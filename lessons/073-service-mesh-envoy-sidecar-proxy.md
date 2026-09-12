# Day 73 — How do 100+ microservices written in five different languages get the same retries, timeouts, and circuit breaking, when one naive retry can turn a single struggling database into 64 requests hitting it at once?

**Date:** 2026-09-12
**Difficulty:** Expert
**Topic:** The service mesh and the sidecar proxy pattern (Lyft's Envoy, born 2015-2016): moving network resilience out of application code and into a language-agnostic infrastructure layer. This lesson sits directly on top of machinery this ledger already built. Day 13 named backpressure and load shedding as the fix for overload; this lesson asks where in the stack that fix should actually live once a company has stopped having one codebase and started having a hundred, in five languages, that all talk to each other. Day 39's distributed tracing solved "which of 40 services was slow"; a service mesh is a big part of how that trace data gets collected uniformly in the first place, from every hop, without every team hand-rolling instrumentation. Day 55's anycast and Day 34's cell-based architecture both isolate blast radius at the network and infrastructure layer rather than in application code, which is exactly the move this lesson makes for resilience logic.
**Stack relevance:** Rare.lab does not run a hundred polyglot microservices today, and standing up a full sidecar mesh now would be solving a problem Rare.lab does not yet have. What transfers immediately is not the topology, it is the discipline the topology exists to enforce: today, Rare.lab's embeddable runtime calls out from a customer's browser to Supabase Postgres, to Cloudflare R2 for content-addressed scene JSON, and eventually to a shader-compile backend, and each of those call sites almost certainly has its own bespoke, probably-inconsistent retry and timeout logic wherever it was written. That is precisely the "every language reimplements resilience differently" problem this lesson opens with, just with client-side JavaScript and backend workers standing in for Lyft's PHP, Python, and Go services. The lesson's real answer for Rare.lab today is: centralize retry budgets, timeouts, and circuit breaking into one shared client wrapper (or one edge Worker) that every call path goes through, before Rare.lab ever needs a real sidecar mesh. The mesh itself becomes relevant the day Rare.lab's backend splits into more than two or three independently deployed services (asset pipeline, shader compiler, runtime config) that all need the same resilience behavior without copying code between them.

---

## 1. The company and the breaking number

**Lyft, 2015: a monolith outgrowing itself.** When Matt Klein joined Lyft's infrastructure team in May 2015, Lyft was already partway through the migration every hyper-growth startup eventually makes: a single PHP monolith backed by MongoDB had started sprouting roughly thirty independent services, with more being added constantly, written in whatever language the team building them preferred. That meant PHP, Python, and soon Go, Java, and Node.js, all calling each other over the network, all in production, all at once. This is a real, common shape: a company doesn't choose to become polyglot on purpose, it accumulates polyglot the way any fast-growing engineering org accumulates technical decisions made by different teams at different times.

**The breaking number is not Lyft's alone, it is arithmetic.** Google's own Site Reliability Engineering book states the mechanism plainly: if a request fans out through three layers (say, a frontend, a backend, and a database-facing service) and each layer, on seeing a failure, retries up to three times (four attempts counting the original), the total number of attempts that can land on the bottom-most layer is not 4, it is 4 x 4 x 4 = 64. A single user tapping "request a ride" once can, in the worst case, generate sixty-four separate attempts hitting whichever service sits at the bottom of that call chain, and it does so exactly when that service is already too slow or too overloaded to answer the first attempt, let alone the other sixty-three. Multiply that by real traffic volume and a struggling dependency does not recover, it drowns.

**The scale Lyft actually reached.** By early summer 2016, the project Klein had been building to address this, Envoy, was fully deployed at Lyft for all edge and service-to-service networking, forming a mesh across more than a hundred services and carrying millions of requests per second. That is the number that makes "just write careful retry logic in each service" stop being a plan: a hundred-plus services, in five languages, each independently deciding how to retry, how long to wait before giving up, and when to stop hammering a dependency that has clearly given up answering, is not a resilience strategy, it is a hundred uncoordinated resilience strategies that can, and eventually will, amplify each other's failures.

---

## 2. Why the naive (demo) design dies

**The obvious version:** give every service's client code its own retry logic, its own timeout value, and, if a team is disciplined, its own circuit breaker, implemented as a shared library imported into whichever language that service happens to be written in.

**Death one: a shared library is not actually shared once there is more than one language.** A resilience library written in Python does nothing for the Go services, the Java services, or the Node.js services; each language needs its own reimplementation of the same retry, timeout, circuit-breaking, and load-balancing logic, ported and kept in sync by hand. In practice this means the logic drifts: the Python version gets a bug fix the Go version never receives, the Java version's circuit breaker trips at a different error threshold than the Node.js version's, and nobody company-wide can answer the simple question "what does this system do when a dependency is slow," because the honest answer is "it depends which language wrote the caller."

**Death two: retry storms amplify exactly where the SRE book's math says they will.** Without one place that owns the total retry budget across a call chain, each layer's client code retries independently, in ignorance of how many retries every other layer is already doing. The 4 x 4 x 4 = 64 amplification is not a hypothetical edge case, it is what independently-authored, per-language retry logic produces by default the moment a dependency several hops down starts failing. The system's own well-intentioned resilience code is what turns a partial slowdown into a total outage, the same retry-death-spiral shape this ledger has already named in Day 13.

**Death three: nobody can see the problem while it is happening.** Lyft's own stated motivation for Envoy was almost entirely about this: the network is inherently unreliable, and when something went wrong, it was nearly impossible to tell where, because the load balancers of the time (AWS's Elastic Load Balancers) did not even expose percentile latency metrics, only averages, which hide exactly the slow-tail behavior that matters. With a hundred-plus services each instrumented differently, or not instrumented at all, in five different languages, there is no single dashboard, and no shared vocabulary of "p99 latency for calls from service A to service B," to even start diagnosing a cascading failure while it is unfolding. Compare this directly to Day 39's distributed tracing problem: you cannot afford to record everything, but you also cannot debug anything if what little you do record isn't even comparable across services.

**The real-world version:** this is not a hypothetical. Klein has described spending three months building an initial version of what became Envoy specifically because Lyft's move to microservices, while organizationally valuable, had made the network itself an unreliable, poorly observed dependency that every team was independently and inconsistently trying to work around.

---

## 3. The architecture

```
Client (Lyft rider or driver app)
  - job: make one logical request; know nothing about how many
    internal services will end up handling it
  - analogy: a customer calling a company's single published phone
    number, with no idea how many departments that call will be
    transferred through internally

        |
        v
Edge Envoy (ingress proxy at the perimeter of the mesh)
  - job: terminate TLS, do the first layer of routing and rate
    limiting, and hand the request into the internal mesh in a
    uniform shape regardless of what the client sent
  - analogy: a building's front desk and mailroom, which opens and
    sorts every incoming letter the same way before any specific
    office ever sees it

        |
        v
Sidecar Envoy proxy (one per service instance, out-of-process,
running as its own container next to the application container)
  - job: intercept every outbound and inbound network call for the
    application instance it sits beside, and apply retries, timeouts,
    circuit breaking, load balancing, outlier detection, and mTLS
    encryption uniformly, no matter what language that application
    happens to be written in
  - analogy: giving every employee, regardless of what language they
    speak, an identical personal assistant who handles all their
    phone calls: redialing on a busy signal according to company
    policy, refusing to keep calling a number that has stopped
    answering, and always placing calls over the same secure line

        |
        v
Application container (business logic only)
  - job: implement the actual product feature; make plain local
    network calls to "localhost" and let its sidecar handle
    everything about how that call actually reaches the network
  - analogy: the employee just does their job in whatever language
    they're most productive in, and never has to think about dial
    tones, busy signals, or wiretapping

        |
        v
Control plane (Istio's Pilot, or an equivalent discovery and
configuration service; Envoy calls this the xDS API)
  - job: continuously compute the current, correct picture of which
    hosts exist, which routes are valid, and what the retry/timeout/
    circuit-breaker policy should be, then push that picture out to
    every sidecar in the mesh as pure data, not code
  - analogy: a company directory and policy memo that updates itself
    and is silently re-delivered to every assistant's desk the moment
    anything changes, so no assistant is ever working from a stale
    org chart

        |
        v
Load balancing and outlier detection inside each sidecar
  - job: choose which specific healthy host, among several
    identical instances of the callee service, gets this particular
    request, and temporarily stop sending traffic to any host that
    has been failing on its own recent calls
  - analogy: the assistant keeping a private, constantly updated
    mental note of which coworkers haven't been picking up lately,
    and quietly routing around them for a while without needing
    permission from head office first

        |
        v
Central telemetry sink (shared metrics, logging, and tracing
backend that every sidecar reports to identically)
  - job: receive the exact same shape of latency, error-rate, and
    trace data from every single hop in the mesh, regardless of
    which language produced either end of that hop
  - analogy: every assistant filling out the identical call log
    form at the end of every call, so a supervisor can compare call
    volume and failure rates across the entire building on one sheet
```

A service mesh is deliberately not a replacement for the queue-based, asynchronous pipelines this ledger covered in Day 9 and Day 42. It solves synchronous, request-response networking between services that expect an answer right away. Genuinely bursty, fire-and-forget, or multi-hop asynchronous work still belongs behind a queue; a mesh just makes the synchronous hops that remain uniformly resilient and observable.

---

## 4. The transferable mechanisms

- **Push resilience logic out-of-process, into one language-agnostic layer.** The sidecar's entire reason for existing is that retries, timeouts, circuit breaking, load balancing, and TLS do not need to live inside the application's own process or its own language's library ecosystem. They can live in a separate, battle-tested proxy binary that sits beside every instance and speaks plain network protocols to whatever runs next to it. This is the single idea that makes a hundred-plus services in five languages tractable: fix a bug once, in one place, and every service gets the fix on its next sidecar rollout, with zero code changes to the application itself.

- **Retry budgets, not per-call retry counts.** The fix for the 4 x 4 x 4 = 64 amplification problem is not "retry less" at any one layer in isolation, it's capping the total retry volume as a system-wide budget (for example, no more than some fixed percentage of a service's total request volume may ever be retries, in any given window), enforced at the one layer that can see the whole picture. A single caller's per-request retry count means nothing if forty other callers are each also independently retrying against the same struggling dependency at the same time.

- **Circuit breaking: stop sending load to something that is already failing.** Once a cluster of upstream hosts crosses a threshold of concurrent requests or error rate, the sidecar stops sending new requests to it and fails fast instead, rather than queuing more work behind a dependency that has already shown it cannot keep up. This trades a guaranteed, fast, explicit failure now for the alternative, an unbounded queue of hopeful requests that all eventually time out anyway, just slower and after consuming far more resources first.

- **Outlier detection: eject the one bad host, not the whole cluster.** A cluster of ten identical service instances rarely fails all at once; usually one instance is slow or erroring while the other nine are healthy. Outlier detection watches each individual host's own recent success rate and temporarily removes just that host from the load-balancing pool for a cooldown period, independent of whatever the control plane's shared configuration says, then quietly lets it back in to see if it has recovered. This is the same instinct as Day 16's hot-key handling, applied to a misbehaving server instead of a misbehaving cache key.

- **Configuration as continuously pushed data, not as a deploy.** The xDS control plane pattern means a fleet-wide policy change, a new timeout value, a new routing rule, a certificate rotation, is delivered to every sidecar as a live data update, not as a code change requiring a hundred separate service redeployments in five different languages. This is the same "decide from locally-held, centrally-updated state" instinct Day 71 used for edge rate-limit configuration, generalized to an entire mesh's routing and resilience policy.

- **Uniform telemetry as a side effect, not an afterthought.** Because every network hop in the system now passes through the same proxy software, every hop produces the same shape of latency percentile, error rate, and trace data automatically, regardless of what wrote either end of that hop. Lyft's own original motivation leaned heavily on this: the problem was not just that failures happened, it was that nobody could see clearly which of a hundred-plus services was actually the slow one.

---

## 5. The trade-offs

**Availability over strict correctness, chosen deliberately for network decisions.** Outlier detection will sometimes eject a host that was only transiently slow, and circuit breaking will sometimes fail a request fast that might have eventually succeeded if it had just waited a bit longer. Both are accepted false positives. The mesh is explicitly built on the belief that a system which sometimes gives up early on a recoverable request, but never lets one struggling dependency drag the whole call chain down with it, is more available overall than a system that tries to be maximally "fair" and patient to every single request, one at a time, until the whole chain collapses under sixty-four-fold retry amplification.

**Latency and resource cost, paid on every single instance, all the time.** Adding a sidecar means every request now takes at least two additional network hops inside the same machine or pod, once from the application into its own sidecar, and once from the destination sidecar into its own application, rather than a single direct hop. Each hop adds real, if small, latency, typically sub-millisecond to low-single-digit milliseconds depending on load, and each sidecar is its own running process consuming its own CPU and memory, meaning the total compute footprint of the fleet goes up, not down, in exchange for network resilience and observability that no longer needs separate, inconsistent implementation in every language. This is the same cost-versus-latency dial this ledger has seen before: paying continuously, in infrastructure, for insurance that only clearly earns its keep on the day something actually goes wrong.

**A new dependency on the control plane, deliberately designed to be safe to lose.** Pushing configuration from a central control plane to every sidecar creates a natural worry: what happens if that control plane itself goes down? The data-plane and control-plane split that Envoy popularizes is built around each sidecar continuing to operate on the last configuration it successfully received if the control plane becomes unreachable, rather than failing closed the instant it loses contact with it. This trade accepts eventually-stale configuration (a host that was removed five minutes ago might still be in a sidecar's local list a little longer than ideal) in exchange for the data plane, the part of the system actually carrying live production traffic, never being made to depend on the control plane's own uptime to keep functioning.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is the **retry storm**, the same shape as a metastable failure: a slow dependency causes timeouts, timeouts trigger retries at every layer above it that doesn't know any other layer is also retrying, those retries add load precisely to the dependency that is already struggling, which makes it slower still, which produces more timeouts, which produces more retries. The Google SRE book's 4 x 4 x 4 = 64 figure is this loop's arithmetic made concrete: it is not a pathological worst case dreamed up for a textbook, it is the default outcome of independently-written, well-intentioned retry logic at every layer of a multi-hop call chain, with nobody positioned to see, let alone control, the total volume of attempts stacking up at the bottom.

The senior fix does not add more retries, more timeout tolerance, or more capacity at the struggling layer and hope it catches up. It breaks the loop directly, by moving the decision of "should this call be retried at all, and how many total retries are allowed system-wide right now" out of each of the hundred-plus independently-written client codebases and into one shared layer that can actually see, and cap, the aggregate picture: a retry budget enforced centrally, and a circuit breaker that starts refusing new load to a cluster the moment it crosses a failure threshold, rather than queuing yet another hopeful attempt behind all the others already waiting. This is the same instinct Day 13 named for backpressure and load shedding, relocated one level down the stack: instead of teaching every application, in every language, to individually behave well under pressure, remove the application's ability to behave badly in the first place by taking the decision out of its hands entirely.

---

## Sources

- [Envoy joins the CNCF, Lyft Engineering (Matt Klein)](https://eng.lyft.com/envoy-joins-the-cncf-dc18baefbc22): primary source for the claim that by early summer 2016 Envoy was fully deployed at Lyft for all edge and service-to-service networking, forming a mesh across more than a hundred services and transiting millions of requests per second. Direct fetch was blocked by this session's network egress policy; described here from search-indexed summaries of the post.
- [Introducing Envoy, Lyft Engineering (Matt Klein), September 2016](https://eng.lyft.com/envoy-10c1990e6c1c): primary source for Lyft's original stated motivation, that the network was inherently unreliable and, when problems occurred, it was nearly impossible to determine the source, compounded by AWS ELBs of the time not exposing percentile latency metrics. Direct fetch blocked this session; described here from search-indexed summaries.
- [Site Reliability Engineering, Chapter 22: Addressing Cascading Failures, Google SRE Book](https://sre.google/sre-book/addressing-cascading-failures/): primary source for the retry-amplification arithmetic used as this lesson's breaking number, a request fanning out through three layers each retrying up to three times (four attempts) can produce as many as 4 x 4 x 4 = 64 attempts at the lowest layer, along with the retry-budget and single-layer-ownership mitigations this lesson reuses.
- [CNCF hosts Envoy, CNCF, September 13, 2017](https://www.cncf.io/blog/2017/09/13/cncf-hosts-envoy/): primary source for Envoy's acceptance into the CNCF as an incubating project.
- [Cloud Native Computing Foundation Announces Envoy Graduation, CNCF, November 28, 2018](https://www.cncf.io/announcements/2018/11/28/cncf-announces-envoy-graduation/): primary source for Envoy becoming the third project to graduate from the CNCF, after Kubernetes and Prometheus.
- [Service mesh data plane vs. control plane, Envoy Proxy blog (Matt Klein)](https://blog.envoyproxy.io/service-mesh-data-plane-vs-control-plane-2774e720f7fc): primary source for the architectural split between the data plane (sidecars carrying live traffic) and the control plane (the system computing and pushing configuration), and the design principle that the data plane should keep operating on its last known configuration rather than fail if it loses contact with the control plane. Direct fetch blocked this session; described here from search-indexed summaries.
- Istio's own project history (open sourced by Google, IBM, and Lyft in May 2017, using Envoy as its data plane) corroborated across multiple secondary sources including Tetrate's and Solo.io's Istio architecture write-ups, used here only for the data-plane/control-plane split's broader adoption context beyond Lyft itself.
- Day 13 (backpressure and load shedding), Day 16 (the hot-key and celebrity problem), Day 34 (cell-based architecture and shuffle sharding), Day 39 (distributed tracing and sampling at scale), Day 55 (anycast global server load balancing), Day 71 (distributed rate limiting at the edge): the ledger's own prior lessons this one directly reuses and recombines.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of every Lyft and Envoy engineering blog post cited above (eng.lyft.com, blog.envoyproxy.io, mattklein123.dev, www.envoyproxy.io, and www.usenix.org were all blocked on direct fetch), so the specific figures (the hundred-plus services and millions of requests per second by early summer 2016, the original motivation around ELB percentile metrics and network unreliability, and the data-plane/control-plane design philosophy) are summarized here from search-indexed excerpts and corroborating secondary sources rather than quoted from a full direct read, consistent with how this ledger has flagged network-blocked sources on Day 69, Day 70, Day 71, and Day 72. The Google SRE book's retry-amplification figure and the CNCF acceptance and graduation dates were independently corroborated across multiple sources and are treated as solid.
