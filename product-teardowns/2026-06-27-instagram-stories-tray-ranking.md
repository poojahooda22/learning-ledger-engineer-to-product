# Instagram Stories: the order of the circles, and why they vanish at midnight

**Date:** 2026-06-27
**Product:** Instagram
**Feature:** The Stories tray (the row of circles at the top) - how it is ordered, and how the stories behind it expire after 24 hours

---

## 1. The user

It is 8:10 AM. Priya is on the Mumbai local, standing room only, one hand on the
rail and the other holding her phone. She opens Instagram out of pure reflex,
the way some people check the time. The first thing her thumb lands on is the row
of little circles across the top of the screen. Some have a bright pink-and-orange
ring around them, which means there is something new she has not seen yet. Some
are grey, already watched.

She has fourteen minutes until her stop. She is not here to read captions or hunt
for anything. She wants to glance at what the handful of people she actually cares
about did since last night, tap through them, maybe reply to one with a single
emoji, and put the phone away. The whole thing is muscle memory.

Priya follows about 800 accounts. Roughly 120 of them posted a story in the last
24 hours. She will look at maybe 15 before her stop. The question the app has to
answer in the few hundred milliseconds before she looks down is brutally simple to
say and hard to do: which 15, and in what order.

## 2. The real problem

Here is the pain, described like a friend would.

Priya follows her sister, her two college roommates, a few school friends, a
cricket meme page, three clothing brands, a couple of celebrities, and a long tail
of people she met once and felt awkward not following back. That is 800 accounts.
On a busy night, 120 of them post stories. If the app just showed them to her in
the order they were posted, the meme page that fires off 8 stories a day and the
brand running a sale would bury the one story she actually wants: her sister, who
posted a photo of their grandmother at 6 AM.

So the real problem is attention triage. There is too much, she has too little
time, and the cost of getting the order wrong is not a crash or an error message.
It is quieter and worse: she gets bored, the circles feel like noise, and one day
she just stops opening them. Nothing breaks. She simply drifts away.

The second, sneakier problem is the disappearing act. The whole promise of a story
is that it is gone in 24 hours. That is what makes Priya post one of herself
yawning on the train without a second thought. It is low stakes because it is
temporary. But "temporary" is a promise the backend has to keep, billions of times
a day, without a giant nightly broom sweeping the floor. If a story lingers past
its deadline, the promise is broken.

## 3. The feature in one sentence

The Stories tray is a per-person, freshly ranked shortlist of the people you follow
who posted in the last 24 hours, ordered by how likely you are to care, sitting on
top of a storage system engineered so each story quietly deletes itself when its day
is up.

## 4. Jobs to be done

What is Priya really hiring this feature to do?

- "Show me what the people I actually care about did, before I show me anyone else."
- "Tell me at a glance what is new (the colored ring) versus what I have already seen
  (the grey ring), so I do not waste a tap."
- "Let me peek without committing. One tap in, swipe through, swipe down to leave."
- "Let me post something dumb and fun and trust that it will be gone tomorrow."

Notice what is not on the list. She is not hiring it to discover strangers. That is
what Explore and Reels are for. The Stories tray is about the people she has already
chosen. That single fact shapes the entire engineering, as we will see: the candidate
set is small and known, so almost all the effort goes into ordering, not finding.

## 5. How it works for the user

The visible experience is four moving parts.

1. **The tray.** A horizontal row of circles at the top of the home feed. Your own
   "add story" circle is first. Then the accounts you follow that have unseen stories,
   colored ring first, roughly in the order the app thinks you care.
2. **The ring.** A bright gradient ring means unseen. A grey ring means seen. Once
   you watch someone's story, their ring goes grey and they slide toward the right.
3. **The viewer.** Tap a circle and it opens full screen, auto-advancing through that
   person's stories, then rolling straight into the next person's. Thin progress bars
   at the top show how many segments are left.
4. **The exit.** Swipe down and you are back. Swipe right and you go backward. Tap the
   right edge to skip forward, the left edge to go back.

A real example of the order Priya sees this morning, left to right: her own circle,
then her sister (colored ring, posted at 6 AM), then her roommate Anjali (colored),
then her school friend Rohan (colored), then the cricket meme page (colored, 8 new),
then a clothing brand (colored), then a celebrity she watches sometimes (grey, she
saw it last night). Her sister did not post first in clock time. Anjali posted at
2 AM, hours earlier. But the app put her sister first because Priya watches every
single one of her sister's stories and often replies. That is the ranking talking.

## 6. The actual flow, step by step

Tap by tap.

1. Priya opens the app. The client asks the server: give me my Stories tray.
2. The server figures out the candidate set: of the 800 accounts she follows, which
   posted a story still inside its 24-hour window? Say 120.
3. For each of those 120, the server pulls a "seen or not" bit for Priya and a bundle
   of signals (how often she watches them, how recently, whether she replies).
4. A ranking model scores all 120 for Priya specifically. It runs in the data center,
   not on her phone.
5. The server returns an ordered list: unseen first, ranked by predicted interest,
   seen ones after. The phone just renders the order it was handed. It does not sort.
6. Priya taps her sister's circle. The viewer opens, plays the 6 AM photo for 5
   seconds, advances to the sister's second story, then auto-rolls into Anjali.
7. The moment a story is shown, the client fires a lightweight "seen" event back to
   the server: viewer Priya, story id 17_29..., seen at 08:11:04.
8. That seen-event flips the ring grey and makes sure that if Priya closes the app and
   reopens it on her laptop at lunch, her sister's story already shows as watched.
9. Twenty-four hours after the sister posted, that story stops being eligible. It
   drops out of every tray and the media itself is scheduled for deletion.

The two interesting halves are step 4 (the ordering) and step 9 (the vanishing). The
rest is plumbing. Let us go under the hood on both.

## 7. Under the hood, like the engineer

### 7a. The tray is a ranking problem, and ranking is two halves

Every ranking feature, whether it is Amazon search or the YouTube home feed, splits
into two jobs that are easy to confuse: **matching** (which candidates are even
eligible) and **ranking** (in what order). The Stories tray is unusual because the
matching half is almost free, and that changes everything downstream.

**Matching: the candidate set is small and self-limiting.** Priya's candidates are
not "all stories on Instagram." They are "stories from accounts Priya follows that
are younger than 24 hours." Two filters do all the work:

- **The follow graph.** Instagram stores who-follows-whom as a graph. Get Priya's
  following list. This is a known, bounded set: 800 edges, not 2 billion users. At
  Meta this kind of lookup is served by TAO, their graph cache that sits in front of
  sharded MySQL and answers "give me the objects connected to this node" in
  milliseconds. (TAO is documented Meta infrastructure; that Stories specifically
  rides on it is reasonable inference, not a published Stories detail.)
- **The 24-hour freshness filter.** Of those 800, keep only the ones with a live
  story. This is where the story id design pays off, and we will get to it in 7c.

So matching hands ranking a list of about 120 items. Compare that to Amazon, where
matching has to claw a few thousand candidates out of 350 million listings using an
inverted index. The Stories candidate set is tiny because Priya already did the
matching herself, years ago, every time she tapped "Follow." The catalog is
pre-filtered by the social graph. This is the single most important fact about the
feature: **you never rank more than a few hundred items per person, no matter how
big Instagram gets.** Ranking cost is bounded by how many people you follow, not by
how many people exist.

**Ranking: a multi-task model predicts several actions, then blends them.** Now the
real work. Instagram does not have one "interest score." It predicts the probability
of several distinct actions and combines them into a single value. This is confirmed
engineering. Meta's own Transparency Center says the Stories system makes predictions
about "how likely you are to tap into a story, reply to a story, or move on to the
next story," and Adam Mosseri has stated the top signals are likelihood to tap, to
like, and to reply.

Mechanically, picture a model that for each candidate account outputs a short vector
of predictions:

- P(tap into this story)
- P(like it)
- P(reply to it)
- P(skip past it quickly)

and then a value function collapses them into one number, something shaped like:

`score = w1*P(tap) + w2*P(like) + w3*P(reply) - w4*P(skip)`

Sort the 120 candidates by that score, put unseen above seen, and you have the tray.
The weights w1..w4 are tuning dials the team sets to trade off, say, replies (a strong
closeness signal) against quick taps (cheap engagement). Instagram's own engineering
write-up describes exactly this design: they "use models with ranking losses and
point-wise models" so they have "better control in the final value function and can
fine-tune trade-offs between key engagement metrics." This is one model predicting
many heads, a multi-task model. Meta's 2025 engineering post on scaling to 1000+
models names the Stories tray model directly: `ig_stories_tray_mtml`. The `mtml`
stands for multi-task, multi-label, which is precisely the several-predictions-in-one
shape described above.

**What features feed it.** From the Transparency Center, the concrete inputs include:
how much time Priya spends viewing stories on average, which device she is on, how
many times she has viewed stories from this specific account, the total time she has
spent on this author's stories, the strength of connection (are they friends on
Facebook, friends-of-friends), and the ratio of replies to views for this author.
This is why her sister ranks first: high view count, high total watch time, frequent
replies. The cricket meme page gets a lot of quick taps but few replies, so it ranks
below the people she is close to even though it posts more.

**The data structures in play here.** A hash map from account id to that account's
freshest story metadata. An array of (account, feature-vector) pairs, 120 long, that
the model scores. A tiny seen-set per viewer (more on that next). And underneath the
matching step, a graph (the follow network) served from cache. No trees, no inverted
index, because there is no text query and no million-item catalog to search. The
shape of the data structure follows the shape of the problem: a known small set to
order, not a giant set to search.

**The position-bias trap, and the real fix.** Here is a subtle bug that Instagram
publicly described solving. People tap the first circle more than the fifth circle
just because it is first, not because it is better. If you train your model on raw
behavior, it learns "things in position 1 get tapped a lot," and the top of the tray
freezes in place because the model keeps rewarding whoever is already on top. Their
fix, from the 2018 engineering post: feed the position itself into the model as a
sparse feature, wired into the last fully connected layer, so the model can soak up
"this got tapped partly because it was in slot 1" into the position feature and keep
the rest of the score honest. At serving time you hold position neutral. This is the
same trick YouTube later published with its "shallow tower." Without it, new and
close friends could never climb past whoever the model happened to rank first
yesterday.

### 7b. The colored ring is a per-viewer seen-set, and it is its own little problem

The ring color looks trivial. It is not, at scale. "Has Priya seen story 17_29...?"
is a question asked billions of times a day, and the answer is different for every
one of the 2 billion-plus people on Instagram.

The naive idea is to store, for each story, the full list of everyone who has seen
it. That list can be hundreds of millions of entries long for a celebrity's story.
Now multiply by every live story. It does not fit, and you do not need it for the
ring anyway. For the ring, you only ever ask the question from one viewer's side:
"which of the 120 accounts in my tray have I already fully watched?"

So the practical shape (this part is class-of-problem inference, since Instagram has
not published the exact store) is a compact per-viewer seen structure: for viewer
Priya, a set of story ids (or a high-water mark per author, "Priya has seen
everything from the sister up to story 17_29...") that the tray builder checks while
ordering. Sets and high-water marks are cheap. A "seen" event in step 7 is an idempotent
write: mark story X seen by viewer P. Idempotent matters because the client may send it
twice on a flaky train connection, and seeing a story twice should still leave it
seen once, exactly like the delivery-receipt ticks in WhatsApp or a Stripe idempotency
key. The cross-device sync Priya notices at lunch (story already grey on her laptop)
falls out for free, because seen-state lives on the server, not on the phone.

### 7c. The vanishing act: 24-hour expiry without a nightly broom

This is the most quietly clever part, and it leans on a piece of Instagram engineering
that is fully public: their ID design.

Every object on Instagram, including every story, gets a 64-bit id that is not random.
It is packed like this (this is documented Instagram engineering):

- **41 bits:** a millisecond timestamp, measured from a custom epoch.
- **13 bits:** which logical shard created it (Instagram ran 13 bits' worth, thousands
  of logical shards mapped onto a smaller number of physical Postgres machines).
- **10 bits:** a per-shard counter, mod 1024, so a single shard can mint 1024 ids per
  millisecond before it has to wait for the next tick.

Two beautiful consequences fall out of putting the timestamp in the high bits:

1. **Sorting by id is sorting by time.** `ORDER BY id` gives you newest-first for free,
   no separate created_at index to maintain in RAM. For a feature whose whole life is
   "show me the newest, hide anything older than 24 hours," the sort key and the
   freshness key are the same number.
2. **Expiry is a subtraction, not a search.** "Is this story alive?" is answered by
   reading the timestamp out of the top 41 bits of its own id and checking if
   `now - timestamp < 24h`. You do not need a separate expiry column, an index on it,
   or a job that scans for stale rows. The deadline is encoded in the identity of the
   object itself.

Why does this matter so much? Because the search snippet said it plainly: Instagram
takes in millions of stories and has to expire millions of stories per second as the
24-hour line sweeps across the globe. A cron job that runs at midnight and deletes
"yesterday's stories" is a disaster at that rate. Midnight in which timezone? And the
delete storm would hammer the database in one giant spike.

The pattern that actually scales (this combination is the standard solution for this
class of problem; the exact Stories implementation is not fully public, so treat the
specifics as well-grounded inference):

- **Lazy expiry on read.** When the tray is built, the freshness filter simply drops
  any story whose embedded timestamp is older than 24 hours. Expired stories vanish
  from the product instantly without anyone deleting anything. The user-facing promise
  is kept the microsecond the deadline passes, by arithmetic, not by a cleanup job.
- **Asynchronous physical cleanup.** The actual bytes get reclaimed later, in the
  background, off the hot path. Because the story id carries its own timestamp, a
  janitor process can stream through and reclaim anything past its deadline at its own
  pace. There is no rush, because lazy expiry already made the story invisible. A
  short-TTL cache (think Redis-style expire) can hold the live story metadata so the
  hot read path never even touches the slow store.
- **Media lives in object storage, metadata in the graph.** The photo or video itself
  is a blob in Meta's blob storage (Haystack for hot photos, f4 for warm, both
  published Meta systems). The tray only ever moves around small metadata: ids,
  author, timestamp, seen-bit. You never drag a 4 MB video through the ranking path;
  you drag a pointer.

### 7d. The scale story at three tiers

**1,000 stories (a small app, or one creator's audience).** Everything is trivial.
Load all candidate stories into memory, score them with the model, sort, return. A
single server does it. You could even sort on the client and no one would notice. At
this tier none of the cleverness above earns its keep.

**100,000 stories (a mid-size network, or a popular city).** Now you cannot rebuild a
person's tray from raw data on every app open, because lots of people open the app at
once. The pressure points: the follow-graph lookups and the per-story metadata reads.
The fixes are caching and read replicas. Put the follow lists and the freshest-story-
per-account in a fast cache (TAO does exactly this) so the common reads never hit the
database. Serve reads from replicas so the write path is not the bottleneck. The model
scoring of 120 items per person is still cheap; the database reads are what you protect.

**10 million-plus live stories, 2 billion-plus users (real Instagram).** Three things
break and three patterns save them.

- **Fan-out.** Do you precompute everyone's tray when a story is posted (fan-out on
  write) or build it when they open the app (fan-out on read)? For a celebrity with 400
  million followers, fan-out on write means 400 million tray updates for one post,
  which is insane. So stories use fan-out on read: build the tray on open, pulling from
  a cache of "newest story per followed account." This is the standard hybrid (push for
  small accounts, pull for huge ones) and it is reasonable inference for Stories given
  Meta's published feed architecture. The candidate set stays bounded by how many
  accounts you follow, which is the whole reason this is affordable.
- **Storage and id contention.** Billions of objects need ids minted fast without a
  global counter becoming a bottleneck. The 41-13-10 bit id solves this: each shard
  mints its own ids locally, 1024 per millisecond, no coordination with any other
  shard. Sharding by a stable key (user/shard id baked into the object id) gives linear
  horizontal scale. Add machines, add shards, no re-partitioning.
- **The expiry storm.** As covered, you never run a global delete sweep. Lazy expiry by
  reading the embedded timestamp makes stories disappear from the product for free, and
  background reclamation handles the bytes off the hot path. The deadline travels inside
  each object's identity, so there is no central list of "things to delete" to contend on.

And one more that only bites at the very top: **1000+ models to keep alive.** Meta's
2025 post is titled "Journey to 1000 models" for a reason. The Stories tray model is
one of over a thousand ranking models in production, each needing training pipelines,
checkpoints, and low-latency inference services. The scaling problem at the very top is
not any single model. It is the meta-problem of operating a thousand of them without
the system collapsing under its own complexity, which is why that post exists at all.

## 8. The retention and habit mechanic

The Stories tray is one of the most efficient habit loops Instagram has, and it works
on three gears.

**Gear one: the unseen ring is a tiny open loop.** A colored ring is an unfinished
task sitting in your visual field. Humans hate leaving those alone. Priya sees three
colored rings and feels a small, almost unconscious pull to clear them, the same way an
unread badge pulls. Watching turns the ring grey, which is a micro-reward of completion.
The loop is: see color, tap, clear to grey, feel done. It repeats every single app open.

**Gear two: the 24-hour deadline manufactures urgency.** Because a story is gone
tomorrow, there is a real cost to not looking today. This is the same scarcity engine as
a Snapchat snap or a limited-time sale. It is the reason people check stories several
times a day rather than letting them pile up like emails. The ephemerality is not just a
storage choice. It is the retention mechanic. The thing that makes the backend hard (it
must vanish on time) is the exact thing that makes the user come back (it will vanish, so
look now).

**Gear three: ranking protects the loop.** If the first circles were junk, the loop would
die. By putting Priya's sister and close friends first, the model makes sure the very
first taps of every session are rewarding, which trains her to keep tapping. The metric
this moves is retention, specifically daily and session frequency. Stories are widely
credited as a major driver of Instagram's time-spent and daily-active growth after their
2016 launch, precisely because they convert "I follow these people" into "I check in on
these people several times a day."

A real observed example of the loop in motion: Priya watches her sister's story every
morning. The model sees that pattern, keeps the sister pinned to the front, which makes
the first tap reliably good, which keeps Priya opening the tray, which produces more
watch-and-reply signal, which pins the sister even harder. The loop feeds itself. That is
a habit, built out of one ranked row of circles.

## 9. The lesson for Rare.lab

Bake the deadline into the identity of the object, not into a side table you have to scan.

Rare.lab compiles node graphs into shippable shaders and runs an embeddable runtime.
You will accumulate a flood of disposable artifacts: per-edit shader compilations, cached
texture bakes, preview renders, autosave snapshots, A/B variants of an effect. Almost all
of them are stale within minutes or hours, the same way a story is stale in 24. The naive
plan is a `created_at` column plus a sweeper job that periodically scans for old rows and
deletes them. That plan does not survive growth. The scan gets slower as the table grows,
and the delete spike contends with live compiles at the worst possible moment.

Do what Instagram's story id does. Give every cached artifact an id that carries its own
creation timestamp in the high bits (a Snowflake-style 64-bit id: timestamp, then a
worker/shard id, then a per-worker counter). Two payoffs land immediately. First,
"is this bake still valid?" becomes a subtraction on the id itself, no extra column, no
index, no lookup. Stale previews disappear from the editor the instant their deadline
passes, by arithmetic, on read. Second, `ORDER BY id` is `ORDER BY time` for free, so
"show me the latest compile for this node" needs no separate time index eating RAM. Then
reclaim the actual GPU memory and disk lazily in the background, off the compile hot path,
because the id already told you what is dead. Never let a cleanup sweep stand between a
user and their next frame. The deadline should live inside the thing, so that expiry costs
you nothing on the path that has to stay fast.

---

## Sources

- Instagram Engineering, "Lessons Learned at Instagram Stories and Feed Machine Learning"
  (Thomas Bredillet, 2018). Value function blending ranking-loss and point-wise models;
  position-bias correction via a sparse position feature in the last fully connected layer;
  Caffe2; co-learned sparse embeddings.
  https://instagram-engineering.com/lessons-learned-at-instagram-stories-and-feed-machine-learning-54f3aaa09e56
- Engineering at Meta, "Journey to 1000 models: Scaling Instagram's recommendation system"
  (2025). The `ig_stories_tray_mtml` model named; operating 1000+ production ranking models.
  https://engineering.fb.com/2025/05/21/production-engineering/journey-to-1000-models-scaling-instagrams-recommendation-system/
- Meta Transparency Center, "Instagram Stories AI system." The named predictions (likelihood
  to tap, reply, move on) and the concrete input signals (average view time, device, times
  viewed from this account, total time on author, connection strength, reply-to-view ratio).
  https://transparency.meta.com/features/explaining-ranking/ig-stories/
- Instagram Engineering, "Sharding & IDs at Instagram." The 64-bit id: 41-bit ms timestamp,
  13-bit shard, 10-bit per-shard sequence (1024 ids/shard/ms); ORDER BY id equals ORDER BY
  created_at.
  https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c
- Engineering at Meta, "Scaling the Instagram Explore recommendations system" (2023). Meta's
  two-stage retrieval-and-ranking pattern and multi-task ranking, context for the tray model.
  https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/
- About Instagram, "Instagram Ranking Explained" (Adam Mosseri). Stories top signals stated as
  likelihood to tap, like, reply.
  https://about.instagram.com/blog/announcements/instagram-ranking-explained
