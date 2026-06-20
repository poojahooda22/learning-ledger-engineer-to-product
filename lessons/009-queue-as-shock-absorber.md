# Day 9: How a Queue Keeps a System Alive When 14 Million Users Show Up at Once

**Date:** 2026-06-20
**Difficulty:** Intermediate-Advanced
**Stack relevance:** Cloudflare Queues, Cloudflare Workers, Supabase Postgres, Redis/Upstash, async job processing, Rare.lab compile pipeline

---

## 1. Named company + the breaking number

**Ticketmaster, Taylor Swift Eras Tour presale, November 15 2022.**

Fourteen million people tried to buy tickets in a single presale window. Ticketmaster's systems collapsed for hours. Millions of verified fans, who had already passed a pre-verification step, were dropped from virtual queues mid-purchase or shown error pages.

**The breaking number: 14 million concurrent sessions arriving inside a 10-minute window onto checkout infrastructure sized for roughly 100,000 concurrent users. That is a 140x spike.**

Second reference point: **Shopify Black Friday 2021** peaked at 40 million requests per minute (roughly 667,000 per second) against their checkout service. They handled it. Ticketmaster did not. The architectural difference: Shopify put a durable queue between the user's click and the database write. Ticketmaster wrote directly to the database.

---

## 2. Why the naive (demo) design dies

**The naive version:** one web server, one Postgres database. User clicks "Buy." App server opens a DB connection, runs a transaction (check inventory, charge card, write order, decrement seats), returns 200 OK.

At 500 users per second this works fine. At 23,000 users per second (14 million over 600 seconds), three things break simultaneously.

**Break 1: Database connection pool exhausts.**
Postgres default max connections is 100 to 500. Each checkout transaction holds a connection open for 200 to 500ms (inventory read, card auth network round trip, order write). At 23,000 req/sec, you need 23,000 times 0.3 seconds = 6,900 simultaneous connections. You have 200. The other 6,700 threads sit blocked. The app server runs out of threads. New requests get "connection refused." The site is down.

**Break 2: The thundering herd begins.**
When users see an error, they hit refresh. Now 14 million people are each refreshing every 5 seconds. That is not 14 million requests. That is 14 million times 12 per minute = 168 million requests per minute. The retry amplifies the original spike by 12x. Load rises. More errors. More retries. Positive feedback. The system has no escape from this loop.

**Break 3: The inventory row becomes a hot key.**
Every checkout must touch the same "remaining seats" counter for a popular section. In Postgres that is one row with a SELECT FOR UPDATE / UPDATE seats SET count = count - 1. Under 23,000 concurrent writes, serialization waits stack up. P99 latency goes from 50ms to 30 seconds. Timeouts fire. Retries pile on top of the existing queue of lock-waiters. The spiral accelerates.

**The root cause is coupling.** The rate at which users arrive is directly and synchronously coupled to the rate at which the database writes. No slack exists between them.

---

## 3. The architecture: top-to-bottom, layer by layer

```
[14 million users]
        |
        v
[Virtual Waiting Room / Rate Gate]     -- Metered entry: 5,000 users/sec past this point
        |
        v
[CDN / Edge (Cloudflare)]              -- Serves static checkout shell from 200+ PoPs
        |
        v
[Load Balancer]                        -- Distributes POST /checkout across app nodes
        |
        v
[Stateless App Servers (N nodes)]      -- Validates token, enqueues job, returns 202 Accepted
        |                                 (This takes 5ms. No DB touched.)
        v
[Message Queue (SQS / Kafka / Cloudflare Queues)]  -- THE SHOCK ABSORBER
        |   Queue depth grows during spike; drains slowly as workers consume
        v
[Worker Pool (auto-scaled)]            -- Pulls jobs at the rate the DB can absorb
        |
        v
[Redis / Upstash Cache]                -- Atomic inventory counter; idempotency key store
        |
        v
[Database Primary (Postgres / DynamoDB)]  -- Receives writes at controlled, bounded rate
        |
        v
[Read Replicas]                        -- Order-status reads; leaderboard queries
```

**Each layer and its single job:**

**Virtual Waiting Room.** The bouncer at a concert. It issues numbered tokens to every user who shows up. Only N tokens per second are allowed past. Everyone else sees a live "position 47,823 in queue" counter. This converts a thundering herd into a smooth, metered stream. Implemented as a Redis sorted set: key = user token, score = join timestamp. Position = ZRANK(key). Pop the front at a fixed tick rate. The user sees their number count down. This predictability eliminates rage-refreshes, which dramatically reduces amplified load.

**CDN / Edge.** Serves the static HTML, CSS, and JavaScript for the checkout page from 200+ global edge nodes. The React shell never hits your origin server. Only POST /checkout and GET /order/:id hit origin. This alone cuts 85 to 90% of your total request volume.

**Stateless App Server.** Does exactly three things: (1) validate the waiting-room token, (2) check user-level rate limit, (3) write one message to the queue and return HTTP 202 Accepted. No database write. No card charge. Done in 5ms. The server can now handle 50,000 requests per second without strain because each request is trivially cheap.

**Message Queue.** The central insight of this entire lesson. A queue decouples arrival rate from processing rate. During the spike, the queue fills up. That is not a failure. A full queue is not a crashed database. SQS stores messages durably across three Availability Zones. Kafka persists to disk. Cloudflare Queues replicates across regions. The queue holds jobs safely and releases them at the rate downstream can absorb. Queue depth is your shock absorber: it grows fast, it drains slowly, and nothing downstream crashes because nothing downstream ever sees the spike.

**Worker Pool.** A fleet of workers (Lambda functions, ECS tasks, Go goroutines, Cloudflare Workers) that pull from the queue. Each worker does the real work: atomically decrement inventory in Redis, run the card charge, write the order to Postgres. Workers autoscale based on queue depth. Queue depth over 10,000 messages: add 50 workers. Queue depth under 100 messages: scale back to 5 workers. This elasticity means you pay for compute only during the spike, not always.

**Redis Cache.** Inventory counts live in Redis, not in Postgres. A Redis DECR on a counter is atomic and handles 100,000 operations per second on a single node. No locks. No transactions. No contention. The logic is: DECR seats:section_A. If the returned value is negative, INCR back and return SOLD_OUT. This replaces the entire "SELECT FOR UPDATE / UPDATE" pattern in Postgres that was your hot key problem.

**Database Primary.** Now only receives order writes at the rate workers process jobs. At 2,000 workers doing 10 jobs per second each = 20,000 writes per second. With PgBouncer connection pooling in front of Postgres, this is manageable. Primary for writes. Read replicas for the order-status polling that users do while waiting.

---

## 4. The transferable mechanisms

**Queue as shock absorber (accept-fast, process-slow).** Any system where arrival rate can spike above processing rate needs a buffer. The queue is that buffer. You accept the request immediately (5ms), acknowledge it (202 Accepted), and process it asynchronously at whatever rate is safe downstream. Used by: Stripe (payment jobs), SendGrid (email delivery), YouTube (video transcoding), LeetCode (code execution -- the video transcript example above shows exactly this pattern: queue between API server and containers). If the user does not need the result within the same HTTP response, use a queue.

**Atomic counter in Redis instead of a locked row in Postgres.** Redis DECR is single-threaded and atomic by design. It returns the new value. If new value is below 0, INCR back and reject. This replaces "SELECT FOR UPDATE" under high concurrency and is 100x faster per operation. Pattern: any counter that many writers touch simultaneously (inventory, rate limit tokens, concurrent connection count, free-tier quota) belongs in Redis or a KV store, not a relational row.

**Idempotency key.** A user might click "Buy" twice. The network might drop the response. The worker might crash mid-charge and retry. Without idempotency, the user gets charged twice. The fix: before enqueueing, the app server generates a UUID (the idempotency key). The worker stores this key in Redis with a 24-hour TTL before starting work. If the same key arrives again (from a retry), return the cached result immediately and skip all processing. This is the exact pattern Stripe uses on every payment endpoint and that was covered in the Day 6 Stripe lesson. It composes cleanly with queues.

**Virtual waiting queue (metered entry).** Turning a spike into a stream. A Redis sorted set lets you serve position updates in O(log N) and pop users in join order in O(1). The gate allows N users per second through. Users see live queue position via short-poll every 3 seconds or WebSocket push. Predictability reduces user abandonment and rage-refreshes. This is the engineering version of the Disney FastPass system: you know you will wait, you know how long, so you stop trying to cut the line.

**Exponential backoff with jitter on retries.** When a request fails, the client should not immediately retry. Wait 1 second, then 2 seconds, then 4 seconds, with plus or minus 25% random jitter. Jitter breaks synchrony. Without jitter, all 14 million clients who timed out at the same moment retry at the same moment. With jitter, retries are smeared across a time window. Marc Brooker's 2015 AWS paper proves this reduces collision probability from O(N squared) to roughly O(N). Every SQS consumer SDK implements this by default. You should too.

**Dead-letter queue (DLQ).** Every queue should have a DLQ. If a job fails 3 times (card processor down, worker crashes, inventory reservation expired), move the message to the DLQ instead of retrying forever or dropping it silently. Workers monitoring the DLQ can: send the user an email ("Sorry, your order could not be completed"), issue a refund if a partial charge happened, or page on-call if the failure rate is high. Without a DLQ, silent data loss is almost inevitable at scale.

---

## 5. The trade-offs

**Consistency vs. availability.**

The queue makes this an AP system (Available, Partition-tolerant) in CAP terms. You trade some consistency for availability.

What "some consistency" means concretely: when the user clicks "Buy" and gets 202 Accepted, they do not have a confirmed order. The order might still fail (inventory ran out before the worker got to the job; card declined). The user must poll /order/:id or receive a webhook callback. They cannot assume success from the 202.

The alternative (synchronous checkout that holds the HTTP connection open until the DB write succeeds) is CP: consistent, but unavailable under spike. The thread pool exhausts. The site goes down.

**Per data type, which trade-off is made:**

| Data type | Consistency choice | Why |
|-----------|-------------------|-----|
| Inventory count | Optimistic (DECR then verify) | Redis atomic; over-sell risk is minimal and recoverable |
| Order record | Strong (worker commits to DB before dequeuing) | Never lose an order |
| Order status visible to user | Eventual (poll or webhook) | Acceptable for purchase; user expects a few seconds |
| Queue job itself | At-least-once delivery (SQS default) | Idempotency key prevents double-processing |

**Cost vs. latency.** The queue adds end-to-end latency. Synchronous checkout: 2 to 5 seconds. Async via queue: 10 to 60 seconds. This is acceptable for a ticket purchase (user expects to wait) but not for a search result (user expects under 200ms). Only use async queues for operations where the user can tolerate a wait or can be notified later via push or email.

**Worker scaling cost.** A spike of 1 million jobs might spin up 1,000 Lambda functions for 10 minutes. At AWS Lambda pricing (roughly $0.0000166 per GB-second at 512MB), a 10-second job costs $0.0000833. One million jobs = $83. Acceptable for the elasticity. But if queue depth stays permanently high, your base worker count is wrong. Tune the steady-state worker pool to your average throughput, not to the peak.

---

## 6. The systems-thinking lens: the retry death spiral

**The feedback loop that kills systems without backpressure:**

```
DB is slow (10s latency, hot row contention)
  --> User request times out after 30s
    --> Client retries automatically
      --> More retries hit the already-slow DB
        --> DB gets slower (more lock contention)
          --> More timeouts
            --> More retries
              --> DB falls over
                --> App servers get 500s
                  --> Load balancer removes unhealthy nodes
                    --> Remaining nodes get 2x traffic
                      --> They fall over too
```

This is a **retry death spiral**, a specific instance of a metastable failure. The system has two stable states: "working fine" and "completely down." Once load pushes past a tipping point, the positive feedback loop (failures cause retries cause more failures) drives the system irreversibly to the bad stable state. Adding more database nodes does not help because the additional capacity attracts more retries proportionally.

**The senior engineer's approach breaks the loop before it starts.**

Three interventions, each targeting a different part of the loop:

**Backpressure at the gate.** The virtual waiting room caps the arrival rate at N req/sec regardless of how many users are hammering the front door. The DB never sees the spike. The loop cannot begin. This is the most important intervention because it is proactive, not reactive.

**Jitter on client retries.** If the gate is breached (something upstream fails and requests get through), jitter prevents synchronized retry waves. The spiral stays kinetic instead of resonant. Resonance (all 14 million clients retrying in the same second) is what causes collapse. Jitter detunes the resonance.

**Circuit breaker.** After N% of requests fail within a 10-second window, the circuit breaker opens: all new requests fail immediately with 503, no work attempted. Workers, caches, and databases get breathing room to drain their queues and recover. The circuit probes with a single request after a cool-down period. If the probe succeeds, the circuit closes. This is the Hystrix pattern from Netflix and is now native in Envoy proxy and AWS App Mesh. The circuit breaker breaks the positive feedback by stopping the retries that are amplifying the failure.

**The structural lesson.** Capacity is not the fix for a metastable failure. Adding more DB nodes during a retry death spiral does not help; more nodes accept more retries, fill up, and crash. The fix is to make the feedback loop impossible: rate-limit at entry, jitter on retry, circuit-break on failure. The queue is the mechanical implementation of all three: it bounds the processing rate (backpressure), holds jobs safely while things recover (slack), and isolates the database from the arrival rate (decoupling).

---

## 7. The lesson for Rare.lab

Rare.lab runs Supabase Postgres with RLS, Cloudflare R2 for content-addressed scene JSON, and an embeddable runtime with a shared WebGL context.

**What you already do right:**
- Cloudflare handles CDN. Static assets (JS, CSS, WASM runtime) are served from edge PoPs. Origin never sees GET traffic at scale.
- R2 stores immutable, content-addressed scene JSON. The same scene hash is never written twice. This is already idempotent storage.
- Supabase includes PgBouncer connection pooling, which protects Postgres from connection exhaustion.

**Your next ceiling:**
When Rare.lab launches publicly (a Product Hunt feature, a press mention, or a viral tweet), a wave of 5,000 users simultaneously saving shader graphs and triggering compile jobs will hit your Cloudflare Workers. Each compile job is CPU-intensive and takes 1 to 5 seconds. Running those synchronously inside a Worker will hit the 50ms CPU wall and return timeouts. Users will retry. You know how this story ends.

**The pattern to apply right now:**
1. Use Cloudflare Queues (in GA, free tier: 1 million messages per month) between the save endpoint and the compile worker.
2. POST /scenes/:id/compile returns 202 Accepted with a job ID immediately.
3. A Queue consumer Worker runs the compile job, writes the artifact to R2, updates the job status in Supabase.
4. Client polls GET /jobs/:id every 2 seconds (or you push via a Supabase Realtime subscription on the jobs table).
5. The artifact key in R2 is a content hash of the input graph. If the same graph is submitted twice, the idempotency check hits R2 and returns the cached artifact. Zero recompute.

**The atomic counter pattern also applies.** If you add a free-tier compile quota (e.g., 50 compiles per day per user), store that counter in Cloudflare KV or Upstash Redis, not in a Supabase row. Row contention on a shared quota counter is one of the most common Supabase scaling mistakes at launch.

---

## References (with summaries)

Every reference below is summarized so you can decide what to read first and why.

---

### Engineering blogs and papers

**1. "Exponential Backoff and Jitter"**
Marc Brooker, AWS Architecture Blog, 2015
https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/

This is one of the most important short papers in distributed systems. Brooker shows with math and simulation that pure exponential backoff still causes synchronized retry storms because all clients backed off for the same duration and wake up together. Adding random jitter (a uniform random number within the backoff window) breaks the synchrony. The paper proves "full jitter" (random between 0 and cap) reduces total request collisions from O(N squared) to roughly O(N). Short read, 10 minutes. Includes working pseudocode. After reading this you will never write a retry loop without jitter again.

**2. "The Log: What every software engineer should know about real-time data's unifying abstraction"**
Jay Kreps, LinkedIn Engineering, 2013
https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying

This is the essay that motivated Kafka. Kreps argues that the append-only log is the single data structure behind databases, distributed systems, stream processing, and message queues. He explains how LinkedIn replaced their messy point-to-point internal queue infrastructure with Kafka, a single unified log where every consumer reads at its own offset. The key insight: a log does not disappear after one consumer reads it (unlike traditional queues). Every consumer gets the full history. This enables replayability, multiple consumers from the same stream, and audit logs for free. Long read (10,000 words) but dense. Read section 1 and 3 if time is short.

**3. "Circuit Breaker"**
Martin Fowler, martinfowler.com, 2014
https://martinfowler.com/bliki/CircuitBreaker.html

Fowler's definitive short post on the circuit breaker pattern. He uses the analogy of an electrical circuit breaker: when current exceeds safe levels, the breaker trips. Unlike a fuse (one-time, throws the request away), a breaker resets after a cool-down. Fowler defines the state machine: Closed (pass requests) -> Open (fail immediately, no work attempted) -> Half-Open (probe: send one request to test if downstream recovered). 10-minute read. The pattern is now in every service mesh (Istio, Envoy) and every major framework (Resilience4j for Java, Polly for .NET). After reading this you will understand why Kubernetes liveness probes and load balancer health checks are not enough by themselves.

**4. "Designing Data-Intensive Applications"**
Martin Kleppmann, O'Reilly, 2017 (book, not free but the best single engineering book of the last decade)
https://dataintensive.net/

Chapter 11 covers stream processing and message queues in depth. Chapter 12 covers the future of data systems. Chapter 9 covers consistency and consensus. Most relevant chapter for this lesson: Chapter 11, specifically the section "Messaging Systems" (pages 443 to 464 in the hardcover). Kleppmann explains exactly-once delivery, at-least-once delivery, consumer groups, and log compaction. He covers Kafka and RabbitMQ side by side. If you own one engineering book, own this one.

**5. "Shopify's Black Friday Cyber Monday Infrastructure" (yearly engineering retrospectives)**
Shopify Engineering Blog
https://shopify.engineering (search "BFCM" or "infrastructure flash sale")

Shopify publishes a post-mortem/retrospective after every Black Friday. The 2021 and 2022 posts are the most detailed. Key facts: 40 million requests per minute peak (2021), 100% of inventory operations go through Redis counters (not Postgres rows), all checkout is async (202 Accepted, poll for result), all infrastructure is pre-scaled before the known spike window. These posts are honest about what failed in earlier years (2014: the database was the bottleneck; 2016: checkout timeouts; 2019: first clean run). Reading the failure years alongside the success years shows exactly which architectural decisions made the difference.

---

### YouTube videos

**6. "Message Queues: From Zero to Scale"**
Gaurav Sen, YouTube
Search: "Gaurav Sen message queue" on YouTube

Gaurav Sen is the clearest system design teacher on YouTube. This video (approximately 18 minutes) walks through why queues exist, the difference between push (broker delivers to consumer) and pull (consumer fetches from broker), when to use RabbitMQ vs Kafka vs SQS, dead-letter queues, visibility timeouts, and consumer groups. He draws each concept as a diagram while explaining it. Watch this before reading the Kleppmann chapter. The visual foundation makes the book much easier to absorb.

**7. "Apache Kafka in 100 Seconds"**
Fireship, YouTube
Search: "Fireship Kafka 100 seconds" on YouTube

A 2-minute high-density visual explainer of Kafka. The key distinction Fireship draws: traditional queues delete messages after a consumer reads them (like a vending machine). Kafka keeps messages in a log for a configurable retention period; every consumer reads at its own offset (like subscribing to a newspaper). Multiple consumers can read the same log independently without interfering. This distinction matters when you want analytics, replay, or multiple downstream services consuming the same event stream. Watch this before the Gaurav Sen video if you have never heard of Kafka.

**8. "System Design: Ticketmaster at Scale (Taylor Swift Case Study)"**
ByteByteGo, YouTube
Search: "ByteByteGo Ticketmaster scale" on YouTube

Alex Xu (author of System Design Interview volumes 1 and 2) does a 12-minute breakdown of exactly what failed at Ticketmaster in November 2022 and what the correct architecture looks like. He covers: virtual waiting room design, inventory reservation with TTL (reserve a seat for 10 minutes while the user completes checkout), async order processing via queue, and the hot-key problem on popular sections. The animation quality is high and the architecture diagram is cleaner than anything in this lesson. Strongly recommended as a companion to reading this lesson.

**9. "Everything You Need to Know About Message Queues"**
Hussein Nasser, YouTube
Search: "Hussein Nasser message queue" on YouTube

Hussein Nasser is a backend engineer who teaches deeply practical content. This video (approximately 40 minutes) is longer than the others but covers things the shorter videos skip: the difference between synchronous and asynchronous consumers, how message ordering is guaranteed (or not) across partitions, what happens when a consumer crashes mid-processing (visibility timeout and re-delivery), and how to handle poison messages (messages that cause the consumer to crash every time). The poison message problem is what makes the DLQ essential. After this video, queue design patterns will feel like second nature.

**10. "Distributed Systems in One Lesson" (keynote)"**
Tim Berglund, GOTO Conferences, YouTube
Search: "Tim Berglund distributed systems one lesson GOTO" on YouTube

A 45-minute conference keynote that explains the CAP theorem with physical analogies (a whiteboard in a distributed classroom). Berglund explains why you cannot have consistency, availability, and partition tolerance simultaneously, and walks through what real systems like Cassandra, HBase, and DynamoDB chose and why. Directly relevant to the trade-offs section of this lesson. The whiteboard classroom analogy makes CAP click in a way that no diagram I have seen does. Free on YouTube.

---

### Articles (Medium, Stack Overflow, and blogs)

**11. "Building Scalable Systems: The Role of Message Queues" (AWS Developer Guide)**
https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html

The AWS SQS documentation is unusually good. The concepts guide (not the API reference) explains visibility timeouts, message retention periods, dead-letter queues, long polling vs short polling, and FIFO queues vs standard queues. Key paragraph to find: the section on "Visibility Timeout" explains exactly how at-least-once delivery works and why your consumer must delete a message only after successful processing. If you delete before processing and the worker crashes, the job is lost. The docs are dense but everything is there.

**12. "Little's Law: The one thing every engineer should know about queues"**
Various authors on Medium (search "Little's Law queues engineering")

Little's Law states: L = lambda times W, where L is the average number of items in the queue, lambda is the average arrival rate, and W is the average time an item spends in the system. This equation tells you how to size your worker pool. If you expect 10,000 jobs per minute (lambda = 167/sec) and each job takes 2 seconds (W = 2), you need L = 167 times 2 = 334 jobs in the system at any moment. To keep queue depth bounded, you need 334 workers (or workers capable of 334 concurrent jobs). The math is simple but most engineers do not use it. Every capacity planning conversation about queues should start here.

**13. Stack Overflow: "When should I use a message queue instead of a database?"**
https://stackoverflow.com/questions/1322842

A classic question with a practical top-voted answer that engineers bookmark. The key rule from the top answer: use a queue when (a) the caller does not need the result within the same HTTP response, (b) the callee might be temporarily unavailable, or (c) the processing rate must be decoupled from the arrival rate. The thread also covers when NOT to use a queue: when you need the result to render the next screen (user clicks "search" and expects results instantly), when ordering of results matters but the queue is not FIFO, or when message latency would make the UX feel broken. Short thread, worth reading the top 3 answers.

**14. "How Discord Stores Trillions of Messages"**
Discord Engineering Blog, 2023
https://discord.com/blog/how-discord-stores-trillions-of-messages

Discord processes roughly 4 billion messages per day. The blog post is about their move from Cassandra to ScyllaDB for message storage, but the most relevant section for this lesson is how Kafka sits in front of the message store. Every message sent by a user flows: client -> API server -> Kafka topic -> ScyllaDB consumer workers. The Kafka layer absorbs burst writes (a streamer announces a raid to 50,000 viewers who all type in chat simultaneously) and ensures ordered delivery to ScyllaDB. The post is honest about what failed (Cassandra hot partitions on high-traffic servers) and why the pattern of queue-before-database solved the problem. Discord is also a good model for Rare.lab because both operate at a mix of small casual usage and sudden massive bursts.

**15. "Real-Time Ticket Sales System Design" (community discussion)**
Reddit r/ExperiencedDevs and r/SystemDesign
Search: "Ticketmaster system design queue" on Reddit

Several long Reddit threads, posted in November 2022 immediately after the Ticketmaster collapse, contain working engineers explaining what likely went wrong technically and what they would have done differently. These threads are valuable because they are unpolished opinions from practitioners, not marketing. Common themes: virtual queues need push updates (polling at 14 million users per second is itself a DDoS), inventory should be reserved (not just checked) at queue entry time with a TTL, and the checkout flow should be a state machine with a dedicated seat-hold step before payment. Sorted by "top" you get the highest-signal answers near the top.

---

### Research papers (for going deeper)

**16. "Kafka: A Distributed Messaging System for Log Processing"**
Jay Kreps, Neha Narkhede, Jun Rao, LinkedIn, SIGMOD 2011
Available on the ACM Digital Library and via search for "Kafka LinkedIn paper 2011"

The original Kafka paper. 6 pages. Explains the design choices that make Kafka different from traditional message brokers: sequential disk I/O (faster than random I/O for a log), zero-copy data transfer (sendfile syscall bypasses user space), consumer-managed offsets (broker does not track what each consumer has read), and partition-based parallelism. The numbers from 2011: 50MB per second write throughput, 600MB per second read throughput on a single broker. The architectural decisions in this 6-page paper influence every queue and stream system built since 2011.

**17. "The Tail at Scale"**
Jeffrey Dean, Luiz Andre Barroso, Google, Communications of the ACM, 2013
Search: "The Tail at Scale Dean Barroso" for a free PDF

Google engineers explain why the 99th percentile latency matters more than the median in distributed systems, and why high percentile latency is unavoidable in large systems. The key insight relevant to queues: in a system with 1,000 sequential steps, even if each step has 0.1% probability of being slow, the probability that at least one step is slow is nearly 100%. Dean and Barroso describe "hedged requests" (send the same request to two backends, take whichever responds first) and "tied requests" as latency mitigation. This paper teaches you to think about tail latency as a system property, not a bug to be fixed in one component.

---

*Next lesson: consistent hashing and sharding -- how do you distribute data across N servers so that adding or removing one server does not require reshuffling everything?*
