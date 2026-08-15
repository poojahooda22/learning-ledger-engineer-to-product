# Day 54 — How does Dream11 re-rank 40 million fantasy-team entries during one cricket match without a live `ORDER BY` collapsing under the writes?

**Date:** 2026-08-15
**Difficulty:** Expert
**Topic:** Real-time leaderboard ranking at scale: why sorting a live, constantly-updated table on every read is the failure mode almost every leaderboard design reaches for first, how Dream11 turned "rank 40 million entries" into a bounded, scheduled batch job (Spark) feeding a key-value serving store (Aerospike) instead of a query, why the serving store is tuned AP instead of CP on purpose, and the general precompute-and-serve pattern this teaches for any read-heavy, write-heavy ranked view.
**Stack relevance:** Rare.lab doesn't run contests, but the shape recurs anywhere a ranked or aggregated view has to stay current under a stream of small updates while many viewers read it live, a public gallery of shader packs sorted by popularity, a "trending nodes" panel, or usage rankings across an org's projects. The lesson underneath, never compute a sorted view inside the request path, applies the moment any list needs both "frequently written" and "frequently read" at once.

---

## 1. The company and the breaking number

**Dream11**, India's largest fantasy sports platform, engineering blog posts "Leaderboard @ Dream11" (2017) and "Scaling to 700k TPS: A Story of High Volume Writes with Aerospike for Dream11's Leaderboard" (2025). During a single big cricket match, users build fantasy teams inside contests that range from head-to-head (2 teams) up to mega-contests with 400,000 teams. Across all the simultaneous contests running on one match, that adds up to as many as **40 million entries that all need a correct, freshly-sorted rank inside their contest, refreshed on a 60-second SLA**. Sustained across the length of a match, that SLA works out to roughly **666,667 leaderboard rows recomputed and re-ranked every second**, not as a burst, for the 3 to 4 hours the match runs.

That number alone would strain a database. What actually breaks a naive design first is what generates the updates: fantasy points change on every real-world scoring event in the match, a run scored, a wicket taken, a catch held, and those events hit every one of the up to 400,000 teams in a mega-contest at once, because most fielders and batters are shared across many users' lineups. So it isn't 666,667 independent single-row updates trickling in, it's a wide fan-out: one ball bowled in the real match can instantly change the point totals, and therefore the *rank order*, of hundreds of thousands of rows simultaneously. By 2023 to 2024, Dream11 was carrying **100 million-plus registered users**, with recorded peaks of **5.5 million concurrent users during the IPL 2020 final** and later **10.56 million concurrent users at peak**, all of whom can have the same live leaderboard open, refreshing to watch their rank move in real time. The read load and the write load spike on the exact same trigger, the same ball being bowled, at the exact same moment.

## 2. Why the naive (demo) design dies

**Version one: one relational table, `SELECT * FROM contest_entries WHERE contest_id = X ORDER BY points DESC`.** Every time a scoring event lands, an application server runs an `UPDATE` on every affected row, and every time a user opens or refreshes the leaderboard, the API runs that `ORDER BY` fresh. This is the natural first build, a leaderboard *is* a sorted list, so sorting a list sounds like the right primitive. It fails for three separate reasons.

**The sort cost is paid on every single read, and it doesn't get cheaper as the contest fills up.** A 400,000-row `ORDER BY` is not free, and unlike a static catalog, this table has no stable index to lean on, because the column you're sorting by is being rewritten by every scoring event. Postgres and MySQL both have to redo real work here: an index on `points` needs to be updated on every write, and a live read racing a live write either blocks or reads a stale mid-sort snapshot depending on isolation level. Run that `ORDER BY` for even a fraction of the 5.5 million concurrent viewers Dream11 has seen, and the database is now doing a full sort per request, for a table that a thousand other requests are simultaneously rewriting.

**Read volume dwarfs write volume, by orders of magnitude, and a naive design serves both from the same live table.** The 666,667 writes/second figure is already large, but the number of people watching a rank move is much larger than the number of scoring events causing it, everyone in a 400,000-team contest can be staring at the same leaderboard screen at once, each one polling or holding a live connection. A design that computes rank at read time multiplies an already-large write rate by a much larger read fan-out and asks one table to absorb both.

**Rank is a whole-contest property, so it can't be sharded away the way most write-heavy tables can.** The standard fix for "too many writes to one table" is to shard by some key and spread the writes across machines. But a leaderboard's central operation, "who is in 1st place," is a global aggregate over the *entire* contest. If contest_id 42's 400,000 rows were split across ten shards to spread the write load, computing one true rank ordering would require pulling candidate rows back from all ten shards and merging them for every single read, which reintroduces the exact sort-on-read cost the sharding was meant to avoid, just now with a network hop added.

## 3. The architecture

```
[Clients: millions of phones/browsers, contest leaderboard screen open]
   analogy: a stadium full of fans checking the scoreboard, not each
   fan doing their own arithmetic off the raw play-by-play feed
   |
   v
[Load balancer -> stateless leaderboard READ API tier]
   analogy: many ticket windows, none of them owns any data,
   any window can answer any user's request
   |
   v
[Serving cache/store: per-contest PRE-RANKED list, key = contest_id]
   analogy: a scoreboard operator posts the finished, already-sorted
   standings on the wall; fans read the wall, nobody re-tallies it
   themselves
   |
   ^
   | (overwritten wholesale on a fixed cadence, not patched row by row)
   |
[Aerospike, AP mode, sharded by contest_id, 21+ node cluster]
   analogy: the warehouse of scoreboards, one per contest, restocked
   on a clock, never "corrected" mid-shift
   |
   ^
[Distributed batch compute: Apache Spark job, recomputes and
 re-sorts every contest's standings each cycle from raw scoring
 events, writes the whole new sorted list back]
   analogy: a large team of scorekeepers who redo the full tally
   from scratch every 10-60 seconds, rather than each fan interrupting
   them mid-count to ask "am I winning yet"
   |
   ^
[Live match scoring event stream: run scored, wicket, catch, etc.]
   analogy: the radio commentary feed, the one true source of what
   actually happened on the field
```

The critical shape: the arrow from clients to the answer never touches a live sort. Reads always hit a pre-computed, already-ranked object. Writes never touch that object directly either, they accumulate as raw events, and a separate compute tier turns the accumulated events into a fresh ranked snapshot on a schedule, then swaps it in.

## 4. The transferable mechanisms

**Precompute the expensive operation, serve a cheap lookup.** The sort, the actual O(N log N) work of ordering 400,000 rows, happens exactly once per refresh cycle inside Spark, off the request path entirely. Every one of the millions of concurrent reads then costs a single keyed fetch from Aerospike, not a sort. This is the same offline-think, online-lookup split that shows up in search ranking and recommendation systems: do the expensive work once, in the background, and let every live request be a lookup against the answer.

**Bounded staleness as a deliberate SLA, not an accident.** The 60-second (and, in the 2025 architecture, roughly 10-second) refresh window means a user's displayed rank can lag the true state of the match by that long. That staleness window is a chosen number, not a failure, it converts "rank must be correct at all times" (expensive, contention-heavy) into "rank must be correct as of the last snapshot, and the next snapshot is always coming soon" (cheap, embarrassingly parallel).

**Choose the partition/shard key to match the query you actually run, not the entity's own identity.** Entries are keyed and sharded by `contest_id`, not by `entry_id` or `user_id`, because the only query that matters is "give me the full ranked list for this contest." A mock system-design interview walkthrough of a LeetCode-style coding-contest platform, reviewed as background material for this lesson, arrives at the identical conclusion from a completely different domain: a submissions table for a live contest leaderboard should be indexed by `competition_id`, because a per-user `submission_id` lookup is irrelevant to "who's in first place right now" and would force a scatter-gather across the whole table to answer it. Same mechanism, two unrelated products.

**Distributed batch recomputation absorbs fan-out that a row-by-row update model can't.** Instead of translating one scoring event into up to 400,000 individual row `UPDATE`s (the naive design's failure mode), Spark treats the whole contest as one distributed computation over the accumulated event log each cycle, spreading the work across cores and nodes. The fan-out cost is paid once per cycle in aggregate, not once per affected row per event.

**Pick AP over CP for data that is about to be superseded anyway.** Aerospike is run for this workload in AP (available, partition-tolerant) mode rather than strongly consistent mode. That's a real, named trade-off, not a default: since the whole point is a full refresh in the next 10 to 60 seconds, a brief window of one replica lagging another after a network hiccup is cheaper to accept than paying the coordination cost to prevent it, because the correction is arriving on a clock regardless.

## 5. The trade-offs

**Consistency vs. availability, split by data type, inside the same platform.** The *raw scoring events* and, critically, the *final settlement numbers that decide who gets paid a real cash prize* have to be exactly right, this is money, not a game score. The *live rank shown on screen while the match is still running* only has to be approximately right, as of the last refresh. Dream11's architecture treats these as two different consistency problems: strict correctness for the ledger that pays out, best-effort freshness for the number a user glances at every few seconds during play. Conflating the two, demanding CP-level rigor for the on-screen rank too, is exactly what a naive single-table design does by accident.

**Cost vs. latency.** A 21-node Aerospike cluster on `r5n.16xlarge` instances plus a Spark cluster large enough to re-rank 56 million entries across 2 million contests inside a 10 to 60-second window is real, continuously-running infrastructure, sized for peak match traffic, not average traffic. Outside live matches that capacity is mostly idle. This is the same shape as scheduled over-provisioning seen elsewhere in this series (Duolingo's pre-warmed worker fleet, Day 53): known, recurring load (a match has a start time) justifies paying for capacity ahead of the spike rather than reacting to it.

**Freshness vs. compute cost, tuned by the refresh interval itself.** Shrinking the cycle from 60 seconds to 10 seconds, which Dream11's later architecture did, buys users a leaderboard that feels closer to real-time, at the direct cost of running the full Spark recomputation six times more often. That interval is a dial, not a constant, and the right setting depends on how much staleness the product can tolerate versus how much the compute costs.

## 6. The systems-thinking lens

The failure mode a naive live-sort design walks into is a **read-amplification feedback loop tightly coupled to the same event that drives writes**.

Here's the loop: a wicket falls in the match. That single real-world event changes the point totals, and therefore the rank order, for a large fraction of every contest's entries at once, a genuine write spike. But it's also the most exciting possible moment for a fan to check their rank, so the same event triggers a simultaneous read spike, potentially millions of people refreshing within the same few seconds. If reads are served by computing the sort live, every one of those reads now competes with every one of those writes for the same table, at the exact moment both are at their peak, driven by the same trigger. Slower reads make impatient users refresh more, adding load to a system that is already struggling, the same self-reinforcing shape as a retry storm: the fix people reach for under pressure (refresh again) is the thing making the underlying problem worse.

The senior fix doesn't try to make the live sort faster; it removes the coupling between the trigger and the request path entirely:

- **Decouple the read path from the write path completely.** Reads only ever touch a finished, precomputed snapshot. No read, no matter how many happen in the same second, can add work to the compute tier, because reads and computation are now different systems talking to different data.
- **Batch the fan-out instead of applying it event by event.** One wicket doesn't trigger 400,000 individual row updates; it becomes one more input into the next scheduled recomputation cycle, however many events land in that window get resolved together, once.
- **Turn "always correct" into "correct as of a bounded, known interval."** That's the lever that lets the system convert an unpredictable spike (a wicket can fall at any second) into predictable, schedulable work (a recomputation that runs on a fixed clock regardless of how many events accumulated).

The general principle: when reads and writes spike together because they share the same real-world trigger, adding capacity to the shared live path doesn't break the coupling, it just raises the ceiling the same feedback loop eventually hits again. Separating "what changed" from "what's currently displayed," with a scheduled, bounded-staleness handoff between them, is what actually breaks the loop.

---

## Sources

- [Leaderboard @ Dream11: Scale to serve 100+ million users, Dream11 Engineering (2017)](https://blog.dream11engineering.com/leaderboard-dream11-4efc6f93c23e)
- [Leaderboard @ Dream11, mirrored on tech.dream11.in](https://tech.dream11.in/blog/2017-10-24_Leaderboard--Dream11-4efc6f93c23e)
- [Scaling to 700k TPS: A Story of High Volume Writes with Aerospike for Dream11's Leaderboard, Pratyush Bansal, Level Up Coding (2025)](https://levelup.gitconnected.com/scaling-to-700k-tps-a-story-of-high-volume-writes-with-aerospike-for-dream11s-leaderboard-0a226bdf1adf)
- [Serving millions of concurrent users in real-time with Aerospike, Dream11 customer story, Aerospike](https://aerospike.com/resources/customer-stories/dream11-aerospike-customer-story/)
- [Build a real-time gaming leaderboard with Amazon ElastiCache for Redis, AWS Database Blog](https://aws.amazon.com/blogs/database/building-a-real-time-gaming-leaderboard-with-amazon-elasticache-for-redis/)
- Secondary, unlinked reference used for this lesson: a mock system-design interview transcript (a Google software engineer interviewing on a LeetCode-style contest platform, covering the contest-leaderboard schema and the choice to index submissions by `competition_id`), supplied as background material rather than a public citable article; its architectural conclusion is summarized in Section 4 and independently corroborated by the Dream11 sources above.

---

*Inference vs. fact, stated plainly: the 40 million entries / 400,000-team-mega-contest / 60-second SLA figures and the "666,667 records/second" sustained rate come from Dream11 Engineering's 2017 "Leaderboard @ Dream11" post; the 700,000 TPS peak, the 21-node `r5n.16xlarge` Aerospike cluster, the 56 million player records across 2 million contests, and the 15GB dataset come from the 2025 "Scaling to 700k TPS" piece; the 5.5 million concurrent users at the IPL 2020 final and the 10.56 million peak concurrent users come from Aerospike's Dream11 customer story. The network policy in this environment blocked direct retrieval of blog.dream11engineering.com, tech.dream11.in, levelup.gitconnected.com, aerospike.com, and aws.amazon.com at the time this lesson was written, so these figures are relayed through search-indexed summaries of those pages rather than a direct fetch; the same numbers appeared consistently across independent mentions of each source, which is why they're presented as fact rather than heavily hedged. The AP-vs-CP framing, the read-write coupling as the specific feedback loop, the comparison to Day 53's scheduled-provisioning pattern, and the generalization to Rare.lab's stack are this lesson's own analysis layered on top of the documented architecture, not claims made verbatim by Dream11 or Aerospike.*
