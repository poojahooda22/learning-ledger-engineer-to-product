# Instagram Reels: how the next video is chosen for you, out of billions, in the time your thumb travels up

Date: 2026-08-07
Product: Instagram (Meta)
Feature: The Reels recommendation and ranking system (the vertical video feed of mostly strangers, and how it picks the very next clip after every swipe)

A note on what this teardown adds to the ledger. We covered the Instagram Explore
grid on 2026-07-20 (candidate generation plus a multi stage ranking funnel) and
the Stories tray on 2026-06-27 (ranking the small ring of people you already
follow). Reels is a third, different shape, and it is the one that now eats half
the app. Explore is a grid of photos you scan with your eyes. The Stories tray
ranks people you chose. Reels is a single, full screen, autoplaying video feed of
people you did NOT choose, delivered one at a time, where the system gets a fresh,
brutally honest grade after every single swipe: did she watch, or did she flick it
away in under a second. The core signal is not a like. It is time. This teardown
is about the funnel that turns a pool of billions of videos into the one clip now
filling Ananya's screen, and how it re-decides in the roughly 300 milliseconds her
thumb is in the air.

---

## 1. The user

Ananya, 29, Bengaluru. It is 9:20 on a Tuesday night. Dinner is done, the dishes
can wait, and she is lying on the sofa with her phone held vertical. She taps the
little clapperboard icon at the bottom of Instagram and a cooking video is already
playing full screen: a pair of hands folding paneer into a green marinade, twelve
seconds long, loud text on top that says "the trick nobody tells you." She watches
it twice without deciding to. Then she flicks up with her thumb. A new video. Flick
up. Another. Flick up. Another.

She is not searching. She has not typed anything. She is not even choosing between
options on a screen. There is exactly one video in front of her at a time, and her
only two moves are keep watching or flick it away. Twenty minutes later she looks
up and does not fully remember the last thirty clips. That trance is the product
working exactly as designed.

Multiply Ananya by the planet. Reels are played more than 200 billion times per
day across Instagram and Facebook, up from roughly 100 billion in late 2024 (Mark
Zuckerberg, Meta Q3 2025 earnings call). Reels are reshared more than 4.5 billion
times per day (Meta). In the United States, Reels made up about 46 percent of all
time spent inside the Instagram app in 2025, up from 37 percent in 2024. Half the
app is now this one vertical feed.

---

## 2. The real problem

Here is the pain, described like a friend would. Ananya opens the app tired and a
little bored. She wants to feel something in the next second, not the next minute.
If the first clip is boring, she is gone, either to a better clip or out of the app
to WhatsApp. The system has no title she typed, no search box, no list she is
scanning. It has one shot, then the next, then the next, and every one of them has
to earn the swipe.

Now flip to the platform's side of the same pain. When Ananya opened Explore or
searched "watch," she told the system what she wanted. In Reels she tells it
nothing up front. The catalog is not a few thousand Netflix titles and it is not
Amazon's 350 million products. It is billions of short videos, and millions of new
ones are uploaded every day, most of them from people Ananya has never heard of and
does not follow. The system has to pick, right now, the single clip most likely to
stop her thumb, out of that ocean, for her specifically, and then do it again a
fraction of a second later when she swipes. And it has to do this simultaneously for
billions of people. That is the problem: no query, an almost infinite and constantly
refreshing pool, a one at a time delivery, and a grade after every swipe.

A plain "most popular Reels today" feed fails, the same way "most popular titles"
failed on the Netflix homepage: it shows the same twenty viral clips to everyone,
and Ananya, who wants paneer recipes and stand up comedy, does not want the same
feed as her cousin who wants cricket and cars. A "only people you follow" feed fails
too, because Ananya follows maybe 400 accounts and they do not post enough good
short video to fill a twenty minute trance. The whole magic of Reels, and of TikTok
before it, is that it mostly shows you strangers, and it is right often enough that
you keep swiping.

---

## 3. The feature in one sentence

Reels is a per person, one video at a time, vertically scrolling feed that predicts,
before you have watched anything, which unseen clip out of billions will hold your
attention next, learns from whether you actually watched it, and immediately uses
that to pick the one after it.

---

## 4. Jobs to be done

What is Ananya really hiring Reels to do at 9:20 on a Tuesday?

- "Fill the next twenty minutes so I do not have to think or choose." The job is
  effortless time. Zero decisions. The feed decides for her.
- "Make me feel something small and good, fast." A laugh, a recipe she saves, a
  dance she sends to her sister. Little hits, one after another.
- "Show me my world without me asking." Paneer recipes, Bengaluru cafes, the
  comedians she likes, but also the occasional surprise she did not know she
  wanted. Familiar enough to be comfortable, surprising enough to not be boring.
- "Do not make me work." No search, no browsing a grid, no reading titles. Just
  play, and let her thumb be the only interface.

And what the creator on the other side is hiring it to do: "Take my twelve second
paneer clip and, if it is genuinely good, put it in front of strangers who will
love it, even though I have 800 followers." Reels is the rare feed where a nobody
can reach a million people in a day if the clip earns it. That promise is why
creators keep posting.

---

## 5. How it works for the user (the visible experience)

Ananya taps the Reels tab. A video is already playing, full screen, sound on,
looping. There is no grid, no list, no "choose one of these." Down the right edge:
a heart (like), a speech bubble (comment), a paper plane (share, usually a DM to a
friend), and a three dot menu that hides "Not interested" and "Save." At the bottom:
the creator's handle, a caption, and the audio track name.

She does four things, mostly without thinking:

- She watches, or she does not. If the first second grabs her she stays. If not she
  flicks up almost instantly.
- She flicks up to go to the next clip. This is the main action, and it happens
  hundreds of times a session.
- Once in a while she double taps to like, or holds and taps the paper plane to send
  a clip to her sister on DM, or taps Save on a recipe she wants later.
- Very rarely she opens the menu and taps "Not interested," usually after two or
  three clips of something she is sick of.

That is the entire surface. The genius and the danger are the same: the cost of
consuming one more is a single thumb flick, and the feed never ends.

---

## 6. The actual flow, tap by tap

Follow one real session.

1. Ananya taps the Reels tab at 9:20:03. The app sends a request to Meta's servers:
   "user 4471..., give me a page of Reels." It also ships along recent context: what
   she watched in the last few minutes, her rough location (Bengaluru), the time,
   the device.
2. The server does not send one video. It sends a small ranked batch, maybe 10 to
   20 clips, so the next few swipes are instant and do not each need a round trip.
   The phone starts playing clip number one (the paneer video) and quietly
   pre loads clips two and three in the background.
3. Ananya watches the paneer clip fully, twice. The phone is recording this the
   whole time: play started, still watching at 3 seconds, still watching at 6, hit
   the end, looped, looped again, total watch time 26 seconds on a 12 second clip.
   No tap needed. Watch time is the signal, and it is collected passively.
4. She flicks up. In the roughly 300 milliseconds her thumb is moving, clip two is
   already on screen and playing, because the phone pre loaded it. Meanwhile the
   phone streams the paneer engagement back to the server: "watched 2.1x through,
   no like, no share." That single fact will reshape what comes later in the
   session.
5. Clip two is a comedy skit. She watches 4 seconds, does not laugh, flicks up. The
   phone logs "skipped after 4 seconds, under one full play." That is a soft
   negative.
6. Clip three is another recipe. She watches fully, then taps the paper plane and
   sends it to her sister. A DM share. This is a strong positive, worth far more
   than a like (more on why below).
7. By now the phone is near the end of the batch it was given. Around clip 8 or so,
   it quietly asks the server for the next batch, and the server generates it fresh,
   this time already knowing about the paneer watch, the comedy skip, and the recipe
   share from this very session. The feed has adjusted mid scroll.
8. Twenty minutes and roughly 60 clips later, Ananya locks her phone. Every one of
   those 60 clips left a grade behind: watch time, completion, likes, shares, saves,
   skips. Tomorrow night's feed is a little more paneer, a little less of that
   comedian.

The thing to hold onto: the phone renders and measures, the server decides. The
phone never picks which billions-wide video comes next. It just plays what it was
handed, and reports back honestly.

---

## 7. Under the hood, like the engineer

This is the heart of it. First, an honesty note on sources. Meta has published the
detailed architecture of the Instagram Explore recommender ("Scaling the Instagram
Explore recommendations system," Engineering at Meta, August 2023) and the
operational story of running the model fleet ("Journey to 1000 models," Engineering
at Meta, May 2025). Meta has NOT published a blow by blow of the Reels ranker
specifically. So: the funnel shape, the two tower retrieval, the two stage ranking,
the multi task model, and the value model below are confirmed for Instagram's
recommendation systems as a class, in Meta's own words. Where I map them onto Reels
specifics (watch time as the dominant label, the exact clip level signals) I lean on
Meta's public statements about Reels ranking, including Adam Mosseri's 2026 comments,
and I label the rest clearly as grounded inference.

### The shape: a funnel, not a sort

You cannot score billions of videos for one person in 300 milliseconds. Nothing can.
So Reels, like every large recommender, is a funnel: start with billions, cut to
thousands, cut to hundreds, cut to a handful, and only the last tiny stage runs the
expensive brain. Meta describes exactly this for Explore: "a multi stage funnel
approach, starting with thousands of candidates and narrowing down the number of
candidates to hundreds as we go down the funnel." Two rules make a funnel work:
cheap stages run on many items, expensive stages run on few items, and each stage
only has to be good enough to not throw away what the next stage would have loved.

Walk it top to bottom with Ananya's session.

### Stage 0: Retrieval (billions to thousands), the two tower trick

The first job is to pull a few thousand plausible clips out of a pool of billions,
fast, for Ananya specifically. You cannot score them one by one; there are too many.
The trick is the two tower neural network, and it is the single most important data
structure idea in the whole feature.

Picture two separate neural nets. One is the user tower: it takes everything about
Ananya (who she follows, what she watched tonight, the paneer she just loved, her
location, time of day) and boils it down to a list of, say, 256 numbers. That list
is her embedding, a point in a 256 dimensional space. The other is the item tower:
it takes a video (the paneer recipe, its audio, its captions, who made it, who else
watched it to the end) and boils it down to its own 256 number embedding, a point in
the same space. The training trick is that videos Ananya would love land near her
point, and videos she would flick away land far away. "Near" means the dot product
of the two number lists is high.

Why two separate towers instead of one model that eats a (user, video) pair together?
Because of caching, and this is the whole reason the design exists. Every video's
embedding can be computed once, ahead of time, the moment it is uploaded, and stored.
It does not depend on Ananya. So you precompute embeddings for the entire catalog and
load them into a special index. Then, at request time, you only have to compute ONE
fresh embedding, Ananya's, and ask the index: "which stored points are nearest to
this one?" Meta states this directly for Explore: they use the two tower model
because of "its cacheability property," store item embeddings in "a service that
supports online approximate nearest neighbors search," and "during online retrieval,
the user tower generates a user embedding on the fly to find the most similar items
in the ANN service."

That phrase, approximate nearest neighbors (ANN), is the other key data structure.
Finding the truly nearest points among billions by checking every one is far too
slow. So the index is built as a graph or a tree that lets you hop toward the
neighborhood of the answer and check only a few thousand candidates instead of
billions. The common structures are HNSW (a layered "skip list for geometry," you
land in a rough region from a coarse top layer and refine downward) and IVF/FAISS
style clustering (bucket the space into cells, only search the few cells near the
query). We covered this machinery in Lesson 33 (vector search and ANN at scale). The
point for Reels: turning "find me relevant videos" into "find me nearby points" is
what makes billions to thousands possible in milliseconds.

And crucially, retrieval is not one source. Meta says the retrieval stage "consists
of multiple candidate retrieval sources," some heuristic, some ML, some real time
"capturing most recent interactions" and some pre generated "capturing long term
interests." For Ananya that means one source pulls "videos similar to the paneer clip
you just watched" (real time, this session), another pulls "recipe and Bengaluru food
content you have liked for months" (long term), another pulls "what is blowing up right
now near you" (trending), another pulls "clips your sister and friends resent recently."
Each source coughs up a few hundred, and they get pooled into a few thousand candidates.

### Stage 1: First stage ranking (thousands to hundreds), the cheap judge

Now there are a few thousand candidates. Still too many for the heavy model. So a
lightweight ranker scores all of them and keeps the top few hundred. Meta's clever
move here is distillation: the first stage ranker is trained to predict what the
expensive second stage would have said. In Meta's words, the label is "PSelect =
media in top K results ranked by the second stage." The little model learns to
imitate the big model's verdict cheaply, so it rarely throws away a clip the big
model would have loved. It is a fast intern trained to guess what the senior editor
will pick, so the senior editor only ever reads a short list.

### Stage 2: Second stage ranking (hundreds to a handful), the expensive brain

Now, for the surviving few hundred, the heavy model runs. This is the multi task,
multi label (MTML) neural network. Instead of predicting one number, it predicts many
probabilities at once for each (Ananya, video) pair: probability she watches most of
it, probability she completes it, probability she likes it, probability she comments,
probability she sends it to a friend on DM, probability she saves it, probability she
taps "Not interested." Meta describes exactly this for Explore: the second stage "predicts
the probability of different engagement events (click, like, and so on) using the multi
task multi label (MTML) neural network model," which is "much heavier than the two towers
model." It is heavy because it looks at the full rich features of both the user and the
item together, no caching shortcut, which is exactly why it is only allowed to see a few
hundred survivors and not billions.

### The value model: turning many predictions into one number

The MTML model hands back a fistful of probabilities per clip. But the feed needs one
ordering. So a value model combines them into a single score, and this formula is where
the platform's priorities live. It is, in essence, a weighted sum:

score = w1 x P(watch) + w2 x P(complete) + w3 x P(like) + w4 x P(share) +
        w5 x P(save) + w6 x P(comment) - w7 x P(not interested) - ...

Meta confirms this general form: rankings come from "a linear combination of engagement
predictions," combining predicted actions "in the same way for everyone," with "hundreds
of additional terms" for context, and the weights tuned to "maximize long term retention."
The weights are the editorial policy of the product, expressed as numbers.

And here is the Reels specific twist, the thing that makes Reels different from a photo
feed. The dominant terms are about time, not taps. Adam Mosseri confirmed in January 2026
that the top Reels signals are watch time, likes per reach, and DM shares, and that
completion rate is the single most powerful signal for reach. Critically, a DM share is
worth far more than a like: Meta has indicated shares to friends are on the order of 3 to
5 times as valuable as a like for reaching non followers, because a share is Ananya
spending her own social capital, the strongest possible vote. This is why, in her session,
watching the paneer clip twice (26 seconds on a 12 second video) and sending the second
recipe to her sister moved her feed hard toward recipes, while a passive like would have
barely nudged it.

There is a symmetric negative side, and it is sharp. Skipping a clip in under a second,
tapping "Not interested," muting, or hiding are strong negative terms (the minus signs in
the formula). A skip rate above roughly 40 percent on a clip is a clear signal the opening
second is not landing, and the system stops widening its distribution. Ananya flicking away
the comedy skit after 4 seconds was a soft down vote; if thousands of people do the same in
the clip's first small test audience, that clip stops spreading. Reels grades on attention
captured, and it treats a fast skip as the honest answer that a like never is.

### The final reranking: do not show six paneer clips in a row

One more light pass sits on top. Even after the value model ranks the handful, you do not
just show them in raw score order, because the top six might all be paneer recipes and Ananya
would get bored of her own interest. A reranking step enforces diversity and business rules:
space out same creator and same topic clips, mix in some freshness, weave in ads (Reels now
carry more than half of Instagram's ad placements). This is the same "rank the row in the
context of the whole page" idea we saw in the Netflix homepage teardown (2026-08-06): local
score is not enough, the sequence as a whole has to be good.

### Where the sorting happens (say it plainly)

Server side. All of it. The phone computes nothing about billions of videos. The two tower
ANN lookup, the first stage cut, the MTML scoring, the value model combination, the diversity
rerank: every step runs in Meta's data centers, and the phone receives a small ready made
ranked batch of 10 to 20 clips plus the video bytes to stream. The phone's only jobs are to
play the clip, pre load the next two, and honestly report back what Ananya did. This is the
same offline-think, online-lookup split that runs through this whole ledger: do the heavy
combinatorial work off the hot path, and make the live request as close to a keyed read as
you can.

### The scale story at three tiers

Tier one, 1,000 videos and a few hundred users. There is no funnel. You score every video
for every user with the good model and sort. A single machine and a Postgres query would do
it. Two tower retrieval, ANN, distillation: all pointless over engineering at this size. The
naive O(users x videos) scan is fine.

Tier two, 100,000 videos and a few million users. Now scoring every video per request starts
to hurt, and per user precompute volume grows. This is where the funnel earns its keep: build
the two tower ANN index so retrieval is a nearest neighbor hop instead of a full scan, add the
cheap first stage ranker so the expensive model only sees a few hundred items, and cache item
embeddings so you are not recomputing them per request. Watch time and engagement events pour
in as a firehose, so you need a streaming pipeline (Kafka style logs, Lesson 42) to fold
tonight's paneer watch into features fast. What broke at the previous tier: scoring cost per
request. What saved you: the funnel and the ANN index.

Tier three, billions of videos, more than 2 billion users, millions of recommendation requests
per second (Meta's stated scale). Now new things break that did not exist before. First, the
sheer volume of models. Instagram runs more than 1,000 ML models in production (Meta, "Journey
to 1000 models," May 2025), and the bottleneck stops being any one model's math and becomes
operational: launching, monitoring, and not letting one sick model quietly poison the feed.
Meta's answer was infrastructure, not a smarter net: a central model registry built on
Configerator (their config system), an automated deployment pipeline that cut new model launch
from days to hours, and unified health metrics combining calibration and normalized entropy to
catch a mispredicting model automatically. Second, serving cost. Running heavy neural nets for
millions of requests per second is a hardware problem solved with batching and accelerators
(the same continuous batching idea as LLM serving, Lesson 44) and heavy caching of the cacheable
tower. Third, freshness at scale: millions of new clips a day must get embedded and become
retrievable within minutes, or trending content is stale before it spreads, so the item tower
runs as a streaming job on upload, not a nightly batch. What breaks at this tier is no longer the
algorithm, it is the fleet: a thousand models, a firehose of events, and a millions per second
request rate. What survives it is boring, unglamorous platform work, a registry, a deploy pipeline,
health metrics, streaming features, and it is exactly the work that lets the interesting models
keep shipping.

---

## 8. The retention and habit mechanic

Reels is, mechanically, a variable ratio reward schedule, the same schedule that runs a slot
machine, and that is not a metaphor, it is the design. Each swipe costs almost nothing (one thumb
flick) and pays off unpredictably: most clips are fine, some are boring, and every so often one is
so good Ananya laughs out loud or saves a recipe or sends it to her sister. Unpredictable rewards
on a near zero cost action is the most habit forming loop known, and the vertical, one at a time,
autoplaying, never ending feed is the purest possible delivery of it. There is no natural stopping
point, no end of page, no "you are all caught up." The feed just refills.

The loop that brings her back tomorrow: the feed got measurably better tonight because she watched
and shared, so tomorrow's first few clips are more likely to land, so the very first swipe pays off
faster, so she stays longer, so it learns more. Watch, learn, better feed, longer watch. The
flywheel spins on watch time, which is why the value model weights watch time and completion above
likes: the metric it optimizes is the metric that keeps her.

Which metric does it move? All three, but retention first and revenue right behind it, and the real
numbers are stark. Reels grew from about 100 billion daily plays in late 2024 to more than 200
billion in 2025 (Zuckerberg, Q3 2025 call). It rose from 37 percent to 46 percent of US time in the
Instagram app in a single year. And it converts straight to money: Reels crossed a 50 billion dollar
annual ad run rate across Instagram and Facebook (Meta, October 2025 earnings), and more than half of
all Instagram ad placements now run inside Reels, up from about 35 percent in 2024. The habit loop and
the revenue line are the same line. Every extra minute Ananya swipes is both a retention tick and an
ad impression, which is precisely why the feed is built to never end.

A concrete observed example of the mechanic in the wild: the "sent this to my sister" moment. Reels
counts 4.5 billion reshares per day, and the value model prizes a DM share at several times a like
specifically because it doubles as retention (Ananya came back to send it) and as growth (her sister
now opens the app to watch it). One strong action, a share, feeds the loop from both ends. That is the
mechanic, not a guess: Meta weights the action that most reliably brings two people back.

---

## 9. The lesson for Rare.lab

Rare.lab is a node based editor for AI shaders and visual effects that compiles to shippable code, plus
an embeddable runtime. The Reels lesson is about how you serve a preset or effect gallery, and it is a
scalability lesson wearing a recommendations coat.

The concrete lesson: build effect discovery as a two tower retrieval plus a cheap-then-expensive ranking
funnel, and never score your whole effect library live on the hot path.

Here is the mapping, made specific. As Rare.lab grows to tens of thousands of community effects and
presets, a creator opening the "suggest an effect" panel is Ananya opening Reels: they have not typed a
query, they want the right effect for THIS scene surfaced now, and you cannot afford to evaluate every
effect against their project live. So do what Reels does.

- Precompute an embedding for every effect the moment it is published (its node graph shape, what inputs
  it needs, its performance cost, which projects used it, its visual output). This is the cacheable item
  tower. It never depends on the current user, so it is computed once and stored in an ANN index (HNSW or
  FAISS, Lesson 33).
- At request time compute one fresh embedding for the current context (this project's style, the device's
  GPU budget, what the creator just added) and do a nearest neighbor lookup to pull a few hundred candidate
  effects out of tens of thousands, in milliseconds, instead of scanning them all.
- Then run a cheap first stage filter (does this effect even compile for this target device and frame
  budget?) before the expensive stage (a full quality-and-fit score). Cheap stage on many, expensive stage
  on few. A mobile target with a 16 millisecond frame budget should never even reach the expensive scorer
  for a 4K-only effect.
- Combine the final signals with an explicit weighted value model, and make performance a first class,
  heavily weighted term, not an afterthought. Reels weights watch time above likes because watch time is
  what it actually wants. Rare.lab should weight "renders inside the frame budget on this device" above
  "looks impressive in a demo," because a shader that drops frames is the flick-up-in-one-second of visual
  effects: the user abandons it instantly. Bake the frame budget constraint into the ranking score, the way
  Reels bakes the skip into the negative terms, so a too-heavy effect is ranked down before it is ever
  suggested, not filtered out after it janks.

And the operational half, from "Journey to 1000 models": when you have many compiled effect variants and
scoring models in play, your bottleneck stops being any single model and becomes the fleet. Invest early in
a registry of effects and their compiled per-device variants, an automated publish pipeline, and health
metrics that catch a bad or slow effect automatically, because at scale the unglamorous plumbing is what
lets the interesting effects keep shipping. Two tower to find candidates fast, cheap-then-expensive to rank
them, performance as a weighted term in the score, and boring infra to run it all: that is the Reels recipe,
and it is exactly the shape a scalable effects gallery needs.

---

## Sources

Confirmed engineering, Meta:

- Scaling the Instagram Explore recommendations system, Engineering at Meta, August 2023.
  https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/
  (the multi stage funnel, two tower retrieval and its cacheability, ANN service, first stage distillation
  with the PSelect label, second stage MTML model)
- Journey to 1000 models: Scaling Instagram's recommendation system, Engineering at Meta, May 2025.
  https://engineering.fb.com/2025/05/21/production-engineering/journey-to-1000-models-scaling-instagrams-recommendation-system/
  (over 1,000 models in production, model registry on Configerator, automated deployment, calibration and
  normalized entropy health metrics)
- Instagram Feed Ranking System Card, Meta AI.
  https://ai.meta.com/tools/system-cards/instagram-feed-ranking/
  (predicted actions such as PLIKE, PCOMMENT, PFOLLOW combined into one relevance score)
- Instagram Explore ranking, Meta Transparency Center.
  https://transparency.meta.com/features/explaining-ranking/ig-explore/

Reels signals and scale (Meta statements and reporting):

- Mark Zuckerberg, Meta Q3 2025 earnings call: more than 200 billion daily Reels plays, 50 billion dollar
  annual ad run rate. Coverage: https://www.tubefilter.com/2025/10/30/meta-reels-ad-revenue-q3-2025-earnings-report/
- Most of Instagram's ads ran on Reels in 2025, CNBC, January 2026.
  https://www.cnbc.com/2026/01/20/most-of-instagrams-ads-ran-on-reels-in-2025-data-shows.html
- Adam Mosseri (2026) on top Reels signals (watch time, likes per reach, DM shares) and completion rate as
  the strongest reach signal, and negative signals (skip rate, Not interested). Aggregated: Hootsuite,
  https://blog.hootsuite.com/instagram-algorithm/ ; Later, https://later.com/blog/how-instagram-algorithm-works/

Background and independent explainers:

- How Instagram Reel Uses Recommender Systems, GeeksforGeeks.
  https://www.geeksforgeeks.org/machine-learning/how-instagram-reel-uses-recommender-systems/
- The Engineering behind Instagram's Recommendation Algorithm, Quastor.
  https://blog.quastor.org/p/engineering-behind-instagrams-recommendation-algorithm-dc9c

Related ledger lessons: Lesson 7 (feed ranking at scale), Lesson 18 (search ranking internals),
Lesson 33 (vector search and ANN at scale), Lesson 42 (Kafka partitioned commit log), Lesson 44
(LLM inference serving and continuous batching).

Inference labels: the funnel shape, two tower retrieval, ANN service, first stage distillation, second stage
MTML model, and value model are confirmed by Meta for Instagram's recommendation systems as a class (Explore
and Feed). Their exact application to the Reels ranker, and the specific per clip signal weights, are grounded
inference built on Meta's public Reels statements, and are labeled as such in the text above.
