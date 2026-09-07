# Gmail Smart Reply: the three one-tap answers under your email

Date: 2026-09-07
Product: Gmail
Feature: Smart Reply (the three suggested one-tap replies at the bottom of an email)

A note on how this sits next to an earlier teardown. On 2026-06-21 this ledger
covered Gmail Smart Compose, the grey ghost text that finishes your sentence as
you type. Smart Reply is the other half and it is built on the opposite idea.
Smart Compose generates new text from scratch with a language model. Smart Reply
does not generate anything at read time. It picks three whole answers out of a
fixed, pre-approved list. One is open-ended writing, the other is closed-set
retrieval. Holding them side by side is the whole point.

---

## 1. The user

Ravi is a sales lead in Pune. He gets about 120 emails a day and reads most of
them on his phone between meetings, in the lift, in an auto. His thumb is his
only input device. A colleague writes: "Can you join the 4pm sync today?" Ravi
knows the answer instantly. The answer is "Yes, see you there." But typing that
on a phone keyboard, with autocorrect fighting him, takes eight to ten seconds
he does not have, so the email sits unanswered until evening, and by evening he
has forgotten it.

## 2. The real problem

Most email replies are short and boring. "Sounds good." "Thanks!" "I will look
into it." "No, that does not work for me." The thinking took half a second. The
typing takes ten seconds and a lot of thumb friction. On mobile that ten seconds
is the whole cost of replying, and it is enough to make people not reply. The
pain is not deciding what to say. The pain is the mechanical act of saying it on
a tiny keyboard. That is a machine problem, not a human one.

## 3. The feature in one sentence

Smart Reply reads the email you just opened and offers three short, tappable,
ready-made responses that each say something different, so a full reply is one
thumb tap instead of ten seconds of typing.

## 4. Jobs to be done

- "When the answer is obvious, let me send it without typing."
- "Give me choices that actually differ, so at least one fits, including a way to
  say no."
- "Do not clutter my screen with suggestions when no short reply makes sense
  (for example a long legal contract or a newsletter)."
- "Never put words in my mouth that are wrong, rude, or embarrassing." A reply I
  tap is a reply I sent. It has to be safe every single time.

## 5. How it works for the user

Ravi opens the "Can you join the 4pm sync?" email. Under the message, three grey
chips appear:

`Yes, I will be there.`  `Sorry, I can't today.`  `What is it about?`

He taps the first. Gmail drops that text into the reply box. He can send as is or
edit first. One tap, done. For a different email, say a shipping confirmation
from Amazon or a promotional newsletter, no chips appear at all. The feature
stays silent when no short reply fits. That silence is a designed behavior, not a
bug, and it matters as much as the suggestions themselves.

## 6. The actual flow, step by step

1. Ravi taps the email to open it.
2. Gmail sends the message content to the server (the heavy model does not run on
   the phone in the classic Gmail version).
3. A first small model, the triggering model, asks one yes/no question: is this an
   email a short reply even makes sense for? For roughly 89 out of 100 emails the
   answer is no, and nothing more happens. No chips, no cost.
4. For the roughly 11 out of 100 that pass, a second model scores possible
   responses and picks the best few.
5. A diversity step throws away near-duplicate suggestions and makes sure the set
   is not all "yes" flavored, forcing in a "no" option when it makes sense.
6. Three chips come back to the phone and render under the email.
7. Ravi taps one. The text lands in the compose box. He sends or edits.

The user sees three chips. The server did four distinct jobs: decide whether to
answer, score answers, diversify them, return them. Steps 3 and 4 are two
different halves, the same match-then-rank split this ledger keeps finding.

## 7. Under the hood, like the engineer

This is the heart of it. Google published two papers that lay the system out
plainly, so most of this is fact, not guess. The 2016 KDD paper "Smart Reply:
Automated Response Suggestion for Email" (Kannan et al.) describes the first
production version. The 2017 paper "Efficient Natural Language Response
Suggestion for Smart Reply" (Henderson et al.) describes the faster rebuild. I
will label anything that is my inference.

### The shape of the problem: a closed set, not open generation

The single most important design choice: Smart Reply does not write text freely.
It chooses from a fixed, curated list of allowed responses, the "response set."
Every chip you ever see was pre-approved and stored ahead of time. This is the
opposite of Smart Compose, which generates fresh tokens with a language model.

Why lock the output to a list? Three reasons, all about safety and speed.

- Safety. A reply you tap is a reply you send under your own name. A free
  generator can produce something rude, wrong, or weird. A curated list cannot,
  because a human already vetted every entry. The paper is explicit that the
  response set is filtered to only responses that meet quality requirements.
- Speed. Scoring against a known list is far cheaper than generating and can be
  precomputed, which is the trick that made it survive at scale (see below).
- Diversity control. If you own the whole list of possible answers, you can group
  them by meaning and guarantee the three you show actually differ.

So the catalog here is a list of maybe a few thousand short human responses like
"Sounds good!", "Sure, what time?", "I can't make it, sorry." Think of it as the
product catalog in an Amazon search teardown, except the items are sentences.

### Piece 1: the triggering model (the cheap gate)

Before any expensive work, a small feedforward neural network decides: should we
even offer suggestions here? It reads features of the email, including a word
n-gram representation, through an embedding layer that covers a vocabulary of
about one million words and three fully connected hidden layers (2016 paper). It
outputs one probability: is a short reply likely useful?

It fires for only about 11% of emails. That is a load shedder disguised as a UX
choice. The expensive model runs on roughly one in nine emails, not all of them.
At Gmail scale that one decision cuts the heavy compute by nearly 90% before it
starts. Concrete example: the "4pm sync?" email scores high and passes. A Myntra
sale newsletter scores low and is dropped, so Ravi never sees three awkward chips
under an ad.

Data structure here: a plain vector of n-gram features fed to a small MLP. Cheap,
stateless, embarrassingly parallel. You can run it on every email without
thinking about cost.

### Piece 2: response selection, the 2016 way (seq2seq LSTM + a trie)

For the 11% that pass, the first production system used a sequence-to-sequence
LSTM. Given the email as input, the LSTM computes the probability of a response
sequence word by word, the same seq2seq framework used for translation.

The clever part is how they force the LSTM to only ever produce a whitelisted
response. A naive beam search would generate free text and might wander off into
something unapproved. Instead, they organized the entire permitted response set
into a trie (a prefix tree, the same structure the Google autocomplete teardown
on 2026-06-16 used for query suggestions). Then they ran a left-to-right beam
search that keeps only hypotheses that still exist as a path in the trie.

Walk it with a real example. The email is "Want to grab lunch on Friday?" The
beam search starts. At the first word it considers "Sure", "Sorry", "What",
"Yes", each of which begins at least one real response in the trie. It expands
"Sure" to "Sure, what" to "Sure, what time?" because that whole path exists in
the trie. It cannot expand "Sure, elephant" because no approved response starts
that way, so that branch dies immediately. The trie prunes the LSTM's imagination
down to only legal sentences.

Cost of this search is O(b l) for beam width b and maximum response length l,
with l typically 10 to 30 words (2016 paper). That is tiny and independent of how
big the catalog is, exactly like walking a trie costs letters-typed and not
catalog-size in the autocomplete teardown. Matching (find legal candidates) and
ranking (the LSTM probability score) happen together in this beam walk.

### Piece 3: building the response set (offline graph clustering)

Where does the approved list come from, and how are responses grouped by meaning
so the system knows "Yes!" and "Sure thing" are the same intent? This is offline
work, done ahead of time, and it is a graph problem.

They built a graph of responses and used semi-supervised label propagation. Start
with about 100 intent clusters, each seeded with only 3 to 5 hand-labeled example
responses (for instance a cluster "affirmative" seeded with "Yes", "Sure",
"Sounds good"). Then propagate those labels across the graph so unlabeled
responses inherit the intent of their neighbors. Run label propagation for 5
iterations, freeze the assignments, randomly sample 100 fresh responses as
candidate new clusters, and repeat until it converges (2016 paper). Responses are
canonicalized first using a dependency parser that extracts the semantic
structure, so "Can you send it tomorrow?" and "Could you send that tomorrow?"
collapse to one canonical form.

The output is a clean list of allowed responses, each tagged with an intent
cluster. That tag is what powers diversity next. Note the pattern this ledger
keeps hitting: the expensive thinking (clustering millions of real responses) is
done offline and frozen. The live path never clusters anything.

### Piece 4: diversity (the part most systems skip)

Here is a failure mode a naive top-3 would hit. The LSTM loves agreeable replies.
For "Want to grab lunch?" the top three scores might be "Sure!", "Sure, sounds
good", "Yes, sounds good." Three ways of saying yes. Useless, because if Ravi
wanted to decline he has no chip to tap.

Smart Reply fixes this with two rules (2016 paper):

- Omit redundancy. Walk the ranked list and skip any response whose intent
  cluster is already represented. So once "Sure!" is picked from the affirmative
  cluster, "Yes, sounds good" is dropped because it is the same intent. The next
  chip must come from a different cluster, say a question: "What time?"
- Enforce a negative. The LSTM is biased toward positive replies, so a "no" like
  "Sorry, I can't make it" naturally scores low and never surfaces. To fix this
  they run a second LSTM pass with the search restricted to only the negative
  responses in the set, pull the best "no", and include it. That is why you so
  often see exactly one decline option in the three.

Result for our lunch email: `Sure, what time?`  `Sounds good!`  `Sorry, I can't
Friday.` One question, one yes, one no. Three real choices, not three yeses. The
intent cluster tag from Piece 3 is the key that makes this cheap: diversity is
just "one chip per cluster" plus "force in the negative cluster."

### Piece 5: the 2017 rebuild (dot products, and precompute everything)

The seq2seq LSTM was accurate but expensive to run for hundreds of millions of
messages a day. In 2017 they rebuilt the scorer as a feedforward dual-encoder and
it is a beautiful piece of systems thinking.

The idea: score how well a response y fits a message x with a single dot product.

- Encode the message x into a vector h_x using a feedforward network over n-gram
  embeddings.
- Encode each response y into a vector h_y the same way.
- The fit score is S(x, y) = h_x dot h_y. Train so good message-response pairs get
  a high dot product.

Now the payoff. Because the response side never depends on the incoming email,
every response vector h_y can be computed once, offline, and stored. The 2017
paper says exactly this: precompute the "response side" for all potential
responses. At read time the server does only two things: encode the one incoming
email into h_x, then find the response vectors with the highest dot product.

That last step, "find the highest dot product against a big pool of fixed
vectors," is nearest-neighbor search. The paper notes that because all response
vectors are precomputed, you can hand them to an external approximate
nearest-neighbor library and search the pool fast. This is the identical move as
the Spotify Discover Weekly teardown (Annoy index over precomputed song vectors)
and the YouTube candidate-generation teardown (ANN over video vectors). Live cost
becomes one small encode plus one nearest-neighbor lookup, and it barely grows as
the response list grows.

The result, in Google's words: the same quality as the seq2seq approach at a
small fraction of the computational requirements and latency. Smart Reply usage
grew to roughly 12% of replies in Inbox on mobile.

### The scale story at three tiers

Tier 1, about 1,000 possible responses. You could brute-force it. Run the LSTM,
or compute the dot product of the email against all 1,000 response vectors live,
and sort. A single machine handles this. No trie needed, no ANN index. At this
size the honest engineering answer is "just score them all."

Tier 2, about 100,000 possible responses, and thousands of emails per second.
Now scoring every response with an LSTM per email is too slow. Two things save
you. The 2016 answer: the trie plus beam search, so cost is O(b l) and does not
depend on the 100,000. The 2017 answer: precompute all 100,000 response vectors
once, and the live path is one encode plus a dot-product scan (or an index) over
them. What breaks at the jump to this tier is per-email model runs; the fix is to
stop generating and start looking up.

Tier 3, hundreds of millions of messages per day, Gmail scale. Now even a cheap
per-email model run, multiplied across the whole planet's inbox, is a serious
bill. Three defenses stack:

- The triggering model sheds about 89% of the load before the expensive model
  runs. This is the single biggest lever.
- The response side is fully precomputed and frozen; the live path is one input
  encode plus an approximate nearest-neighbor lookup, so adding more responses
  costs almost nothing at read time.
- The response set itself is bounded (a curated list, not the open space of all
  English sentences), which is what makes precompute and ANN even possible.

What would break at the next tier, in my inference, is freshness and personal
tone: a single global response set cannot capture how a specific team writes.
Google's own later work (a per-user layer, on-device variants for newer Gmail and
Chat) points that way, but the exact current production internals past 2017 are
not public, so I am labeling that as inference.

## 8. The retention and habit mechanic

Which metric does it move? Primarily engagement and retention, by removing the
friction that makes people abandon a reply. The observed proof point Google
published: Smart Reply drives roughly 10% (2016) growing to about 12% (2017) of
all replies sent on mobile in Inbox. More than one in ten replies on the phone
became a single tap.

The loop is subtle. Every time Ravi taps a chip and it was exactly right, the
feature earns a little trust. Next time he glances at the chips first before
reaching for the keyboard. Over weeks, checking the chips becomes the default
first move when he opens an email on his phone. The habit is not "open Gmail," it
is "let Gmail answer for me," and once that reflex forms the keyboard becomes the
fallback rather than the default. That is why the silence matters: if chips
showed up on newsletters and were useless, the reflex would never form. The
triggering model protects the habit by only showing chips when they are likely to
be right, so the hit rate stays high and trust compounds.

Revenue is indirect. More replies sent means more time in Gmail means the product
is stickier, which matters for Workspace subscriptions, but Google frames the win
as replies-assisted, an engagement number.

## 9. The lesson for Rare.lab

Rare.lab suggests the next node in a shader graph. The temptation is to have an AI
freely generate a node or a snippet of shader code. Smart Reply says do the
opposite, and the reason is exactly the scalability and safety bias Rare.lab
needs.

Build a curated, precomputed, closed set of suggestions, not a free generator.

1. Keep a vetted library of known-good node subgraphs (a glow block, a Voronoi
   distortion, a chromatic aberration pass), each one already proven to compile
   and run on the target runtime. This is your response set. Like Smart Reply's
   list, every suggestion a user can tap is guaranteed safe, meaning it always
   compiles, because a human or a build step already vetted it. No suggestion ever
   produces a broken shader.
2. Encode the current graph context into a vector once, and precompute a vector
   for every library subgraph offline. Rank suggestions by dot product and pull
   the top few with an approximate nearest-neighbor lookup. The live editor path
   is one small encode plus one index lookup, constant in the size of your
   library, so the suggestion panel stays instant whether the library has 500
   blocks or 50,000. This is the 2017 dot-product rebuild applied to shader
   graphs.
3. Steal the diversity rule. Tag each block with an intent (color, distortion,
   lighting, masking). Show one suggestion per intent so the three chips are a
   glow, a warp, and a grade, not three glows. A node editor with three
   near-identical suggestions is as useless as three "yes" replies.
4. Steal the triggering gate. Do not pop a suggestion after every single node.
   Run a cheap classifier that decides when a suggestion is actually likely
   wanted (for example right after the user adds an output node but the graph
   looks unfinished). This sheds compute and, more importantly, keeps the
   suggestion hit rate high so users learn to trust and rely on the panel.

The one-line version: constrain the output to a curated set that is guaranteed to
compile, precompute the expensive half offline, and make the live path a cheap
ranked lookup with a diversity filter. That is how a suggestion feature stays
fast, safe, and habit-forming at scale.

---

## Sources

- Kannan, Kurach, Ravi, Kaufmann, Tomkins, Miklos, Corrado, Lukacs, Ganea, Young,
  Ramavajjala. "Smart Reply: Automated Response Suggestion for Email." KDD 2016.
  https://research.google/pubs/smart-reply-automated-response-suggestion-for-email/
  (PDF: https://research.google.com/pubs/archive/45189.pdf)
- Henderson, Al-Rfou, Strope, Sung, Lukacs, Guo, Kumar, Miklos, Kurzweil.
  "Efficient Natural Language Response Suggestion for Smart Reply." 2017.
  https://arxiv.org/abs/1705.00652 (PDF: https://arxiv.org/pdf/1705.00652)
- Google AI Blog. "Efficient Smart Reply, now for Gmail." May 2017.
  http://ai.googleblog.com/2017/05/efficient-smart-reply-now-for-gmail.html
- The Morning Paper (Adrian Colyer). "Smart Reply: Automated response suggestion
  for email." Nov 2016.
  https://blog.acolyer.org/2016/11/24/smart-reply-automated-response-suggestion-for-email/
- Shagun Sodhani. Reading notes on the Smart Reply paper.
  https://gist.github.com/shagunsodhani/da411f15b71ed6a664f9d5ac46409b42
