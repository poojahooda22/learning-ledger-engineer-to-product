# Spotify Wrapped: the once-a-year batch job that reads half a billion people's whole year

Date: 2026-07-21
Product: Spotify
Feature: Wrapped (the annual personalized year-in-music recap, and the data pipeline that builds it)

A note on what this teardown is. Every other Spotify feature in this ledger
(Discover Weekly, instant playback) is a live system: the user does a thing, a
server answers in milliseconds. Wrapped is the opposite animal. It is a giant
batch job that runs once, in the dark, over a full year of listening for every
user on Earth, and then goes quiet for eleven months. It is the purest example
in this whole ledger of "do the expensive thinking offline, serve the live path
as a cheap keyed lookup." So it is worth tearing down precisely because it does
not look like the others.

---

## 1. The user

Meet Ananya. She is a second-year engineering student in Pune. It is the first
week of December. She is between two lab submissions, half-scrolling Instagram
on her phone at 11pm, and her whole feed is suddenly the same thing: friends
posting colorful vertical cards. "My top artist was Taylor Swift." "I was in the
top 0.5% of Arijit Singh listeners." "97,412 minutes this year."

She has not opened Spotify in maybe four days. But now she wants to know her
number. She opens the app. There is a card at the top of Home: "Your 2026
Wrapped is here." She taps.

Ananya is not doing anything technical. She is doing something social. She wants
to know who she was this year, told back to her as a story, and she wants
something worth posting.

## 2. The real problem

Here is the honest version, said like a friend would say it.

A year of your listening is a huge, boring pile of numbers. 40,000 individual
play events. Timestamps. Track IDs. Skip flags. On its own it is landfill. No
human wants to read a spreadsheet of their own scrobbles.

The pain Wrapped removes is not "I cannot see my data." Spotify has always had
your data. The pain is "I cannot feel my data." Ananya cannot look at 40,000
rows and go "wow, I really spiraled into sad Punjabi songs in July after the
breakup." The raw data hides the story inside itself.

And there is a second, quieter pain, this one Spotify's: Ananya drifts. She is a
free user, she uses YouTube too, some months she barely opens the app. Spotify
needs a reason, once a year, to make her come back and remember why she likes
being a Spotify person specifically. Not a discount. A mirror.

## 3. The feature in one sentence

Wrapped is a once-a-year batch pipeline that reads every play event you logged
all year, boils it down to a handful of personal "data stories" (top artist, top
songs, total minutes, top genre, a listening personality), and hands them to the
app as one pre-built, ready-to-share slideshow.

## 4. Jobs to be done

What is Ananya really hiring Wrapped to do?

- "Tell me who I was this year in a way I can feel in five seconds." Not a table.
  A number and a name. "97,412 minutes. Taylor Swift."
- "Give me something worth posting." The card has to be designed to screenshot.
  Her identity, made shareable.
- "Rank me." Humans love a percentile. "Top 0.5% of Arijit listeners" is candy.
  It turns private listening into a leaderboard she did not know she was on.
- "Remind me that this app knows me." The emotional payload. A year of loyalty,
  reflected back, so switching to another app feels like abandoning a diary.

Notice none of these are "compute statistics." The statistics are the raw
material. The job is a feeling.

## 5. How it works for the user

Ananya taps the Wrapped card. The screen goes full-bleed, portrait, no player
chrome. It is a sequence of animated slides, one tap or auto-advance each,
exactly like an Instagram Story:

- "You listened to 97,412 minutes of music in 2026."
- "Your top artist was Taylor Swift. You were in her top 0.5% of listeners."
- "Your top 5 songs" scrolling up one by one, number 1 held for a beat.
- "Your top genre was Pop, but you had a wild indie phase in July."
- "Your 2026 listening personality: The Adventurer."
- A final share card: everything summarized, a big "Share" button.

She taps Share. It drops a pre-rendered vertical image straight into her
Instagram Story with the Spotify logo baked in. Total time from tap to post:
maybe 40 seconds. The whole thing feels instant and hand-made for her.

It is not hand-made for her in that moment. It was made for her weeks ago and
was sitting in a database waiting. That gap is the entire engineering story.

## 6. The actual flow, step by step

1. Ananya opens the app. The Home service checks: does this user have a Wrapped
   payload ready? It does one keyed lookup by her user ID.
2. The lookup hits a row that was written weeks earlier. That row already holds
   all her data stories: minutes, top artist, percentile, top 5 songs, genre,
   personality label.
3. The app pulls that small bundle of values (a few kilobytes, not a year of
   events) and pours them into templated slides. In 2023 Spotify moved these
   animations to Lottie so the same vector animation renders on iOS, Android and
   web without shipping video.
4. She taps through. No server round trip per slide. The payload is already on
   the phone.
5. She taps Share. The app composes the share card locally (or fetches a
   pre-rendered one) and hands it to the OS share sheet.

The live path is: one keyed read, then local rendering. That is it. Nothing about
that path scales with the number of songs she played. It is constant work no
matter how heavy a listener she is. All the heavy lifting happened before she
ever tapped. Now we go there.

## 7. Under the hood, like the engineer

This is the heart of it. Wrapped is Spotify's single largest data job of the
year. Spotify has publicly called it the largest Google Cloud Dataflow job ever
run on the platform (2018), and then broke that record again. So how do you read
a year of listening for hundreds of millions of people without setting money on
fire?

### The raw material: an event stream, not a database

First, understand the input. Spotify does not store "Ananya's stats" anywhere. It
stores raw events. Every time a track plays, the client emits a small event:
user ID, track ID, timestamp, how long it played, whether it was skipped. Those
events are collected by Spotify's event delivery system, validated, and streamed
through a message bus (Kafka-style) into the data lake, which sits on object
storage backed by Google Cloud. Spotify's platform ingests on the order of
trillions of data points; the company has publicly described handling 1.4
trillion data points in its data platform.

So the year of Ananya's music is not a tidy profile. It is 40,000 tiny event
rows scattered across enormous daily log files, mixed in with everyone else's
billions of events. Wrapped's job is to reassemble those scattered rows into one
per-person story.

### The core operation is a group-by, and the enemy is shuffle

Strip away the marketing and Wrapped is three classic data operations:

- Group by user. Gather every event belonging to Ananya into one place.
- Aggregate. Count her minutes, count plays per artist, per genre.
- Rank / top-K. Keep only her top 5 songs, her number 1 artist. Throw the rest
  away.

The data structures are humble and exactly right. A hash map keyed by user ID to
accumulate per-user counts. Inside each user, a hash map from artist ID to a play
count. To get "top 5 songs" you do not sort all 40,000 events; you keep a small
bounded top-K selection (a min-heap of size 5, or a partial sort), so cost per
user is roughly linear in her events, not N log N over everything. To turn "top
0.5% of Taylor Swift listeners" into a real number you need the flip side too: a
per-artist aggregation across all users, then a percentile cut.

The expensive word hiding in "group by user" is shuffle. In a distributed job,
Ananya's 40,000 events start life spread across thousands of machines, because
they were written on thousands of different days by thousands of different log
writers. To gather them onto one machine so you can count them, the system has to
physically move key-value pairs across the network so that all records with the
same user ID land together. That all-to-all data movement is the shuffle. It is
the single most expensive thing a big data pipeline does: disk writes, network
transfer, disk reads, for petabytes of data. Shuffle is the enemy. The whole
Wrapped optimization story is a war on shuffle.

### Attempt one: one giant job with a giant shuffle (works, but hurts)

The naive design is one huge Dataflow pipeline. Read all the year's events, do a
massive GroupByKey on user ID, then compute every data story inside that grouped
stream. It works. Spotify shipped early Wraps roughly this way. But it has two
problems at Spotify's scale. First, that single GroupByKey shuffles essentially
the entire year of listening data at once, which is the costliest possible
operation run on the largest possible input. Second, all the data stories are
welded into one monolithic job. If the "top genre" logic has a bug, you re-run
the whole planet's shuffle to fix it. Iteration is agony.

### Attempt two: Bigtable as the assembly point, one row per user

The 2019 Wrapped was the decade-in-review edition, so the input was not one year
but ten. It was about 5 times larger than the 2018 job. If shuffle cost scaled
with it, the bill would have been brutal. Instead Spotify ran that 5x-larger job
at roughly three-quarters of the previous cost. Here is the trick.

They decomposed the monolith. The problem was split into three stages: data
collection, aggregation, and transformation. Then, crucially, the data stories
themselves were split into many independent jobs. Top artist is one job. Top
genre is another. Minutes listened is another. Because most data stories do not
depend on each other, they can run in parallel and be iterated on separately.

The clever part is where they land. Every data story job writes its output to the
same Bigtable row, keyed by user ID, but into a different column family. Think of
Bigtable as one giant table with one row per user. The "top artist" job writes
into Ananya's row, column family A. The "top genre" job writes into the same
row, column family B. They never have to be joined by a shuffle, because Bigtable
is doing the assembly by key for free. Bigtable is a sorted key-value store; a
write to a known row key is a cheap keyed put, not an all-to-all move. Bigtable
becomes a remediation layer between Dataflow jobs, so the pipeline can process
and store more data in parallel instead of forever regrouping it. When the app
finally needs Ananya's Wrapped, her whole story is already sitting in one row,
ready to read.

This is a materialized view. You precompute the answer, keyed by exactly the key
you will look it up by later, so the live read is O(1).

### Attempt three: Sort Merge Bucket, killing shuffle at the source

For Wrapped 2020 Spotify went further and attacked the shuffle itself, not just
the reassembly. The tool is Sort Merge Bucket (SMB), which grew out of Andrea
Nardelli's 2018 master's thesis and became a module in Scio, Spotify's open
Scala API for Apache Beam.

The idea is beautiful and old (databases have done sort-merge joins for decades).
The reason a join or group-by needs a shuffle is that matching keys are scattered
and unsorted. So do not leave them scattered. When you write a dataset, write it
into a fixed number of bucket files chosen by a hash of the join key, and sort
the records inside each bucket by that key. Do the same for the second dataset
with the same bucketing scheme. Now bucket 7 of "events" and bucket 7 of "track
metadata" are guaranteed to hold the same slice of key space, already sorted. To
join them you just open the matching bucket files and merge-sort them together,
walking both in lockstep. No moving key-value pairs across the network. No
shuffle. The expensive sorting was paid once, at write time, and reused on every
future read.

The payoff numbers Spotify reported are the headline: for Wrapped 2020 they
joined on the order of 1 petabyte of data with no conventional shuffle and no
Bigtable intermediate, and cut Dataflow costs by roughly 50% versus the previous
Bigtable-based approach. Collocating similar records also compressed better, for
about a 50% storage reduction, and it let them avoid scaling the Bigtable cluster
up two to three times to survive the peak. And this all rides on Google Cloud
Dataflow's autoscaling and dynamic work rebalancing, so the fleet of workers
grows and shrinks with the job instead of being hand-sized.

### The scale story at three tiers

Tier 1, about 1,000 users. This is a laptop. Load a year of events into memory,
one hash map from user ID to a counts object, loop once, keep a size-5 heap per
user for top songs. Done in seconds. No Dataflow, no Bigtable, no shuffle. The
whole Wrapped fits in RAM. At this size all the machinery above is pure
over-engineering.

Tier 2, about 100,000 users. Now the year of events does not fit on one machine,
so you go distributed: one Dataflow job, read the logs, GroupByKey on user ID,
aggregate. This is where shuffle is born. It still works fine. The shuffle is
real but the data is small enough that Dataflow moves it without drama. A single
monolithic job is acceptable here.

Tier 3, hundreds of millions of users, hundreds of billions of events, petabytes.
The monolithic shuffle is now the bill and the bottleneck. The single GroupByKey
would shuffle the entire year at once. This is where the two survival moves kick
in. First, decompose into many independent per-data-story jobs and assemble them
by writing every job's output into the same per-user Bigtable row under separate
column families, so reassembly is keyed puts, not a shuffle. Then go one level
deeper and pre-bucket-and-sort the inputs with SMB so the joins themselves need
no shuffle at all, joining a petabyte with merge-sorts and cutting cost in half.
What breaks at each tier is always the same thing: the amount of data you have to
move all-to-all. Every fix is a way to move less of it.

### Fact versus inference

Fact, from Spotify's own engineering posts and press: Wrapped is the largest
Dataflow job of the year; 2019 was about 5x 2018 at about three-quarters the
cost; they used Bigtable as an intermediate store with per-user rows and
per-story column families; they decomposed collection, aggregation and
transformation into separate jobs; SMB joined roughly 1PB shuffle-free for
Wrapped 2020 at about half the cost; pipelines are Scio on Dataflow; 2023 used
Lottie for animations.

Inference, clearly labeled: the exact internal data structures (size-5 heaps,
the precise hash-map layouts, the specific percentile algorithm for "top 0.5%")
are my well-grounded reconstruction of how this class of aggregation is normally
built, not published Spotify internals. The trillions-of-events figure is
Spotify's stated platform scale, not a Wrapped-specific count.

## 8. The retention and habit mechanic

Wrapped is the most successful retention machine in this entire ledger, and the
mechanic is unusual: it is not a daily loop, it is one enormous annual spike that
is designed to be shared, not just consumed.

The loop is social, not solo. Ananya's Wrapped is built to be posted. The final
slide is a share card, pre-sized for Instagram Stories, logo baked in. When she
posts "97,412 minutes, top 0.5% Taylor Swift listener," she is not just bragging.
She is advertising Spotify to every friend who does not have a card yet, and
making them feel left out until they open the app and generate their own. Each
share manufactures the next person's craving. That is a genuine viral loop, not a
metaphor.

The real observed numbers are staggering. In 2023, about 227 million users shared
their Wrapped, generating an estimated 2.3 billion social media impressions
across 170 markets and 35+ languages. The #SpotifyWrapped hashtag has tens of
billions of views on TikTok. In 2022, roughly 400 million posts about Wrapped
appeared on X in the three days after launch. And the metric that matters most:
in the first week of December 2020, Spotify saw about a 21% jump in mobile app
downloads right after Wrapped dropped.

Which metric does it move? All three, but the standout is activation and
reactivation. Wrapped drags dormant users like Ananya back into the app once a
year with a reason no discount could buy, and it acquires brand-new users through
the shares of existing ones. The revenue link is indirect but real: a reopened,
re-engaged user is a user who might convert to Premium, and the annual ritual
hardens the emotional switching cost. A decade of Wraps is a decade of your
identity stored in one place. Leaving Spotify starts to feel like burning a
diary.

The habit is annual, but it is a ritual, and rituals are stickier than daily
nags precisely because they are rare and anticipated. People count down to
Wrapped. Nobody counts down to a push notification.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader code plus an embeddable
runtime. The Wrapped pipeline hands you a precise, three-part lesson, all biased
toward the thing you care about: keeping the runtime cheap while the hard work
scales.

The one-line version: do the whole expensive job offline, decompose it into
independent per-key passes that all write into the same keyed row, and pre-sort
your inputs so recombination is a merge, never a shuffle. Then the live path is
one keyed read.

Concretely, three moves lifted straight from Wrapped:

1. Precompute a per-target "wrapped." Wrapped's live path is one lookup into a
   per-user row that was built weeks earlier. Do the same for shader variants.
   Offline, for each shader crossed with each device capability bucket (GPU
   family, memory, feature set), compile and measure the variant, and store the
   finished artifact keyed by (shader_id, device_bucket). At runtime the client
   computes its bucket once and does one keyed read to get a pre-validated,
   already-measured shader. No compiling on the user's phone in the hot path,
   the same way Ananya never re-reads her year when she taps.

2. Decompose into independent passes that write to the same row. Spotify made top
   artist, top genre and minutes into separate jobs that each write a different
   column family of the same user row, so a bug in one does not re-run the
   planet. Do that with your compile pipeline. Make "optimize register
   allocation," "generate the low-power variant," and "bake the shadow LUT"
   independent passes that each write their own column of the same
   (shader_id, device_bucket) artifact row. One pass changing does not force a
   full recompile of everything, and independent passes run in parallel.

3. Bucket and sort by the join key so you never shuffle. This is the SMB lesson
   and it is the most valuable one for you. When you rebuild artifacts across a
   catalog of thousands of shaders and dozens of device buckets, do not do an
   all-to-all regroup every build. Write your intermediate IR bucketed and sorted
   by the exact key you will recombine on (say, target device bucket), so
   assembling the final per-device bundle is a cheap merge of already-aligned
   buckets, not a full shuffle. Spotify joined a petabyte this way at half the
   cost. Your build farm will feel the same win: the cost that used to scale with
   catalog-times-devices collapses toward a linear merge.

The through-line with the rest of this ledger: offline-think, online-lookup, and
the deepest cost in any big recombination job is moving data all-to-all. Kill the
shuffle by pre-sorting on the key you already know you will use. Ananya's tap is
one read because a petabyte of sorting happened while she was asleep.

---

## Sources

- Spotify Engineering, "How Spotify Optimized the Largest Dataflow Job Ever for Wrapped 2020": https://engineering.atspotify.com/2021/02/how-spotify-optimized-the-largest-dataflow-job-ever-for-wrapped-2020
- Spotify Engineering, "Spotify Unwrapped: How we brought you a decade of data" (Wrapped 2019): https://engineering.atspotify.com/2020/02/spotify-unwrapped-how-we-brought-you-a-decade-of-data
- Spotify Engineering, "Big Data Processing at Spotify: The Road to Scio (Part 1)": https://engineering.atspotify.com/2017/10/big-data-processing-at-spotify-the-road-to-scio-part-1
- Scio documentation, "Sort Merge Bucket": https://spotify.github.io/scio/extras/Sort-Merge-Bucket.html
- Scio project (GitHub): https://github.com/spotify/scio
- TechCrunch, "How Spotify ran the largest Google Dataflow job ever for Wrapped 2019": https://techcrunch.com/2020/02/18/how-spotify-ran-the-largest-google-dataflow-job-ever-for-wrapped-2019/
- ByteByteGo, "How Spotify Built Its Data Platform To Understand 1.4 Trillion Data Points": https://blog.bytebytego.com/p/how-spotify-built-its-data-platform
- Forbes, "Spotify Wrapped 2023 ... How It Became A Viral And Widely Copied Marketing Tactic": https://www.forbes.com/sites/conormurray/2023/11/28/spotify-wrapped-2023-comes-soon-heres-how-it-became-a-viral-and-widely-copied-marketing-tactic/
- Sprout Social, "Spotify Wrapped: What marketers can learn from the viral campaign": https://sproutsocial.com/insights/spotify-wrapped/
- KTH thesis (Andrea Nardelli), "Sort Merge Buckets: Optimizing Repeated Skewed Joins": https://kth.diva-portal.org/smash/get/diva2:1334587/FULLTEXT01.pdf
