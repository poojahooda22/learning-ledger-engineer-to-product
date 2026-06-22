# YouTube: the recommendation funnel (home feed and "watch next")

Date: 2026-06-22
Product: YouTube
Feature: Recommendations (the home feed and the "up next" sidebar that pick which videos to show you out of billions)

---

## 1. The user

It is 11:10 pm on a Wednesday. Arjun is in bed. He opened YouTube for one reason:
he wanted to see the last over of the India vs Australia match he missed. He
watched the highlight. Forty seconds. The plan was to put the phone down.

He did not put the phone down. Below the video, the "up next" list already had a
queued video glowing: "Bumrah's yorker masterclass, every wicket." He tapped it.
After that came a press conference clip, then a 2011 World Cup final flashback,
then somehow a video about how cricket bats are made in a factory in Meerut. It
is now 11:52 pm. Arjun never searched once after that first clip. The app kept
handing him the next thing, and the next thing kept being good enough to tap.

That handing-you-the-next-thing is the feature. It is the home screen full of
tiles when you open the app cold, and it is the "up next" rail and the autoplay
that fires when a video ends. Most of what Arjun watched tonight, he did not go
looking for. YouTube went looking for him.

## 2. The real problem

Here is the honest version, friend to friend. YouTube has more than a billion
videos. Five hundred hours of new video get uploaded every minute. Arjun has the
attention span of a tired man in bed at midnight. He will give the home screen a
few seconds. If nothing grabs him, he closes the app and opens Instagram.

So the problem is brutal and specific. Out of a corpus the size of a small
country's worth of footage, pick the roughly twenty tiles to put on Arjun's home
screen right now, and pick the one video to autoplay after his cricket clip ends,
such that he taps and keeps watching. Get it wrong and he is gone. Get it right
and he stays for forty more minutes.

You cannot solve this by scoring every video for every user. Score one billion
videos for two billion logged-in users every time someone opens the app and you
would need a data center the size of a city block just to answer one tap, and you
would still blow the latency budget by a factor of a thousand. The whole art is
how to avoid looking at almost everything, while still finding the few hundred
videos worth a careful look.

## 3. The feature in one sentence

For every screen and every video ending, YouTube narrows more than a billion
videos down to a few hundred you might like, then carefully orders that short
list to maximize how long you will keep watching, and it learns what works from
how billions of people actually behave.

## 4. Jobs to be done

- "I came for one cricket clip. Now find me the next good thing so I do not have
  to think." (Arjun)
- "Do not make me search. I do not know what I want, just show me." (the lean-back
  mood)
- "Keep it fresh. I was here this morning, give me a reason to come back tonight."
- For a creator: "Put my new upload in front of the people who will actually
  watch it to the end."
- For YouTube: "Turn one opened app into a long session, and one session into a
  daily habit, so the ad inventory and the watch time keep growing."

## 5. How it works for the user

Arjun never sees a setting called "recommendations." It is the product. When he
opens the app cold, the home screen is the recommendation system. When a video
ends and the next one starts on its own, that is the recommendation system. When
the sidebar shows "up next," that is the recommendation system.

It feels like a single magic list. It is not. Under the surface there are two
completely different machines doing two completely different jobs, and they run
one after the other. The first machine is fast and rough. It takes the billion
videos and throws away all but a few hundred. The second machine is slow and
careful. It takes those few hundred and puts them in the exact order Arjun is
most likely to keep watching. The user only ever sees the output of the second
machine. The whole design of the system is that split, and we will spend most of
this report on it.

## 6. The actual flow, step by step

1. Arjun finishes the cricket highlight. The video player fires an event: this
   session, this user, this video just ended, here is the context (time of day,
   device, the watch history so far tonight).
2. The candidate generation service builds a compact picture of Arjun right now.
   It is a list of numbers (an embedding) summarizing what he has watched, searched,
   and subscribed to, plus context like "11:52 pm, phone, in India."
3. That picture is used as a key to look up the few hundred videos closest to it
   out of the whole corpus. This is a nearest-neighbor lookup, not a scan. It
   returns maybe a few hundred candidates: more cricket, some other sports, a
   couple of things from channels he watches.
4. Those few hundred candidates go to the ranking service. Ranking pulls richer
   features for each one (how many times Arjun has seen this thumbnail already,
   how long since he last watched this channel, the video's freshness) and scores
   each candidate with a prediction: if we show Arjun this, how long will he watch.
5. Ranking sorts the few hundred by that predicted watch time. Server side. The
   phone never sorts anything.
6. The top result becomes the autoplay "up next." The next several fill the
   sidebar. On a cold home-screen open, the same pipeline fills the grid of tiles.
7. Arjun taps "Bumrah's yorker masterclass." That tap, and how long he watches it,
   becomes a training label. Tomorrow's model is a little better because of tonight.

The user felt one smooth thing. Underneath, two machines ran in sequence, and a
feedback loop quietly logged the result.

## 7. Under the hood, like the engineer

This is the heart of it. YouTube has published the real architecture in two
landmark papers: "Deep Neural Networks for YouTube Recommendations" (Covington,
Adams, Sargin, RecSys 2016) for the two-stage funnel, and "Recommending What
Video to Watch Next: A Multitask Ranking System" (Zhao et al, RecSys 2019) for
the modern ranking layer. A third paper (Chen et al, WSDM 2019) describes the
reinforcement-learning version of candidate generation. Where I go past what they
state, I label it inference.

### The one idea that makes everything possible: matching, then ranking

Recommendation is the same shape as search, and it splits into the same two
halves. Search has matching (find the documents that contain the words) and
ranking (order them by relevance). Recommendation has the exact same split, and
YouTube names it directly: **candidate generation** and **ranking**.

- Candidate generation is matching. Input: more than a billion videos. Output: a
  few hundred. Job: high recall, be fast, do not miss the good stuff. It is
  allowed to be rough.
- Ranking is precision. Input: a few hundred. Output: the same few hundred in the
  best order. Job: be exactly right about the top handful, because that is what
  Arjun actually sees.

Why two stages and not one good model? Because the two stages face opposite
constraints. Stage one must be cheap enough to run against a billion items, so it
must be simple per item. Stage two can afford a heavy model with hundreds of
features, because it only ever runs on a few hundred items. Trying to do both jobs
with one model means either you are too slow to touch the whole corpus, or too
dumb to order the final list well. The split lets each half specialize. This is
the single most important design decision in the whole system.

### Stage one: candidate generation, posed as an absurd classification problem

Here is the clever trick from the 2016 paper. They framed "what should Arjun watch
next" as an **extreme multiclass classification** problem. The classes are the
videos themselves. So the question becomes: out of roughly a million video classes,
which class is Arjun about to "pick" (watch) next, given everything he has watched
so far?

A neural network learns two things at once:

- A **user vector**: take Arjun's watch history (a sequence of video IDs), his
  search history, his coarse features (age of account, location, device), average
  the embeddings of his watched videos into a fixed-length vector, push it through
  a few fully connected layers. Out comes one vector, say 256 numbers, that is
  "Arjun, right now." Call it `u`.
- A **video vector** for every video, `v_j`, also 256 numbers.

The model is trained so that the probability Arjun watches video `j` next is
proportional to `exp(u · v_j)`, run through a softmax over all the video classes.
In plain terms: the closer Arjun's vector points in the same direction as a
video's vector, the more likely he watches it. The cricket-highlight vector and
the "Bumrah yorker" vector end up pointing the same way because the people who
watch one tend to watch the other. That is collaborative filtering, learned as
geometry.

You cannot compute a softmax over a million classes for every training example.
That would be a million dot products per step. So they train with **negative
sampling** (a sampled softmax): for each true watched video, sample a few thousand
random videos as negatives, and only push the math through those. This is the same
trick word2vec uses for words. It turns an impossible sum over a million classes
into a cheap sum over a few thousand.

### The payoff: serving becomes a geometry lookup, not a scan

Here is why posing it that way is genius. At serving time you do not need the
softmax at all. You computed every video's vector `v_j` offline and stored it. When
Arjun shows up, you compute his one vector `u` live, and the videos most likely to
be watched are simply the ones whose `v_j` is closest to `u` in dot-product space.

Finding the closest few hundred vectors to a query vector out of a million is a
solved problem: **approximate nearest neighbor (ANN) search**. You build the index
offline (tree-based, or hashing, or graph-based like HNSW). At query time it
returns the top few hundred in sub-linear time without comparing against all
million. The 2016 paper says it plainly: at serving, picking the N most likely
videos "reduces to a nearest neighbor search."

So the entire expensive part, learning what Arjun is like and what every video is
like, happens offline in big batch training. The live path is: build one user
vector, hit one ANN index, get a few hundred IDs. Cheap. Constant-ish in corpus
size because ANN does not scan. This is the same pattern as Spotify serving an
Annoy index and Netflix doing a few dot products: do the thinking offline, serve a
lookup.

### One real feature worth calling out: "example age"

A recommender trained on watch logs has a nasty bias: it learns to love old videos,
because old videos have had years to rack up views. But users want fresh stuff. The
2016 paper added a feature they call **example age**: how old the video was at the
moment of the training example. Feed that in, and at serving time set it to zero
(or near zero) for "right now." The model learns the natural freshness curve (a
video gets a burst of attention right after upload) instead of flattening it. This
is why a video uploaded two hours ago about today's match can still reach Arjun
tonight, instead of being buried under a 2011 classic with a hundred million views.

### Another real choice: predict the next watch, not a random held-out watch

When they pick the training label, they do not hold out a random video from Arjun's
history and ask the model to guess it. They hold out the **future**: take a point in
time, use everything before it as input, predict the very next watch. Why? Because
consumption is asymmetric and sequential. People watch episode 1 then episode 2,
or they binge a creator's catalog in order. A model that can peek at the future
(predict a middle video using videos that came after it) learns an unrealistic task
and does worse live. Predicting strictly forward in time matches how the feature is
actually used: what comes next.

They also cap the number of training examples **per user** at a fixed amount, so a
person who watches 500 videos a day does not drown out a person who watches 5. That
keeps the model from overfitting to power users.

### Stage two: ranking, and why the target is watch time, not clicks

Now we have a few hundred candidates for Arjun. The ranking network scores each one.
It looks similar to candidate generation (embeddings plus fully connected layers)
but it gets to be richer, because it only runs on a few hundred items, not a billion.
It pulls features the cheap stage could not afford: how many times Arjun has already
been shown this exact thumbnail (impression fatigue), time since he last watched this
channel, the language match, the video length.

The crucial decision is **what the model optimizes**. Early recommenders optimized
clicks. That is a trap. Clicks reward clickbait: a shocking thumbnail and a title in
all caps gets the tap, then the user bails in five seconds and feels cheated. YouTube
instead optimizes **expected watch time**. The 2016 ranking model does this with a
neat trick called **weighted logistic regression**: positive examples (videos that
were watched) are weighted by how long they were watched, negatives get weight one.
The odds the model learns then come out proportional to expected watch time. So a
thumbnail that gets a tap but a quick bounce scores low. A video that gets watched to
the end scores high. The system is built to reward "was this actually worth the
viewer's time," which is exactly the thing a clickbait optimizer destroys.

### The modern ranking layer: many goals at once, and fixing its own bias

The 2019 "Watch Next" paper upgraded ranking in two ways that are worth knowing.

First, **multitask learning with MMoE** (Multi-gate Mixture-of-Experts). One video
has many signals: did you click, how long did you watch (engagement), and did you
like it, share it, or dismiss it (satisfaction). These can conflict. A video can be
engaging but leave you unsatisfied. So they train one model to predict several of
these at once, using a shared bank of "expert" sub-networks with a separate gate per
task that decides how much of each expert to use. It is cheaper than training a
separate model per goal and it lets the goals share what they have in common.

Second, the **shallow tower for selection bias**. Here is the subtle problem. The
training data is polluted by the system's own past behavior. A video got clicked
partly because it was shown in the number-one slot, not because it was the best video.
If you train naively, the model learns "things in slot one get clicked" and chases its
own tail. Their fix: a small side network (the shallow tower) takes the bias features
(like the position the item was shown in) and predicts a bias term. The main model
predicts true user value. They add them in training, then **drop the shallow tower at
serving**. The result: the model learns the unbiased "would Arjun like this" signal,
separated from the "it was in slot one" effect. This is propensity correction built
into the network.

### The reinforcement-learning version: optimizing for tomorrow, not just the next tap

The 2019 WSDM paper (Chen et al) reframed candidate generation as reinforcement
learning. Instead of "predict the next watch," treat each recommendation as an action
in a policy whose reward is long-term satisfaction, with an **action space on the
order of millions** of videos. The hard part is that the training data was logged by
older policies, so they apply **off-policy correction** (importance weighting) and a
novel **top-K correction** because you recommend a slate of K videos, not one. YouTube
reported this was one of its most impactful launches for engagement. The point for us:
the objective crept from "next click" toward "long-term value of the session," which is
the right metric for a habit product.

### The scale story at three tiers

**1,000 videos.** Trivial. You do not need two stages, or embeddings, or ANN. You
score all 1,000 for the user with one model and sort. A single database query with an
`ORDER BY score DESC LIMIT 20` does it. Latency is nothing. This is the tier where a
new startup lives, and where building YouTube's architecture would be a waste.

**100,000 videos.** Now a per-request scan starts to bite. Scoring 100,000 candidates
with a rich ranking model, per user, per page load, for many users at once, blows the
latency budget. This is where the two-stage idea earns its keep. You introduce a cheap
recall stage that cuts 100,000 down to a few hundred (even simple co-visitation counts
or a small embedding plus ANN), and you only run the heavy ranker on those few hundred.
What broke at the previous tier: you could afford to score everything. What survives:
score a shortlist.

**10 million and beyond (YouTube's real tier: more than a billion videos, around two
billion logged-in monthly users, 500 hours uploaded per minute).** A scan is now
physically impossible per request. Everything depends on moving cost off the live path:

- Video vectors are computed offline in batch and stored. The live request never
  recomputes them.
- An ANN index over those vectors is built offline and replicated. The live request
  does one approximate nearest-neighbor lookup, sub-linear, returning a few hundred IDs.
  It never touches most of the corpus.
- The expensive model training (about a billion parameters, hundreds of billions of
  examples per the 2016 paper) runs offline on dedicated hardware. Serving is a forward
  pass on a short list plus a sort of a few hundred numbers.
- The freshness problem (500 hours a minute) is handled by the example-age feature and
  by refreshing embeddings and indexes frequently, so a video from this afternoon is
  reachable tonight.
- Read scale (billions of feed builds a day) is handled the usual way: heavy caching,
  read replicas, and sharding the candidate stores. The user vector is small and
  self-routing, so requests parallelize cleanly.

The through-line at every tier: the corpus grows, but the live path is engineered to
stay roughly constant in cost, because the only thing that touches the whole corpus
(building the index) happens offline.

## 8. The retention and habit mechanic

The loop is the product. It runs like this:

1. Arjun opens the app or finishes a video.
2. The system hands him a next video tuned to his taste right now.
3. It is good enough that he watches. Autoplay removes even the decision to continue:
   the next one just starts.
4. That watch is logged and feeds tomorrow's model, so the next hand-off is a touch
   better.
5. Go to step 1. The session stretches from one cricket clip to forty minutes.

Two design choices make this loop sticky rather than annoying. First, optimizing
**watch time instead of clicks** means the loop tends to feed Arjun things he actually
enjoys, not things that trick him, so the habit feels good instead of cheap. Second,
the **freshness** built in by example age gives him a different home screen tonight
than this morning, so there is always a reason to reopen. Neal Mohan, then YouTube's
chief product officer, said at CES in January 2018 that recommendations drive about
**70 percent of watch time** on the platform, and that the average mobile viewing
session runs **more than an hour**. The metric this moves is the big one: retention and
total watch time, which is the engine ad revenue rides on. It moves activation too
(the cold home screen has to land a tap), but its real home is retention: the daily,
almost reflexive reopen.

The dark side is real and worth naming: an engagement loop this strong can pull people
toward sensational or extreme content, because that holds attention. YouTube's later
moves (the satisfaction signals and the multitask "did you actually value this" targets
in the 2019 paper) are partly an attempt to steer the loop toward "time well spent"
rather than raw time. The lesson is that the objective you optimize becomes the product
you ship.

## 9. The lesson for Rare.lab

Steal the two-stage funnel and the offline/online split, and apply it to your asset and
node library.

Rare.lab is a node-based shader editor that compiles to shippable code, plus an
embeddable runtime. As the library of nodes, effects, materials, and community presets
grows from a few hundred to tens of thousands, the "search and recommend an effect" box
and the "you might also want this node" panel face exactly YouTube's problem: too many
items to score richly per keystroke, and a hard latency budget because it sits inside an
editor where any lag is felt.

Do not score the whole library on every interaction. Split it:

- **Candidate generation, offline and cheap.** Precompute an embedding for every node,
  effect, and preset (from its tags, its graph shape, its parameter signature, and which
  effects co-occur in real shipped projects). Build an ANN index over those embeddings
  offline. When a user is building a glow effect and reaches for "what next," compute one
  query vector from their current graph and do a single nearest-neighbor lookup to pull a
  few hundred candidates. Sub-linear, constant in library size, no scan.
- **Ranking, online and rich, on the shortlist only.** Score just those few hundred with
  the expensive signals: does this node's output type match the current socket, will it
  compile cleanly on the user's target (WebGL2 vs WebGPU), what is its measured GPU cost,
  how often does the user reuse it. Sort server side. Show the top handful.

And carry over the objective lesson directly. Do not optimize the suggestion panel for
clicks on a node. Optimize it for the equivalent of watch time: nodes that get **kept in
the final compiled shader and shipped**, not nodes that get dragged in and deleted ten
seconds later. A suggestion that gets clicked then removed is your clickbait. Weight your
training signal by "survived to ship," exactly as YouTube weights by watch time, and your
suggestions will pull users toward effects that actually work on their target hardware
instead of flashy nodes that tank the frame rate. The funnel keeps you fast at scale; the
right objective keeps the suggestions worth trusting.

---

## Sources

- Paul Covington, Jay Adams, Emre Sargin. "Deep Neural Networks for YouTube
  Recommendations." RecSys 2016. https://research.google/pubs/deep-neural-networks-for-youtube-recommendations/
- Mirror PDF of the 2016 paper: https://cseweb.ucsd.edu/classes/fa17/cse291-b/reading/p191-covington.pdf
- Zhe Zhao et al. "Recommending What Video to Watch Next: A Multitask Ranking System."
  RecSys 2019 (MMoE plus shallow tower for selection bias). https://daiwk.github.io/assets/youtube-multitask.pdf
- Minmin Chen, Alex Beutel, Paul Covington, et al. "Top-K Off-Policy Correction for a
  REINFORCE Recommender System." WSDM 2019. https://arxiv.org/abs/1812.02353
- The Morning Paper summary of the 2016 paper (Adrian Colyer). https://blog.acolyer.org/2016/09/19/deep-neural-networks-for-youtube-recommendations/
- Tubefilter, "YouTube Says 70% Of All Watch Time Is Driven By Its Own Recommendations"
  (Neal Mohan, CES January 2018). https://www.tubefilter.com/2018/01/11/youtube-most-watch-time-driven-by-recommendations/
- Quartz, "YouTube's recommendations drive 70% of what we watch." https://qz.com/1178125/youtubes-recommendations-drive-70-of-what-we-watch
