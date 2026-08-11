# Day 53 — How does a leaderboard rank millions of live scores without falling over?

**Date:** 2026-08-11
**Difficulty:** Expert
**Topic:** Real-time leaderboards at scale: why "ORDER BY score DESC" on a relational table dies under live, high-frequency score updates, and the three production answers to it, an in-memory ranked structure (Redis-style sorted set: skip list plus hash table, O(log N) insert and rank instead of O(N log N) per-page sort) for Peloton-style single-shared-event leaderboards, cohort sharding down to a fixed small N (Duolingo's leagues of 30) so no leaderboard ever has to rank millions in the first place, and batch settlement (Codeforces' post-contest Elo recompute) for the cases where exact fairness matters more than an instant number.
**Stack relevance:** Rare.lab's community gallery and any future "most-remixed node graph" or per-device performance-benchmark board (which submitted shaders hit 60fps on a mid-range GPU, ranked fastest-to-slowest) is a leaderboard problem the moment it has more than a few hundred entries and updates on every new remix or benchmark run, not on a nightly cron. The mechanism choice here, in-memory ranked structure versus fixed cohort versus batch settlement, maps directly onto which of those three Rare.lab boards actually needs to be live.

---

## 1. The company and the breaking number

**Peloton**, in its 2025 Thanksgiving "Turkey Burn" live class, put **more than 32,000 riders into one shared, real-time leaderboard at the same moment**, and the event's service mesh peaked at **452,000 requests per second**, processing 7.27 billion requests across the event, according to AWS's published account of how Peloton engineers its largest live fitness events (Peloton has also run a Guinness World Record cycling class at 27,556 simultaneous riders). Every one of those 32,000 people is pedaling a bike that streams a fresh output reading to the server every few seconds, and every one of them is staring at a leaderboard panel that is supposed to show their current rank among the pack, updating live, for the entire duration of the class.

That single number, one shared leaderboard, tens of thousands of concurrently changing scores, each viewer expecting their own position to update within about a second, is the number that breaks a design that looks completely reasonable on a whiteboard: "store scores in a table, `ORDER BY score DESC` when someone opens the leaderboard." It works instantly in a demo with 20 rows. It does not survive contact with 32,000 people whose numbers are all moving at once.

## 2. Why the naive (demo) design dies

**Version one: one Postgres table, one column for score, `SELECT * FROM leaderboard ORDER BY score DESC LIMIT 100` on every page load, plus `SELECT COUNT(*) FROM leaderboard WHERE score > :mine` to tell a rider their own rank.** This is the version everyone builds first, because relational tables are the default hammer and "just sort it" sounds free. It fails on three fronts, and they compound.

**Sorting the whole board on every read does not amortize.** A `SELECT ... ORDER BY` over an unindexed or heavily-updated column re-sorts (or re-walks a churning index) on every single request. With 32,000 people each refreshing their leaderboard view, that is tens of thousands of full sorts per minute, not one sort shared by everyone, because there is no cached, standing answer to "who's currently on top," only a query that recomputes it from scratch every time it's asked.

**"What's my own rank" is worse than "who's on top," and it is asked far more often.** Nobody needs their own exact position O(1) times, they need it constantly, every few seconds, for the whole ride. `COUNT(*) WHERE score > mine` is an O(N) scan per rider per request. Multiply that by 32,000 riders each polling every few seconds and the database is doing millions of full-table scans a minute for a number that, realistically, only needs to be approximately right.

**Every score update is a write to the exact same hot structure the reads are hammering, and a B-tree index on a constantly-reordering column is expensive to maintain.** Each bike's telemetry update is an `UPDATE` that potentially reshuffles that row's position in the sort order, which means index maintenance, on the same table 32,000 riders are concurrently reading. Readers doing full sorts queue behind writers holding locks; writers queue behind readers scanning the same pages. This is the same collision-course shape as trying to run analytics queries against your live transactional table, except here both sides are the live, latency-sensitive path, not one background report. This is exactly the kind of load a naive single-primary relational design cannot absorb at Peloton's Turkey Burn scale, and it is why the real system looks nothing like "one table, one sort."

## 3. The architecture

```
[Clients: bike/tablet console, phone app — displays rank + nearby
 riders, subscribes to live updates]
   analogy: a runner's wristwatch showing their current 5K split
   without radioing the race organizer every second to ask
   |
   v
[Edge / CDN: static app shell, images, class metadata — NOT the
 live score data, which is far too volatile to cache at the edge]
   |
   v
[Load balancer -> stateless app tier: ingests each rider's telemetry
 event, translates "output = 210 watts at t=12:04" into a score-
 update call; also stateless read handlers serving rank queries]
   analogy: a bank of ticket-window clerks, any of them can serve
   any customer, none of them personally remembers anyone
   |
   v
[In-memory ranked structure, one per live class/shard: a sorted
 set (skip list + hash table) keyed by rider, scored by output.
 ZADD/ZINCRBY to update a score in O(log N); ZRANK/ZREVRANGE to
 read a rider's position or the current top 100 in O(log N) / O(log N + M)]
   analogy: a leaderboard scroll at a real cycling velodrome that a
   human updates by sliding a name-tile up or down the board, not
   one that gets rewritten from a fresh printout every lap
   |
   v
[Async fan-out / pub-sub: publish "rank changed" deltas to
 subscribed viewers over a push channel (WebSocket / long-lived
 stream), on a fixed cadence, instead of each client re-polling
 and re-sorting]
   analogy: a scoreboard operator announcing only the changes
   ("now in third: Priya") instead of re-reading the entire
   standings sheet aloud after every single lap
   |
   v
[Durable store, written asynchronously behind the live tier:
 Postgres/DynamoDB holds the source-of-truth history, workout
 records, and final results — the thing an audit, a payout, or a
 personal-record badge can trust, even though it is not what the
 live rank number is read from mid-class]
   |
   v
[Sharding by class, not by score range: each live class gets its
 OWN sorted set with N in the hundreds or low thousands, not one
 global structure ranking every Peloton member on Earth against
 each other]
```

Two other real products solve the same problem by refusing to let N get large in the first place, and it's worth drawing both, because "shrink N" beats "optimize the read" as a first move.

**Duolingo's Leagues sidestep the scaling problem entirely.** According to Duolingo's own product explanation of Leagues, each weekly league is a cohort of **30 randomly assigned learners** with similar habits, ranked by XP earned that week; only the top slice of each 30-person group promotes, and the losing majority in each tier demotes or stays. There is no such thing as a global Duolingo leaderboard a learner competes on. This means the "rank 30 people by a number" query never needs a skip list, a sorted set, or a cache at all, a scan over 30 rows is cheap enough to just run, every time, on a plain indexed table. (How Duolingo actually stores and computes this internally is not publicly documented in engineering detail; the cohort-of-30 product mechanic itself is, and the inference that a 30-row scope needs no dedicated ranking data structure is this lesson's own reasoning, clearly labeled as such, not a quoted architecture claim.)

**Codeforces separates "live, approximate" from "final, exact" by settling in one batch pass after the contest closes.** During a round, standings shown to participants are live but explicitly provisional. Rating changes, the number that actually matters long-term, are computed once, for the whole contest at once, using a seed-vs-rank Elo-style formula (Mike Mirzayanov's own published description of the Codeforces rating system: each participant's expected finishing position, their "seed," is compared against their actual rank, and the gap drives the rating delta) rather than being recalculated incrementally after every single submission. That formula fundamentally needs the whole field's final results at once to compute anyone's seed correctly, so trying to keep it "live and exact" the way a Peloton rank number is live would be solving the wrong problem: the fair answer only exists once the contest is over.

## 4. The transferable mechanisms

**Sorted set as the core primitive: an ordered structure that updates in O(log N), not a table you re-sort on every read.** A skip list, a probabilistic linked structure where each node gets a randomly chosen "height," gives expected O(log N) insert, delete, and rank lookup, paired with a hash table for O(1) "what is this specific member's current score." This is the mechanism behind Redis's sorted-set commands (`ZADD`, `ZINCRBY`, `ZRANK`, `ZREVRANGE`) and it is the single biggest lever here: it turns "recompute the whole order" into "fix one element's position," the same difference as inserting into a sorted array versus re-sorting the whole array from scratch on every write.

**Shrink N before you optimize the read of N.** Duolingo's 30-person leagues and Peloton's per-class sharding are the same move seen twice: don't build one structure that has to rank everyone against everyone, partition first (by class, by cohort, by region, by time window) so each individual ranked structure only ever has to handle a small, bounded N. This is the same instinct as cell-based architecture (bounding blast radius by partitioning users into small independent cells) applied to ranking cost instead of failure isolation.

**Push deltas, don't make everyone re-poll and re-sort.** Publish "rank changed" events to subscribers on a fixed cadence instead of having every client hit the read path on its own schedule. This is the same shape as a WebSocket game-state broadcast or a chat app's message fan-out: the expensive computation happens once, centrally, and the result is pushed out, rather than being independently recomputed once per viewer.

**Separate the cheap live estimate from the expensive exact answer.** A rider mid-class needs "you're roughly 12th" within a second; nobody needs that number to be transactionally exact down to the millisecond. Codeforces takes this to its logical extreme: the live number during a contest is explicitly provisional, and the number that actually counts is computed once, later, when it can be gotten exactly right. Decide, per feature, which side of that line you're on before building the read path.

**Batch settlement as a legitimate design, not a fallback.** When a ranking's fairness formula genuinely needs the whole data set at once (Codeforces' seed calculation needs every participant's final result to know anyone's expected finish), fighting to make that "real-time and always consistent" is solving a problem nobody has. Compute it once, correctly, in a controlled batch job, and be explicit that the live number shown beforehand was always an approximation.

**Bound the read, not just the write.** Whether it's capping how many entries a "top 100" view ever returns (Peloton, Redis `ZREVRANGE ... LIMIT 100`) or capping cohort size outright (Duolingo's 30), every one of today's real systems puts an explicit ceiling on how much of the ranked set any single read has to touch, rather than trusting an index to make an unbounded scan cheap enough.

## 5. The trade-offs

**Live rank display: availability and low latency over strict consistency.** A rider's on-screen rank being a second or two stale, computed from the last broadcast tick rather than the literal instant their last pedal stroke landed, is the correct trade. Nobody's ride is ruined by their number lagging a beat; everyone's ride is ruined if the leaderboard panel spins or errors out because the system was trying to guarantee millisecond-perfect ordering across 32,000 concurrent writers.

**Final results and anything with a prize or a rating attached to it: consistency wins, and it wins completely.** The moment a number is going to be written into a permanent record, a personal-record badge, a contest rating, a payout, "eventually correct" is not good enough, it has to be exactly correct and auditable against the durable store, computed once and not recomputed differently on a retry. This is why the durable store sits behind the live tier as the actual source of truth, and why Codeforces refuses to call its live standings final.

**Cost vs. latency: an in-memory ranked structure per shard costs real RAM, and it is worth it exactly where the read volume justifies it.** Keeping a live sorted set resident in memory for an active Peloton class is a good trade, tens of thousands of rank reads per second need an O(log N) structure, not a disk-backed sort. Keeping one for Duolingo's 30-person league is not worth building at all, a plain indexed table scan over 30 rows is already fast enough; adding a sorted set there would be complexity spent on a problem that sharding already solved for free.

## 6. The systems-thinking lens

The feedback loop worth naming here is a **recompute stampede**, a variant of thundering herd, but triggered by legitimate write volume instead of a cache-expiry timer.

Picture the failure: the "top 100" leaderboard view is cached and invalidated on every score-changing write. At the exciting moment of the class, the final sprint, when everyone's output is spiking at once, score updates arrive in a burst. Every one of those writes invalidates the cached top-100 snapshot. Every client watching the board, which had been polling or subscribed expecting a fresh snapshot, refetches. Now the system has to recompute and resend the full leaderboard view at exactly the moment write volume is highest, which is also exactly the moment the most people are staring at their screens waiting for it to update. The very thing that made the moment exciting, everyone's numbers moving at once, is what causes the read-side stampede that makes the board freeze right when riders care most.

Buying a bigger cache node doesn't touch this loop; the problem is that write rate is directly driving both invalidation rate and refetch rate at the same time. The senior fix decouples them:

- **Broadcast on a fixed cadence, not on every write.** Recompute and push the top-100 snapshot every 250ms to 1 second, regardless of whether one score changed or a thousand did in that window. This caps read-side work at a constant rate no matter how bursty the writes get, the same move as batching Kafka consumer flushes instead of committing per message.
- **Push deltas to already-subscribed clients instead of inviting a refetch.** A client holding a WebSocket connection that receives "rider 4821 moved from 12th to 9th" never has to ask for anything; there is no herd, because there is no request to stampede on.
- **Rate-limit or debounce cache invalidation itself.** If a cache-and-invalidate pattern is used anywhere in the stack, treat "someone wrote" as a signal to schedule a refresh, not as a command to refresh immediately and let every waiting reader pile onto the recompute.

The general principle again: any time write volume is allowed to directly drive read-side or broadcast-side volume, a burst on one side becomes a stampede on the other, precisely when the system is already under the most load. Break the coupling with a fixed cadence, not with more capacity thrown at an uncapped rate.

---

## Sources

- [How Peloton Engineers the World's Largest Live Fitness Events on AWS, AWS Industries Blog](https://aws.amazon.com/blogs/industries/how-peloton-engineers-the-worlds-largest-live-fitness-events-on-aws/)
- [Stats for Peloton's Thanksgiving Day 2025 Classes (Turkey Burn 2025), Peloton Buddy](https://www.pelobuddy.com/stats-turkey-burn-2025/)
- [Peloton Stays Online During the 2024 Turkey Burn / Thanksgiving Day Classes, Peloton Buddy](https://www.pelobuddy.com/2024-turkey-burn-stats/)
- [Peloton Secures GUINNESS WORLD RECORDS Title for the Largest Live Cycling Class, PR Newswire](https://www.prnewswire.com/news-releases/peloton-secures-guinness-world-records--title-for-the-largest-live-cycling-class-300561448.html)
- [Open Codeforces Rating System, Mike Mirzayanov, Codeforces](https://codeforces.com/blog/entry/20762)
- [How Duolingo Leaderboards and Leagues Work, Duolingo Blog](https://blog.duolingo.com/duolingo-leagues-leaderboards/)
- [Build a Real-Time Leaderboard with Redis Sorted Sets, Redis](https://redis.io/tutorials/howtos/leaderboard/)
- [Real-time leaderboard and ranking solutions, Redis](https://redis.io/solutions/leaderboards/)

---

*Inference vs. fact, stated plainly: the AWS/Peloton blog and the Codeforces rating-system post were reached only through search-engine-indexed summaries in this environment (direct page fetches to aws.amazon.com, codeforces.com, redis.io, and duolingo.com's own blog were all blocked by this session's network egress policy), so the 32,000+ rider count, the 452,000 requests-per-second peak, and the 7.27-billion-request total for Turkey Burn 2025 are drawn from those indexed summaries rather than a directly-read primary article, and should be treated as reliable-but-secondhand rather than independently verified against the source page. The Duolingo cohort-of-30 mechanic and the Codeforces seed-vs-rank batch formula are the same way: confirmed via indexed summaries of Duolingo's and Codeforces' own posts, not a full direct read. Redis's skip-list-plus-hash-table implementation of sorted sets and the O(log N) complexity of ZADD/ZRANK/ZREVRANGE are well-established, widely documented facts about Redis's public data-type design, stated here from general knowledge of that documentation rather than a page fetched during this run. Everything framed as "this lesson's own reasoning" (Duolingo's likely internal simplicity at N=30, the recompute-stampede feedback loop and its fix) is explicitly this lesson's synthesis, not a claim sourced to any company.*
