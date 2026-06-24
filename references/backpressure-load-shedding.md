# References: Backpressure and Load Shedding (Lesson 013, 2026-06-24)

## Engineering Blogs

- https://blog.x.com/engineering/en_us/a/2010/the-anatomy-of-a-whale — Twitter's Fail Whale post-mortem
- https://netflixtechblog.com/introducing-hystrix-for-resilience-engineering-13531c1ab362 — Netflix Hystrix introduction
- https://netflixtechblog.com/keeping-netflix-reliable-using-prioritized-load-shedding-6cc827b02f94 — Netflix prioritized load shedding
- https://netflixtechblog.com/enhancing-netflix-reliability-with-service-level-prioritized-load-shedding-e735e6ce8f7d — Service-level load shedding
- https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/ — Marc Brooker on retry/backoff/jitter
- https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/ — AWS jitter simulation results
- https://sre.google/sre-book/handling-overload/ — Google SRE Chapter 21
- https://sre.google/sre-book/addressing-cascading-failures/ — Google SRE cascading failures

## Library Docs

- https://resilience4j.readme.io — Resilience4j (circuit breaker, rate limiter, bulkhead, retry)
- https://redis.io/docs/latest/develop/use-cases/rate-limiter/ — Redis rate limiting patterns
- https://redis.io/tutorials/howtos/ratelimiting/ — Redis 5 rate limiter algorithms
- https://github.com/ReactiveX/RxJava/wiki/Backpressure — RxJava backpressure operators
- https://github.com/Netflix/Hystrix — Netflix Hystrix (maintenance mode)

## Research Papers

- https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf — Metastable Failures in Distributed Systems (HotOS 2021)
- https://sigops.org/s/conferences/hotos/2025/papers/hotos25-106.pdf — Analyzing Metastable Failures (HotOS 2025)

## Community and Practitioner Articles

- https://medium.com/@jayphelps/backpressure-explained-the-flow-of-data-through-software-2350b3e77ce7 — Backpressure explained (Jay Phelps)
- https://medium.com/helpshift-engineering/load-shedding-in-web-services-9fa8cfa1ffe4 — Load shedding in web services (Helpshift Engineering)
- https://medium.com/@Realblank/backpressure-patterns-in-go-from-channels-to-queues-to-load-shedding-0841c9fe5607 — Backpressure patterns in Go
- https://www.tedinski.com/2019/03/05/backpressure.html — Backpressure conceptual explainer
- https://binodmahto.medium.com/backpressure-and-load-shedding-in-system-design-9ee6b5dbeb7d — Backpressure and load shedding overview
- https://kalyanaj.medium.com/metastable-failures-in-distributed-systems-what-causes-them-and-3-things-you-can-do-to-tame-them-8fd56d593950 — Metastable failures explained
- https://gist.github.com/rponte/8489a7acf95a3ba61b6d012fd5b90ed3 — Little's Law and backpressure

## YouTube Channels

- https://www.youtube.com/@hnasr — Hussein Nasser (backpressure, circuit breakers, rate limiting)
- https://www.youtube.com/c/ByteByteGo — ByteByteGo (rate limiting, resiliency patterns)
- https://bytebytego.com/guides/resiliency-patterns/ — ByteByteGo resiliency patterns guide
- https://www.youtube.com/channel/UCRPMAqdtSgd0Ipeef7iFsKw — Gaurav Sen (capacity planning, load balancing)

## Incident Post-Mortems

- https://danluu.com/cache-incidents/ — Decade of cache incidents at Twitter
- https://www.infoq.com/articles/anatomy-cascading-failure/ — Anatomy of cascading failures
