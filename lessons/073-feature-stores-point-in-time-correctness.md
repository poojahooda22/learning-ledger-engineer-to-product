# Day 73 — How does Uber serve 10 million ML predictions a second without a single model quietly learning the wrong thing?

**Date:** 2026-09-13
**Difficulty:** Expert
**Topic:** Feature stores and point-in-time correctness. Not a new data structure or a new protocol, but the same discipline problem this ledger met on Day 72: a failure mode that never crashes anything, never pages anyone, and shows up only as a model that is a little worse than it should be, for reasons nobody can see on a dashboard. The mechanism at the center is the point-in-time join, a specialized as-of join that this ledger's earlier work on joins (Day 46, distributed joins across shards) and time-ordering (Day 27, TrueTime and external consistency; Day 31, causal consistency) already built the vocabulary for. What's new is applying "don't let a query see data from the future" to machine learning training data specifically, where the failure is not a wrong row, it's a permanently over-optimistic model.
**Stack relevance:** Rare.lab does not run ML feature stores today, but it already has the exact shape of this problem, just wearing different clothes. A shader graph in Rare.lab's node-based editor gets evaluated twice, once live in the editor's preview (likely an interpreter walking the graph, or a fast in-browser compile, so an artist sees instant feedback), and once for real when it is compiled to the shippable code a customer's page actually runs. If those two evaluations use even slightly different logic (a different float-precision path, a node that's implemented one way for interactive preview and another way for the production compiler, a function available in the editor's sandbox but resolved differently in the embeddable runtime), the artist ships something that "looked right in the editor" and renders differently for the end customer. That is training-serving skew wearing a graphics hat. Section 4 below names the exact fix Uber uses, one shared transformation implementation invoked from both code paths instead of two independent reimplementations, and Section 6 names the feedback loop that makes ignoring this compound quietly over time rather than fail loudly on day one.

---

## 1. The company and the breaking number

**Uber, and the 10,000 features nobody could promise were computed the same way twice.** Uber's ML platform, Michelangelo, went into production around 2016 and Uber first wrote about it publicly in 2017. The problem it was built to solve was not "our models are too slow," it was that Uber had dozens of teams each building their own one-off pipeline to compute the same kinds of signals (a rider's average trip distance, a driver's recent acceptance rate, a restaurant's current prep-time trend for Uber Eats), and every team's pipeline for training a model looked nothing like that same team's pipeline for serving that model live. One offline job, usually Python or Spark, computed a feature by scanning historical data for training. A separate online service, in a different language, recomputed something that was supposed to be the same feature at request time, live. Nobody had a way to prove the two matched.

**The breaking number: it fails silently, at whatever scale you're running.** By the platform's more recent published figures, Michelangelo had grown to over 400 active ML projects, more than 5,000 models in production, more than 20,000 model training jobs a month, and was serving upward of 10 million real-time predictions a second at peak, up from roughly 250,000 predictions a second at sub-10ms P95 latency when the platform first launched. A shared feature store held on the order of 10,000 reusable features consumed across the company. The dangerous part of this number is not its size, it's that a single mismatched feature definition doesn't throw an exception. It doesn't fail a health check. It just means every one of those millions of predictions per second is quietly built on a feature value the model never actually saw during training, and the only symptom is a slow, hard-to-attribute decline in accuracy that could just as easily be blamed on "the market changed" or "the model needs retraining."

**The other half of the breaking number: joining billions of rows without looking into the future.** To train a model at all, Uber has to answer a much harder version of an ordinary database join: for every historical trip, what was the exact value of "driver's acceptance rate" at the exact moment that trip's outcome (the label) happened, not before, and critically, not after. Getting this wrong even slightly, by including a feature value that was only computed or updated after the labeled event occurred, silently leaks the future into the training set. The model then learns a pattern that will never be available to it at real prediction time, tests beautifully offline, and quietly underperforms the moment it goes live, because production can never give it information from the future the way the corrupted training set accidentally did.

---

## 2. Why the naive (demo) design dies

**The obvious version:** every ML team writes its own feature logic twice. Once as a batch job (Python notebook, or a Spark script) that scans historical data to build a training set. Once as a small service, or inline code in the serving path, that recomputes "the same" feature from live production data at request time. Two authors, two languages, two code paths, one intention.

**Death one: the two implementations drift, and nothing detects it.** A data scientist's offline feature might define "trip distance so far" using the full, cleaned, deduplicated trip history. The online service computing "the same" feature at serving time, under a tight latency budget, might use a cheaper approximation, a slightly different rounding rule, or a different definition of "so far" because the production database it's reading from doesn't have the same shape as the offline warehouse. Both versions look reasonable in isolation. Neither team knows the other's version disagrees, because there is no shared source of truth to diff them against. This is training-serving skew: the classic, well-documented failure mode where offline and online features silently diverge, called out explicitly as its own numbered rule ("measure training/serving skew") in Google's widely cited internal machine-learning engineering guidance, precisely because it is common enough across the industry to deserve its own named category of bug.

**Death two: naive joins leak the future, and offline validation can't catch it.** Building a training set naively often means "join the label to the latest known value of each feature," rather than "join the label to the feature value that was true at that instant." At small scale, with a handful of features and a small history, an engineer might eyeball this and catch an obvious leak. At Uber's scale, with roughly 10,000 candidate features, many of them continuously updated by streaming jobs, and billions of historical rows, nobody is eyeballing this by hand. A model trained on leaked future data will show excellent offline accuracy, because it is effectively cheating on a test it wrote for itself, and only reveals the problem in production, after it has already shipped, when it degrades against real time-ordered traffic it was never actually equipped to handle.

**Death three: separately maintained pipelines don't scale with headcount.** Uber's own account of the problem was not primarily "our systems are too slow," it was that data scientists were spending most of their time on plumbing, hand-building and hand-maintaining redundant feature pipelines per team, per model, rather than on modeling. At a company running thousands of models across hundreds of active projects, letting every team reinvent "how do I compute a driver's recent rating" from scratch is not just a correctness risk, it's an organizational scaling failure: the number of near-duplicate, slightly-inconsistent feature pipelines grows roughly with the number of models, not with the number of genuinely distinct signals actually being computed.

**The real-world version:** this is exactly the problem Uber has described publicly as motivating Michelangelo's feature store component, Palette: teams independently reimplementing the same signals, with no shared definition and no guarantee that a model's training-time view of a feature matched its serving-time view, at a company where getting an ETA or a fraud score wrong at 10 million predictions a second is not a rounding error, it's the product.

---

## 3. The architecture

```
Event sources (rider app taps, driver GPS pings, trip completions,
Eats order and delivery events)
  - job: emit the raw facts the entire feature layer is built from
  - analogy: every register in every store phoning in each sale the
    instant it happens, instead of someone auditing receipts later

        |
        v
Streaming ingestion and near-real-time feature jobs (Flink jobs
computing streaming SQL aggregates, continuously updating feature
values as new events arrive)
  - job: keep fast-changing features (a driver's last five minutes
    of acceptance behavior, a restaurant's live prep-time trend)
    fresh to the second, not batched overnight
  - analogy: a scoreboard that updates the instant a point is
    scored, instead of only being posted after the game ends

        |
        v
Batch feature jobs (Spark/Hive SQL scanning the historical
warehouse for slower-moving features: a rider's lifetime average
trip distance, a driver's account age)
  - job: compute the features that don't need second-by-second
    freshness, over the full depth of historical data, at much
    lower cost per row than a streaming job would
  - analogy: doing full inventory once a night rather than
    re-counting the whole warehouse after every single sale

        |
        v
Shared transformation logic (one definition per feature, expressed
once, e.g. through Michelangelo's DSL/"Transformer" layer)
  - job: be the single place a feature's meaning is defined, so the
    exact same logic runs whether it's being computed for a
    historical training row or for a live request a millisecond
    from now
  - analogy: one master recipe card taped to the wall that every
    cook, day shift or night shift, is required to follow exactly,
    instead of each cook trusting their memory of "how we make this"

        |
        v
Dual-tier feature store (Hive for offline/historical storage
optimized for big scans and joins; Cassandra for online storage
optimized for single-key point lookups under strict latency)
  - job: store the same logically-defined features twice, in two
    physical shapes tuned for two very different access patterns,
    rather than forcing one storage engine to be good at both
  - analogy: keeping a library's full archive in a warehouse for
    researchers doing deep lookups, while keeping this week's most
    requested titles on a fast-access shelf right by the front desk

        |
        v
Point-in-time (as-of) join for training-set construction
  - job: for every labeled historical event, pull the feature
    values that were true at that exact moment, and reject any
    feature value that was only computed or updated afterward
  - analogy: reconstructing what the scoreboard actually said at
    minute 43 of a game, not what it says now, when building a
    highlight reel that has to feel honest

        |
        v
Online prediction service (queries the Cassandra-backed feature
store live, at request time, via the same shared transformation
logic, to augment the request with fresh feature values before
scoring)
  - job: serve the model's input vector, made of the same features
    it trained on, computed the same way, in single-digit
    milliseconds, at millions of requests a second
  - analogy: a ticket window that looks up your account balance in
    real time before approving a purchase, using the exact same
    account-balance definition the bank's own statements use

        |
        v
Feature and prediction monitoring (drift dashboards comparing the
distribution of features and predictions in production against
what training expected)
  - job: catch the skew that slips through despite shared code,
    because a shared definition can still drift if an upstream
    event schema quietly changes
  - analogy: a bathroom scale that gets checked against a known
    reference weight periodically, not trusted forever just because
    it worked correctly on day one
```

---

## 4. The transferable mechanisms

- **One shared transformation, invoked from two contexts, instead of two separate reimplementations.** The single highest-leverage fix here is not a smarter algorithm, it's an organizational one: define a feature's logic exactly once, in a form that can be executed identically by an offline batch job and an online serving path, rather than trusting two different teams, in two different languages, to independently arrive at the same answer forever. This is the same instinct behind Day 6's guarded ledger write and Day 12's idempotency key: don't rely on two callers agreeing by convention, make the system itself incapable of disagreeing.

- **Point-in-time (as-of) joins as the correctness primitive for time-ordered training data.** Instead of joining a label to "whatever a feature currently equals," the join is keyed on (entity, timestamp) and explicitly excludes any feature value whose own timestamp comes after the label's. This is the ML-training-specific cousin of Day 27's external consistency and Day 31's causal consistency: the guarantee isn't about global agreement on wall-clock time, it's about never letting an effect appear to have happened before its cause.

- **Storage tiers matched to access pattern, fed from one canonical definition.** A warehouse-style store (Hive) is good at scanning billions of rows for a training join; it is terrible at single-key point lookups in single-digit milliseconds. A wide-column store (Cassandra) is the opposite. Rather than forcing one engine to do both badly, the same logical feature is materialized into both, and the shared transformation layer is what keeps them from silently disagreeing about what "the same feature" means. This is Day 19's caching-strategy instinct (right storage engine for the read pattern that will actually hit it) applied to two producers of ground truth instead of one producer and one cache.

- **Freshness as a tunable choice per feature, not a platform-wide constant.** A feature like "driver's rating over their lifetime" barely changes minute to minute, and a nightly batch job is more than fresh enough. A feature like "driver's acceptance rate in the last five minutes" is stale and nearly useless if computed by the same nightly batch job. Routing each feature to a streaming (Flink) or batch (Spark/Hive) pipeline based on how fast it actually changes avoids paying streaming-infrastructure cost for signals that don't need it, while still giving genuinely time-sensitive signals the freshness they require.

- **A shared, versioned feature registry to make reuse safe, not just possible.** Making ~10,000 features available to every team only helps if a team adopting an existing feature can trust its definition won't change out from under their already-trained model. A central registry with versioned feature definitions turns "reuse" from a risk (silent redefinition breaking someone else's model) into a genuine efficiency win (thousands of models sharing a much smaller number of actually-distinct, well-tested signals).

- **Monitor the thing you can't fully prevent.** Even with shared code, upstream schema changes, partial outages, or a backfill gone wrong can still cause offline and online feature distributions to drift apart. Drift dashboards comparing production feature and prediction distributions against training-time expectations are the safety net underneath the shared-logic guarantee, catching the cases the architecture alone doesn't.

---

## 5. The trade-offs

**Consistency versus cost, paid as literal duplicated storage and compute.** Materializing every feature into both a scan-optimized offline store and a latency-optimized online store, kept in sync by a shared transformation layer, costs meaningfully more than picking one storage engine and living with its weaknesses on the other side. Uber accepts this cost because the alternative, a model quietly trained on data it will never see in production, is a correctness failure with no error message, which is far more expensive to discover after the fact than the storage bill is to pay up front.

**Freshness versus infrastructure complexity, chosen per feature.** Routing every feature through a streaming pipeline would maximize freshness uniformly but would mean running expensive, operationally heavier streaming infrastructure (Day 40's exactly-once stream processing machinery) for signals that genuinely do not need it. Uber's answer is not "streaming everywhere" or "batch everywhere," it's a deliberate per-feature choice, accepting operational complexity (two different pipeline types to maintain) in exchange for not over-paying for freshness nobody needs, and not under-paying for freshness a fraud or ETA signal genuinely requires.

**Availability versus strict correctness on the online path.** The online feature store (Cassandra) is a wide-column store built around tunable, generally eventually-consistent reads under normal operation, favoring being available and fast over guaranteeing every reader sees the absolute latest write instantly. For a prediction service answering in single-digit milliseconds, a feature value that is a few hundred milliseconds behind the absolute latest streaming update is a non-event; the model was never trained to expect microsecond freshness anyway. This is the same CAP dial Day 22's leaderless replication and Day 14's multi-region active-active have turned before, applied here to feature freshness instead of a shopping cart or a viewing position.

---

## 6. The systems-thinking lens

The feedback loop worth naming is **silent skew compounding into feedback poisoning.** A small mismatch between a feature's training-time definition and its serving-time computation does not crash anything and does not show up as an error rate. It shows up, if at all, as a model that performs slightly worse than its offline evaluation promised. Left unmeasured, that gap doesn't stay a one-time cost: many production ML systems feed their own live predictions and the resulting user behavior back into the next round of training data (an ETA model's predictions shape what riders do next, which becomes part of the data the next model version trains on). A model quietly degraded by skew produces subtly worse live behavior, which becomes subtly worse future training data, which trains the next model on an already-corrupted signal. Nothing alarms. Nothing pages anyone. The system just gets a little worse, repeatedly, in a direction nobody chose.

This is the same shape of danger this ledger named on Day 72 as latent-failure normalization, a gap that grows precisely because nothing forces it to be measured, just relocated from infrastructure redundancy to model correctness. The senior fix here is identical in spirit: don't hope two independently maintained things stay in sync, make it structurally impossible for them to silently diverge, and instrument the seam anyway in case something still slips through. One shared transformation logic, invoked from both the offline and online path, removes the most common source of skew entirely rather than trying to catch it after the fact. Point-in-time joins remove the future-leakage failure mode at the training-set-construction step, before a corrupted model can even be trained. And drift monitoring exists precisely because "we made it structurally hard to diverge" is not the same claim as "it is now impossible to diverge," the same honest distinction Day 72 drew between having a backup and having a tested backup.

For Rare.lab, the same fix applies directly to the editor-preview versus compiled-runtime seam described above: the way to prevent "it looked right in the editor" from ever becoming a customer-facing bug is not more QA passes after the fact, it's making the preview and the compiled shippable output run through the same underlying node-evaluation logic, so there is structurally nothing left to drift.

---

## Sources

- [Meet Michelangelo: Uber's Machine Learning Platform, Uber Engineering Blog](https://eng.uber.com/michelangelo-machine-learning-platform/): primary source for Michelangelo's original 2017 architecture, the motivating problem (teams independently rebuilding inconsistent feature pipelines, training-serving skew as a named failure mode), the Hive/Cassandra dual-store design, and the original-era throughput and latency figures (on the order of 250,000 predictions/sec at peak, sub-10ms P95). Direct fetch was blocked by this session's network egress policy (uber.com and eng.uber.com are both blocked); the account here is drawn from search-indexed summaries and secondary write-ups rather than a full direct read, consistent with how this ledger has flagged network-blocked primary sources before (Day 69 through Day 72).
- [Scaling Machine Learning at Uber with Michelangelo, Uber Blog](https://www.uber.com/en-US/blog/scaling-michelangelo/): primary source for the more recent scale figures used in Section 1 (400+ active ML projects, 20,000+ monthly training jobs, 5,000+ production models, roughly 10 million real-time predictions per second at peak). Direct fetch blocked this session; summarized from search-indexed excerpts.
- [Michelangelo Palette: A Feature Engineering Platform at Uber, InfoQ](https://www.infoq.com/presentations/michelangelo-palette-uber/): secondary source (conference talk write-up) corroborating Palette's architecture: Hive for offline/historical storage, Cassandra for low-latency online storage, batch features via Spark/Hive SQL, near-real-time features via Flink streaming SQL, and a shared Transformer/DSL layer executing identical feature logic offline and online. Direct fetch blocked this session; summarized from search-indexed excerpts.
- [Michelangelo PyML: Introducing Uber's Platform for Rapid Python ML Model Development, Uber Blog](https://www.uber.com/en-US/blog/michelangelo-pyml/): referenced for platform context on Michelangelo's evolution; direct fetch blocked this session.
- [Rules of Machine Learning: Best Practices for ML Engineering, Google Developers](https://developers.google.com/machine-learning/guides/rules-of-ml): widely cited industry reference that names training/serving skew as its own explicit, numbered engineering rule, used here to establish that this is a well-documented, cross-industry failure mode rather than an Uber-specific quirk. Direct fetch blocked this session (developers.google.com blocked by network egress policy); referenced from established public knowledge of this document's content rather than a fresh direct read.
- Day 6 (Stripe correctness under load), Day 12 (idempotency and exactly-once delivery), Day 14 (multi-region active-active), Day 19 (caching strategies and invalidation), Day 22 (leaderless replication, quorums, and vector clocks), Day 23 (content-addressed storage and Merkle DAGs), Day 27 (TrueTime, Spanner, and external consistency), Day 31 (session guarantees and causal consistency), Day 40 (stream processing, exactly-once, and watermarks), Day 46 (distributed joins across shards), Day 72 (chaos engineering and regional failover): the ledger's own prior lessons this one directly reuses and recombines.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of every Uber engineering blog post and the Google Rules of ML page cited above (uber.com, eng.uber.com, and developers.google.com were all blocked, along with medium.com, applyingml.com, and hellosde.com, three secondary sources this session also attempted to fetch for cross-checking). The architectural claims here (the Hive/Cassandra dual-store split, the shared Transformer/DSL logic, the batch-versus-streaming feature split, and the throughput/latency figures in both their original 2017 and more recent forms) are corroborated across multiple independent search-indexed summaries of Uber's own posts, an InfoQ conference write-up, and an MLOps-focused database entry, rather than quoted from a full direct read of any single primary source. The core claims are treated as solid because the specific numbers (250K then 10M predictions/sec, sub-10ms P95, roughly 10,000 shared features, 5,000+ production models) recur consistently across independently-indexed summaries rather than appearing in only one place.
