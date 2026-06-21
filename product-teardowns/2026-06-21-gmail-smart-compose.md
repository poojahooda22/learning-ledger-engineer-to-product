# Gmail Smart Compose: the gray ghost text that finishes your sentence

Date: 2026-06-21
Product: Gmail
Feature: Smart Compose (inline sentence completion while you type)

---

## 1. The user

Meet Anjali. She runs partnerships at a mid-size SaaS company. By 11 a.m. she
has already written nine emails that all sound the same. "Thanks for reaching
out." "Let me loop in my colleague." "Happy to jump on a call next week." "Please
find the deck attached." She is not writing literature. She is writing the same
forty phrases in a slightly different order, all day, every day.

She is on the Gmail web compose window. Her cursor is blinking. She types
"Thanks for your" and pauses for half a second to think of the rest. In that
half second a soft gray phrase appears after her cursor: "email. Let me know if
you have any questions." She presses Tab. The gray text turns black and becomes
hers. She just wrote ten words by typing four and tapping one key.

That gray ghost text is Smart Compose.

## 2. The real problem

Typing the same boilerplate is slow, boring, and a tax on your hands. Anjali
does not need help being creative. She needs help not retyping "Let me know if
you have any questions" for the four-thousandth time.

The deeper pain is interruption. Every word she types is a tiny context switch
between thinking and spelling. If a tool could handle the predictable middle of
the sentence, she could stay in the thought and let the machine handle the
keystrokes. On mobile this is even sharper. Typing on glass is slow, and finishing
a sentence with one tap instead of twenty is the difference between replying now
and replying later.

The catch is brutal and specific. The help has to appear faster than she can
notice. If the suggestion shows up even a fifth of a second too late, it lands
after she has already typed the next letter, the prediction is now wrong, and it
flickers and annoys her. A late suggestion is worse than no suggestion. This is
the whole engineering story in one sentence: predict the rest of a human's
sentence, correctly, in less time than it takes them to press the next key.

## 3. The feature in one sentence

Smart Compose watches what you have typed so far, plus the context around the
email, and predicts the rest of your current sentence as gray inline text you can
accept with the Tab key.

## 4. Jobs to be done

What is Anjali really hiring Smart Compose to do?

- "Finish the boring half of my sentence so I can keep moving." She types
  "Sounds good, I'll" and wants " follow up with you next week" handed to her.
- "Save my hands." Fewer keystrokes per email, especially on mobile.
- "Match the moment." On a Friday afternoon she wants "Have a great weekend!"
  not "Have a great day!" The suggestion should know it is Friday.
- "Never embarrass me." It must not suggest something wrong, offensive, or
  presumptuous about a person. A bad guess in her own outgoing email is her
  reputation, not Google's.
- "Stay out of the way." If it has nothing good to say, it should say nothing.

That last two are the reason this is hard. The bar is not "be helpful sometimes."
The bar is "be helpful often, instant always, and embarrassing never."

## 5. How it works for the user

The visible experience is almost nothing, which is the point.

Anjali types in the Gmail compose box. As she types, a faint gray continuation
appears inline, right after her cursor, on the same line. It is not a dropdown.
It is not a popup. It looks like the email already contains those words, just
dimmed.

- Press **Tab** (or the right arrow) to accept the whole suggestion.
- Keep typing to ignore it. If her next letters match the suggestion, the gray
  text just shrinks as her black text catches up. If they diverge, the gray
  vanishes and a new suggestion may appear.
- On mobile, a swipe accepts.

She never opens a menu. She never picks from a list of five options. There is
exactly one suggestion at a time, take it or leave it. That single-suggestion
design is a deliberate latency and simplicity choice, and we will see why under
the hood.

Real example. She types "Let me know if you need anything" and the model adds
" else." She types "I hope you" and it offers " are doing well." She types
"Looking forward" and it offers " to hearing from you." None of these required a
menu. Each was one Tab.

## 6. The actual flow, step by step

1. Anjali opens a reply to an email whose subject is "Q3 partnership renewal."
2. Gmail already knows the context: the subject line, the body of the email she
   is replying to, her locale (en-US), and the current date and time (Friday,
   4:50 p.m.).
3. She types "Thanks for the". The client sends the current prefix plus that
   context to the Smart Compose service.
4. The service runs its language model, does a short beam search for the most
   likely continuation, scores it, and checks the score against a confidence
   threshold.
5. The continuation " update" clears the threshold. It comes back and renders as
   gray text after her cursor. Total round trip: tens of milliseconds. She does
   not perceive a delay.
6. She keeps typing " update on the". A new request fires. The model now
   suggests " renewal timeline." Gray text again.
7. She presses Tab. The gray turns black. The sentence is now "Thanks for the
   update on the renewal timeline."
8. She types "Have a" near 5 p.m. on Friday. Because the model sees the time
   context, it suggests " great weekend!" She taps Tab and sends.

Note what did not happen. There was no spinner. There was no list. There was no
"loading suggestions." Every keystroke that paused for even a moment got a fresh
answer back before her finger reached the next key, or it got nothing and stayed
silent.

## 7. Under the hood, like the engineer

This is the heart of it. Smart Compose is a language model prediction problem
wrapped in a savage latency budget. The published source is the KDD 2019 paper
"Gmail Smart Compose: Real-Time Assisted Writing" by Chen et al., and the 2018
Google AI blog post. I will mark what is confirmed by those and what is grounded
inference about how this class of system is built.

### The shape of the problem

Confirmed. Given a prefix (what you have typed in the current email) and a set of
context fields (the subject, the previous email body, your locale, the date and
time), predict the most likely continuation of the current sentence.

This is autocomplete, but a fundamentally different animal from the Google Search
autocomplete we tore down on 2026-06-16. Search autocomplete matches your prefix
against a finite catalog of past queries using a prefix trie. The candidate set
is a list of real strings that real people have typed before. You walk the tree
and rank what you find.

Smart Compose has no catalog of sentences to match against. The space of valid
English email continuations is effectively infinite. You cannot store it. You
have to generate it, token by token, from a model that learned the statistics of
language. Matching becomes generation. That is why the data structure at the
center is not a trie. It is a neural network plus a beam search.

### The text unit: wordpieces, not words

Confirmed. The model does not work in whole words. It works in subword units
called wordpieces, and the vocabulary is multilingual. "Thanksgiving" might be
two pieces, "Thanks" + "giving." "renewal" might be "renew" + "al."

Why subwords and not words? Two reasons.

- A whole-word vocabulary for every language Gmail supports would be enormous and
  full of rare words you can never learn well. Wordpieces cap the vocabulary at a
  manageable size (tens of thousands of pieces) while still being able to spell
  any word, including ones never seen in training, by gluing pieces together.
- Partial word completion. If Anjali has typed "renew", the model can finish the
  current piece into "renewal" because it predicts at the piece level. A
  word-level model would be stuck mid-word.

The data structure here is a hash map from wordpiece string to an integer id, and
the reverse. Every token in and out of the model is one of those integers. The
model never sees letters. It sees id sequences like [884, 12, 9050].

### The model: they picked the fast architecture on purpose

Confirmed, and this is the most important engineering decision in the paper.

They compared three designs:

- **Sequence-to-sequence with attention** (like machine translation). The encoder
  reads the subject and previous email; the decoder generates the continuation
  while attending back to the encoded context. In their tests this gave the best
  prediction quality. The baseline seq2seq had encoder and decoder each built
  from two 1024-dimensional LSTM layers.
- **Language model A (LM-A).** Drop the heavy attention encoder. Instead, encode
  each context field (subject, previous email) as a bag of words: average the
  word embeddings in that field into a single vector. Feed those context vectors
  into a recurrent language model that then generates the continuation.
- **Language model B**, a second variant of folding context into the LM.

Seq2seq won on quality. They shipped LM-A anyway. Read that again. They knowingly
gave up some accuracy to ship the faster model, because the attention mechanism in
seq2seq recomputes over the whole context on every step and that cost is fatal at
a per-keystroke latency budget. The bag-of-words context trick collapses the
subject and previous email into a couple of fixed vectors once, so the decode loop
stays cheap. This is the entire philosophy of the feature: the right answer that
arrives in 200 ms is the wrong answer.

Inference, well grounded. The recurrent core is an LSTM. At each step it holds a
hidden state vector, takes the previous wordpiece id, and outputs a probability
distribution over the whole wordpiece vocabulary for the next piece. The
expensive part of each step is the final matrix multiply from the hidden state to
the vocabulary-sized output, a tens-of-thousands-wide vector. That single matmul,
repeated for every token of the suggestion, is most of the compute.

### Generating the suggestion: beam search over a heap

Confirmed. To turn token-by-token probabilities into a whole suggested phrase,
they run beam search. Beam search keeps a heap of the m best partial sequences so
far. At each step it extends every candidate by one more token, scores the new
longer candidates, and keeps the best m. The score of a finished suggestion is its
length-normalized log probability. Length normalization stops the search from
unfairly preferring very short completions just because short strings multiply
fewer probabilities together.

Walk it concretely. Anjali has typed "Have a". The beam starts from that state.
Step one expands to candidates like " great" (high prob), " good" (high), " nice"
(medium), " wonderful" (low). Keep the top few. Step two expands " great" into
" great weekend" and " great day", expands " good" into " good day", and so on.
After a few steps the best-scoring full phrase is " great weekend!" because it is
Friday evening and the time context nudged the weights that way. That phrase is
the one suggestion that gets returned.

Why a heap? Because at every step you need the m highest-scoring candidates out of
many, and a heap gives you that top-m maintenance cheaply. This is the same "keep
only the top-k" move you see in every ranking system, here applied to half-built
sentences instead of search results.

### The silence switch: confidence-based triggering

Confirmed, and this is the feature's secret weapon for never embarrassing you.

After beam search produces its best suggestion, the system compares that
suggestion's confidence score to a threshold. If the score is below the threshold,
it shows nothing. No gray text at all. The threshold is tuned to hit a target
trigger rate, how often a suggestion appears at all.

This is why Smart Compose feels smart rather than annoying. It is allowed to stay
quiet. A search box has to return something. Smart Compose is designed to return
nothing most of the time and only speak when it is confident. Raising the
threshold means fewer but more accurate suggestions; lowering it means more
suggestions but more wrong ones that flicker and get ignored. That single dial is
the product's whole personality.

### Personalization: a tiny n-gram model riding alongside the giant neural one

Confirmed. The global neural model captures how everyone writes. But Anjali has
her own phrases. She always signs off "Cheers, A" and always writes "circle back
on this thread." A giant neural model retrained per user is impossible at billions
of users. So they bolt on a small per-user n-gram language model that learns each
person's frequent phrases, and they blend its score with the global model's score
by linear interpolation.

Concrete example. The global model, seeing "Talk", might lean toward " to you
soon." Anjali's personal n-gram model has seen her type "Talk soon, thanks" a
hundred times. Blended, the suggestion shifts toward her habit. The big model
gives general fluency; the tiny model gives her fingerprint. The n-gram model is
cheap to store and update because it is just counts of short phrase sequences in a
hash map, not a neural net.

### Fairness: the predictions that were deliberately removed

Confirmed in spirit by the paper and Google's writing on it. The model is not
allowed to guess gendered pronouns about a named person, because guessing wrong is
both offensive and a privacy leak. If you write "I am meeting Dr. Patel and",
the model will not gamble on " he" or " she." Those predictions were suppressed.
This is a content rule layered on top of the statistical model, and it matters
because the output goes out under the user's name, not Google's.

### Serving: where TPUs saved the feature

Confirmed, and this is the scale punchline.

The first version ran inference on standard CPUs. Average serving latency was in
the hundreds of milliseconds. That is a dead feature. By the time the gray text
appears, Anjali has typed three more letters and the suggestion is stale.

They offloaded the heavy model computation to TPUs (tensor processing units,
Google's matrix-math accelerator chips). Average latency dropped to tens of
milliseconds, and a single machine could serve far more requests at once. The
published target was a roughly 100 ms ceiling so the user perceives no lag, and
the system hit a 90th-percentile latency near 60 ms. Nine out of ten suggestions
come back in under 60 ms.

Training used a full TPUv2 Pod and converged in under a day, which is what let
them iterate on model designs at all.

Inference, well grounded. To get from hundreds of ms to tens, beyond just faster
hardware, the standard toolkit applies and the paper gestures at several of these:

- **Quantization.** Store and compute the model weights in 8-bit integers instead
  of 32-bit floats. Smaller, faster, and the quality loss is tiny for this task.
- **Encode the context once.** The bag-of-words context vectors for subject and
  previous email do not change as Anjali types. Compute them once per email, cache
  them, and reuse on every keystroke. Only the prefix changes.
- **Cap the suggestion length and the beam width.** Do not generate a paragraph.
  Generate a short phrase with a narrow beam. Every extra token and every extra
  beam is more matmuls on the clock.
- **Batch requests on the accelerator.** TPUs are happiest doing many predictions
  in one big matrix operation. Group concurrent users' requests into a batch so
  the expensive vocabulary matmul is amortized.

### The scale story at three tiers

The "catalog" for Smart Compose is not items in a database. It is users typing
keystrokes per second. Each keystroke that pauses is a full neural inference plus
beam search. So scale here means inference requests per second, not rows.

- **1,000 users.** Trivial. A handful of CPU machines could serve every keystroke.
  At this tier you would not even reach for TPUs. You would just run the model and
  ship. Latency per request on CPU, a few hundred ms, is annoying but survivable
  for a beta. This tier hides the real problem.

- **100,000 users.** Now the per-request cost matters. If even a fraction are
  typing at once, and each active keystroke is a hundred-millisecond CPU
  inference, your fleet balloons and your tail latency drifts past the point where
  the gray text feels late. What breaks: CPU inference cost per request. What you
  do: move inference to accelerators, quantize the model, cache the context
  encoding so only the prefix is recomputed, and turn the confidence threshold up
  so you simply do not spend compute on low-value suggestions. Staying silent is
  also a way to save machines.

- **1.5 billion users (Gmail's real scale).** Confirmed user count from the paper.
  Now the math is unforgiving. Even with most keystrokes triggering nothing, the
  active prediction volume is enormous and global. What breaks at this tier:
  per-machine throughput. A CPU box serving tens of requests per second cannot
  carry it without a server farm the size of a city. What they did: TPU serving,
  which both cut latency to tens of ms and multiplied requests per machine;
  batching concurrent requests into single accelerator operations; geographic
  distribution so a user in Mumbai hits a nearby data center and the network round
  trip itself does not eat the 100 ms budget; and the confidence gate doing
  double duty as a load shedder, since every suppressed suggestion is compute not
  spent.

The through-line across tiers: the live path must stay cheap and constant per
keystroke. Heavy training runs offline on a TPU Pod. The per-user n-gram models
are tiny hash-map count tables. The context is encoded once and cached. The model
is quantized and runs on accelerators. What reaches Anjali's cursor is a short
beam search over a small, fast model, returned in tens of milliseconds, or
nothing at all.

## 8. The retention and habit mechanic

Smart Compose builds habit through a tight, rewarding micro-loop that repeats
dozens of times per email session.

The loop: type a few words, pause, see a useful gray completion, press Tab, feel a
small hit of "that just saved me." Type the next clause, pause, Tab again. Each
Tab is a tiny reward delivered in under a second. Over a workday Anjali accepts
the suggestion hundreds of times. The Tab key becomes muscle memory. After a week
she expects the gray text to be there, and an email client without it feels slow
and manual by comparison.

Which metric does it move? Primarily **retention** and **engagement stickiness**,
not direct revenue. Smart Compose makes Gmail feel faster and more modern than
rival mail clients, which raises switching cost. Once your fingers expect Tab to
finish your sentence, moving to a mail app that does not do this feels like a
downgrade. Google reported that Smart Compose saves users from typing billions of
characters per week in aggregate, which is the observable proof the loop is firing
at scale. Saved keystrokes is the activation-and-retention signal here: a user who
accepts suggestions is a user who has internalized the habit and will keep coming
back to the surface that gives it.

The honesty of the mechanic matters too. Because it stays silent when unsure, it
rarely trains users to distrust it. A feature that interrupts with wrong guesses
teaches people to ignore it. Smart Compose's confidence gate protects the habit
loop by only ever spending the user's attention on a suggestion likely to be
accepted.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. The Smart Compose lesson is direct,
and it is about where you spend your latency budget.

**Ship the fast model on the live path, not the best model. Keep the quality
work offline and cached.**

Smart Compose had a more accurate architecture available (seq2seq with attention)
and deliberately shipped the faster one (LM-A with bag-of-words context), because
a suggestion that arrives after the user's next keystroke is worthless. They moved
the expensive context encoding out of the inner loop (compute it once per email,
cache it, reuse on every keystroke) and only let the cheap, prefix-dependent part
run live.

Apply this to Rare.lab's node graph. When a user is dragging a slider on a noise
node and the shader should update in real time, you are in the same place Smart
Compose was: you owe a correct-enough result inside a perceptual deadline (one
animation frame, about 16 ms at 60 fps), and the perfect result that arrives two
frames late is a stutter the user will hate. So split your pipeline the way they
split theirs:

- **Encode the unchanging context once.** The parts of the graph upstream of the
  edited node do not change while the user drags one slider. Compile and cache
  their output once when the drag starts. Only recompile from the edited node
  downstream on each frame. This is exactly their "encode the subject once, only
  recompute the prefix" move, applied to a shader DAG.
- **Have a fast preview path and a quality compile path.** While the user drags,
  run a quantized or lower-precision preview (fewer samples, half-resolution,
  cheaper math) that fits the frame budget. When the user lets go, run the full
  quality compile to shippable code. Smart Compose's quantized TPU model versus
  its offline TPU-Pod training is the same two-speed split.
- **Add a confidence gate of your own.** If recompiling a heavy subgraph cannot
  finish inside the frame, do not ship a half-frame stutter. Hold the last good
  frame and update when ready, the visual equivalent of staying silent rather
  than showing a late, wrong suggestion.

The principle in one line: decide your perceptual deadline first, then design the
live path to always hit it, pushing every expensive computation that can be
precomputed or cached out of the inner loop. Correct-and-on-time beats
perfect-and-late, every time, for anything a human is watching in real time.

---

## Sources

- Chen, Mia Xu, et al. "Gmail Smart Compose: Real-Time Assisted Writing." KDD 2019.
  Paper: https://arxiv.org/abs/1906.00080 (PDF: https://arxiv.org/pdf/1906.00080)
- KDD 2019 accepted paper listing:
  https://www.kdd.org/kdd2019/accepted-papers/view/gmail-smart-compose-real-time-assisted-writing
- ACM Digital Library entry: https://dl.acm.org/doi/10.1145/3292500.3330723
- Google Research publication page:
  https://research.google/pubs/gmail-smart-compose-real-time-assisted-writing/
- Google AI Blog: "Smart Compose: Using Neural Networks to Help Write Emails" (2018):
  https://research.google/blog/smart-compose-using-neural-networks-to-help-write-emails/
- Paper summary (weak-learner): https://www.weak-learner.com/blog/2019/11/03/gmail-smart-compose/

Note on sourcing. The model architecture choice (LM-A bag-of-words context over
seq2seq), wordpiece tokenization, beam search with a heap and length-normalized
confidence scoring, the confidence-based triggering threshold, the per-user n-gram
personalization blended by linear interpolation, the 1.5 billion user scale, the
"hundreds of ms on CPU to tens of ms on TPU" serving result, the ~60 ms 90th
percentile latency, and TPUv2 Pod training are all stated in the KDD 2019 paper
and the Google AI blog. Items labeled inference (8-bit quantization specifics,
request batching, caching the context vector across keystrokes, geographic
distribution, and the LSTM output matmul being the dominant cost) are the standard
way this class of low-latency neural serving problem is solved and are consistent
with the paper, but the exact production details are not all individually spelled
out in the public sources.
