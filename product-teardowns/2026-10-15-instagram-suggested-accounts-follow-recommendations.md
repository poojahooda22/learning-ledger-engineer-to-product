# Instagram "Suggested for you": how it picks the next account to follow

Date: 2026-10-15
Product: Instagram
Feature: Account recommendations ("Suggested for you" / who to follow), the row of accounts Instagram tells you to follow

This teardown is about the little row of faces, not the feed of posts.

You follow someone, and right under them a strip slides open: "Suggested for
you", four or five accounts with a blue Follow button next to each. The same
strip lives on your profile, on the "Discover people" screen, and sometimes
stitched into your home feed. The question is simple to ask and hard to answer:
out of roughly two billion accounts, which four does it put in front of you, and
why those four.

A note on neighbours in this ledger, because Instagram shows up a lot here.
We have already torn down the Explore page recommender (2026-07-20), Reels
ranking (2026-08-07), the Stories tray (2026-06-27), home-feed fan-out
(2026-09-04), and photo storage (2026-07-11). This one is a different target on
purpose. Explore and Reels recommend CONTENT: a post or a clip to look at right
now. This feature recommends an ACCOUNT: a long-lived relationship, a follow edge
you add to the social graph. The payoff is different (a follow pays back for
months, a Reel pays back for nine seconds), the candidate pool is different
(accounts, not media), and the first move is a walk of the follow graph rather
than a pull from a content index. It reuses a lot of the same machinery, and
where it does I say so.

The spine of the whole thing: recommending an account is the same two-half
problem as every search and feed teardown in this ledger. First MATCHING (find a
few hundred plausible accounts out of billions), then RANKING (order those few
hundred). The twist is where the candidates come from. A search box starts from
words. A feed starts from who you follow. This starts from the shape of the
graph around you plus a learned sense of "accounts like the ones you already
like". The expensive thinking (embeddings, graph walks) is done offline and
cached. The live tap is a cheap lookup, a nearest-neighbour query, and a light
score. Offline-think, online-lookup, the same pattern as Discover Weekly
(2026-06-13) and autocomplete (2026-06-16).

---

## 1. The user

Meet Riya. She is 23, in Bengaluru, two weeks into actually using Instagram
after years of just lurking. Tonight she finally followed Virat Kohli
(@virat.kohli), because the T20 match was on and everyone in her hostel was
posting about it.

The instant she taps Follow, a row unfolds under his name: @imrohit,
@bcci.official (placeholder handle for the example), @anushkasharma,
@cricket.india. A blue Follow button beside each one. She does not go looking for
this. It arrives on its own, in the half second after her tap.

She is not an engineer and does not care how it works. She cares that two of
those four are people she actually wanted and did not know how to find.

## 2. The real problem

Here is the pain, said plainly, like a friend would say it.

An empty Instagram is boring. A feed with four accounts in it shows you the same
four posts over and over. The product only comes alive once you follow thirty or
forty accounts you genuinely care about. But nobody wants to sit and type names
into a search box one at a time. You do not even know the handles. Is it
@virat, @viratkohli, @virat.kohli, @imvkohli? You would get three wrong before
you got one right.

And the deeper problem: the accounts Riya would love most are ones she has never
heard of. A small cricket-stats account that posts the exact nerdy graphics she
likes. She cannot search for a thing she does not know exists. Search can only
find what you can name. This feature exists to hand you the accounts you could
never have typed.

For Instagram the pain is sharper still. A new user who follows fewer than a
handful of accounts in their first week mostly never comes back. The feed has
nothing in it, so there is no reason to open the app tomorrow. Getting Riya from
four follows to thirty in her first few sessions is close to life or death for
whether she becomes a real user. This row is the machine that does that.

## 3. The feature in one sentence

Out of about two billion accounts, Instagram picks a handful you are most likely
to follow and keep engaging with, and shows them to you at the exact moments you
are most open to following someone new.

## 4. Jobs to be done

What is Riya really hiring this row to do.

- "Fill my empty feed fast, without me having to know any handles." She wants
  thirty good follows, not a search chore.
- "Introduce me to accounts I would never have found on my own." The nerdy
  cricket-stats account she did not know existed.
- "Read the room. I just followed a cricketer, so show me cricket, not cooking."
  The suggestion has to match the moment.
- "Do not waste my taps. Every suggestion I ignore is a tiny insult." Four slots,
  all four should be plausible.

And Instagram's own job, riding underneath: turn a thin new account into a dense
one quickly, because follow count in week one predicts whether Riya is still here
in month three.

## 5. How it works for the user

The visible experience is almost nothing, which is the point.

Riya taps Follow on @virat.kohli. A row titled "Suggested for you" expands in
place, right under the account she just followed. Four accounts, each with a
circular profile photo, the handle, a one-line reason sometimes ("Followed by
accounts you follow" or "Popular"), and a Follow button. She can tap Follow on
any of them without leaving the screen, swipe the row sideways to see more, or
tap the little x to dismiss one she does not want.

The same row shows up in other places. On her own profile there is a
"Discover people" chevron that opens a full screen of suggestions. Every so often
a "Suggested for you" card slides into her home feed between posts. When she
first made the account, the onboarding flow showed her a grid of accounts to
follow before she had done anything at all.

Each place is the same engine with a different amount of screen. The post-follow
row is the highest-intent moment, so it is the one tuned hardest.

## 6. The actual flow, step by step

Tap by tap, from Riya's thumb to the four faces.

1. Riya taps Follow on @virat.kohli. The app writes the new follow edge
   (Riya to Virat) and, in the same breath, fires a request for suggestions with
   a bit of context: who she just followed, where she is in the app (post-follow
   row), her account id.
2. The server does NOT go searching two billion accounts live. It assembles a
   candidate set of a few hundred accounts from sources that were mostly
   precomputed (next section).
3. Those few hundred get scored and ordered by a ranking model.
4. Accounts she already follows, accounts she has dismissed before, blocked
   accounts, and accounts that fail integrity checks are filtered out.
5. The top few survivors come back. The app draws them into the row.
6. All of this finishes in the time it takes the Follow button to animate from
   "Follow" to "Following", a few hundred milliseconds. If it were slow, the row
   would pop in late and feel broken, so speed is a feature, not a nicety.

Riya taps Follow on @imrohit and @cricket.india, ignores the other two. Those
taps and non-taps are logged. They become training data for the next version of
the model, and they immediately reshape what she sees next. Follow @cricket.india
and the very next row leans harder into cricket media accounts.

## 7. Under the hood, like the engineer

This is the heart of it. The feature is a MATCHING half and a RANKING half, the
same split as every search teardown in this ledger. Matching finds a few hundred
candidates out of billions. Ranking orders those few hundred. They are different
problems solved with different tools, and conflating them is the classic mistake.

A sourcing note before we go deep. The single most detailed public engineering on
this machine is written up for Instagram's Explore system, not for the account
row specifically: Meta's 2019 post "Powered by AI: Instagram's Explore
recommender system" and the 2023 engineering.fb.com post "Scaling the Instagram
Explore recommendations system". The account-specific signals (the follow-graph
walk, region, contacts) are confirmed separately on Meta's own transparency page
for Suggested Accounts. Explore recommends content and this row recommends
accounts, but they share the same backbone: candidate generation then ranking,
ig2vec account embeddings, FAISS nearest-neighbour retrieval, the IGQL query
layer, Two Tower retrieval, and a cascade of rankers. Where I bridge from the
Explore write-ups to the account row, I label it INFERENCE. Where Meta states a
thing about accounts directly, I label it CONFIRMED.

### The matching half: four ways to find candidates

Instagram does not have one source of candidates. It has several, each a
different idea of "accounts Riya might like", and it unions them. CONFIRMED by
Meta's transparency page, the signals include "followers of accounts you already
follow", "popular accounts in your region", and accounts tied to "topics you have
recently engaged with", plus synced phone contacts.

**Source 1: the two-hop walk of the follow graph (friends of friends).**

Model the whole thing as a directed graph. Each account is a node. Each follow is
an edge from follower to followed. Riya is a node. She follows @virat.kohli. Virat
is followed by, and follows, other accounts.

The candidate idea: walk two hops out from Riya. Hop one is @virat.kohli. Hop two
is everyone connected to Virat. @imrohit shows up because he and Virat share a
huge overlap of followers, which is the classic "triangle closing" signal:
if many of the people who follow A also follow B, then a new follower of A
probably wants B. LinkedIn's engineering team describes this exact
friends-of-friends triangle-closing idea as one of the first signals any
People-You-May-Know system reaches for, and it is the same shape here.

The data structure is an adjacency list. For each account you store the list of
account ids it follows, and separately the list of ids that follow it. A follow
is an O(1) append. A two-hop walk is "read Riya's follow list, then for each
account in it read its follower list, and count how often each candidate appears".
Count the overlaps and @imrohit and @bcci.official float to the top because they
co-occur with Virat for millions of people.

Concrete number, and this is where it gets scary. Suppose Riya follows 500
accounts, and each of those follows 500 accounts. Two hops is 500 times 500 equals
250,000 raw candidates, many of them duplicates. That is just from one user with a
modest follow count. For a user who follows 2,000 accounts, each with a million
followers, the naive two-hop neighbourhood is billions of (account, path) pairs.
You cannot compute that live on every tap. What survives the next tier is about
how you tame this number, and we get there in the scale story.

**Source 2: ig2vec account embeddings plus nearest-neighbour search.**

The graph walk only finds accounts structurally near you. It misses the nerdy
cricket-stats account that no one Riya follows happens to follow yet. For that you
need a notion of "similar in taste", not "near in the graph".

CONFIRMED (Meta 2019): Instagram built ig2vec. It is word2vec, the famous text
embedding trick, pointed at accounts instead of words. The analogy is exact.
In word2vec you take sentences and learn that words appearing in similar contexts
get similar vectors ("king" and "queen" land near each other). In ig2vec you take
the SEQUENCE of accounts a user interacts with in a session, treat that sequence
as a "sentence" and each account id as a "word", and learn that accounts people
engage with in the same sessions get similar vectors. VentureBeat, reporting on
Meta's work, put it as ig2vec treating "the accounts a user interacts with as word
sequences in a sentence".

So @virat.kohli ends up with a 128-or-so-dimensional vector (the exact dimension
is not public; INFERENCE that it is in the low hundreds, which is the usual
range). @imrohit lands near him in that space because cricket fans engage with
both in the same sittings. The nerdy stats account also lands near him, even
though almost no one follows both yet, because the handful of people who do engage
with it engage with Virat in the same session. The embedding captures taste that
the raw graph has not connected yet. That is the whole reason to have embeddings
on top of the graph.

Now the retrieval. You have Virat's vector and you want the accounts whose vectors
are closest. Closeness is cosine similarity or dot product. Doing this exactly
against two billion vectors on every tap is impossible: exact nearest-neighbour is
a linear scan, two billion dot products per request. So Instagram uses FAISS,
Meta's own approximate-nearest-neighbour library. CONFIRMED (Meta): embeddings are
"loaded offline, allowing for fast similarity search at inference time using
FAISS". FAISS builds an index (think of it as bucketing similar vectors together
so a query only compares against a few promising buckets, not all two billion)
and trades a tiny bit of accuracy for being thousands of times faster. This is the
same move Spotify made with Annoy for Discover Weekly (2026-06-13): precompute
vectors offline, serve the live query as an approximate-nearest-neighbour lookup.
Different library, identical idea.

INFERENCE (labeled): for the account row, Instagram seeds the ANN query with
embeddings of the accounts Riya has engaged with (Virat, plus her other recent
follows and likes), fetches the nearest account vectors, and those become
candidates. Meta's 2023 Explore post confirms the retrieval stage uses a
"Two Towers neural network" with "caching on both the user and item side": one
tower turns the user into a vector, the other turns each account into a vector,
both towers are run offline and cached, and the live step is just the ANN lookup
between them. That is why it is fast. Nothing heavy runs at tap time.

**Source 3: contacts and cross-app graph.** CONFIRMED (transparency page): synced
phone contacts feed suggestions. If Riya let Instagram read her contacts, the
people in her phone who are on Instagram become high-confidence candidates. This
is pure identity matching (hash the phone numbers, look up which hashes have
accounts), not learning, and it is the strongest single signal for "people you
actually know" as opposed to "accounts you might like".

**Source 4: popularity and region priors.** CONFIRMED (transparency page):
"popular accounts in your region". For a near-empty new account with almost no
signal, you fall back to "what do people in Bengaluru follow a lot right now".
This is the cold-start answer and it is deliberately dumb, because dumb and
non-empty beats clever and empty.

**Stitching the sources together: IGQL.** CONFIRMED (Meta 2019): engineers do not
write bespoke pipelines per surface. They write queries in IGQL, a declarative
query language for recommendations, where you can declare several candidate
generators and a "combiner" that blends their outputs with weights. So the
post-follow row might be "70 percent graph-walk candidates seeded on the account
just followed, 20 percent ig2vec neighbours, 10 percent region-popular, unioned
and deduped". Changing the mix is a config change, not a rewrite. That is a real
engineering lever: the same engine serves onboarding, the profile screen, and the
in-feed card by just reweighting sources.

At the end of the matching half you have maybe a few hundred to a couple of
thousand candidate accounts, pulled mostly from precomputed structures, deduped,
with the ones Riya already follows removed. Meta's transparency page for Explore
quotes "up to 1500" items fetched in retrieval for that surface; INFERENCE that
the account row is the same order of magnitude, a few hundred to ~1,500. Now rank
them.

### The ranking half: a cascade of cheap-then-expensive models

You have, say, 500 candidate accounts and you need the best 4. You could run one
giant neural network on all 500. You do not, because the giant network is
expensive and running it 500 times per tap, times hundreds of millions of taps a
day, would cost a fortune and blow the latency budget. So you build a funnel:
cheap model thins the herd, expensive model judges the survivors. CONFIRMED (Meta
2023): Explore uses exactly this, a "multi-stage" funnel of "retrieval,
first-stage ranking, second-stage ranking and final reranking", narrowing
"thousands of candidates" to "hundreds as we go down the funnel".

**First-stage ranker: a distillation model.** CONFIRMED (Meta): the first pass is
"a super-lightweight model" whose "training objective is to optimize for NDCG
ranking loss over the main ranking model's output". Read that twice. They trained
a cheap model to IMITATE the expensive model's ordering. It is not trying to
predict reality; it is trying to predict what the expensive model would say, which
is a much easier target and runs in a fraction of the time. The 2019 arXiv paper
on Instagram's personalization gives a concrete cascade: take ~500 eligible
candidates, a first pass picks the 150 most relevant, a neural network narrows to
50. Numbers are from 2019 and will have drifted, but the shape is the point:
500 to 150 cheaply, then spend real compute only on the 150.

**Second-stage ranker: the heavy multi-task model.** CONFIRMED (Meta): the heavy
stage is a "multi-task multi-label (MTML)" neural network. For the account row the
tasks are the things Riya might do: probability she follows, probability she stays
followed in 30 days (does not unfollow), probability she engages with that
account's posts after following, probability she taps "see less" or dismisses.
INFERENCE on the exact label set for accounts; CONFIRMED that MTML predicts a set
of engagement probabilities for Explore. One model, many predicted numbers per
candidate.

**The value model: turning many numbers into one.** CONFIRMED (Meta describes a
value model for Explore): the predicted probabilities get combined into a single
score so you can sort. Conceptually it is a weighted sum:

  score = w1 times P(follow) + w2 times P(still-followed-in-30-days)
          + w3 times P(engages-after) - w4 times P(dismiss)

The exact weights are not public (INFERENCE on the formula shape; the idea of a
value model combining predicted engagement probabilities is CONFIRMED). The
important design choice is the 30-day term. If you only optimised P(follow), you
would flood Riya with clickbait accounts that are easy to follow and quickly
unfollowed. By putting weight on "still followed in 30 days" you steer toward
follows that stick. This is the same lesson as YouTube moving from clicks to watch
time (2026-06-22): optimise the long-term good, not the cheap tap.

Sort the 50 survivors by value score, run the final filters (already-following,
previously-dismissed, blocked, integrity and safety checks), take the top 4 for
the row. Done.

Walk Riya's real query end to end. She follows @virat.kohli.
Matching: graph walk from Virat surfaces @imrohit, @bcci.official, @cricket.india
(huge follower overlap with Virat); ig2vec neighbours of Virat's vector add
@anushkasharma (co-engaged) and the nerdy @cricketing.numbers stats account
(close in taste space, far in graph); region-popular adds a couple of generic
Bengaluru accounts. Union and dedupe to ~400 candidates.
Ranking: distillation model drops the generic region accounts and most of the
noise down to ~150, MTML scores the rest, the value model rewards @imrohit and
@cricket.india (high follow and high stick probability) and @cricketing.numbers
(lower follow probability but very high engage-after, the nerdy-match bonus).
Top 4 come back: @imrohit, @bcci.official, @anushkasharma, @cricket.india, with
@cricketing.numbers just behind in the swipe. Riya follows two. The system logs
all of it.

### The scale story: 1,000, 100,000, 10 million plus

The thing that grows here is not a searchable catalogue of text. It is the SIZE OF
THE GRAPH and the number of accounts you must embed and the number of two-hop
candidates a single tap can spawn. Watch what breaks at each tier.

**1,000 accounts (a tiny new app).** Just do it live. Walk the two-hop
neighbourhood with a double loop over adjacency lists, count overlaps in a hash
map, sort, return top few. A thousand accounts is nothing; the whole graph fits in
memory on one box. No embeddings, no FAISS, no cascade. Building ig2vec here would
be over-engineering a problem you do not have. Ship the double loop.

**100,000 accounts.** The two-hop walk starts to hurt. A popular account with
50,000 followers means walking from it touches 50,000 nodes, and a user following
a few hundred such accounts spawns millions of candidate pairs per tap. Two fixes.
First, cap and sample: do not walk all 50,000 followers of Virat, sample a few
thousand, which is enough to find the strong overlaps. Second, precompute. Stop
computing suggestions on the tap. Run a nightly batch job that computes a
candidate set per user and caches it, so the tap is a cache read. This is the
first appearance of the offline-think, online-lookup split, and it is forced on
you by the two-hop blow-up. Now you also want ig2vec, because at 100k accounts
"similar in taste" starts beating "near in graph" for quality, and you can train
embeddings offline nightly.

**10 million plus accounts, up to Instagram's ~2 billion.** Everything must be
precomputed and approximate, and the live path must touch almost none of the
graph. Four walls and what they did about each.

- Wall 1: exact nearest-neighbour is dead. Two billion dot products per request is
  impossible. Fix: FAISS approximate nearest-neighbour, which only compares
  against a few promising buckets. CONFIRMED. Accuracy drops a hair, speed goes up
  thousands of times.
- Wall 2: recomputing embeddings and candidates live is dead. Fix: Two Tower with
  "caching on both the user and item side" (CONFIRMED 2023). Both towers run
  offline. The item (account) vectors are a cached index. The user vector is
  cached and refreshed. The live tap is one ANN query against cached vectors, no
  model forward pass over the catalogue.
- Wall 3: the graph is too big for one machine. A single adjacency list for
  "followers of a mega-account" can be hundreds of millions of ids. Fix: shard the
  graph by account id across many machines, so reading Virat's follower list and
  Riya's follow list are keyed reads to specific shards, not a scan. This is the
  same shard-by-key move as Notion-by-workspace (2026-06-25) and Gmail-by-mailbox
  (2026-10-03): make the hot read a keyed lookup on one shard.
- Wall 4: ranking 500 candidates with the heavy model, hundreds of millions of
  times a day, is too expensive. Fix: the distillation cascade. Cheap imitation
  model on the 500, heavy MTML only on the ~150 survivors. CONFIRMED.

The through-line at every tier: push the expensive work (graph walks, embedding
training, candidate caching) OFFLINE into nightly and streaming batch jobs, and
reduce the live tap to cheap cached lookups plus one ANN query plus a light score.
The tap must finish inside the Follow-button animation, a few hundred
milliseconds, no matter whether the graph has a thousand nodes or two billion.

## 8. The retention and habit mechanic

The loop is tight and it is aimed at the most fragile moment in a user's life: the
first week.

The cue is a fresh follow. Riya follows @virat.kohli, which tells the system "she
is in cricket mode and she is in a following mood right now". The response is to
strike immediately, in the same half second, with four more cricket accounts.
The reward is instant: two of the four are accounts she wanted. The next time she
follows someone, the row is even better tuned because it just learned from those
two taps. Follow, get rewarded, follow again. Each loop makes the feed less empty,
and a less-empty feed is the only reason to open the app tomorrow.

Which metric does it move? Primarily ACTIVATION and early RETENTION, not revenue
directly. The industry-wide framing (Facebook's own growth lore, widely reported)
is that getting a new user to a threshold number of connections in their first
days is the single strongest predictor of whether they stick. This row is the
dedicated machine for racing a new account past that threshold. It is why the
highest-tuned placement is the post-follow moment, the exact instant a user has
shown both intent and a topic.

A real observed example of the loop working as designed: the onboarding grid. When
Riya first created the account, before she had done anything, she was shown a
screen of accounts to follow, pre-filled from contacts and region-popular accounts
(CONFIRMED signals). That screen exists because an account with zero follows has
nothing to come back to. The grid manufactures the first ten follows so the feed
is non-empty on day one, and the post-follow row keeps the ball rolling from
there. The suggestion engine is not a nice-to-have bolted on the side; it is the
on-ramp for the entire product.

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor for shaders and visual effects that compiles to
shippable code, plus an embeddable runtime. The feature that maps onto this
teardown is "suggest the next node", the recommendation Rare.lab should offer when
a creator is wiring up a graph: they just dropped a Noise node, so what node do
they most likely want next, a Color Ramp, a Displace, a Fresnel.

The lesson, biased to scalability and performance as asked:

Build node suggestions as the SAME two-half, offline-think-online-lookup machine,
and never let the live editor do the expensive work.

Concretely, three moves lifted straight from this teardown.

1. Learn a node2vec embedding offline, the exact ig2vec trick. Every shader graph
   your users build is a "sentence" and every node is a "word". Nodes that appear
   near each other across thousands of real graphs (Noise then Color Ramp then
   Displace) get similar vectors. Train this nightly on your corpus of graphs, not
   on the user's machine. Now "what pairs well with a Noise node" is a cached
   nearest-neighbour question, not a rule you hand-write and have to maintain.

2. Split matching from ranking, and make matching a graph walk plus an ANN query,
   both served from precomputed structures. Matching: from the node just placed,
   pull the cached co-occurrence neighbours (the graph-walk half) and the embedding
   neighbours (the taste half), union them to a few dozen candidates. Ranking: a
   small model orders those few dozen using the creator's own recent actions. The
   editor must stay at 60 frames per second while someone drags a wire, so the live
   suggestion must be a lookup and a light sort, never a model trained or a corpus
   scanned at interaction time. Precompute the embeddings and the co-occurrence
   table offline; the editor only ever reads them.

3. Rank for the node that STAYS in the graph, not the node that gets clicked.
   Borrow the value model's 30-day term. A suggestion that gets accepted and then
   deleted two nodes later is worse than no suggestion. Weight your ranker toward
   "accepted and still present when the graph compiles", so you steer creators
   toward nodes that actually belong, not flashy nodes that get added and ripped
   out. Optimise the graph that ships, the way Instagram optimises the follow that
   sticks.

One line: suggesting the next node is the same machine as suggesting the next
account, so precompute a node2vec embedding and a co-occurrence table offline,
serve suggestions as a cheap nearest-neighbour lookup plus a light rank at
interaction time, and rank for the node that survives to compile, not the node
that merely gets clicked.

---

## Sources

- Meta AI, "Powered by AI: Instagram's Explore recommender system" (2019). Primary
  source for the two-stage candidate-generation-then-ranking design, IGQL, ig2vec
  account embeddings, FAISS similarity search, and the distillation first-pass
  ranker. https://ai.meta.com/blog/powered-by-ai-instagrams-explore-recommender-system/
- Meta Engineering, "Scaling the Instagram Explore recommendations system"
  (engineering.fb.com, 2023). Primary source for the multi-stage funnel
  (retrieval, first-stage, second-stage, final reranking), the Two Towers retrieval
  network with user-side and item-side caching, the MTML ranker, and the value
  model. https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/
- Meta Transparency Center, "Instagram Suggested Accounts AI system". Primary
  source for the account-specific signals: followers of accounts you follow,
  popular accounts in your region, topics you engaged with, and contacts.
  https://transparency.meta.com/features/explaining-ranking/ig-suggested-accounts/
- Meta Transparency Center, "Instagram Explore AI system" (the "up to 1500" / top
  100 retrieval funnel numbers).
  https://transparency.meta.com/features/explaining-ranking/ig-explore/
- "How Personal is Machine Learning Personalization?", arXiv:1912.07938 (2019).
  Source for the concrete ranking cascade numbers (roughly 500 candidates to 150
  to 50). https://arxiv.org/pdf/1912.07938
- VentureBeat, "Facebook details the AI technology behind Instagram Explore".
  Secondary coverage confirming ig2vec as word2vec over account sequences and the
  account-embedding retrieval pipeline.
  https://venturebeat.com/ai/facebook-details-the-ai-technology-behind-instagram-explore/
- LinkedIn Engineering, "People You May Know". Source for the friends-of-friends
  triangle-closing candidate signal as the canonical first move in this class of
  problem. https://engineering.linkedin.com/teams/data/artificial-intelligence/people-you-may-know

Fact-versus-inference note: the signals for account suggestions (follow-graph
walk, region, contacts, topic engagement), the two-stage matching-then-ranking
design, ig2vec, FAISS retrieval, IGQL, Two Tower retrieval with caching, the
distillation first-pass ranker, the MTML second-stage ranker, and the value model
are all CONFIRMED by the Meta sources above, though the deepest writeups are for
the Explore (content) surface. Applying that exact backbone to the account row,
the specific candidate-source weighting, the exact embedding dimension, the exact
value-model weights, and the exact per-stage candidate counts for the account row
are INFERENCE, labeled as such in the text.
