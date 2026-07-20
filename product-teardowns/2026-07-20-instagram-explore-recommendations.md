# Instagram Explore: the grid of things you never asked for

Date: 2026-07-20
Product: Instagram
Feature: The Explore page recommender (candidate generation plus multi-stage ranking)

A note on sourcing. Instagram has published two solid engineering posts on this
exact system: "Powered by AI: Instagram's Explore recommender system" (2019) and
"Scaling the Instagram Explore recommendations system" (Engineering at Meta,
August 2023). Meta also keeps a public "Instagram Explore AI system" transparency
page. The stage names, the candidate counts (500 to 150 to 50 to 25), ig2vec,
the Two Towers retrieval model, FAISS, and the headline scale numbers (65 billion
features extracted and 90 million model predictions every second) come from those.
Where I fill a gap with "this is how this class of problem is solved," I label it
inference.

---

## 1. The user

Meet Ananya. She is 29, lives in Pune, and makes pottery on weekends. On her phone
she follows maybe forty accounts: three home-baking pages, a bunch of pottery
studios, two friends from college, a couple of dog accounts, and one account that
only posts photos of old Ambassador cars.

It is 4pm on a Tuesday. She has a ten minute gap between meetings and a cup of
chai going cold next to her. She has already seen everything her forty accounts
posted today. So she taps the little magnifying glass at the bottom of Instagram.

She is not searching for anything. She did not type a word. She just wants
something new and good to look at for the length of a chai break.

## 2. The real problem

Here is the honest version, the way you would say it to a friend.

Ananya follows forty accounts. Instagram has billions of posts. The gap between
"the forty accounts she chose" and "the best thing on the entire app for her right
now" is enormous, and she is never going to close that gap by hand. She is not
going to search "pottery wheel throwing technique." She does not know the name of
the studio in Jaipur that would blow her mind. She cannot browse a catalog of
billions of posts.

So the product has a job that the user cannot do for herself: out of billions of
posts made by strangers she has never heard of, pick the roughly two dozen that
will make her forget her chai is going cold. And it has to pick them in the third
of a second between her tap and the grid appearing.

The pain, plainly: discovery without a query. No search box does this. The user
gives you almost nothing (a tap) and expects magic (a wall of things she will
love from people she does not follow).

## 3. The feature in one sentence

Explore is the grid you get when you tap the magnifying glass: a personalized wall
of posts and reels from accounts you do not follow, chosen for you in real time
out of billions of candidates by a funnel that first gathers a few thousand likely
matches and then ranks them down to the two dozen you actually see.

## 4. Jobs to be done

What is Ananya really hiring Explore to do?

- "Show me something new and good without me having to ask." Fill a boredom gap
  with zero effort from her.
- "Introduce me to accounts I would have loved but never found." Grow her world
  past the forty she already picked.
- "Read my taste, not just my follows." She follows two friends out of politeness;
  she does not want more of their brunch. She wants more pottery.
- "Do not waste my ten minutes." Every one of the two dozen tiles should earn the
  tap.

For Instagram the job is blunter: keep her on the app past the moment she ran out
of feed. Explore is the answer to "I've seen everything, now what."

## 5. How it works for the user

She taps the magnifying glass. Instantly a grid appears: a mosaic of square
photos and taller video tiles, some bigger than others. A pottery wheel reel. A
sourdough loaf being scored. A golden retriever failing to catch a ball. A studio
in Jaipur she has never heard of.

She did not type anything. She did not pick a category. The grid is just there,
and it is uncannily on point: mostly pottery and baking and dogs, the exact three
veins of her taste, plus one or two wildcards to see if she bites.

She taps the pottery reel. It opens full screen. She swipes up. Another reel.
Another. The grid quietly reloads more in the background so she never hits a wall.
If she taps the three dots and says "Not interested" on a fitness influencer, that
kind fades out over the next visits.

## 6. The actual flow, step by step

1. Ananya taps the magnifying glass (the Search and Explore tab).
2. Her phone sends a request to Instagram's servers: "give me Ananya's Explore
   grid." It carries her user id and a bit of context (device, session).
3. On the server, the candidate generation stage runs first. It looks at the
   accounts Ananya has interacted with (liked, saved, lingered on) and calls these
   her seed accounts. Using account embeddings it finds accounts topically similar
   to those seeds, then pulls fresh, eligible media from all of them. This produces
   a large pool of candidate posts.
4. From that pool it takes a sample of 500 candidate posts and hands them to the
   ranking stage.
5. Ranking runs in passes. A super-light distillation model scores all 500 and
   keeps the top 150. A heavier model with the full set of features scores those
   150 and keeps the top 50. A final ranker plus business and integrity rules
   trims to the roughly 25 posts that will fill the visible grid.
6. Those 25 come back to her phone already in order. Her phone does not rank
   anything. It just lays them out in the mosaic.
7. She scrolls. As she nears the bottom, the phone quietly asks for the next page,
   and the funnel runs again with fresh candidates.
8. Every tap, save, swipe-away, and "Not interested" she does is logged and feeds
   the next visit's seeds and the next model retraining.

The whole thing from tap to grid is a fraction of a second. Everything expensive
(the account embeddings, the ranking models, the precomputed features) was built
long before she tapped.

## 7. Under the hood, like the engineer

This is the heart of it. The core idea is the same one that shows up in almost
every teardown in this ledger: split the work into matching and ranking, and do
the expensive thinking offline so the live request is cheap. Instagram states this
plainly: they split Explore into a candidate generation stage (also called sourcing)
and a ranking stage.

### The scale of the haystack

Explore is one of the largest recommendation systems on Instagram, with hundreds
of millions of people using it. The pool of things it could show is billions of
posts. You cannot score billions of posts for one user in 300 milliseconds. If you
tried to loop over a billion posts and run even a cheap model on each, at, say, a
microsecond per post, that is 1,000 seconds. Ananya's chai would be stone cold and
her meeting would be over.

So the funnel exists to turn "billions" into "a few thousand" into "500" into "25"
as fast as possible, spending the most compute per item only on the last, smallest
sets. Meta's own framing: most large-scale recommender systems use a multi-stage
funnel, starting with thousands of candidates and narrowing to hundreds as they go
down.

### Half one: candidate generation (turning billions into a few thousand)

The clever move here is account embeddings, built with a framework Instagram calls
ig2vec. It is word2vec, but for Instagram accounts.

Quick reminder of word2vec: you feed it sentences, and it learns a vector for each
word such that words appearing in similar contexts (like "king" and "queen") end
up close together. ig2vec does the same trick but treats the sequence of accounts
a single user interacts with as a "sentence," and each account id as a "word." If
Ananya likes a post from claybymeera, then one from potterybarn_india, then one
from wheelthrown_jaipur, those three account ids sit next to each other in her
"sentence." Do this across hundreds of millions of users and pottery accounts end
up clustered together in vector space, baking accounts in another cluster, dog
accounts in another. Nobody labeled them "pottery." The co-interaction did.

Concretely: claybymeera becomes a vector like [0.11, -0.83, 0.42, ...]. So does
wheelthrown_jaipur. Because so many pottery-loving users interacted with both,
those two vectors sit very close. A baking account like poornas_bakes sits far
away in a different direction.

Now the retrieval. Ananya's seed accounts are the ones she has interacted with.
For each seed, Instagram wants the nearest accounts in embedding space. That is a
nearest-neighbor search over millions of account vectors. You cannot scan all
millions per request, so Instagram uses FAISS, its own approximate nearest neighbor
(ANN) library, to find close vectors without touching the whole set. ANN trades a
tiny bit of accuracy for a huge speedup: instead of comparing against every one of
millions of vectors, it uses an index (think of it as pre-clustered buckets of
vectors) so it only looks in the few buckets likely to contain neighbors.

So for the seed claybymeera, ANN returns wheelthrown_jaipur, glazestudio_goa, and
a dozen more topically similar pottery accounts. Repeat across all her seeds, pull
recent eligible media from all those accounts, and you have a big candidate pool.
Instagram then samples 500 candidate posts from it and passes them on.

The 2023 upgrade: Two Towers. The newer system generalizes ig2vec into a Two Tower
neural network. There are two separate networks. One (the user tower) eats
user-side features and outputs a user embedding. The other (the item tower) eats
post-and-author features and outputs an item embedding. They are trained so that
the dot-product similarity between a user embedding and an item embedding predicts
engagement (a like, a save). Now retrieval is symmetric and richer than word2vec:
you can feed in any user or item feature, and learn from multiple objectives at
once.

The scale trick inside Two Towers is precomputation. Item embeddings for the whole
catalog are computed ahead of time and loaded into the ANN (FAISS) service. Only
the user tower runs live: when Ananya taps, her user embedding is computed on the
fly from her freshest features, then used to query the ANN index for the most
similar items. So the live cost is one embedding computation plus one ANN lookup,
not a scan of billions. Offline think, online lookup, again.

Data structures worth naming here:
- A hash-map style embedding table: account id to vector. O(1) lookup of an
  account's vector.
- An ANN index (FAISS): the pre-clustered structure that makes "find nearest
  vectors" cost far less than scanning the full set.
- The candidate pool itself is just an array of post ids, later sampled down to 500.

### Half two: ranking (turning 500 into 25)

Matching found plausibly relevant posts. Ranking decides the order, and it is a
different problem: precise scoring of a small set. Instagram runs it as a funnel of
passes so it spends heavy compute only where the set is small.

Pass one, the distillation model. Score all 500 candidates with a super-lightweight
model and keep the top 150. The word "distillation" is doing real work: Instagram
trains this tiny model to imitate the output of the heavy ranking model. It learns
to approximate the expensive model's scores cheaply. So the first cut is almost as
smart as the final ranker but runs at a fraction of the cost. This is the key to
affording a big first set.

Pass two, the lightweight-but-full-featured model. Take the 150 survivors and run
a neural network that uses the full set of dense features. Keep the top 50. This is
where the real engagement modeling kicks in.

The engagement model is a multi-task multi-label (MTML) neural network. Instead of
predicting one thing, it predicts the probability of many actions at once from one
shared network: probability Ananya likes it, saves it, comments, shares, watches
the whole reel, and so on. One network, many heads. This is efficient (features are
computed once and shared) and it captures that these actions are related.

Pass three, the final ranker and the value model. Take the top 50 and combine each
post's predicted actions into a single number with a weighted formula. The public
transparency description says Explore predicts how likely you are to do things like
like, save, or comment, and combines those. In shorthand (inference on the exact
weights, which are private):

    Value(post) = w1 * P(like) + w2 * P(save) + w3 * P(share)
                + w4 * P(watch-through) + ... - w_neg * P(see-less / hide / report)

Saves and shares usually get a higher weight than a like, because they predict
deeper satisfaction, and negative signals ("See less," hide, report) subtract. Sort
the 50 by this value, apply integrity and business rules (do not show three posts
from the same author in a row, filter policy-violating or previously-seen content),
and the top ~25 become Ananya's grid.

Where the sort happens: on the server, not the phone. The phone receives an ordered
list of 25 post ids and just renders them. Same discipline as Amazon search, Swiggy
search, YouTube. The client never sorts a catalog.

Why these data structures and models:
- A neural net (not a tree of if-statements) because taste is a smooth, high-
  dimensional thing. Embeddings capture "pottery-ish" and "cozy-ish" as directions,
  which hand-written rules never could.
- MTML (multi-task) because the actions share signal. Learning "will she save it"
  and "will she like it" together, from shared features, is cheaper and more
  accurate than five separate models.
- Distillation because you want the recall of scoring 500 items with the accuracy
  of the heavy model, without paying the heavy model's cost 500 times.
- The value model is a plain weighted sum on purpose: it is the one knob product
  managers can tune (turn up the weight on saves, dial down on cheap likes) without
  retraining the deep net.

### The scale story at three tiers

At 1,000 posts (a tiny app, one city of pottery fans):
You do not need any of this. Fetch all 1,000 candidate posts, run one model over
each, sort in memory, return the top 25. A single server does it in milliseconds.
No embeddings, no funnel, no FAISS. Building ig2vec here would be over-engineering.

At 100,000 posts (a mid-size app):
Scoring 100,000 posts per request with a deep model starts to hurt latency and
cost. This is where you split into matching and ranking. You precompute embeddings,
add an ANN index so retrieval only touches likely-relevant posts, and you introduce
the two-pass ranker so the heavy model only sees a few hundred candidates. You add
read replicas and caching so many users' requests do not stampede one database.
What broke: the "score everything" loop got too slow. What survived it: the funnel.

At 10 million and beyond (billions of posts, hundreds of millions of users):
Now even the funnel has to be industrialized. Meta's 2023 numbers: the system
extracts 65 billion features and makes 90 million model predictions every second.
You cannot hit those numbers with naive serving. The survival moves:
- Precompute and cache aggressively at every stage. Item embeddings are computed
  offline and sit in the ANN service ready to query. Meta explicitly credits
  "clever use of caching and pre-computation in different ranking stages" as what
  lets them run heavier models at every stage without blowing the latency budget.
- Distillation to make the first pass cheap enough to cover 500 candidates.
- Two Towers so the only live embedding work is the user tower; everything item-
  side is precomputed.
- FAISS/ANN so nearest-neighbor over millions of accounts is sublinear, not a full
  scan.
- Sharding and replication (inference, standard for a system this size): shard the
  candidate stores and the feature stores, replicate for the enormous read volume,
  so no single machine is the bottleneck and a hot post does not melt one box.
What breaks at this tier if you do nothing: feature extraction and model inference
volume. 90 million predictions a second is not something you brute-force; it is
something you earn with precomputation, distillation, and caching so that the live
request is a handful of lookups plus a small amount of fresh scoring.

## 8. The retention and habit mechanic

Explore is a variable-reward slot machine wearing a friendly grid.

The loop: Ananya finishes her feed (a fixed, finite reward: she has seen the forty
accounts). Then she taps Explore (a variable, near-infinite reward: a fresh grid
every single time, tuned to her). The uncertainty is the hook. She does not know
what is in the grid, but she has learned it is usually good. That "usually good,
occasionally amazing" pattern is exactly the schedule that builds a reflex. The tap
becomes automatic, the way you check a fridge you know is empty.

The self-reinforcing part: every action she takes sharpens the next grid. She saves
a glaze tutorial, so glaze accounts move up her seeds. She swipes past a fitness
reel, so that vein cools. The grid feels like it is reading her mind because it
literally trained on her last visit. A brand-new competitor app cannot feel this
way on day one; it has no seeds for her. That gap is a moat made of her own history.

Which metric it moves: primarily retention, and secondarily time-spent (which
feeds ad revenue). Explore's job is to catch the user at the exact moment they
would have closed the app ("I've seen everything") and hand them a reason to stay.
Meta has said hundreds of millions of people use Explore to discover new content;
it is one of the main surfaces that converts "done with my feed" into "still
scrolling." The concrete observed example: the Explore grid auto-loads more content
as you scroll and opens into a swipe-up reel viewer, so a single tap turns into a
ten minute session with no natural stopping point. The design removes the edge of
the cliff.

## 9. The lesson for Rare.lab

Rare.lab has its own billions-versus-milliseconds problem, just in a different
costume. At runtime, your embeddable shader runtime has to pick, for this exact
frame on this exact device, the right visual-effect variant out of a large space of
possible shader configurations (quality levels, feature toggles, precision modes,
LOD tiers). You cannot afford to evaluate and compile that whole space on the hot
path, the same way Explore cannot score billions of posts in 300ms.

Steal the funnel and steal the distillation.

The funnel: split "pick the right shader variant" into candidate generation and
ranking. Offline, precompute a small candidate set of variants that are known-good
for a device class (built via an embedding of device capability: GPU family,
memory, feature support, thermal headroom), exactly like item embeddings sitting
ready in the ANN service. At load time, compute one "device embedding" live (cheap,
like the user tower) and retrieve the handful of pre-validated candidate variants
for that device. Then rank only that handful by predicted cost and quality. The
expensive per-shot shader compile only ever touches the tiny survivor set, never
the full space.

The distillation trick is the sharper, more specific lesson. Explore affords a big
first pass (500 candidates) by training a super-lightweight model to imitate the
heavy ranker's scores. Do the same for shader selection. Your "heavy ranker" is the
real thing: actually compiling a variant and measuring its frame time on hardware,
which is far too slow to do per-frame. So train a tiny, cheap predictor that
imitates that expensive measurement, a distilled cost model that takes a variant
plus a device embedding and predicts frame time and quality in microseconds. Use
the distilled predictor as your first-pass filter over all candidate variants, and
reserve real compilation and measurement for the top few it surfaces. You get the
recall of considering many variants with the cost of measuring only a couple.

And keep the value-model knob: combine your predictions (predicted fps, predicted
visual fidelity, predicted power draw) into one weighted score with tunable weights,
so a game shipping on battery-constrained phones can turn up the power-saving weight
without you retraining anything. Precompute the expensive part, keep the live path a
lookup plus a light score, and expose one weighted dial for the product to tune.
That is the whole Explore playbook, pointed at shaders.

---

## Sources

- Instagram Engineering, "Powered by AI: Instagram's Explore recommender system"
  (2019). ig2vec account embeddings, seed accounts, candidate generation, sampling
  500 candidates, the distillation first pass, MTML ranking.
  https://instagram-engineering.com/powered-by-ai-instagrams-explore-recommender-system-7ca901d2a882
- Engineering at Meta, "Scaling the Instagram Explore recommendations system"
  (August 9, 2023). Multi-stage funnel (retrieval, first-stage ranking, second-stage
  ranking, final reranking), Two Towers retrieval, caching and precomputation, the
  65 billion features and 90 million predictions per second figures.
  https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/
- Meta Transparency Center, "Instagram Explore AI system." How Explore predicts and
  combines engagement signals like likes, saves, and comments.
  https://transparency.meta.com/features/explaining-ranking/ig-explore/
- Engineering at Meta, "How Instagram suggests new content" (December 10, 2020).
  Account embeddings and suggested content.
  https://engineering.fb.com/2020/12/10/web/how-instagram-suggests-new-content/
- Engineering at Meta, "Journey to 1000 models: Scaling Instagram's recommendation
  system" (May 21, 2025). Model-serving scale for Instagram recommendations.
  https://engineering.fb.com/2025/05/21/production-engineering/journey-to-1000-models-scaling-instagrams-recommendation-system/
- VentureBeat, "Facebook details the AI technology behind Instagram Explore."
  Third-party recap of ig2vec and the two-stage system.
  https://venturebeat.com/ai/facebook-details-the-ai-technology-behind-instagram-explore/
- FAISS (Facebook AI Similarity Search), the approximate nearest neighbor library
  used for retrieval. https://github.com/facebookresearch/faiss
