# Netflix, the "Top 10 in Your Country Today" row

Date: 2026-10-25
Product: Netflix
Feature: The non-personalized "Top 10 in [Country] Today" row, the one with the big red numbers 1 to 10 stacked on the corner of each poster.

A note on scope so this does not collide with earlier teardowns. On 2026-08-06 I tore
down the Netflix personalized homepage: which titles fill a row, the order inside a row,
and the harder problem of ranking rows against each other (page-wise generation, GenPage).
That teardown mentioned "Trending Now" in one line as "a windowed streaming aggregate" and
moved on. It never built the counting machine. This teardown is only about ONE specific
row, and it is the one row on the whole home screen that is NOT personalized: the numbered
"Top 10 Today" list that is identical for every account in the same country. The hard
problem flips. Personalized rows are a ranking-and-matching problem over your taste. The
Top 10 is a pure COUNTING problem over a firehose of play events, with zero personalization.
Different machine, different enemy, so it earns its own day.

---

## 1. The user, in their day

It is 9:10pm in Pune. Ananya just finished dinner, she is on the sofa with the TV remote,
and she has about ninety minutes before she wants to sleep. She opens Netflix. She does not
know what she wants to watch. She is not in the mood to read five paragraphs of plot
summary for twenty different titles. She scrolls down a little and sees a row she trusts
more than any other: a horizontal strip of ten posters, each with a huge red number in the
corner, 1 through 10, under a label that says "Top 10 in India Today." She looks at number
1, "The Night Agent," number 3, "Wednesday," and thinks, if this many people are watching
it right now, it is probably not a waste of my evening. She clicks number 3.

That is the whole moment. A tired person, limited time, decision fatigue, and one row that
answers the only question that matters at 9:10pm: what is everyone else watching.

---

## 2. The real problem, like a friend would say it

"I don't know what to watch" is the single biggest reason people close Netflix without
pressing play. The catalog is huge, the personalized rows are good but they all feel like
guesses aimed at you, and sometimes you do not want a guess aimed at you. You want to know
what is actually popular. Not "popular for people like you." Popular, full stop. The real
pain is social: you heard someone at work mention a show, you forgot the name, and you want
to find the thing everyone is talking about this week so you are not left out of the
conversation tomorrow. The old personalized rows cannot answer that, because they are tuned
to you, not to the crowd. The Top 10 row is the crowd made visible.

---

## 3. The feature in one sentence

The Top 10 row shows the ten most-watched titles in your country right now, ranked 1 to 10
with a visible number, identical for everyone in that country, refreshed daily.

---

## 4. Jobs to be done

What is Ananya really hiring this row to do?

- "Tell me what is safe to pick so I stop scrolling." A ranked list is a decision shortcut.
- "Tell me what is culturally current so I am not the last to know." It is a social radar.
- "Give me permission to watch the obvious thing." Number 1 is an excuse to not overthink.
- "Show me the crowd, not a guess about me." It is the one row that is not a mirror.

The quiet fourth job matters most. Every other row on the screen is Netflix predicting
Ananya. This row is Netflix reporting a fact about everyone else. That difference is why
people trust the red numbers more than they trust "Because you watched."

---

## 5. How it works for the user (the visible experience)

Ananya sees a single horizontal row, ten posters, each with a bold number 1 to 10 overlaid
in Netflix red. The order is the ranking. Number 1 is the most-watched title in India
today. The row looks the same on her phone, her mother's phone, and her neighbor's TV,
because it is national, not personal. It changes day to day. If she opens the app tomorrow,
a title may have jumped from 6 to 2, or fallen off the list entirely. There is also a
separate public website, top10.netflix.com, that publishes a WEEKLY global and per-country
version of the same idea, and that one is where the press gets its "Squid Game is number 1
in 93 countries" headlines.

The important user-facing fact: the number is doing persuasion work. A poster with a red "1"
on it gets more clicks than the identical poster without the number. The ranking is not just
information. It is a nudge.

---

## 6. The actual flow, step by step

1. Ananya taps the Netflix icon. The TV app wakes and calls home for her home screen.
2. The server assembles her page: Continue Watching, a few personalized rows, and the Top 10
   row. Each row arrives as a small ordered list of title IDs plus artwork pointers.
3. For the Top 10 row specifically, the server does NOT compute anything live. It reads a
   tiny precomputed list for her country (India) that was built by a batch job earlier.
   That list is ten title IDs in rank order. Example payload, simplified:
   `["80236318" (#1 Night Agent), "81231974" (#2 ...), ...]`, ten entries, nothing more.
4. The app draws ten posters, overlays the red numbers 1 to 10 in order, done.
5. Ananya taps number 3. Now the normal playback machinery takes over (that is the adaptive
   bitrate and Open Connect story from earlier teardowns, not this one).
6. As she watches, the app emits playback events (started, still playing heartbeats, stopped).
   Those events are the raw fuel that, days later, will decide tomorrow's and next week's
   Top 10. She is both a reader of the list and, invisibly, a voter in it.

The whole read path is: fetch ten IDs, draw ten posters. The TV never sorts anything. All
the hard work happened offline, before she opened the app. Hold that thought, it is the
spine of section 7.

---

## 7. Under the hood, like the engineer

### 7.1 The trap: this is not a sort problem

The naive mental model is "sort the catalog by views, take the top ten." That model is
wrong about where the difficulty lives, and seeing why is the whole point of this teardown.

Netflix's entire catalog is small. Netflix's own first engagement report, "What We Watched"
(covering January to June 2023), listed "more than 18,000 titles" and said that set
represented about 99% of all viewing on the service (Netflix, about.netflix.com). Eighteen
thousand is nothing. You can sort 18,000 numbers a million times a second on a laptop. If
the Top 10 were "sort 18,000 titles," it would be a homework problem, not an engineering
feature.

So the catalog is not the enemy. The enemy is the EVENT STREAM. That same report counted
"nearly 100 billion hours viewed" in six months (Netflix, about.netflix.com). The 2023 top
title, "The Night Agent: Season 1," alone pulled 812,100,000 hours (Netflix engagement
report, reported by ETCentric and others). Those hours are made of billions and billions of
tiny playback events streaming in from hundreds of millions of devices every single day. The
hard problem is not ordering a short list. The hard problem is COUNTING a firehose cheaply
and continuously, so that by the time you want the list, the per-title totals already exist.

That is the real two halves of this feature, and they are different from the two halves of a
search feature. In search the halves are matching (find candidates) then ranking (order
them). Here the halves are:

- Half one, COUNT: turn a never-ending river of "member X watched title Y for Z seconds in
  country C" events into a running total per title per country. This is the expensive half.
- Half two, SELECT TOP-K: pick the ten largest totals per country. This is the cheap half,
  as long as half one did its job.

### 7.2 What a "view" even is (the counting rule keeps changing)

Before you can count, you must define the unit, and Netflix has rewritten this definition
three times, which tells you how load-bearing it is.

- Original rule: a title counted as watched when an account completed about 70% of it
  (reported by Tubefilter, 2020).
- January 2020 (Q4 2019 shareholder letter): the "2-minute rule." Watching at least 2
  minutes counted as a view. Netflix said this made reported numbers about 35% higher on
  average than the old rule (Netflix Q4 2019 letter, via Tubefilter). Critics pointed out a
  movie with a 2-minute opening credits sequence could score a "view" from someone who
  bailed immediately (BGR, Cheddar).
- November 2021: for the Top 10 and the new public site, Netflix switched to total HOURS
  VIEWED over a rolling window, originally 28 days for the "most popular" all-time lists
  (BGR; site launch reported by Advanced Television, 17 Nov 2021).
- June 2023: Netflix added "Views," defined as hours viewed divided by the title's runtime,
  so a 10-hour series and a 100-minute film can be compared without the long series winning
  automatically. The qualifying window for the "most popular" lists was extended from 28
  days to 91 days (multiple outlets summarizing Netflix's methodology update, incl. TVLine).

Why an engineer cares: the counting unit determines what you accumulate. If the unit is
"hours," every heartbeat event must carry a duration and you SUM durations. If the unit is
"views = hours / runtime," you still sum hours and then divide by a per-title constant at
read time. Netflix also hired EY, an outside accounting firm, to audit these numbers,
because the whole feature only works if people believe the count is honest (Netflix,
reported widely). A ranking is only as trustworthy as its counting rule, and Netflix pays an
auditor to defend the rule. That is a product decision wearing an engineering costume.

### 7.3 The data structures, tier by tier

Now the actual mechanics. I will walk the real example query: "produce the Top 10 in India
Today," and show how the machine changes shape as the numbers grow. Confirmed Netflix
infrastructure is cited; where I describe the standard way this class of problem is solved
and Netflix has not published the exact internals, I label it INFERENCE.

#### Tier 1: 1,000 titles, one small country, thousands of plays a day

At this size you do the obvious thing and it is correct. Keep one hash map in memory:
`Map<titleId, double hoursToday>`. Every time a play-stop event arrives for India, add its
duration to the right bucket. At 9pm, to get the Top 10, iterate the roughly 1,000 entries
once, keep a 10-slot min-heap (if a title's hours beat the smallest in the heap, swap it
in), and you have the answer in a millisecond. Equivalent in SQL it is literally
`SELECT titleId, SUM(hours) FROM plays WHERE country='IN' AND day=today GROUP BY titleId
ORDER BY 2 DESC LIMIT 10`. One box, one query, done. Nothing is sharded. A min-heap of size
10 over 1,000 items is 1,000 comparisons, trivial. Concrete: title "The Night Agent" sits in
bucket 80236318 and you keep adding 0.73 hours, 0.91 hours, 1.00 hours as each member
finishes an episode.

What breaks going up: you cannot keep re-scanning the raw plays table once "plays" is
billions of rows a day, and you cannot keep it all in one box's memory across every country.

#### Tier 2: 100,000 plays a day, many countries, read millions of times

Two things change. First, you stop computing on read. Ananya opens the app millions of times
a day across India, and you will not run a GROUP BY for each open. So you flip to
offline-think, online-lookup, the same spine as Discover Weekly (06-13) and the homepage
(08-06): a nightly (or hourly) batch job computes the ten IDs per country and writes them to
a tiny key-value row, `top10:IN -> [id1..id10]`. Every app open is now an O(1) cache read of
ten IDs, not a sort. The TV never counts.

Second, you stop storing raw events forever and start storing TIME-BUCKETED COUNTERS.
Instead of a billion rows, keep per-title per-country per-hour (or per-day) sums:
`(titleId, country, hourBucket) -> hoursViewed`. This is a rollup. To get "today" you sum 24
hourly buckets. To get the weekly public list you sum 168 hourly buckets, Monday to Sunday.
Rollups collapse a firehose into a grid you can actually query. Netflix's analytics stack
leans on Apache Druid for exactly this shape of time-bucketed, roll-up-friendly data, and
Druid's TopN query is built to return the top K per dimension quickly (Druid is a known part
of the Netflix data platform; Druid's TopN is approximate by design for ranks past a
threshold, exact results need a groupBy). INFERENCE on the exact Top 10 wiring, CONFIRMED
that Druid + rollups are the house style.

What breaks going up: at true Netflix scale the raw event volume is so large that even
writing one counter row per title per country per hour, and re-summing from durable storage,
is a heavy batch job, and you want fresher-than-nightly signals for the in-app row. You also
cannot cheaply keep EXACT counts for every (title, country) pair in fast memory when the
event stream is trillions of events.

#### Tier 3: billions of events a day, 190+ countries, near-continuous

This is where the real Netflix machinery shows up, and it splits into a streaming path and a
batch path.

The pipe. Every device emits playback events. Those events ride the Keystone pipeline,
Netflix's petabyte-scale event stream. Keystone is built on Apache Kafka for transport and
Apache Flink for stream processing (CONFIRMED: Netflix Tech Blog and multiple talks; a 2018
O'Reilly talk abstract put Keystone at roughly 3 trillion events and 12 PB per day, and a
2025 data-mesh talk cited 20,000+ Flink jobs and thousands of Kafka topics). A Flink job can
route and aggregate these events into sinks like Kafka, Apache Iceberg (for batch history),
and Druid (for fast time-bucketed queries). So the counting half literally runs as streaming
aggregation: Flink maintains windowed sums of hours per (title, country) as events fly by,
and lands rollups in Druid and history in Iceberg.

The counting trick when exact is too expensive. If you want a near-real-time "what is hot
right now" signal across every country without holding an exact counter for every one of
18,000 titles in every one of 190+ countries in hot memory, you reach for approximate
heavy-hitter structures. The textbook pair:

- A Count-Min Sketch: a small 2D array of counters with a few hash functions. You hash
  (title, country) into each row and increment. To read an estimate you take the minimum
  across rows. It uses fixed memory no matter how many distinct titles you see, and it never
  undercounts, it can only overcount a little (Cormode and Muthukrishnan, 2005). You trade a
  little accuracy for bounded memory.
- A min-heap of the current top K, fed by a Space-Saving style algorithm (Metwally, Agrawal,
  El Abbadi, 2005), or in practice Redis's Top-K, which uses the HeavyKeeper algorithm: a
  count-min-like hash table paired with a min-heap of the top K, with exponential decay so
  heavy hitters stick and small flows fade (Redis Top-K docs). Redis's own docs admit it can
  briefly misrank items and converges over time. For a Top 10 that is fine. Nobody riots if
  rank 7 and rank 8 swap for an hour. The HEAD of the list, number 1 and number 2, is what
  people look at, and the head is exactly what these structures nail, because the whole point
  of a heavy-hitter sketch is that the biggest flows are the ones it tracks most reliably.

This is the key judgment call of the whole feature. EXACT counting of a trillion-event
firehose, per country, continuously, is enormously expensive. APPROXIMATE counting with a
count-min sketch plus a min-heap is cheap and bounded, and the error lands exactly where it
does not matter (the tail, ranks 8 to 10 wobbling) and avoids where it does (the head is
rock solid). So you use approximate structures for the live in-app daily row, and you use the
heavier exact batch job (Flink over Iceberg history) for the audited WEEKLY public numbers
that EY signs off on and the press quotes. Two counting paths, matched to two different needs:
cheap-and-fresh for the home screen, exact-and-slow for the headline. INFERENCE on Netflix's
exact choice of sketch; CONFIRMED that the streaming/batch split (Kafka + Flink + Iceberg +
Druid) is how Netflix is built.

The select half, finally. Once per-country totals exist (exact in Druid/Iceberg, or
approximate in a sketch), picking the Top 10 is the cheap half: a size-10 min-heap scan, or
a Druid TopN, per country. Write the ten IDs to `top10:IN`. For 190 countries that is 190
tiny lists, a few kilobytes total, replicated to the read caches that serve the home screen.
Ananya's 9pm open reads ten IDs. The counting cost was paid hours ago by a Flink job; her
read costs almost nothing.

### 7.4 Walking the real query end to end

"Top 10 in India Today," 25 Oct 2026, 9:10pm:

1. All day, Indian devices emit playback events. "Member a7f watched 80236318 for 2,730
   seconds." These hit Kafka, then Flink.
2. Flink keeps a windowed sum: hours for (80236318, IN, today) ticks up past, say, 9.4
   million hours as the evening peak rolls across India. A count-min sketch tracks the hot
   (title, country) pairs; a min-heap holds the current top ten for IN.
3. The daily batch (and/or the streaming top-k) finalizes the ranking. The Night Agent leads
   India today, Wednesday is third, and two titles that were in yesterday's list fell off.
4. The job writes `top10:IN = [80236318, ..., ...]`, ten IDs, to the serving store.
5. Ananya opens the app. The home-page service reads `top10:IN`, gets ten IDs, attaches
   artwork, ships the row. The TV overlays red 1 through 10.
6. She taps number 3. Her watch session tonight becomes new events that feed tomorrow's list.

The loop is closed: the crowd's watching produces the ranking, the ranking nudges the next
person's watching, and that watching produces tomorrow's ranking.

---

## 8. The retention and habit mechanic

The Top 10 row is a bandwagon machine, and bandwagons are one of the oldest, strongest pulls
in consumer product. The visible number is social proof rendered as a single glyph. A poster
that says "1" is saying "millions of people chose this, you probably should too," and that
lowers the risk of pressing play, which is the exact friction that makes people quit the app.

The metric it moves is engagement and retention, not revenue directly (Netflix is a flat
subscription, so there is no per-click revenue). The mechanism is: reduce "I don't know what
to watch" abandonment, convert an open into a play, and manufacture cultural moments that
make the whole service feel alive and worth keeping. "Squid Game" is the canonical real
example. It hit number 1 in 90-plus countries and became a schoolyard and water-cooler topic
worldwide, and the Top 10 row plus the public weekly site turned that into a visible,
self-reinforcing wave: being on the list put it in front of more people, which kept it on the
list. "Wednesday" and the Wednesday dance did the same. The weekly public top10.netflix.com
list adds a second habit loop aimed at a different user, the press: every week journalists
have a fresh, rankable, quotable dataset, so Netflix gets a recurring news cycle it does not
pay for. One feature, two retention loops, one for viewers (watch what is hot) and one for
the press (write about what is hot), and both feed the same cultural-relevance flywheel.

A real observed subtlety: because the list is national and not personalized, it is the one
place Netflix deliberately breaks its own personalization. That is a bet that FOMO about the
crowd sometimes beats a tailored guess, and the bet is strong enough that Netflix gives the
row a permanent slot and publishes a whole website to amplify it.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code,
plus an embeddable runtime. The exact shape of this feature maps straight onto a "trending"
surface you will eventually want: most-used nodes, hottest effect templates, the "Top 10
shaders people shipped this week." The lesson is about how to count, not what to show.

1. Separate the two halves, and never confuse the cheap one for the expensive one. Ranking
   your top ten templates is trivial (ten numbers in a heap). COUNTING usage across every
   editor session and every embedded runtime instance is the expensive half. If a "trending"
   feature feels hard, it is almost always the counting that is hard, never the sorting. Put
   your engineering budget there.

2. Offline-think, online-lookup, with zero aggregation on the frame path. The editor must
   never run a GROUP BY to render a "trending nodes" panel, the same way Netflix's TV never
   sorts. A background job writes a tiny `trending:this_week -> [ten node IDs]` row, and the
   editor reads ten IDs. Your render loop and your compile step are latency-critical; keep
   every aggregation off them completely.

3. Use approximate heavy-hitter counting and put the error where it does not matter. You do
   not need an exact count of how many times the "fractal noise" node was instantiated this
   week. A count-min sketch plus a min-heap (or Redis Top-K) gives you the top ten with
   bounded memory and a stable head, and the only cost is that ranks 8 to 10 may wobble.
   Nobody cares if the 9th most popular node is actually the 10th. Spend exact counting only
   where a human audits the number (billing, a public leaderboard), and spend approximate
   counting everywhere the number is just a nudge. Netflix runs exactly this split: cheap
   approximate for the in-app row, exact-and-audited for the weekly headline.

4. Count per shard and merge, so locality is cheap. Netflix counts per country. You count per
   project, per team, or per runtime region, then merge small top-k lists into a global one.
   Merging ten-element lists is free; re-scanning the global firehose is not.

5. The visible rank is a persuasion tool, use it deliberately. A "1" on a template will pull
   more clicks than the same template unranked. When you surface "most-used effects," you are
   not just informing, you are steering new users toward paths that already work, which
   shortens their time-to-first-shipped-shader. That is your activation metric. Design the
   number to do that job, and keep the counting cheap enough that you can afford to show it
   everywhere.

Bottom line: a "trending" feature is a counting pipeline wearing a tiny UI. Build the bounded
approximate counter and the precomputed cached top-k first, and the row draws itself.

---

## Sources

- Netflix, "Top 10 things about Netflix Top 10" (methodology overview): https://about.netflix.com/news/top-10-things-about-netflix-top-10
- Netflix, "What We Watched: A Netflix Engagement Report" (18,000+ titles = ~99% of viewing, ~100B hours, Night Agent S1 = 812.1M hours): https://about.netflix.com/en/news/what-we-watched-a-netflix-engagement-report
- Advanced Television, "Netflix launches Top 10 website" (17 Nov 2021 launch of weekly site): https://www.advanced-television.com/2021/11/17/netflix-launches-top-10-wesbite/
- TVLine, "Netflix Changes Up Its Viewership Formula" (June 2023 switch to Views = hours / runtime, 28 to 91 day window): https://tvline.com/ratings/netflix-viewership-changes-wednesday-squid-game-most-popular-1235003018
- Tubefilter, "Netflix Now Counts 2 Minutes Of Watching As A View" (Q4 2019 letter, ~35% higher): https://tubefilter.com/2020/01/22/netflix-how-calculates-view-counts-the-witcher-you-number-of-viewers-q4-earnings/
- BGR, "Why Netflix Used The 2-Minute Rule (And Why It Stopped)" (Nov 2021 switch to 28-day hours viewed): https://www.bgr.com/2192308/what-netflix-2-minute-rule-is-explained/
- Digital TV Europe, "Netflix rolls out Top 10 feature following Mexico and UK trials" (Feb 2020 global rollout, daily country-level): https://digitaltveurope.com/2020/02/25/netflix-rolls-out-top-10-feature-following-mexico-and-uk-trials
- InfoQ, "Netflix: Migrating Stream Processing" (Keystone on Kafka + Flink, stream-processing-as-a-service): https://www.infoq.com/articles/netflix-migrating-stream-processing
- StreamNative, "Building the Next Generation of Real Time Data Pipelines: Data Mesh and Streaming SQL at Netflix" (20,000+ Flink jobs, thousands of Kafka topics): https://streamnative.io/videos/dss-virtual-2025-building-the-next-generation-of-real-time-data-pipelines-data-mesh-and-streaming-sql-at-netflix
- Redis, Top-K probabilistic data type (HeavyKeeper: count-min-like table + min-heap, exponential decay, may briefly misrank): https://redis.io/docs/latest/develop/data-types/probabilistic/top-k/
- Cormode and Muthukrishnan, "An Improved Data Stream Summary: The Count-Min Sketch and its Applications" (2005): http://dimacs.rutgers.edu/~graham/pubs/papers/cm-full.pdf
- Metwally, Agrawal, El Abbadi, "Efficient Computation of Frequent and Top-k Elements in Data Streams" (Space-Saving, 2005): https://www.cs.ucsb.edu/sites/default/files/documents/2005-23.pdf
