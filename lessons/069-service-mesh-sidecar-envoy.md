# Day 69 — How does Lyft stop 300 microservices, written in three different languages, from taking each other down with retries?

**Date:** 2026-09-01
**Difficulty:** Expert
**Topic:** The service mesh and the sidecar proxy pattern: how a fleet of polyglot microservices gets one consistent, language-agnostic layer for retries, timeouts, circuit breaking, mTLS, load balancing, and observability, without every team re-implementing that logic in their own language. This ledger has already built two of the three pieces this lesson assembles: Day 39 covered following one request across 40 services without recording all of them (distributed tracing), and Day 47 covered pushing config to every node with no database call in the request path (Cloudflare Quicksilver, push not poll). A service mesh is those two mechanisms, plus a proxy sitting next to every single service instance, aimed at a problem neither of those lessons solved: not "how do I see what happened after the fact," but "how do I make every one of thousands of service-to-service calls behave safely, in the same way, regardless of what language wrote either end of the call."
**Stack relevance:** Rare.lab today is small enough that this lesson's main architecture is premature: a handful of services (the node-based editor's backend, a compiler/build pipeline, the embeddable runtime's asset delivery through Cloudflare R2) do not need a sidecar on every instance to stay safe from each other. But Rare.lab already runs on Cloudflare, and Cloudflare's own edge is a real-world instance of exactly this lesson's core idea at a much larger scale: a purpose-built proxy layer (Pingora, covered below) sitting between every request and the service that handles it, doing retries, connection reuse, and traffic control in one place instead of inside application code. The next ceiling for Rare.lab is the day the shader-compiler backend stops being one process: if AI-assisted shader generation becomes a separate Python service, the WGSL/Rust compiler stays its own process, and the node-graph API stays a third, the moment two of those are in different languages is the moment retries, timeouts, and circuit breaking either get consistently enforced in one place or silently drift apart between them the way Lyft's did.

---

## 1. The company and the breaking number

**Lyft, and the number hiding inside "polyglot."** When Matt Klein joined Lyft's infrastructure team in May 2015, the company had already begun splitting its original PHP monolith apart, and had reached more than 30 independent services. That number by itself is not large. What made it dangerous is that Lyft's engineering culture let each new service pick whatever language fit the job best: the original monolith was PHP, and the services replacing pieces of it were being written in Python and Go, with more languages arriving as teams grew. By the time Lyft finished decomposing the PHP monolith entirely, late in 2018, the backend was a mix of Python and Go services, and the fleet had grown past 300 microservices, moving millions of requests a second at peak, across more than 200 cities of live ride traffic. The breaking number is not the request rate. It is the product of two smaller numbers multiplying against each other: N services times M languages, where every one of those N times M combinations needs the same answer to the same question, "what happens when the service I'm calling is slow, or down, or returning garbage," and nothing was forcing that answer to be the same twice.

**Why a shared library, the obvious fix, does not survive contact with "polyglot."** The standard answer to "many services need the same networking logic" is a shared client library: write the retry policy, the circuit breaker, the load balancing, and the service discovery lookup once, and have every service import it. Twitter had already done exactly this with Finagle, and Netflix had done it with Hystrix, introduced in Netflix's November 2012 techblog post "Introducing Hystrix for Resilience Engineering." Both were genuinely effective, and both shared the same structural limit: they were JVM libraries, written for Scala and Java services, living inside the process that used them. Netflix's own framing of the problem Hystrix solved is worth stating precisely, because it names the failure mode this whole lesson is about: without isolation, "the entire system can grind to a halt from a single unhealthy dependency, even if all other service dependencies are healthy." Hystrix solved that for Netflix's largely-JVM fleet. It could not solve it for Lyft, where the calling service and the called service were routinely written in different languages, and a Python service has no way to import a Java circuit breaker.

**The concrete failure this produced at Lyft: cascading failure and "accidental internal denial-of-service."** Lyft's own infrastructure team described the actual symptom in plain terms: as the number of microservices grew, so did the number of outages caused by cascading failure, or by one internal service accidentally denial-of-service-ing another. The mechanism is mechanical, not exotic. Service A calls service B without a circuit breaker. B gets slow under load. A's calls to B start timing out, so A's own request-handling threads sit blocked waiting on B instead of failing fast. A's thread pool fills up with requests stuck waiting on a struggling B, so A itself becomes slow to everyone calling A, including its retries. Each one of Lyft's languages had its own version of retry logic, if it had any at all, with its own defaults, its own bugs, and no way for an operator to change all of them at once during an incident. A security fix or a new load-balancing algorithm did not mean updating one proxy config; it meant redeploying every service in every language that had its own copy of the logic, correctly, under incident pressure.

---

## 2. Why the naive (demo) design dies

**The obvious version:** each service calls the services it depends on directly, over HTTP or gRPC, using whatever HTTP client its language ships with, plus maybe a shared internal library for the team's language of choice. Retries, if they exist, are a few lines wrapping the call: "try three times, wait a second between tries." Timeouts are whatever the client library defaults to, if anyone thought to set them. Service discovery is a hostname in a config file or an internal DNS entry. This is exactly how Lyft, and nearly every company that starts splitting a monolith into services, begins, because for a first handful of services calling each other a few times a second, it is fast to build and easy to reason about: the call site is one function call away from the network request.

**Death one: the same safety logic has to be built N times, once per language, and it drifts every time.** A retry-with-backoff policy written in Go and one written in Python are never bit-for-bit identical unless someone is actively keeping them in sync, and nobody's job is to do that full time. One team's Python service retries three times with no backoff; another team's Go service retries once with exponential backoff and jitter; a third team's newest Node.js service has no retry logic at all because nobody got around to adding it before the deadline. None of this shows up in a demo with two services and light traffic. It shows up the first time a downstream service degrades under real load, because every caller behaves differently, unpredictably, and nobody can answer "what will happen to our traffic if this dependency gets slow" without reading every caller's source code individually.

**Death two: retries without a circuit breaker turn a struggling service into a dead one.** This is the retry storm, and it is not a hypothetical. If a downstream service starts failing 10 percent of its requests because it is overloaded, naive per-caller retries do not spread that load out, they concentrate it: every failed request becomes two, three, or more requests as callers retry, and the retries land on the same already-struggling service, which fails more of them, which triggers more retries. A service that could have recovered from a brief overload instead gets buried by its own callers' good intentions. Lyft's own engineering account of this class of incident calls it plainly: an internal service accidentally denial-of-service-ing another internal service, from inside the same company, with nobody attacking anyone.

**Death three: there is no consistent, per-hop observability, so nobody can see the failure coming.** In the naive design, each language's HTTP client logs, or doesn't, in its own format, to its own place. Answering "what is the p99 latency between service A and service B, right now" means finding whichever team owns A, hoping they instrumented that specific call, and hoping their metrics use compatible units and time windows to whichever team owns B. At two services this is mildly annoying. At 300 services in three languages, it is not answerable at all during an incident, which is exactly when the answer matters most. Day 39's tracing lesson describes the problem of following one request across 40 services; the naive design here does not even clear the lower bar of knowing, in aggregate, how any two services are currently treating each other.

**Death four: a fix has to be redeployed everywhere, one service at a time, in the middle of the incident it is meant to stop.** If the fix for a retry storm is "lower the retry count and add exponential backoff," the naive design has no single place to make that change. It has to be changed in every language's copy of that logic, in every service that calls the affected dependency, each requiring its own build, test, and deploy pipeline, while the outage is actively in progress. The safety mechanism is exactly as slow as the deploy process, at the exact moment speed matters most.

---

## 3. The architecture

```
Clients (rider app, driver app, third-party API consumers)
  - job: send a request into the system without knowing or caring
    which of hundreds of backend services will end up handling it
  - analogy: a caller dialing one phone number, not needing to know
    which department, floor, or building will answer

        |
        v
Edge proxy (Lyft's own front Envoy; a public-facing Envoy instance
terminating internet traffic before it enters the service fleet)
  - job: TLS termination, authentication, edge routing into the
    right internal service, and the first point where uniform
    rate limiting and observability apply to every request
  - analogy: the building's front desk, checking ID and directing
    you to the right floor, before you're allowed past the lobby

        |
        v
Stateless app tier: one process per service instance, EACH PAIRED
WITH ITS OWN ENVOY SIDECAR running in the same pod or on the same
host, listening on localhost
  - job (app process): pure business logic, in whatever language its
    team chose (Python, Go, and others at Lyft), completely unaware
    of retries, circuit breakers, mTLS, or service discovery
  - job (sidecar): every byte the app process sends to or receives
    from another service passes through this proxy first; it does
    the retrying, the timing out, the encrypting, the load balancing,
    and the reporting, identically, no matter what language the app
    next to it is written in
  - analogy: every apartment gets its own doorman. Tenants never deal
    with the building's security policy directly, the doorman does,
    the same way, for every apartment, regardless of what the tenant
    inside does for a living

        |   (sidecar to sidecar, not app to app, mutual TLS)
        v
Service mesh data plane: thousands of Envoy sidecars talking directly
to each other, forming the actual mesh of connections
  - job: carry every service-to-service call, applying per-call
    policy (timeout, retry budget, circuit breaker state, mTLS
    identity check, load-balancing choice among healthy upstream
    instances) at the proxy layer, invisibly to the app code on
    either end
  - analogy: a phone network's switching layer. Callers dial a name,
    not a route; the switches decide the actual path and can reroute
    around a bad line without either caller knowing it happened

        |   (dynamic config, pushed, not requested per-call)
        v
xDS control plane (Listener Discovery Service, Route Discovery
Service, Cluster Discovery Service, Endpoint Discovery Service,
Secret Discovery Service)
  - job: hold the single source of truth for who is allowed to talk
    to whom, which hosts are currently healthy, what the current
    retry and circuit-breaking policy is, and push that down to
    every sidecar in the fleet within seconds of a change, with no
    sidecar restart required
  - analogy: Day 47's Quicksilver, aimed at service-to-service
    policy instead of firewall rules: a bulletin pushed to every
    doorman's booth at once, not a rulebook each doorman has to
    remember to go re-read

        |
        v
Observability pipeline: stats, distributed traces, and access logs,
emitted by every sidecar on every hop, aggregated centrally
  - job: because every request already passes through a proxy on
    both ends, per-service-pair latency, error rate, and retry count
    become a byproduct of normal operation, not something each team
    has to remember to instrument
  - analogy: a phone company that already logs call duration and
    drop rate on every line, because the switch sees every call
    anyway, not because each caller filled out a form afterward
```

---

## 4. The transferable mechanisms

- **Move cross-cutting network logic out of the process and next to it, not inside it.** The sidecar pattern puts retries, timeouts, circuit breaking, mTLS, and load balancing in a separate process that every app instance talks to over localhost, instead of a library linked into the app. This is what makes the mechanism language-agnostic: a Python service and a Go service run identical sidecars and get identical behavior, something no shared library, JVM-based or otherwise, can offer across languages by construction. Envoy specifically was designed from the ground up for this out-of-process model, in contrast to Twitter's Finagle and Netflix's Hystrix, both in-process JVM libraries built a few years earlier for the same underlying problem.

- **Push policy to every node from one control plane, don't let each node poll for it.** The xDS protocol (Listener, Route, Cluster, and Endpoint Discovery Services) lets Envoy's control plane change routing, retry policy, or which hosts are healthy across every sidecar in a fleet within seconds, without restarting a single proxy. This is Day 47's Quicksilver mechanism again, aimed at service-to-service policy instead of firewall rules: config a system needs to react to instantly has to already be sitting in memory at the edge that needs it, delivered by a stream, not fetched on demand.

- **Circuit break and eject unhealthy hosts before they get more traffic, don't just retry harder.** Circuit breaking caps how much concurrent traffic, and how many retries, a caller is allowed to send toward a struggling dependency, failing fast once a threshold is crossed instead of piling on. Outlier detection goes further and actively removes a misbehaving upstream instance from the load-balancing pool based on its own error rate, the way a phone switch stops routing calls to a line that keeps dropping them. Both are explicitly the fix for the retry-storm death mode from Section 2: they refuse to let a caller's good intentions turn into a denial-of-service attack on its own dependency.

- **Cap retries with a budget, not a fixed count per call site.** A naive "retry three times" policy, multiplied across every caller of a struggling service, can turn a 10 percent failure rate into a full outage through sheer amplification. A retry budget instead limits the total volume of retries a service is allowed to send as a percentage of its normal traffic, so retries can absorb the occasional blip without being able to multiply an overload into a collapse. Lyft's own guidance internally is to circuit-break retries aggressively for exactly this reason: allow retries for sporadic failures, but hard-cap the total retry volume so it structurally cannot explode.

- **Make identity and encryption a property of the network, not something every app implements.** Mutual TLS between sidecars means every service-to-service call is authenticated and encrypted by default, enforced at the proxy layer, without any application code calling a crypto library. A service written by a team that never thought about transport security still gets it, the same way every apartment gets a locked door whether or not the tenant personally installed one.

- **Get observability as a side effect of the architecture, not as an extra task.** Because every call already passes through a sidecar on both the sending and receiving side, per-hop latency, error rate, and retry counts are something the mesh already knows, uniformly, in one format, across every language in the fleet. This is what makes the answer to "what is A doing to B right now" available during an incident instead of requiring an archaeology dig through five different logging conventions.

---

## 5. The trade-offs

**Latency vs. uniform safety, paid per hop.** A sidecar adds an extra network hop, app to local sidecar to remote sidecar to remote app, on every service-to-service call, even though the sidecar-to-app leg is over localhost. Industry measurements of Envoy-based meshes commonly report this overhead in the sub-millisecond range, on the order of roughly two tenths of a millisecond added at the 90th and 99th percentiles; treat the exact figure as an approximate, workload-dependent industry estimate rather than a fixed constant, since it depends heavily on request size, sidecar CPU headroom, and mesh configuration. That cost is paid on every single call, whether or not that particular call ever needed a retry or a circuit breaker to trip, in exchange for every call getting the same safety guarantees without asking the app to opt in.

**Resource cost vs. consistency.** Every service instance now runs two processes instead of one: the app, and its sidecar, each with its own CPU and memory footprint, sometimes called the "sidecar tax." At Lyft's scale, hundreds of services times whatever replica count each runs, that is a meaningful, ongoing infrastructure cost, accepted because the alternative, N teams independently building and maintaining N versions of the same safety logic, is a larger and less visible cost, paid in outages instead of compute bills.

**Operational complexity vs. blast radius reduction.** A service mesh introduces a new piece of infrastructure, the control plane, that did not exist before, and it needs to be reliable itself: if it disappears entirely, sidecars are generally designed to keep running on their last-known-good configuration rather than fail closed, the same "serve stale rather than serve nothing" choice Day 68's feature-flag lesson describes for its SDKs. But a broken push from the control plane, misconfigured routing rules pushed everywhere at once, is now a new way to cause a fleet-wide incident that did not exist when each service configured itself independently. The mesh trades many small, isolated failure modes for fewer, larger ones, betting that centralizing the logic is easier to get right and to fix quickly than getting N independent copies right.

**Below a certain N, the sidecar tax is not worth paying.** A service mesh earns its cost specifically because of the N-services-times-M-languages multiplication described in Section 1. A team with three services in one language does not have this problem: a single shared library, in that one language, solves the consistency question without the extra process, the extra hop, or the extra control plane to operate. Lyft did not start with Envoy: it started with per-language solutions and reached for a mesh only once the number of services and the number of languages had both grown large enough that library-based consistency was no longer realistically maintainable. Adopting the heavier mechanism before that point is optimizing for a scale that has not arrived yet.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is the **retry storm**, the same shape as a thundering herd, but generated from inside the system by its own callers instead of by an external spike. The loop runs: a downstream service slows down under some ordinary load increase, its callers' requests start timing out, each caller retries the failed request believing it is being a good citizen, the retries land on the same already-slow service as additional load, that additional load makes the service slower still, which produces more timeouts, which produces more retries. Nothing in this loop requires an attacker, a bug, or bad luck beyond an ordinary load spike; it is a structural property of "retry on failure" applied without a limit, and it converts a service that could have recovered on its own into one that cannot recover at all until callers stop hammering it.

The naive fix, telling every team to "please add exponential backoff to your retries," does not break this loop, it only slows down how fast it forms, because it still leaves the decision to retry, and how aggressively, scattered across every caller's own code, in whatever language that caller happens to be written in, with no way to change the policy fleet-wide in the middle of an incident. The senior fix is structural, the same shift Day 13's backpressure lesson insists on: move the decision out of each individual caller and into a shared layer that can see the aggregate picture, a circuit breaker that trips based on the callee's actual health, a retry budget that caps total retry volume regardless of how many individual callers exist, and outlier detection that removes a struggling instance from rotation before its callers pile further load onto it. None of those mechanisms are "add more capacity"; they are all versions of "stop the system from doing the thing that is making the problem worse," enforced at a layer no single team's code has to remember to implement correctly, in a language nobody can forget to use, because it isn't inside their code at all.

---

## Sources

- [Scaling productivity on microservices at Lyft (Part 3): Extending our Envoy mesh with staging overrides, eng.lyft.com](https://eng.lyft.com/scaling-productivity-on-microservices-at-lyft-part-3-extending-our-envoy-mesh-with-staging-fdaafafca82f): source for Lyft's transition from a monolith to hundreds of microservices and the framing of Envoy as the solved-problem layer for cascading failure and accidental internal denial-of-service between services; direct fetch of eng.lyft.com was blocked by this session's network egress policy, details drawn from search-indexed excerpts.
- [5 years of Envoy OSS, mattklein123.dev](https://mattklein123.dev/2021/09/14/5-years-envoy-oss/): Matt Klein's own retrospective, source for joining Lyft in May 2015 with 30-plus services already split from the monolith, Envoy reaching full deployment across the edge and service-to-service mesh by early summer 2016 handling millions of requests per second across over a hundred services, and the framing of Lyft's polyglot culture (multiple languages, no single shared library being practical) as the direct motivation for an out-of-process proxy instead of an in-process client library; accessed via search-indexed excerpt.
- [What Powers Lyft? A Deep Look At Lyft Tech Stack, Appscrip blog](https://appscrip.com/blog/lyft-tech-stack-and-infrastructure/): source for the PHP-monolith-to-Python-and-Go-microservices migration completing in late 2018, the eventual 300-plus microservice count, and Envoy's first deployment sitting next to the PHP monolith as a replacement for HAProxy before growing into a full mesh.
- [Introducing Hystrix for Resilience Engineering, Netflix TechBlog, November 2012](http://techblog.netflix.com/2012/11/hystrix.html): primary source for Netflix's own framing of cascading failure, that a system can grind to a halt from a single unhealthy dependency even when every other dependency is healthy, and for Hystrix as the JVM-library-based predecessor approach to the same underlying problem Envoy later solved at the proxy layer instead.
- [Envoy proxy circuit breaking documentation, envoyproxy.io](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking): primary documentation source for circuit breaking and outlier detection as implemented mechanisms in Envoy, and for Lyft's stated internal guidance to circuit-break retries aggressively so sporadic failures can still be retried while total retry volume is capped.
- [xDS Deep Dive: Dissecting the "Nervous System" of the Service Mesh, dev.to](https://dev.to/kanywst/xds-deep-dive-dissecting-the-nervous-system-of-the-service-mesh-3m5i): source for the xDS protocol family (Listener, Route, Cluster, Endpoint, and Secret Discovery Services) as the mechanism by which Envoy's control plane pushes configuration changes across a fleet of sidecars within seconds without a restart.
- [The Sidecar Pattern: Why Every Major Tech Company Runs Proxies on Every Pod, Medium](https://lukasniessen.medium.com/the-sidecar-pattern-why-every-major-tech-company-runs-proxies-on-every-pod-8138d79c597a): source for the general sidecar-per-instance architecture description and the commonly cited sub-millisecond per-hop latency overhead figure (roughly 0.18 ms at p90, 0.25 ms at p99) used in Section 5, treated as an approximate, workload-dependent industry estimate rather than a fixed measured constant.
- [How we built Pingora, the proxy that connects Cloudflare to the internet, blog.cloudflare.com](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/): source for Cloudflare's Pingora, a purpose-built Rust proxy serving more than one trillion requests a day, replacing an NGINX-based architecture and cutting CPU usage by roughly 70 percent and memory usage by roughly 67 percent for equivalent traffic, used in this lesson's stack-relevance note as the large-scale analogue of a dedicated proxy layer that Rare.lab already sits behind via Cloudflare; direct fetch was blocked by this session's egress policy, details drawn from search-indexed excerpts and secondary write-ups.
- [Envoy joins the CNCF, eng.lyft.com via search index / SDxCentral coverage](https://www.sdxcentral.com/news/envoy-joins-kubernetes-prometheus-with-a-cncf-diploma/): source for Envoy's November 2018 graduation from the Cloud Native Computing Foundation, alongside Kubernetes and Prometheus, faster than the norm for graduated projects at the time.
- Day 13 (this ledger, backpressure and load shedding), Day 39 (distributed tracing and sampling at scale), Day 47 (Cloudflare Quicksilver and config distribution), Day 68 (feature flags and progressive rollout): the ledger's own prior lessons this one builds directly on, for the structural loop-breaking framing applied here to retry storms, the per-hop observability problem a mesh solves as a byproduct, the push-not-poll control plane mechanism reused for xDS, and the fail-static default reused for a sidecar's behavior when its control plane is unreachable.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of eng.lyft.com and blog.cloudflare.com, so the figures drawn from those domains rest on search-indexed excerpts and secondary summaries rather than a full read of the original posts. The core numbers this lesson leans on hardest, Lyft's 30-plus services in 2015 growing past 300 by 2018, the PHP-to-Python-and-Go migration, and Envoy's role as a language-agnostic replacement for JVM-only libraries like Hystrix and Finagle, are corroborated across multiple independent sources (Matt Klein's own retrospective, Lyft's engineering blog as indexed, and independent tech-stack write-ups) and are treated as solid. The per-hop latency overhead figure and the exact Pingora efficiency percentages rest on fewer, secondary sources and are treated as labeled industry estimates rather than figures verified against a primary benchmark.
