# Netflix: personalized artwork (which poster you see for a title)

Date: 2026-06-19
Product: Netflix
Feature: Per-member artwork personalization (the thumbnail/poster picked for each title, for each person)

---

## 1. The user

It is 9:40 pm on a Tuesday. Priya has finished dinner, the dishes can wait, and
she has exactly one hour before she wants to sleep. She opens Netflix on the
living room TV. She is not searching for anything. She did not come with a title
in mind. She is in the mode every streaming exec fears and depends on: browsing.
She scrolls the home screen, eyes flicking across rows of rectangular box art,
and somewhere in the next minute or two she will either start something or give
up and open Instagram instead.

What Priya sees on each tile is not a fixed official poster. The image for
"Stranger Things" that lands in front of her may be different from the image her
flatmate sees for the exact same show on his phone in the next room. That single
picture is doing an enormous amount of quiet work.

## 2. The real problem

Here is the honest version, friend to friend. Netflix has thousands of titles.
Priya will look at each tile for a fraction of a second. Research Netflix itself
has cited says a member gives the home screen roughly 90 seconds and looks at
about 10 to 20 titles before deciding to watch or to leave. The box art is often
the single biggest factor in whether she even considers a title. A great show
with a flat, generic poster gets skipped. A mediocre show with a gripping image
gets a click.

So the problem is not "what should we recommend." That is a separate, older
system. The problem is: given that we already decided to show Priya "Good Will
Hunting" in her row, which single picture of it gives her the best chance of
pressing play. The right answer is different for different people. Someone who
loves romance should maybe see Matt Damon and Minnie Driver in a tender moment.
Someone who loves comedy should maybe see Robin Williams, the comedian they
recognize. Same film. Different door in.

## 3. The feature in one sentence

For every title shown to you, Netflix picks one image out of a small set of
candidate images, choosing the one most likely to make *you specifically* press
play, and it learns which image works from how people actually behave.

## 4. Jobs to be done

- "Help me find something worth my one free hour before I give up." (Priya)
- "Make this title look like it is for me, not for everyone." (the pull)
- "Show me a face, a tone, or a moment I already trust." (familiarity)
- For Netflix: "Convert a browsing session into a play, and a play into a finished
  show, so the member renews next month."

## 5. How it works for the user

Priya never sees a setting for this. There is no "personalize my posters"
toggle. It is completely invisible. She just notices, if she notices at all,
that the art feels right. The "Stranger Things" tile shows the kids on bikes
with a warm 80s glow because Netflix has learned she watches a lot of nostalgic
family adventure. Her flatmate, who mainlines horror, gets the darker tile with
the Demogorgon and a blood-red tint for the same show.

The image is also stable within a session. It does not flicker between options
while she scrolls. It holds still so the screen does not feel broken.

## 6. The actual flow, step by step

1. Priya opens Netflix. Her client asks the server to build the home page.
2. The recommendation system decides *which* titles go in which rows. (Not our
   feature. This is the older personalized ranking system.)
3. For each title that made the cut, a second system asks: which image for this
   title, for this member, right now. Call it the artwork selector.
4. The selector pulls the candidate image set for that title (a small list of
   pre-approved images, all of the same show).
5. It scores each candidate for Priya using a model fed with her context (her
   taste history, country, device, time, the row the title sits in).
6. It picks one image, usually the highest scoring one, sometimes a deliberately
   randomized one for learning (more on that below).
7. The chosen image URL goes back with the page. Priya sees the tile.
8. Netflix logs what it showed, why, and what Priya did next: hover, click,
   play, and crucially whether she watched enough to count as a real "take."
9. That log feeds tomorrow's model. The loop closes.

## 7. Under the hood, like the engineer

This is the heart of it. The feature is really two different systems bolted
together, and they solve two different problems. Keep them separate in your head:
**candidate generation** (where do the images come from) and **selection** (which
one do we show you). Matching and ranking, the same split you see in search, just
with pictures instead of documents.

### Half one: candidate generation (the AVA pipeline)

Before anyone can personalize, you need a handful of good images per title. A
two-hour movie is roughly 172,800 frames at 24 fps. You cannot ask a human to
look at all of them, and you cannot pick at random, because most frames are
useless: motion blur, a black transition, the back of someone's head.

Netflix built a system called **AVA (Aesthetic Visual Analysis)** to harvest
candidate frames automatically. The data structure here is, at bottom, a big
table: one row per frame, with a vector of annotations attached. Netflix groups
those annotations into three buckets:

- Visual metadata: brightness, sharpness, color, contrast.
- Contextual metadata: face detection, motion estimation, object detection,
  which actor is on screen, what kind of camera shot it is.
- Composition metadata: depth of field, symmetry, the rule-of-thirds style
  framing principles, plus practical filters like "probability of nudity" so a
  bad frame never becomes a poster.

Computing all of that over every frame of the whole catalog is a brutal batch
job. Netflix runs it on **Archer**, a MapReduce-style media processing platform
that uses containers. The trick that makes it scale: split each video into small
chunks and process the chunks in parallel, so a growing catalog stays within a
predictable time budget. This is classic offline heavy-lifting. It runs once per
title when content is onboarded, not when Priya is browsing.

The output is a ranked shortlist of strong frames per title. Humans and editorial
tools then refine these into the final candidate set: the "image suite." Each
title ends up with a small number, N, of approved images. N is small. Think a
handful up to a couple dozen, not thousands. That smallness matters enormously
for half two, as you will see.

### Half two: selection as a contextual bandit

Now the live problem. Title is chosen, image suite of N candidates is ready, and
a specific member is looking. Pick one image. Netflix frames this as a
**contextual multi-armed bandit**, and the framing is the whole insight.

A plain multi-armed bandit is the slot-machine problem: N levers, each pays out
at some unknown rate, find the best one while spending as little as possible
pulling bad levers. The "contextual" part adds the key twist: the best lever
depends on *who is pulling it*. The context is the member.

Map it onto our problem:

- **Arms** = the N candidate images for this title. For "Good Will Hunting" maybe
  arm 1 is Robin Williams alone, arm 2 is Damon and Driver close together, arm 3
  is the two leads on a park bench.
- **Context** = a feature vector describing the moment: the member's taste
  profile (how much they lean comedy vs romance vs thriller, built from watch
  history), country, language, device, time of day, and which row and rank the
  title sits at.
- **Reward** = the **take fraction**. Did the member actually play this title
  after seeing this image, and did they watch enough of it to count as a genuine
  engagement rather than a 30-second bail. A click that leads to a quick quit is
  not a real win. Netflix optimizes for the quality play, not the bait.

The model learns a function: given (member context, candidate image features),
predict the probability this member takes the title. At serve time you compute
that score for each of the N arms and, in the simplest **greedy** policy, show
the argmax. The image features are not just "image id." They include the AVA
annotations and learned image embeddings, which is what lets the system say
useful things about a brand-new image it has barely shown to anyone, by analogy
to images it knows ("close-up of a recognizable face on a warm background tends
to do well for nostalgia-leaning members").

#### Data structures and algorithms actually in play

- The image suite per title: a small fixed-length **array** of N candidates.
  Small N is deliberate.
- Per-frame annotations: a **table / feature matrix**, one row per frame, dense
  numeric vectors. Produced offline.
- The member context: a **feature vector**, much of it precomputed (taste
  affinities do not change second to second), so the live path does not
  recompute heavy history on each request.
- Selection: for a linear bandit (the workhorse class here, think LinUCB style),
  scoring an arm is a **dot product** of weights and the combined context-image
  vector. N arms means N dot products and one argmax. That is cheap and constant
  in catalog size, which is exactly what you want on the hot path. The expensive
  thinking (training the weights) is offline; the live decision is a few dot
  products and a max over a short array.

#### Exploration: why you cannot always be greedy

If you always show the current best image, you never gather evidence about the
others, and you can get stuck believing a wrong answer forever (the classic
exploit-only trap). So the policy injects controlled randomness:

- **Epsilon-greedy**: most of the time show the predicted-best image, but a small
  fraction of the time show a uniformly random candidate, on purpose, to keep
  learning.
- **Closed-loop / adaptive** schemes (Thompson sampling, LinUCB-style upper
  confidence bounds): vary the amount of randomization by how *uncertain* the
  model is about an arm. Explore the images you are unsure about; stop wasting
  shows on images you already know are bad. Netflix's own write-up describes the
  spectrum from "simple epsilon-greedy with uniform randomness" to "closed loop
  schemes that adaptively vary the degree of randomization as a function of model
  uncertainty."

Every served impression is logged with the image shown and, importantly, the
**propensity**: the probability the policy had of showing that image. You need
that number to evaluate honestly. Which brings us to the cleverest piece.

#### Replay: how you test a new model without launching it

You have a new artwork model. Is it better than the live one. The naive answer is
a live A/B test, which is slow and risks showing real members worse art for
weeks. Netflix instead uses an offline technique called **Replay**.

It works like this. In the logged data, a slice of traffic was served images
chosen *uniformly at random* among the candidates (that is the exploration data,
where every image had an equal chance). Now take your new model. For each logged
event, ask: would my new model have picked the same image that was randomly shown
here. Keep only the events where the answer is yes. Throw the rest away. Over that
matched subset, compute the take fraction. Because the original assignment was
uniform random, that matched subset is an unbiased sample of "what would have
happened if we had run my new policy." No live test needed.

It is beautiful and it is data-hungry, and the data hunger explains a design
choice. With N candidates and uniform random logging, only about 1 out of N
logged events will match your policy's pick. You discard roughly the
(N-1)/N fraction. If N were 10,000 you would throw away 99.99 percent of your
logs and have almost nothing left to measure with. Because N is a handful, you
keep a usable slice. **This is why the candidate set is kept small.** The size of
the action space is not a UX detail; it is what makes off-policy evaluation
statistically possible.

(For grounding: Netflix's published example walks a toy case with a take fraction
of 2/3 over the matched subset. The real systems run on far more data, but the
mechanic is exactly that: match on equal assignment, then average the reward.)

### The scale story, three tiers

**1,000 titles.** You could almost hand-pick posters. A human art team could
choose one strong image per title and ship it. No personalization needed, no
pipeline needed. The problem does not exist yet.

**100,000 titles (catalog scale).** Now humans cannot watch everything to find
good frames, and one global poster per title leaves engagement on the table. This
is where AVA earns its keep: automated frame harvesting and annotation, run as a
parallel batch on Archer so a growing catalog still finishes on a predictable
SLA. Personalization starts to pay because the diversity of members is large
enough that one-size art is clearly suboptimal. What breaks at the edge of this
tier is *evaluation*: you cannot A/B test every model idea on members; replay
becomes essential, and replay forces small N.

**Hundreds of millions of members, billions of decisions, 10M+ requests per
second at peak.** Now the live constraint dominates. The artwork decision sits on
the page-build hot path for every tile of every home screen for every member,
worldwide. It must be a few dot products and an argmax over a short array, server
side, with member features precomputed and cached, never a model retrain or a
catalog scan. Decisions get cached per session so the image is stable and so you
are not recomputing on every scroll. What breaks at this tier and how they
survive it:

- **Replay data sparsity.** Uniform random exploration at planet scale wastes
  real impressions on bad images and still only yields 1/N matched events. Fix:
  closed-loop adaptive exploration to randomize less where the model is already
  confident, squeezing more signal per wasted show.
- **Cold start for new titles.** A brand-new show has zero logged takes. Fix:
  lean on image-level features and embeddings (the AVA annotations) and a pooled
  model so a new image inherits priors from similar images, rather than starting
  blind.
- **Stability vs freshness.** Members hate flickering art, but the model keeps
  learning. Fix: lock the chosen image for the session/context and update between
  sessions, not mid-scroll.

### What is fact and what is inference here

Fact, from Netflix's own engineering writing: the contextual bandit framing, the
take fraction reward, the Replay off-policy evaluation method, the epsilon-greedy
to closed-loop exploration spectrum, the AVA frame-annotation pipeline, the
Archer chunked parallel processing, and the "1 of N images from the title image
suite" formulation. The specific *internal* numbers (exact N per title today,
exact lift, exact feature list, exact model class in production now) are not all
public. The "few dot products, argmax, precompute the context, cache per session"
description of the live path is the well-grounded inference version of how this
class of low-latency linear-bandit serving is built, labeled as inference.

## 8. The retention and habit mechanic

The artwork is the storefront window. The loop it powers is the browse-to-play
conversion inside that fragile 90-second window. Every session where Priya finds
something fast is a session that ends in a play instead of a close. Plays build
into finished shows, finished shows build into "Netflix is where I find things I
like," and that belief is what gets renewed at the end of the month. The metric
this moves is **engagement and retention**, not acquisition. It does not bring
new members; it keeps the existing ones from bouncing on a dead browse.

Real observed example: the same "Stranger Things" is presented with the warm,
kids-on-bikes nostalgia art to a member with a family-adventure history, and with
the dark Demogorgon art to a horror fan. Netflix has publicly used "Good Will
Hunting" (Robin Williams for comedy lovers, Damon and Driver for romance lovers)
and the multi-image suites for titles like "Stranger Things" to illustrate
exactly this. The win is not vanity; it is that the romance fan who would have
scrolled past the comedy-framed poster now stops, recognizes a moment for her,
and presses play. A stopped scroll is a saved session.

## 9. The lesson for Rare.lab

Rare.lab compiles node graphs to shippable shader code and ships an embeddable
runtime. The transferable pattern from Netflix here is not "personalize," it is
**push the heavy, branchy decision offline into a small precomputed candidate set,
then make the live path a constant-time pick, and instrument it so you can judge a
new policy from logs instead of a live test.**

Concretely:

1. **Precompile a small suite of variants, not one mega-shader and not infinite
   on-device compiles.** AVA does the expensive frame analysis once per title
   offline and hands the runtime a short array of N candidates. Rare.lab should
   compile each effect into a small, fixed set of pre-approved variants per
   device class (high-end desktop GPU, mid mobile, integrated, WebGL fallback),
   shipped as an "image suite" equivalent. The runtime then does a cheap lookup
   plus a tiny scoring step to pick the right variant for the actual GPU it woke
   up on, instead of compiling a fresh shader on the user's device at load and
   stalling the first frame.

2. **Keep N small on purpose, because small N is what makes evaluation cheap.**
   Netflix keeps the candidate count low so Replay keeps a usable slice of logs.
   If Rare.lab wants to auto-pick the best-performing variant per device, log each
   pick with its propensity and the resulting frame-time/crash outcome, then use
   replay-style matching to evaluate a new selection policy from existing
   telemetry. No need to ship a risky live A/B of "new auto-tuner" to real games
   if you can match logged decisions and read off the reward. A large variant
   space would make that math collapse, so bias the editor toward a curated few.

3. **Lock the decision per session, learn between sessions.** Just as the poster
   must not flicker mid-scroll, a shader variant must not thrash mid-frame.
   Decide once at scene/load time, cache it, and only revisit on the next load.
   Stability is a feature, and it is also what keeps the hot path free of
   recomputation.

The deep reuse: expensive thinking offline, a few cheap operations on the live
path, and a logging discipline that lets you improve the offline brain without
ever gambling on live users. That is the same spine as Spotify's offline-built
Discover Weekly served as a cheap lookup, and it is the spine Rare.lab's runtime
should be built on.

---

## Sources

- Netflix Technology Blog, "Artwork Personalization at Netflix" (Dec 2017):
  https://netflixtechblog.com/artwork-personalization-c589f074ad76
- Netflix Technology Blog, "AVA: The Art and Science of Image Discovery at
  Netflix": https://netflixtechblog.com/ava-the-art-and-science-of-image-discovery-at-netflix-a442f163af6
- Netflix Technology Blog, "ML Platform Meetup: Infra for Contextual Bandits and
  Reinforcement Learning":
  https://netflixtechblog.com/ml-platform-meetup-infra-for-contextual-bandits-and-reinforcement-learning-4a90305948ef
- Fernando Amat et al., "Artwork Personalization at Netflix," RecSys 2018 (talk
  and paper):
  https://www.slideshare.net/slideshow/artwork-personalization-at-netflix-fernando-amat-recsys2018-118208854/118208854
- Semantic Scholar entry, "Artwork personalization at Netflix" (Gil,
  Chandrashekar et al.):
  https://www.semanticscholar.org/paper/Artwork-personalization-at-netflix-Gil-Chandrashekar/9aa3dd841b2dc52a68e281ffd5ae508e2162d416
- Eppo, "How Netflix, Lyft, and Yahoo use Contextual Bandits for Personalization":
  https://www.geteppo.com/blog/netflix-lyft-yahoo-contextual-bandits
- Analytics India Magazine, "Understanding AVA, the Image Discovery Tool Used by
  Netflix": https://analyticsindiamag.com/ai-features/understanding-ava-image-discovery-tool-used-by-netflix-to-power-its-content-posters/
</content>
</invoke>
