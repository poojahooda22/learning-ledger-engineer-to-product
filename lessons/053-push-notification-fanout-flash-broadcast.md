# Day 53 — How does Duolingo push 6 million phone notifications inside 6 seconds, all timed to one 30-second Super Bowl ad?

**Date:** 2026-08-14
**Difficulty:** Expert
**Topic:** Flash-broadcast push notification fan-out: why a queue built for exactly-once correctness cannot also be the queue that carries the volume, why autoscaling is the wrong tool for a load spike that is over before a new server finishes booting, and how Duolingo pre-warmed a 5,000-worker fleet and split a single queue's job in two to deliver 6M+ notifications with a 95th-percentile latency of 3.9 seconds.
**Stack relevance:** Rare.lab publishes a new scene manifest to Cloudflare R2 and every embedded runtime watching that project has to learn about it. Today that is small fan-out (a handful of viewers per embed), but the same shape shows up the moment one publish event has to reach a large number of simultaneously-connected runtimes at once, a viral embed, a live collaborative session, a broadcast update to every open instance of a shader pack. The Duolingo case is what that problem looks like at its most extreme: one trigger, a hard deadline measured in single-digit seconds, and millions of independent recipients.

---

## 1. The company and the breaking number

**Duolingo**, at QCon London 2024, in a talk titled "How We Created a High-Scale Notification System at Duolingo That Delivered Millions of Messages Within Seconds During a Super Bowl Commercial Break." The setup: Duolingo ran a Super Bowl LVIII (February 2024) ad, and the marketing ask was not "send some notifications eventually." It was a synchronized moment: the instant the ad finished airing, phones across the country should buzz within seconds, so the joke lands while the TV audience is still watching and the "did your phone just do that too" reaction spreads on social media while it's still fresh. The target, reported consistently across the talk's writeups: **6 million-plus notifications delivered within 5 seconds** of the trigger. Duolingo's own previous best, from an earlier, smaller campaign, was **500,000 notifications in 60 seconds**. The new ask was roughly **12x the volume in 1/12th the time**, which is not a 12x harder problem, it's closer to a 144x jump in required throughput.

The specific number that broke the first design wasn't the 6 million. It was a queue limit. Duolingo's system needed **exactly-once delivery**: if the trigger fired twice, whether from a retry, a second system calling it independently, or an operator mis-click, no user should get two buzzes for one ad. The natural tool for that is an AWS SQS **FIFO** queue, which gives built-in deduplication over a 5-minute window. FIFO queues are also throughput-capped: **300 messages per second without batching** (roughly 3,000/second with the maximum batch of 10 messages per API call). Six million notifications at 300 messages a second is **20,000 seconds, about 5.5 hours**. Even at the batched ceiling of 3,000/second it's still 2,000 seconds, over half an hour. Against a 5-second deadline, the exactly-once queue was never going to carry the volume, no matter how much compute sat behind it.

## 2. Why the naive (demo) design dies

**Version one: one queue does both jobs.** A single SQS FIFO queue receives the trigger, deduplicates it, and feeds every downstream worker that actually sends the push. This is the natural first build, because "exactly-once" and "queue" sound like the same requirement. It fails for three separate reasons, not one.

**The correctness guarantee caps the throughput.** FIFO's ordering and dedup guarantees are exactly why it's capped at 300–3,000 messages/second: maintaining strict order and a dedup window costs coordination, and that coordination cost is what the throughput ceiling is paying for. A queue that is fast enough for 6 million messages in 5 seconds cannot also give you that guarantee for free. Trying to force one queue to do both jobs means the weaker of the two requirements, in this case raw throughput, loses.

**Autoscaling reacts to load that has already arrived.** A standard autoscaling group watches a metric (CPU, queue depth), decides more capacity is needed, launches new instances, waits for them to boot and register healthy, then routes traffic to them. That loop typically takes on the order of a minute or more, start to finish. The entire event here is over in 5 seconds. By the time an autoscaler noticed the queue depth spike and started launching instances, the ad would already be over and the moment would be gone. Reactive scaling is built for gradual, organic growth; it structurally cannot respond to a spike shorter than its own reaction time.

**A cold read at zero-hour is a self-inflicted thundering herd.** If every worker's first move, the moment the trigger fires, is to query DynamoDB for "which users get this notification," millions of near-simultaneous reads slam the database at the exact instant the system needs to be fastest. The read pattern that would normally be spread across a day gets compressed into the same one second the notifications themselves need to go out, competing for the same capacity budget.

## 3. The architecture

```
[Trigger: one marketing action, timed to the ad's air time]
   analogy: pressing the button that starts a fireworks show, not
   lighting six million individual fuses by hand
   |
   v
[API Gateway + a Python trigger service on ECS: the single entry point,
 receives the "go" signal exactly once from a human or scheduler]
   analogy: the stage manager's one cue light
   |
   v
[Queue #1 -- SQS FIFO, the CORRECTNESS layer: exists only to
 deduplicate the trigger itself (5-minute dedup window), capped at
 ~300-3,000 msg/sec, and that's fine because it only ever has to
 carry ONE logical event, not six million]
   analogy: a single sign-in sheet at the door, checked once, not
   the door itself
   |
   v
[Queue #2 -- SQS standard, the VOLUME layer: a completely separate
 queue that does the actual fan-out trigger, no ordering guarantee,
 no strict dedup, sized for 120,000 in-flight messages and pushed
 further via batching, because bulk delivery doesn't need FIFO's
 guarantees, it needs throughput]
   analogy: once the sign-in sheet is checked, the doors open on
   every aisle at once, no more one-at-a-time gatekeeping
   |
   v
[Pre-warmed worker fleet -- 5,000 ECS/ASG instances, manually
 provisioned HOURS before the event, not autoscaled in response to it]
   analogy: hiring and seating the entire stadium crew before the
   show starts, not calling more ushers in once the crowd is already
   at the gates
   |
   v
[In-memory prefetch layer -- 20 "interim" worker instances read the
 target user/device-token list out of DynamoDB ahead of time, stage
 it to S3, and load it into memory BEFORE the trigger fires, so the
 5,000 workers never touch a cold database during the burst]
   analogy: printing every guest's name badge the night before,
   instead of looking each one up at the door
   |
   v
[Push gateway workers: batch calls out to FCM (Android) and hold
 persistent HTTP/2 connections to APNs (iOS), respecting each
 platform's own rate limits rather than firing one HTTP request per
 notification]
   analogy: a mail room that bundles outgoing letters into batches
   per carrier route instead of walking each envelope to the post
   office individually
   |
   v
[FCM / APNs -- the last hop, outside Duolingo's control, delivers to
 the OS-level push service on the device, best-effort, no guarantee]
   analogy: once the letter leaves the mail room, its speed depends
   on the postal system, not on how well the mail room is run
   |
   v
[CloudWatch: real-time observability on the delivery curve as it
 happens, since there's no room to debug after the fact, the whole
 event is over in under 6 seconds]
```

Result, per the talk: roughly **800,000 requests/second** sustained through the pipeline, **95% of notifications published within 3.9 seconds**, **99% within 5.7 seconds** of the trigger firing.

The one design decision worth pulling out of the diagram: **the two queues are not redundant, they are doing different jobs on purpose.** Queue #1 answers "has this event already happened" (a question that needs a correct, unique answer). Queue #2 answers "get this payload out as fast as possible" (a question that doesn't need ordering or a dedup window at all, it needs raw throughput). Collapsing them back into one queue is the single change that would have brought the whole design back down to 300 messages a second.

## 4. The transferable mechanisms

**Separate the correctness gate from the volume pipe.** Whenever a system needs both "exactly-once" and "very high throughput," don't ask one component to give you both. Put a small, strict, low-throughput gate in front (a dedup check, an idempotency key lookup) that only has to process the *event*, and let a second, high-throughput, weaker-guarantee stage carry the *payload*. The gate protects correctness; the pipe protects speed. This generalizes past notifications: it's the same shape as validating a request once at the edge and then letting a fan-out job run unconstrained downstream.

**Pre-provision for scheduled spikes; reserve autoscaling for organic ones.** Autoscaling's reaction time is a fixed cost (instance boot, health checks, traffic registration), typically on the order of a minute. Any load event shorter than that reaction time will be over before autoscaling helps at all. If you know the time of the spike in advance, a TV ad slot, a product launch, a scheduled sale, provision the capacity ahead of the clock instead of waiting for a metric to cross a threshold.

**Prefetch the read path before the write burst, not during it.** Loading the target list into memory ahead of time turned a live, synchronized database read (a self-inflicted thundering herd) into a one-time batch job that happened hours earlier, off the critical path. Any time a burst of writes or sends is triggered by a known event, ask whether the data those sends depend on can be staged in advance instead of queried live at zero-hour.

**Idempotency keys let you stop worrying about duplicates everywhere else.** SQS FIFO's dedup ID absorbed the "what if the trigger fires twice" problem in one place, at the front door, so nothing downstream, the fan-out queue, the workers, the push gateways, had to re-implement its own duplicate check. Push the dedup responsibility as far upstream as possible and let everything after it assume the input is already clean.

**Batch to punch through per-call rate limits, on both sides.** SQS FIFO's 300/second-unbatched-vs-3,000/second-batched gap is one instance of a general rule: most hard rate limits are per-*call*, not per-*message*, so batching multiple messages into fewer calls raises effective throughput without needing a different service. The same logic applies going out to FCM and APNs, which both support and expect batched or multiplexed sends rather than one HTTP round trip per device.

## 5. The trade-offs

**Consistency vs. availability, split by data type, inside the same system.** The dedup decision ("has this user already been notified for this event") is treated as something that must be strictly correct, worth accepting a 300 msg/sec ceiling for. The delivery itself is treated as best-effort: FCM and APNs offer no delivery guarantee at all, some device tokens will be stale, some phones will be off or offline, and Duolingo's own success metric was a *percentile* (95% within 3.9s, 99% within 5.7s), not "100% of devices received it." That's a deliberate acceptance that perfect delivery isn't achievable or even necessary when the value of the notification is tied to a fleeting cultural moment; a notification that arrives 40 seconds late has mostly lost its point anyway.

**Cost vs. latency.** Five thousand pre-provisioned worker instances, sitting mostly idle for hours before and after a 30-second commercial break, is a real, deliberate cost. Duolingo paid for capacity that would look wasteful on any normal day, to buy a guaranteed sub-6-second delivery window for the handful of minutes it actually mattered. This is the same trade as over-provisioning for a scheduled sale: cost is cheap and reversible, a missed deadline on a one-time marketing moment is not.

**Freshness vs. read latency on the target list.** Staging the recipient list to S3 and loading it into worker memory ahead of time means that list is a snapshot, not a live query; a user who unsubscribed or changed their device token in the minutes between the snapshot and the trigger firing might get a slightly stale send. Duolingo accepted that small freshness gap in exchange for removing the live database read from the critical path entirely.

## 6. The systems-thinking lens

The failure mode this architecture is built to avoid is a **scheduled thundering herd meeting a reactive fix that can't keep up**.

Here's the loop a naive design falls into: load arrives all at once because it's synchronized to an external clock (the ad's air time), not because it grew gradually. The system's only lever for more capacity is autoscaling, which watches a metric, decides to add servers, and waits for them to boot. But the metric only crosses its threshold once the spike has already started, and the spike is over in 5 seconds while the autoscaler's reaction time is measured in tens of seconds to minutes. The "fix" arrives after the event it was meant to fix has already finished. Worse, if the same low-throughput queue is also carrying the payload, every retry or backlog from the failed first attempt adds more pressure to the one component that was already the bottleneck, the classic shape of a metastable failure: the system doesn't recover on its own once it falls behind, because the thing meant to drain the backlog is the same thing that's saturated.

The senior fix doesn't add more capacity reactively; it changes *when* the capacity decision gets made:

- **Move the scaling decision from reactive to scheduled.** If you know the event's timing in advance, provision ahead of the clock. Autoscaling stays for the traffic you can't predict; scheduled provisioning handles the traffic you can.
- **Separate the low-throughput correctness component from the high-throughput volume component**, so a strict guarantee that only ever needs to process one event doesn't become an accidental bottleneck for six million.
- **Move expensive reads out of the hot path entirely**, prefetching them earlier, so the burst only has to do the one thing it actually needs to do at zero-hour: send.

The general principle: when a spike is shorter than your system's reaction time, reacting faster is not the fix, because you cannot react to something that's already over. The fix is to stop reacting and start scheduling, converting a surprise into an appointment.

---

## Sources

- [How We Created a High-Scale Notification System at Duolingo, QCon London 2024 (InfoQ presentation page)](https://www.infoq.com/presentations/duolingo-high-scale-notification/)
- [QCon London 2024 session page: How We Created a High-Scale Notification System at Duolingo That Delivered Millions of Messages Within Seconds During a Super Bowl Commercial Break](https://qconlondon.com/presentation/apr2024/how-we-created-high-scale-notification-system-duolingo-delivered-millions)
- [InfoQ news recap of the QCon London Duolingo Super Bowl talk, April 2024](https://www.infoq.com/news/2024/04/qcon-london-duolingo-super-bowl/)
- [Duolingo: Sending 6M+ Notifications Within 5 Seconds, David Mosyan on Medium](https://medium.com/@dmosyan/duolingo-sending-6m-notifications-within-5-seconds-c630145038c3)
- [Amazon SQS FIFO queues, throughput and deduplication limits, AWS documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)
- [Sending notification requests to APNs, Apple Developer documentation](https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns)
- [About FCM messages / HTTP v1 send API, Firebase documentation](https://firebase.google.com/docs/cloud-messaging/send-message)

---

*Inference vs. fact, stated plainly: the target (6M+ notifications in 5 seconds), the previous best (500K in 60 seconds), the two-queue split (FIFO dedup queue vs. standard fan-out queue), the 5,000 pre-provisioned workers and 20 interim prefetch workers, the sustained ~800,000 req/sec, and the 95th/99th-percentile results (3.9s / 5.7s) all come from the QCon London 2024 talk, relayed here through the InfoQ presentation listing, InfoQ's own news recap, and a third-party session recap on Medium, since the network policy in this environment blocked direct retrieval of infoq.com, qconlondon.com, and medium.com at the time this lesson was written; the same figures appeared consistently across all independent write-ups, which is why they're presented as fact rather than hedged. The SQS FIFO throughput ceiling (300/sec unbatched, ~3,000/sec batched) is documented, stable AWS behavior. The APNs HTTP/2 connection model and FCM's HTTP v1 API are documented platform behavior, not company-specific claims. The autoscaling-reaction-time framing, the thundering-herd/metastable-failure mechanism, and the "control queue vs. volume queue" naming are this lesson's own analysis of why the documented design choices work, not claims from the talk itself.*
