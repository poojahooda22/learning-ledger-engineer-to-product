# Day 62 — How do you guarantee a username can only exist once, when the table that stores users is sharded by user ID, not by username?

**Date:** 2026-08-23
**Difficulty:** Expert
**Topic:** Global uniqueness constraints on a sharded database: why "make this field unique" is trivial on one box and a genuinely hard distributed-systems problem the instant the table holding it is split across many machines. The forcing example is Discord's 2022 to 2024 "Pomelo" migration, moving hundreds of millions of accounts off `Username#1234` (duplicate usernames disambiguated by a random 4-digit discriminator) onto a single globally unique `@username`, a retroactive uniqueness constraint imposed on an already-sharded population, rolled out in gated cohorts over nearly two years rather than flipped on for everyone at once. The concrete breaking number comes from a different company hitting the same wall from the other direction: Meta's Threads app signing up 1 million users in its first hour and 100 million within five days in July 2023, every one of them typing a username that has to be checked, reserved, and committed with zero duplicate tolerance, worldwide, under sustained signup rates a naive check-then-insert design cannot survive. Why a plain `UNIQUE` index, which solves this completely on one database, silently stops enforcing anything real once the table is sharded by a different key than the one that needs to be unique. How Twitter's Manhattan database, Cassandra's lightweight transactions, DynamoDB's conditional writes, and MongoDB's documented sharding rules all converge on the same real mechanism: shard a small, separate index by the unique field itself, and make the one write into it atomic.
**Stack relevance:** Rare.lab doesn't have this problem yet, a single logical Supabase Postgres instance with a real `UNIQUE` constraint is completely sufficient for any handle or slug it stores today, the same honest "you don't need the heavy machinery yet" verdict Day 61 gave the in-editor node search. But the moment Rare.lab ships public creator handles or shareable slugs for gallery shaders, a field the *user* picks rather than one the system generates, this lesson's mechanism, not its infrastructure, is the thing worth adopting now: reserve the unique string in its own narrow table before writing anything else about the entity, so the shape survives unmodified if Postgres is ever sharded later.

---

## 1. The company and the breaking number

**Discord, 2022 to 2024, and Meta's Threads, July 2023, approached from opposite directions.** Discord launched in 2015 with a username model that never needed global uniqueness: every account was `Username#1234`, a display name plus a random four-digit discriminator, so ten thousand different people could all be `Alex#0001` through `Alex#9999` and the system never had to resolve a collision, because the discriminator did that work. By the early 2020s Discord had accumulated an enormous account base, commonly cited around 150 million monthly active users during the Pomelo period, with aggregator estimates putting total registered accounts near 560 million by 2023 and roughly 200 million monthly actives that year. In May 2023 Discord announced it was scrapping the whole discriminator scheme. According to Discord's own blog post announcing the change, more than 40% of users didn't know or remember their own discriminator, and roughly half of all friend requests failed to connect the right person because someone typed the wrong discriminator or the wrong casing. The replacement was a single, lowercase, 2-to-32-character `@username`, alphanumeric plus period and underscore only, and critically, globally unique across the entire platform, no discriminator left to disambiguate a collision.

That is a retroactive uniqueness constraint imposed on a population that was never designed to need one, and the rollout shows the engineering team treated the claiming process itself as the dangerous part, not just the storage. Per Discord's support documentation and corroborating rollout-tracking sources, the change didn't go live for everyone at once. Staff, partners, and verified server owners got access around May 18, 2023. Nitro subscribers followed on June 10. Every remaining non-Nitro user was let in on June 23, 2023, gated by account creation date, oldest cohorts first. And a meaningful tail of stragglers, accounts that hadn't picked a name during the open window, weren't auto-assigned unique usernames until a second push starting February 12, 2024, finishing around March 4, 2024, nearly two years after the project was announced. Spreading a few hundred million people's attempt to claim a scarce namespace over almost two years, in cohorts, rather than opening the gate to everyone simultaneously, is itself a real, load-bearing architectural decision, and it's the first clue to what makes this problem hard: it isn't really about aggregate volume, it's about how many people are reaching for the *same specific string* at the *same instant*.

For a number that actually quantifies sustained pressure on exactly this kind of check, look at Meta's Threads instead. Threads launched July 5, 2023, and by widely corroborated reporting from multiple outlets, hit 1 million signups within the first hour, 2 million by hour two, 10 million by hour seven, 30 million within 24 hours, and 100 million signups in roughly five days, the fastest consumer app growth on record at the time, beating ChatGPT's prior two-months-to-100-million pace. Averaging the first hour's milestone out gives a sustained rate of roughly 278 signups per second (that division is this lesson's own arithmetic on top of the sourced 1-million-in-one-hour figure, not a number Meta published directly, and instantaneous bursts, especially around a celebrity account joining and millions of people queuing to follow them, were almost certainly far higher). Meta's own engineering account of the launch, relayed through secondary summaries after direct fetches of engineering.fb.com were blocked in this session, describes the infrastructure team getting as little as two days' notice before the launch decision and having to proactively reshard for 100x expected growth, using automation to finish just before traffic opened at midnight UK time. Every one of those signups needed a username, typed by a human, checked for availability, and committed with zero duplicates tolerated, globally, immediately.

The structural point that separates this from every prior scaling lesson in this ledger: Day 26 covered generating a unique ID (a Snowflake ID), and that problem is easy in the sense that the *system* picks the value, so it can always mint a fresh one that has never existed before, no collision is even possible by construction. This lesson is the opposite case. The *user* picks the value. Many different humans, independently, will type the exact same short, desirable string, "alex," "shadow," "admin," "neo," at overlapping moments, and the system has to let exactly one of them win and tell everyone else, correctly and quickly, that they lost. That's not a throughput problem you can solve by adding servers. It's a contested-write problem, and it's the subject of the rest of this lesson.

---

## 2. Why the naive (demo) design dies

**The obvious version:** a `users` table with an `id` column (the shard key, because a table serving hundreds of millions of accounts has to be sharded for the same read and write throughput reasons Day 45 and Day 46 already established) and a `username` column with, in the demo's mental model, "a unique index on it."

On a single, unsharded relational database, this is completely solved. A real `UNIQUE` constraint backed by a B-tree index is enforced by the storage engine itself, atomically, at commit time: two concurrent `INSERT`s for the same username will have one of them rejected by the database, not by application logic racing against itself. This is not the naive design, it's the *correct* design, at the scale where it applies, in the same way Day 61 pointed out that a client-side trie is the correct architecture for a small catalog. The naive design only becomes naive once you shard.

**Death one: the unique index stops enforcing anything real the moment it's sharded by the wrong key.** A `users` table sharded by `id` (so that, per Day 26, a fresh Snowflake ID routes deterministically to one shard) places the `username` column on whichever shard the *user's ID* happens to hash or range to, which has nothing to do with the username's own value. Shard 7 doesn't know what usernames exist on shard 41. A `UNIQUE` index on the `username` column, if the database even lets you declare one on a sharded table without the shard key as a prefix, only ever sees the local slice of rows on its own shard. Two people can both successfully claim "alex," on two different shards, and each shard's own local index is completely happy, because from where either shard sits, it enforced uniqueness perfectly, of the wrong scope.

**Death two: check-then-act is a race condition the moment the check has to fan out.** Because the shard holding a user's row is determined by ID, not by username, "is this username taken" cannot be answered by looking at one shard, it requires scanning every shard, the exact distributed fan-out-and-merge problem Day 46 already named for joins, now sitting on the *write* path of every single signup rather than an occasional analytical query. Say the naive app server does this anyway: issue a scatter-gather read across all shards for `username = 'alex'`, get back zero rows, decide the name is free, then `INSERT` the new user into whichever shard their fresh ID lands on. Between the read returning and the insert landing, there is a gap, and it is exactly the gap two concurrent signup requests for the same name will race inside of. Request A reads zero rows. Request B, milliseconds later but before A's insert has committed anywhere either shard would see it, also reads zero rows. Both conclude the name is free. Both insert. Both succeed. Now two different accounts, on two different shards, both silently hold "alex," and nothing in this design ever notices, because no single component ever looked at both writes at once. This is the textbook time-of-check-to-time-of-use (TOCTOU) bug, and it is structurally the same anomaly Day 29's write-skew lesson described for a single database under weak isolation, now made worse because the "isolation" here spans machines that were never in the same transaction to begin with.

**Death three: even the read side alone collapses under real signup-storm traffic.** Set the write race aside for a moment. Just answering "is this username available" as someone types it, the same live-feedback UX Day 61 covered for search, requires that same all-shards scatter-gather on every keystroke, at whatever the current signup rate is. At Threads' sustained roughly 278 signups a second in its first hour, with real people trying, rejecting, and retrying several candidate usernames before landing on one that's free, the actual read volume against this design is a multiple of the raw signup rate, hitting *every shard* on *every attempt*, at the exact moment growth is spiking hardest and the system can least afford to be doing the most expensive possible query shape. Nothing about adding an index fixes any of these three deaths, because the problem was never that lookups were slow, it's that a `UNIQUE` constraint's atomicity guarantee is scoped to one machine's local B-tree, and sharding by the wrong key throws that guarantee away entirely, silently, with no error, right up until two people are both certain they own the same name.

---

## 3. The architecture

```
Client (signup form, live-typing availability check)
  - normalizes input before it's ever sent: lowercase, strip
    disallowed characters, Unicode-normalize, so "Alex", "alex",
    and a look-alike Unicode variant all resolve to one canonical
    string before uniqueness is even checked
  - job: don't let cosmetic differences hide a real collision
    analogy: a form that folds "St." and "Street" into one spelling
    before checking the address book, not two different addresses

        |  HTTPS: availability check, then a signup attempt carrying
        |  an idempotency key
        v
Edge / API gateway
  - rate limits and sheds load per Day 8 and Day 13, so a signup
    storm degrades gracefully (slower, or a wait screen) instead of
    taking the reservation layer down with it
  - job: protect the narrow, contended layer below from raw internet
    concurrency
    analogy: a single queue line at a ticket window, not everyone
    crowding the glass at once

        |
        v
Stateless signup service
  - holds no state of its own; orchestrates the two writes below and
    nothing else
  - job: sequence the reserve-then-commit steps; never itself decide
    who "wins" a contested name
    analogy: a notary who witnesses and sequences a transaction, but
    doesn't own the property being transferred

        |
        +----------------------------+
        v                            v
Uniqueness index (sharded         User records store
BY THE USERNAME ITSELF,           (sharded by user ID, per
hash or range of the string)      Day 26's ID-generation shape)
  - one small row per claimed       - the "real" account: profile,
    username: {username ->           settings, everything that is
    owner_id, state, ttl}            NOT the contested field
  - written via an atomic            - only written AFTER the
    conditional operation, not         reservation above already
    a select-then-insert:              committed successfully
    Cassandra "IF NOT EXISTS"        - job: hold everything about
    (Paxos underneath, Day 11's        the account that doesn't need
    consensus lesson made              cross-shard coordination,
    concrete), DynamoDB's              scaled the ordinary way
    ConditionExpression:               (Day 45's secondary-index
    attribute_not_exists(username),    shape, not this lesson's)
    or Postgres INSERT ... ON        analogy: the actual apartment
    CONFLICT DO NOTHING on a           lease, signed only after the
    single small cluster that          building's front desk has
    still fits the whole               already confirmed the unit
    namespace on one node's            number isn't double-booked
    write path
  - short TTL on an unconfirmed
    reservation, so an abandoned
    signup flow doesn't squat a
    name forever (same primitive
    as this ledger's Zomato
    seat-lock teardown)
  - job: be the ONE place in the
    whole system where "does this
    string already exist" can be
    answered by looking at exactly
    one shard, because the shard
    key and the uniqueness key are
    now the same thing
    analogy: a single sign-up sheet
    for a raffle prize, not one
    sheet per room in the building

        |  reservation confirmed
        v
Compensation on partial failure
  - if the user-record write fails for any unrelated reason after
    the reservation already succeeded, the signup service explicitly
    releases the reservation rather than leaving it orphaned
  - job: this is a saga (Day 20), not a distributed transaction
    spanning two independently sharded tables; no 2PC coordinator
    holds locks across both stores at once
    analogy: cancelling a hotel hold if the payment on file fails,
    not leaving the room permanently blocked
```

This composite mirrors real, documented mechanisms rather than one company's disclosed exact stack. Twitter's Manhattan database, per its own engineering blog, extended its architecture specifically "to support global uniqueness on any field by performing a compare-and-set on a strongly consistent index" before allowing the insert, which is precisely the sharded-by-the-unique-field index in the diagram above. Cassandra's lightweight transactions (`IF NOT EXISTS`) are implemented via Paxos consensus for exactly this reason, and DataStax's own documentation uses this exact scenario, two users racing to claim the same username, as the canonical example of why an ordinary last-write-wins write is unsafe here and a conditional write is required. DynamoDB's `ConditionExpression: attribute_not_exists(username)` gives the same compare-and-set guarantee in a single round trip, and its documented limitation, that enforcing uniqueness on two independent fields at once (username *and* email, say) needs a `TransactWriteItems` call across two separate conditional items, is a real constraint worth knowing before assuming one conditional write covers every uniqueness need an account has. MongoDB's documentation states the sharding rule most bluntly: a unique index is only automatically enforced across a sharded collection if the unique field is the shard key or has the shard key as a prefix, and its documented workaround for a unique field that is *not* the shard key is exactly this lesson's separate, minimal collection holding just the unique value and a reference, written first, gating everything else. Whether Discord's own username table specifically sits on Cassandra, ScyllaDB (which Discord's own blog confirms it migrated its messages storage to in 2022, cutting message read latency from roughly 200ms to 5ms while going from 177 Cassandra nodes to 72 ScyllaDB nodes), or something else entirely was not found in any public source during this lesson's research, and that gap is stated here plainly rather than guessed at: the mechanism above is the well-documented general pattern other named systems use for this exact problem, not a confirmed description of Discord's internals.

---

## 4. The transferable mechanisms

- **Shard by the constraint, not by the entity.** The single idea underneath the whole architecture: when a field must be globally unique but the table's natural shard key is something else, don't try to make the wrong shard key work harder, build a second, small, separate index physically sharded by the unique field's own hash or range, so that the database's ordinary per-shard atomicity guarantee (the thing a `UNIQUE` index gives you for free on one box) lines up exactly with the scope you actually need it to cover. This is Day 45's secondary-index lesson, specialized to the one case where "secondary" isn't optional, it's the only place uniqueness can be enforced at all.

- **Compare-and-set as the atomic primitive, never select-then-insert across a network hop.** Whatever storage engine sits under the uniqueness index, the write into it has to be a single atomic conditional operation (Cassandra's `IF NOT EXISTS`, DynamoDB's `attribute_not_exists`, Postgres's `ON CONFLICT DO NOTHING`, or a plain B-tree `UNIQUE` insert if the index fits on one small cluster), because the moment "check" and "act" are two separate round trips issued by an application server, a race window exists between them, no matter how fast the network is. This is Day 11's consensus lesson made concrete: Cassandra's lightweight transactions are Paxos, meaning the "atomic conditional write" primitive is, underneath, a small consensus protocol deciding who wrote first.

- **Reserve-then-commit as an ad hoc saga, not a distributed transaction.** The reservation write and the "real" account write live on two independently sharded stores that were never going to be coordinated by one 2PC transaction manager without paying a real latency and availability cost on every signup. Sequencing them as two local atomic steps, with explicit compensation (release the reservation) if the second step fails, is Day 20's saga pattern, applied to exactly the shape of problem sagas were designed for: a multi-step operation across storage boundaries that don't share a transaction coordinator.

- **TTL the reservation.** An unconfirmed claim on a scarce name has to expire, or an abandoned signup flow permanently squats it. This is the identical primitive to this ledger's Zomato District seat-lock teardown, a short-lived hold that auto-releases, and to a distributed lock's lease (Day 24), the same discipline applied to a username instead of a theater seat or a mutex.

- **Normalize before you shard.** Case-folding, Unicode normalization, and stripping the specific punctuation a platform allows must happen *before* the value is hashed to decide which shard of the uniqueness index it lives on, or "Alex" and "alex" can silently land in different partitions and never collide-check against each other at all. Discord's move to an explicitly lowercase-only scheme in Pomelo is this exact discipline made visible in a product decision, not just an implementation detail.

- **Idempotency keys distinguish a retry from a new competitor (Day 12).** A client that times out mid-reservation and retries the same signup attempt must be recognized as *itself*, not a second person racing for the name it may have already won. Tying the retry to an idempotency key generated once, client-side, at the start of the attempt, not to the username being claimed, is what lets the reservation service tell "this is me again" apart from "someone else just showed up."

---

## 5. The trade-offs

**Consistency vs. availability, scoped to exactly one narrow write, not the whole system.** The reservation step for a given username has to be strongly consistent, at least within whatever failure domain guarantees only one write wins, because this is the one place in the entire signup flow where weak consistency produces a visible, unrecoverable-feeling bug: two real humans both believing they own the same handle. Everything else about the account, avatar, bio, preferences, notification settings, can and should be eventually consistent, the same "available first, correct-fast-enough second" posture this ledger's own LeetCode mock-interview transcript reference point argued for a coding platform's non-uniqueness state. The trade-off isn't "pick consistency or availability for the system," it's "spend strong consistency on the one write that structurally cannot tolerate a race, and buy back availability and latency everywhere else that safely can."

**Cost vs. latency, paid once per account, deliberately.** A Paxos-backed conditional write is documented to cost more round trips than an ordinary write, Cassandra's own lightweight-transaction documentation is explicit about this overhead. Paying that cost on the signup hot path is acceptable specifically because it happens exactly once, ever, per account, not on every subsequent read or write to that user's profile. Putting the same consensus round trip on ordinary account activity, instead of concentrating it at the one moment the constraint actually needs enforcing, would be spending this cost in the wrong place.

**Sharding for scale and uniqueness enforcement are in real tension, and the resolution is to shrink the surface, not abandon either.** Sharding scales throughput precisely by giving up any single node's ability to see the whole dataset at once. Uniqueness enforcement fundamentally needs something that *can* see the whole namespace for the field in question, at least a shard, at least a leader, at least a consensus group. The way to have both is to keep the something-that-sees-everything as small and narrow as possible: not the entire multi-hundred-column `users` table, just a thin index of `{username -> owner_id}`, so the "single point that must coordinate" is doing the least possible work it can. Discord's own multi-cohort, nearly-two-year rollout is a real, load-bearing instance of this trade-off played out at the product level rather than the storage level: spreading a few hundred million people's attempt to claim scarce names over months, instead of opening the gate to everyone simultaneously, is choosing to shrink the *concurrency* the uniqueness layer has to survive, because shrinking the *data* it has to hold isn't an option, the whole existing account base needs a name eventually.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is a **hot-key problem** (Day 16), but inverted from its usual shape. Day 16's hot-key lesson was about a popular row being *read* by an enormous number of people at once, a celebrity's profile, a viral post. Here the hot key is a popular *string being written* by many different people at once. When Discord opened a rollout cohort, or when Threads onboarded a wave of new users, a large fraction of them independently reach for the same small set of short, desirable names first, "alex," "admin," "shadow," "neo." Hashing the username into a shard spreads *different* names across *different* partitions perfectly well, that part of sharding still works exactly as designed. It does nothing at all for the fact that thousands of different people, at the same moment, are all hashing to the exact same partition, because they all typed the same word. The write contention concentrates on a tiny number of specific reservation-table partitions, not evenly across the fleet, which is precisely the load pattern that breaks the "shard it and traffic balances out" assumption every other lesson in this ledger has relied on.

The naive instinct, give the hot partition more capacity or a faster consensus round trip, doesn't break this loop, it only raises the concurrency level at which that same partition eventually saturates or its Paxos round trips start queueing behind each other, because the underlying cause isn't insufficient aggregate throughput, it's N people racing for one specific contended key. The senior fix is the same shape Day 13's backpressure lesson and Day 16's hot-key lesson both already taught, applied here with a twist specific to this problem: fail the losers fast, and let the failure itself redirect load away from the hot key, rather than trying to queue everyone up waiting to find out who won. A compare-and-set that returns "already taken" in milliseconds lets 999 of the 1,000 people who all tried "alex" get an instant, honest rejection and move on to typing a *different* string, which is what actually relieves the pressure, because their retry lands on a *different* shard of the uniqueness index, not the same hot one. This is meaningfully different from a generic exponential-backoff retry, which would just make the same crowd hammer the same hot key more slowly. The mechanism that actually breaks the loop isn't slower retries, it's a fast, unambiguous "no" that sends the next attempt somewhere else in the keyspace entirely.

---

## Map to Rare.lab's stack

Rare.lab does not have this problem today, and saying that plainly matters before saying what transfers. A single logical Supabase Postgres instance, with a real `UNIQUE` constraint on whatever column holds a handle or a slug, is not a simplified version of this lesson's architecture, it's the *correct* architecture at Rare.lab's current scale, for the same reason Day 61 called a client-side trie the correct choice for the in-editor node search rather than a compromise. Postgres's B-tree unique index gives atomic, race-free enforcement on one box, exactly the case Section 2 opened by saying is completely solved, and nothing in Rare.lab's current stack, one Postgres instance under Supabase with row-level security, is sharded across multiple physical nodes the way this lesson assumes.

The gap opens the moment Rare.lab ships something a user picks rather than the system generates: a public creator handle (`rare.lab/@name`), or a shareable slug for a gallery shader, the same public gallery Day 61 already flagged as the natural home for content-addressed scenes stored in R2. The first two people who both want the slug `glass` or the handle `neo` are this lesson's entire problem, compressed into a two-person race instead of a Threads-scale storm, but the failure mode is identical in kind, just smaller in volume: without a real atomic constraint gating that specific write, two rows can end up claiming the same public-facing string, and unlike most bugs, this one is visible to strangers on the internet, not just to Rare.lab's own logs.

The concrete thing worth adopting now, before it's forced by an incident, is the *shape*, not new infrastructure: a dedicated `reserved_slugs(slug PRIMARY KEY, owner_id, state, reserved_at)` table, written first, with its own real Postgres `UNIQUE` constraint, gating the write to whatever richer scene or profile record depends on that slug, rather than folding the uniqueness check into a single wide `INSERT` alongside a dozen other columns. At Rare.lab's current single-node Postgres scale this buys almost nothing over a plain constraint on the main table today. What it buys is that the reserve-then-commit protocol, and the narrow, small table doing the actual contested write, is already the right shape to keep working unmodified if Supabase Postgres is ever fronted by shard-aware routing (Citus or similar) or if slug reservation is ever pulled out into its own small service, the same "get the mechanism right while the infrastructure is still simple" discipline Day 59's event-sourcing entry already applied to Rare.lab's single shared WebGL context for a different reason. Only the storage engine enforcing the compare-and-set changes later, the protocol, reserve the name first, in its own place, before writing anything else about the thing that owns it, does not.

---

## Sources

- [Evolving Usernames on Discord, Discord Blog](https://discord.com/blog/usernames): primary source for the discriminator-based-friction numbers (more than 40% of users not knowing their discriminator, roughly half of friend requests failing) and the new lowercase, 2-to-32-character, discriminator-free username format. Direct fetch was blocked by this session's network egress policy; relayed via search-indexed summaries of Discord's own post rather than a first-hand read.
- [New Usernames & Display Names, Discord Support](https://support.discord.com): source for the account-creation-date-gated cohort rollout mechanics and the trademark/reserved-name handling referenced in Section 1 and Section 2. Direct fetch was blocked; relayed via search-indexed summary.
- Discord Wiki (Fandom), "Pomelo" article: source for the specific rollout dates cited (May 18, June 10, June 23, 2023; February 12 to March 4, 2024 for the straggler-assignment phase). This is a fan-maintained wiki, not a Discord-primary source, and is flagged here as lower authority than the official blog and support articles above; the dates are corroborated across multiple secondary rollout-tracking sources but were not independently verified against a first-hand Discord post.
- Statista, Discord monthly active users page: source for the roughly 150 million MAU figure cited around the Pomelo period. Direct fetch was blocked; relayed via search-indexed summary, and this figure should be treated as an approximate, third-party-aggregated number rather than a Discord-disclosed one.
- [How Discord Stores Trillions of Messages, Discord Blog](https://discord.com/blog/how-discord-stores-trillions-of-messages): primary source for the Cassandra-to-ScyllaDB migration, the 177-to-72 node reduction, and the roughly 200ms-to-5ms message read latency improvement referenced in Section 3. This post describes Discord's **messages** storage specifically; this lesson does not claim it also describes the username/uniqueness table, and says so explicitly in Section 3. Direct fetch was blocked; relayed via search-indexed summary.
- ScyllaDB press release, "Discord Chooses ScyllaDB as Its Core Storage Layer": secondary corroboration of the same migration, from the vendor's own announcement. Direct fetch was blocked; relayed via search-indexed summary.
- [Native secondary indexing in Manhattan, Twitter/X Engineering Blog](https://blog.x.com/engineering/en_us/topics/infrastructure/2018/native-secondary-indexing-in-manhattan): primary source for the "compare-and-set on a strongly consistent index" global-uniqueness mechanism described in Section 3, from Twitter's own engineering account of its Manhattan database. Direct fetch was blocked; relayed via search-indexed summary, worth re-verifying directly.
- Twitter/X Engineering Blog, "Strong Consistency in Manhattan" (2016): primary source for Manhattan's consensus-backed replicated log and its LOCAL_CAS / GLOBAL_CAS compare-and-set modes referenced in Section 3 and Section 4. Direct fetch was blocked; relayed via search-indexed summary.
- DataStax / Apache Cassandra documentation on Lightweight Transactions: primary documentation source for `IF NOT EXISTS`, its Paxos implementation, and the canonical two-users-racing-for-one-username example referenced in Section 3 and Section 5. Direct fetch was blocked; relayed via search-indexed summary.
- AWS DynamoDB documentation, Condition Expressions and `TransactWriteItems`: primary documentation source for `attribute_not_exists`-based conditional writes and the multi-field-uniqueness limitation referenced in Section 3. Direct fetch was blocked; relayed via search-indexed summary.
- MongoDB documentation, "Sharded Collections with Unique Indexes": primary documentation source for the shard-key-must-prefix-the-unique-index rule and the documented separate-collection workaround referenced in Section 3. Direct fetch was blocked; relayed via search-indexed summary.
- [How Meta built the infrastructure for Threads, Meta Engineering Blog](https://engineering.fb.com): primary source (relayed through a third-party summary) for the two-days'-notice, 100x-resharding account of Threads' launch infrastructure, and its use of ZippyDB and the Async serverless platform, referenced in Section 1. Direct fetch to engineering.fb.com was blocked by this session's network egress policy; relayed via a search-indexed summary of the post rather than a first-hand read, and worth re-verifying directly.
- Multiple outlets (Forbes, TechCrunch, Time, Al Jazeera, corroborating each other in search results) on Threads' launch-week growth: source for the 1 million/1 hour, 2 million/2 hours, 10 million/7 hours, 30 million/24 hours, and 100 million/5 days milestones referenced in Section 1. Direct fetches were blocked; relayed via search-indexed summaries, consistent across independent outlets.

---

*Inference vs. fact, stated plainly: Discord's discriminator-friction statistics, its new username format rules, its rollout cohort dates, its ScyllaDB migration numbers for message storage, Twitter/X Manhattan's compare-and-set uniqueness mechanism and consensus-backed strong-consistency modes, Cassandra's lightweight-transaction and DynamoDB's conditional-write documentation, MongoDB's sharded-unique-index rules, and Threads' launch-week growth milestones are all documented claims from named, identifiable sources, but every one of them was relayed through this session's web search rather than a first-hand read of the original page, because direct fetches to discord.com, engineering.fb.com, blog.x.com, and every documentation and news domain checked were blocked by this session's network egress policy; none of this lesson's sources were fetched directly, and every figure above should be treated as worth re-verifying before being relied on for anything beyond this lesson's teaching purpose. The roughly 278-signups-per-second sustained-rate figure in Section 1 is this lesson's own arithmetic, dividing the sourced "1 million in the first hour" milestone by 3,600 seconds, not a number Meta published directly, and is flagged as inference for that reason; real instantaneous peaks, particularly around celebrity account launches, were almost certainly higher. Whether Discord's own username-uniqueness table specifically runs on Cassandra, ScyllaDB, or another store entirely was not confirmed by any source found during this lesson's research, and is explicitly left unresolved in Section 3 rather than guessed at; the architecture and mechanisms described there are the well-documented general pattern used by Twitter/Manhattan, Cassandra, DynamoDB, and MongoDB for this exact class of problem, not a confirmed description of Discord's internal implementation. The architecture diagram's specific layering, the hot-key-inverted framing in Section 6, and the entire Rare.lab mapping are this lesson's own synthesis on top of the documented mechanics above, not a claim that Discord, Twitter/X, Meta, or any cited vendor describes their systems in these exact terms.*
