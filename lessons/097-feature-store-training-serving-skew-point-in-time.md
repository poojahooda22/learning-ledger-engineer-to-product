# Day 97: A model that scored 0.91 offline scores 0.78 live, and nothing is "broken". How feature stores (Uber Palette, DoorDash, Feast) keep training data and live data identical at 10 million reads per second

**Date:** 2026-10-24
**Difficulty:** Advanced (the data layer under every ranking and recommendation lesson: Days 7, 33, 54, 56, 61)
**Topic:** Every ranking system so far (feed, search, ads, leaderboards) quietly assumed "the model gets its inputs". Today is where those inputs live, how they are computed ONCE, and how they reach the model in under 10 ms.
**Stack relevance:** Rare.lab will rank presets, suggest nodes and predict "will this shader hit 60 fps on this phone". All of that needs features. Section 9.
**A note on the pasted transcript:** the LeetCode mock interview is already taught in [Day 74](074-leetcode-online-judge-end-to-end.md) (sandbox Day 50, leaderboard Day 54). I did not repeat it. Section 0 uses its interview recipe on today's topic, and Section 10 explains it again in plain language, as you asked.
**Honesty labels:** CONFIRMED = read in a search excerpt or summary today. Important limit: direct page fetches failed (network blocked), so I never read a full primary article. Uber, DoorDash and Feast facts come from search excerpts and secondary summaries. INFERENCE = my reasoning, labeled. MEMORY = standard knowledge I did not re-read today. Numbers from vendor blogs are flagged.

---

## 0. The interview recipe applied to a feature store

The LeetCode candidate went: functional requirements, non-functional requirements, entities, API, naive design, then deep dives. Same recipe.

1. **Functional.** Define a feature once (for example "orders this user placed in the last 30 days"). Get its value for millions of historical training rows. Get its latest value for one user in a live request.
2. **Non-functional.** Online read p99 under about 10 ms. Training and serving values must match. No future data leaks into training. Features reusable by many teams. Fresh enough (seconds to hours, depending on the feature).
3. **Entities.** Entity (user, restaurant, product), feature, feature view or group, timestamp, offline store, online store, materialization job.
4. **API.** `get_historical_features(entity_rows_with_timestamps, feature_list)` for training. `get_online_features(entity_keys, feature_list)` for serving. Plus `write(feature, key, value, event_time)`.
5. **Naive design.** The data scientist writes SQL in a notebook for training. The backend engineer rewrites the same logic in Java for the live service.
6. **Deep dives.** Skew, point-in-time joins, two stores, freshness, hot keys, scale tiers.

---

## 1. Named companies and the breaking numbers

- **The skew number (the one that bites).** You train a "will this delivery be late" model. Offline AUC (a 0 to 1 score of ranking quality, 0.5 is a coin flip) is 0.91. In production it is 0.78. Same model file, same users. The gap is not a bug in the model. It is two different pieces of code computing "average prep time, last 7 days" slightly differently. **Illustrative arithmetic (INFERENCE, not a published number):** if 3 of your 40 features drift by even a few percent, the model, which memorized exact relationships, mis-scores every request.
- **DoorDash (CONFIRMED via search excerpts of the 2020 DoorDash engineering post and a RedisConf 2021 talk).** The online feature store ran on Redis and was close to running out of capacity. The team benchmarked five key-value stores on cost and performance, kept Redis, and rebuilt how features are serialized (protocol buffers, string handling, Snappy compression). Reported: about 3x cost reduction and a 38 percent drop in Redis latency. A secondary newsletter summary says CPU fell from 208 to 72 cores per 10 million reads per second (65 percent). I could not verify that last figure against DoorDash's own page. The talk topics (CONFIRMED titles) include hot keys, stampedes and predictive cache expiry.
- **Uber Michelangelo and Palette (CONFIRMED via a ZenML summary of a 2019 Uber talk, plus the title and excerpt of Uber's own "Palette meta store journey" post).** Palette is a central feature store. Offline history lives in Hive. Low-latency online lookups come from Cassandra (the Uber post excerpt also says Cassandra/Redis clusters, with multiple Cassandra clusters for large use cases). Batch features are ingested offline and auto-synced to the online store. Streaming features land in Cassandra first and are ETL'd to Hive with backfill. A secondary article claims 10 trillion feature computations per day for Michelangelo; I could not verify it.
- **Google (CONFIRMED via mirrors of "Rules of Machine Learning", not the official page).** Rule 29: the best way to train like you serve is to save the features used at serving time and log them for training. Rule 32: reuse code between the training and serving pipelines. Rule 37: measure training/serving skew.

**The breaking number for the naive design: two code paths for one feature gave a 0.91 offline vs 0.78 live gap, and one online store at 10 million reads per second needed 208 CPUs before it was optimized.**

---

## 2. Why the naive design dies

**Failure 1: Two implementations of one feature (training-serving skew).**
- Notebook SQL says `AVG(prep_minutes) over 7 days, excluding cancelled orders`. The production Java says `AVG(prep_minutes) over the last 168 hours`, and includes cancelled orders because a flag was not in the live row.
- Analogy: two chefs given one recipe card in different languages. Both swear it is the same dish. Customers can taste the difference.

**Failure 2: Time travel in the training data (label leakage).**
- You build training rows by joining "orders" to "user stats" using the latest stats table. A row for an order placed in March now sees the user's October order count. The model learns from the future, scores great offline, and fails live.
- Analogy: grading an exam with the answer key visible. Great score, zero knowledge.

**Failure 3: Every request recomputes from raw data.**
- Live request needs "orders in last 30 days" for user 42. Scan the orders table: 200 ms. Ranking 500 candidates, each with 30 features: 15,000 lookups. At 5,000 requests per second that is 75 million lookups per second against a transactional database that was sized for orders.
- Analogy: re-adding the whole shop receipt book every time a customer asks "how much did I spend this month".

**Failure 4: Every team rebuilds the same features.**
- Ten teams each compute "user lifetime orders" ten ways. Nobody knows which is right. (INFERENCE, a standard motivation stated by feature store vendors.)

---

## 3. The architecture, top to bottom

```
Raw sources:  event stream (Kafka), transactional DB (Postgres), warehouse tables
   |
   v
Feature definitions  (ONE place: name, entity key, transformation, TTL, owner)
   |                       job: write the recipe once. Analogy: the single recipe card.
   |
   +--> Batch pipeline (Spark/SQL, hourly or daily)
   |        computes features over history WITH event timestamps
   |
   +--> Streaming pipeline (Flink/Kafka consumer, seconds)
   |        updates fast-changing features (orders in last 10 min)
   v
OFFLINE STORE  (Hive, BigQuery, Parquet on object storage, Delta)
   |   job: keep EVERY past value with its timestamp. Big, cheap, slow.
   |   used by: training jobs doing point-in-time joins.
   |
   |  materialization job copies only the LATEST value per key
   v
ONLINE STORE  (Redis, Cassandra, DynamoDB)
   |   job: one key, one tiny row, p99 under ~10 ms. Small, hot, fast.
   |   Analogy: the offline store is the filing cabinet, the online store is the sticky note on your monitor.
   v
Feature serving API  (stateless service, batch multi-get)
   |   job: fetch 30 features for 500 candidates in one round trip
   v
Model server / ranker  (the app tier from Days 7, 33, 44, 61)
   |
   v
Logged request: the exact feature values USED, written to the offline store
        job: Google Rule 29. Training next week reads what serving saw last week.
```

Annotations:
- **Two stores, one definition.** The offline store keeps history. The online store keeps only the latest value. Different jobs, different databases, one recipe.
- **Statelessness.** The serving API holds no data. Any replica can answer. Scale by adding replicas.
- **The log closes the loop.** It is the cheapest skew killer, since training reads what serving truly saw.

---

## 4. The transferable mechanisms (6)

1. **Define once, run twice.** One feature definition compiles to a batch query (for history) and a streaming or lookup path (for live). Uber and Feast both exist for this (CONFIRMED in concept). Example: `orders_30d` is written once; Spark computes it for 2 years of history, and a stream job keeps today's value fresh.
2. **Point-in-time (as-of) join.** For a training row with label time T, pick the feature value with the latest timestamp that is at or before T (CONFIRMED rule via a Feast-focused summary). Worked example: Asha ordered at 2026-03-05 18:00. Her `orders_30d` was 4 on 03-01 and 9 on 03-20. The row must get **4**, not 9. A plain join on user_id gives the latest value, 9: leakage.
3. **Offline/online split (history vs latest).** Offline keeps all versions (large, scanned in bulk). Online keeps one version per key (small, point-read). Example: 100 million users x 50 features x 8 bytes = 40 GB online, but 2 years of hourly history is 17,520 x larger offline (arithmetic).
4. **Materialization with TTL.** A scheduled job copies the latest value to the online store. A TTL (time to live) on each feature view says how old a value may be. Past it, the lookup returns null rather than a stale lie. Analogy: milk with a date on it.
5. **Log what you served.** Save the feature vector used for each prediction. Then you can retrain on exactly that, and also compare it with a recomputed value to detect skew (Google Rules 29 and 37).
6. **Serialize small, coalesce reads.** DoorDash's lever (CONFIRMED title and summary): compact binary encoding plus compression to triple capacity, one multi-get for many keys, and stampede protection and hot-key handling when a popular store gets hit by many requests at once. Same ideas as Day 80 (request coalescing) and Day 16 (hot keys).

---

## 5. The trade-offs accepted

- **Freshness vs cost.** A streaming pipeline gives second-old features but costs always-on compute and operational pain. A nightly batch is cheap but a user who ordered 5 minutes ago still looks like a new user. Choose per feature: price of a product (hours is fine), orders in the last 10 minutes (seconds needed).
- **Consistency vs availability, per data type.** Online features: availability wins. If the store is slow, serve a default or a slightly stale value, since a ranking that is 1 percent worse beats a 500 error. Offline training data: consistency wins, since the point-in-time join must be exact and reproducible. A payment balance would be the opposite (Day 82), but a feature is an input to a guess, so staleness is tolerable.
- **Cost vs latency.** RAM-backed online stores (Redis) are fast and expensive per GB. Disk-based ones (Cassandra, DynamoDB) are cheaper and a few ms slower. DoorDash's compression work was literally about shrinking the RAM bill.
- **Reuse vs ownership.** A shared store means one team's bad feature can hurt another model. You need owners, versions and monitoring (INFERENCE).
- **Storing everything offline is cheap, until a deletion request arrives** (Day 90): a user's history now lives in the feature log too.

---

## 6. The systems-thinking lens

**The loop that causes failure: the silent feedback loop of stale or skewed features.**
- A model trained on skewed features ranks worse, so users click less on good items, so the training data (built from clicks) gets biased toward whatever was shown. The next model learns from that biased data and is confidently wrong. Nothing crashes, so there is no alert. (INFERENCE, a known feedback-loop pattern in recommender systems.)
- A second loop is the **online store stampede**: a popular key (a viral restaurant) expires in cache, many requests recompute or reload it at once, the store slows, requests time out and retry, adding load (retry spiral, Day 13; thundering herd, Day 16). DoorDash's talk topics (stampedes, predictive expiry, hot keys) are the real-world version.

**The senior fix BREAKS the loop, it does not add servers:**
- Remove the second code path, so skew cannot arise at all (define once).
- Log what was served and measure the difference between served and recomputed values, so the silent loop becomes a visible alert.
- Add a TTL and a default: stale input becomes a known fallback, not a hidden lie.
- Add jittered expiry, request coalescing and hot-key local caches, so one popular key cannot synchronize the herd.

---

## 7. Scale story at three tiers

- **1,000 users, one model:** compute features inside the request with a SQL query. A feature store is over-engineering. The one habit worth keeping: write the feature function once in a shared module so training and serving import the same code. Example: `def orders_30d(user_id, as_of)` used by both.
- **100,000 users, 5 models, 3 teams:** inline SQL now hurts. Live queries hit the transactional DB (Failure 3), and two teams define "active user" differently. Add: a feature table with timestamps (offline), a Redis or Postgres table of latest values (online), a nightly materialization job, and the point-in-time join in training. Open-source Feast does exactly this.
- **10 million+ users, hundreds of models (Uber, DoorDash):** walls in order: (1) the online store is too big or hot for one node, so shard by entity key and compress (DoorDash); (2) streaming freshness needs a separate pipeline with backfill so history matches live (Uber Palette, CONFIRMED in concept); (3) hot keys and stampedes, so local caches and jittered expiry; (4) governance: owners, lineage, versions, deletion; (5) cost, since RAM-heavy online stores dominate the bill. Real breaking number from DoorDash: CPUs per 10 million reads per second (208 down to 72 per the secondary summary).

---

## 8. A worked example, end to end

Ravi in Pune opens a food app at 8:10 pm. The ranker must order 500 restaurants. One feature: `restaurant_avg_prep_minutes_7d`.

**Training week (offline):**
1. A batch job computes `restaurant_avg_prep_minutes_7d` per restaurant per hour and stores `(restaurant_id, event_time, value)`. The Biryani House row on Tuesday 7 pm says 22.4, and on Wednesday 7 pm says 26.0.
2. A training row is an order placed Tuesday 7:30 pm. The point-in-time join picks the latest value at or before 7:30 pm Tuesday: 22.4. Not 26.0 from Wednesday.
3. The model learns "22.4 minutes prep, evening, rain: 31 minutes door to door".

**Serving (online):**
1. A materialization job copies each restaurant's latest value into Redis every 15 minutes: key `rest:BH42`, field `prep_7d`, value 26.0, TTL 2 hours.
2. Ravi's request arrives. The ranker calls the serving API with 500 restaurant ids. One multi-get returns 500 x 30 values in about 5 to 8 ms (target; INFERENCE).
3. The model scores, the server sorts (never the phone), Ravi sees the list.
4. The feature vector used is logged to the offline store.

**The skew detector:** next day, a job recomputes prep averages from raw orders as of the logged time and compares with the logged value. If 4 percent of rows differ by more than 1 percent, alert. Cause found: the batch job excluded cancelled orders, the stream job did not. One definition, two bugs in two paths, caught in a day instead of a quarter.

---

## 9. Map to Rare.lab's stack

**Already used, without calling it a feature store:**
- **Content-addressed immutable scene JSON on R2 plus a manifest.** This is exactly the offline-store idea: immutable versions with a pointer to the latest. A training snapshot could be "manifest version at time T", so point-in-time reproducibility comes for free (Day 23, Day 93).
- **Supabase Postgres with RLS.** A perfectly good online store at the 1,000 to 100,000 tier: a `preset_features` table keyed by preset id, read by primary key.
- **Embeddable runtime with one shared WebGL context.** It can report real measurements (compile time, frame time), which are the best features you will ever have.

**The ceiling (INFERENCE, measure before acting):**
1. **You will want a "will this graph hit 60 fps on this device" predictor.** Features: node count, texture reads, loop bounds, device GPU class, browser. Train on logged runtime measurements. **Define the graph-to-features function ONCE** (a pure function in your compiler package) and import it in both the training script and the live editor. That single shared module is the entire skew defense at your tier.
2. **Point-in-time trap for you:** preset ratings and "fps median" change over time. When training a gallery ranker, join to the rating as of the moment of the impression, not today's rating (otherwise the rating leaks the future, the same trap as the Amazon reviews teardown of 2026-10-24).
3. **Do not build Redis plus Cassandra.** At your scale, a Postgres table of latest values plus an append-only events table (the offline store) is enough. Add a materialization step only when read latency or load on the main DB shows up in your dashboards.
4. **Log the features served** with every suggestion shown. It costs one JSON column and gives you retraining data and a skew check later.
5. **Latency budget:** the editor runs at 60 fps (16.6 ms per frame). Never call a remote feature lookup on the frame path. Fetch suggestions async, cache by graph hash (content-addressed, so cache hits are exact).

Concrete first step: write `extractGraphFeatures(graph)` as one pure function in the compiler package, log its output next to every measured fps sample, and add a test that runs it on a frozen graph and compares to a stored snapshot.

---

## 10. References, summarized in plain language

1. **Google, "Rules of Machine Learning: Best Practices for ML Engineering" (Martin Zinkevich).** I saw only mirrors and excerpts, not the official page. Key rules for us: Rule 29 (save the features you used at serving time and log them for training), Rule 32 (reuse code between training and serving), Rule 37 (measure training/serving skew). Plain version: the cheapest way to avoid skew is to train on what the live system actually saw.
2. **DoorDash engineering, "Building a Gigascale ML Feature Store with Redis" (Arbaz Khan, 2020) and the RedisConf 2021 session by Zohaib Sibte Hassan and Arbaz Khan.** Seen via search summaries only. Story: the Redis feature store was near capacity, five key-value stores were benchmarked, Redis stayed, and a new compact serialization (protobuf plus Snappy and string handling) gave about 3x capacity and 38 percent lower latency. The talk adds hot keys, stampede handling and predictive expiry. Lesson: at 10 million reads per second, bytes per row and cache stampedes are the problem, not the model.
3. **Better Stack newsletter, "How DoorDash Improved Redis to Handle 10+ Million Reads per Second."** Secondary. Adds the 208 to 72 CPU figure and says CockroachDB was used offline. Unverified against DoorDash; use for orientation only.
4. **Uber, "Palette meta store journey" and a ZenML summary of Uber's Michelangelo Palette talk (2019).** Palette: one place for curated and crowd-sourced features, Hive offline, Cassandra (and Redis) online, batch auto-synced to online, streaming features land online first then backfill to offline. Lesson: the sync direction differs by feature type, and backfill keeps history honest.
5. **Feast documentation (open source feature store), via search excerpts.** The point-in-time join rule, offline and online store split, and materialization of only the latest value. Lesson: you can adopt the pattern without building Uber's platform.
6. **Introductory articles (apxml course page on point-in-time correctness, Conduktor glossary, Atlan and Kanerika overviews, Databricks Feature Store description).** Beginner-level restatements of the same ideas. The Atlan 10-trillion and 90 percent figures are unverified. Lesson: the vocabulary is standard across vendors even though the numbers are marketing.
7. **Aerospike Feast benchmark blog.** Vendor-authored comparison; the post itself warns the competitor numbers were taken from different hardware and times. Lesson: never trust cross-vendor latency tables, benchmark with your own key sizes.
8. **Not retrieved today, would read next:** the original Uber Michelangelo post (2017) and Palette talks; Airbnb Zipline (no results surfaced); Chip Huyen's "Designing Machine Learning Systems" chapter on feature stores; Tecton's blog on point-in-time joins. Listed so you can find them, not summarized as if I read them.
9. **The pasted LeetCode mock interview (YouTube).** A Google engineer designs LeetCode. Functional: list problems, view and code in any language, submit with an instant verdict, a live contest leaderboard. Non-functional: availability over consistency, low latency (about 2 to 5 seconds), isolation of untrusted code, 100k contest users and millions overall, no single point of failure. Entities: problem, user, submission, leaderboard. REST APIs with pagination. DynamoDB because there are no joins. Running user code directly on the API server was rejected (security, CPU hogging, no fault isolation), VMs were heavy, containers per language chosen, with the interviewer noting VMs isolate better and suggesting serverless. A queue sits between API and runners with exponential retry, a Redis cache serves the leaderboard instead of polling the database, test cases are stored language-neutral and serialized per language, and the database gets a primary with replicas. Full teaching: [Day 74](074-leetcode-online-judge-end-to-end.md). Link to today: the test cases "stored language-neutral, serialized per language" is the define-once, run-twice idea. One definition of a test (the recipe), many language-specific executions, so the verdict is the same everywhere. A contest ranker that scored "solve speed" with one formula in the leaderboard and another in the profile page would show the same skew.

---

## 11. One-line takeaway

**Write each feature once, store its history with timestamps for training and only its latest value for serving, join training rows to the value as of that moment, and log what you served: skew then becomes a measured number instead of a silent 13-point accuracy loss.**

## Sources

- [Google Rules of Machine Learning (mirror summary, DEV Community)](https://dev.to/steminstructor/google-s-machine-learning-crash-course-43-rules-of-machine-learning-summary-2f0p)
- [Rules of Machine Learning (SlideShare)](https://www.slideshare.net/slideshow/rules-of-machine-learning/244263148)
- [Google production ML testing guide](https://developers.google.cn/machine-learning/testing-debugging/pipeline/production?hl=en)
- [DoorDash engineering, feature store posts](https://careersatdoordash.com/blog/tag/feature-store/)
- [DoorDash engineering, Redis posts](https://careersatdoordash.com/blog/tag/redis/)
- [RedisConf 2021, Redis as a scalable feature store](https://redis.com/redisconf/sessions/rc21-s4/)
- [Better Stack, How DoorDash Improved Redis to Handle 10+ Million Reads per Second](https://newsletter.betterstack.com/p/how-doordash-improved-redis-to-handle)
- [Uber, Palette meta store journey](https://www.uber.com/us/en/blog/palette-meta-store-journey/)
- [ZenML, Uber Michelangelo Palette case study](https://www.zenml.io/mlops-database/uber-michelangelo-michelangelo-palette-feature-engineering-platform-for-consistent-offline-training-and-low-latency-onli)
- [Feast quickstart](https://docs.feast.dev/v0.21-branch/getting-started/quickstart)
- [apxml, point in time correctness](https://apxml.com/courses/feature-stores-for-ml/chapter-3-data-consistency-quality/point-in-time-correctness)
- [Conduktor, Feature Stores for Machine Learning](https://conduktor.io/glossary/feature-stores-for-machine-learning)
- [Aerospike Feast benchmark (vendor)](https://aerospike.com/blog/aerospike-feast-benchmark)
