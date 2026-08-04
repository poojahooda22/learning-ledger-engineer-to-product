# YouTube automatic captions (the "CC" that appears on a video nobody typed captions for)

Date: 2026-08-04
Product: YouTube
Feature: Automatic captions (the ASR pipeline that turns speech into timed text, plus auto-alignment)

---

## 1. The user

Meet Deepa. She is deaf. It is 9pm and her nephew has sent her a link to a
15 minute paneer butter masala tutorial on YouTube. The cook is chatting the
whole time, pointing at the pan, saying numbers out loud: "add two hundred
grams of paneer, then stir." Deepa cannot hear a word of it. The recipe is
locked inside the audio.

Now meet Rohan, sitting on a crowded Mumbai local at 8am, phone in hand, no
earphones. He wants to watch the same tutorial but he cannot turn the sound
on without annoying the whole compartment. And meet Sana in Jakarta, who
speaks Bahasa Indonesia and only a little English, watching the same video and
wishing it spoke her language.

Three different people. None of them can use the audio. The cook never typed
out a single caption. Yet all three tap the little "CC" button and words start
appearing at the bottom of the screen, roughly in time with the cook's mouth.
Nobody wrote those words. A machine listened and wrote them.

## 2. The real problem

Video is a wall of sound. If you cannot hear it, or you cannot play it out
loud, or it is in a language you do not speak, the whole thing is useless to
you. The obvious fix is captions, but captions are expensive. A human
transcriber charges real money and takes hours per hour of video. The person
who uploaded the paneer tutorial is a home cook with 4,000 subscribers. She is
never going to pay someone to transcribe her, and she is never going to sit
there and type it herself.

So without machine help the math is brutal: the tiny fraction of videos made
by big studios with budgets get captions, and the other 99 percent (the home
cooks, the lecture recordings, the 2011 vlog nobody will ever caption) stay
silent forever to anyone who needs text. For a deaf viewer that is not an
inconvenience. That is a locked door on most of the largest video library
humans have ever built.

The friend version of the pain: "I want to watch this, but the sound is no use
to me, and no human is ever going to type it out. Can the computer just listen
and write it down?"

## 3. The feature in one sentence

Automatic captions run speech recognition over a video's audio to produce a
time-aligned text track that appears under the video, generated with zero human
effort from the creator.

## 4. Jobs to be done

- "Let me read what is being said because I cannot hear it." (accessibility)
- "Let me watch this with the sound off without missing anything." (muted
  viewing on a train, in bed, in an open office)
- "Let me watch this in my language even though it was spoken in English."
  (auto-translate rides on top of the caption track)
- "Let me search inside the video, or let YouTube search inside it for me, so
  the right clip surfaces." (the transcript becomes indexable text)
- "Let me jump to the exact moment the cook adds the paneer." (timed text is a
  clickable index into the timeline)

Notice the job is never "give me a perfect literary transcript." It is "give me
enough correct, roughly-timed text that the video stops being a locked box."
That lowered bar is what makes the whole thing shippable.

## 5. How it works for the user

Deepa opens the video. In the player controls there is a "CC" button. She taps
it. Within a moment, a line of text appears at the bottom: "add 200 grams of
paneer and stir." A second or two later it is replaced by the next line. The
words track the cook, not perfectly, but close. Some words are wrong (the cook
said "ghee" and the caption says "gee"), and there is no punctuation in the raw
version, but the recipe is now readable.

Deepa taps the gear icon, picks "Auto-translate," chooses Hindi, and the same
timed lines now appear in Hindi. Sana in Jakarta does the same with Bahasa
Indonesia. Rohan on the train just leaves it in English and reads along with the
sound off.

The creator did nothing. She uploaded a video and went to bed. YouTube labels
the track "English (auto-generated)" so everyone knows a machine made it, not a
human.

There is also a second, quieter feature hiding here. If the creator DOES have a
plain text transcript (say she pasted her recipe script into a text file but did
not bother with timings), she can upload just the words with no timestamps, and
YouTube will figure out the timings for her. That is called auto-alignment. Same
engine, easier job, as we will see.

## 6. The actual flow, step by step

1. Creator uploads the paneer video. The upload also kicks off the transcoding
   pipeline (covered in the 2026-07-17 teardown) and, separately, the audio is
   handed to the speech system.
2. The audio track is pulled out and chopped into tiny overlapping frames,
   about one every 10 milliseconds.
3. The speech recognizer listens frame by frame and produces its best guess at
   the words, each word tied to a time.
4. A second pass adds capitalization and punctuation, decides where one caption
   line ends and the next begins, and (for multi-speaker audio) can mark speaker
   changes.
5. The result is written as a timed-text file: a sorted list of cues, each cue
   being {start time, end time, text}. In WebVTT form one cue looks like:
   `00:00:11.400 --> 00:00:14.000` then the line `add 200 grams of paneer and
   stir`.
6. That file is stored and served from the CDN like any other static asset.
7. Deepa taps CC. Her player fetches the tiny caption file, and as the video
   clock ticks past 11.4 seconds it renders the matching cue. The phone is doing
   nothing clever: it is looking up which cue covers the current timestamp and
   drawing it.

The heavy thinking (steps 2 to 5) happened once, offline, long before Deepa
pressed play. The live path (steps 6 and 7) is a cheap fetch and a lookup. Hold
that thought. It is the spine of this whole ledger.

## 7. Under the hood, like the engineer

This is the heart of the report. Turning "add two hundred grams of paneer" the
sound into "add 200 grams of paneer" the text is one of the genuinely hard
problems in computing, and the way it is solved is a beautiful match-then-decode
pipeline. I will give the confirmed public facts first, then the well-grounded
"this is how this class of problem is solved" version, clearly labeled.

### Confirmed facts to anchor on

- YouTube launched auto-captions in November 2009 (limited), then opened it to
  all English videos in March 2010. The lead engineer was Ken Harrenstien, who
  is himself deaf and pushed for the feature for years (Google announcement,
  NPR, CNN, 2010).
- Google has publicly stated that both auto-captioning and auto-alignment use
  the SAME speech recognition infrastructure that powers Google Voice and Voice
  Search, just trained on different data (Google Research, "Automatic Captioning
  in YouTube," 2010).
- Over the years that infrastructure moved from classic HMM plus deep neural net
  systems decoded over a Weighted Finite State Transducer (WFST) search graph
  (production graphs around 2GB) to modern end-to-end neural models. Google
  published its move to an all-neural, on-device RNN-Transducer recognizer in
  March 2019, quantized down to about 80MB.

### Step one: audio becomes an array of feature vectors

Sound is a pressure wave. A microphone samples it, say 16,000 numbers per second.
Raw samples are useless to a recognizer, so the first move is feature
extraction. Slide a small window (about 25ms wide) along the audio, hop forward
10ms each time, and for each window compute a compact vector of numbers that
describes the sound's frequency content (historically MFCC or PLP features, now
often log-mel filterbanks).

The concrete result: the cook's 15 minute video (900 seconds) becomes an array
of roughly 90,000 feature vectors, one every 10ms. Audio is now a data
structure: a long sequence of fixed-size numeric vectors. Everything downstream
operates on this array, not on the waveform.

Why 10ms? Because a spoken sound (a phoneme) lasts tens of milliseconds, so you
need several frames per phoneme to see it start, hold, and end. The word
"paneer" is roughly the phonemes /p/ /ax/ /n/ /IH/ /r/, and each of those spans
maybe 5 to 15 frames.

### Step two, the classic pipeline: three models composed into one graph

The traditional recognizer (the shape YouTube used for its first decade, and the
shape worth understanding because it makes the pieces visible) is three separate
models stitched together:

1. Acoustic model. Takes each feature frame and outputs probabilities over tiny
   sound units (phone states, often called senones). Early on this was a
   Gaussian mixture over HMM states; from the early 2010s it became a deep neural
   net, then an LSTM that can see context across frames. For our video, at the
   frame sitting at 12.5 seconds, the acoustic model might say "this frame is 70
   percent likely the middle of an /n/ sound."

2. Pronunciation lexicon. A dictionary mapping words to phoneme sequences,
   stored as a finite-state transducer. "paneer" maps to /p ax n IH r/. This is
   what lets the system go from sounds to words.

3. Language model. Probabilities over word sequences, classically an n-gram
   model. It knows that after "grams of" the word "paneer" is far more likely
   than "pioneer," even though the two sound similar. The language model is the
   context that rescues ambiguous audio.

Now the clever engineering. You do NOT run these three as three separate
programs. You compose them, offline, into one giant Weighted Finite State
Transducer, the HCLG graph (H for the HMM structure, C for context, L for the
lexicon, G for the grammar/language model). A WFST is a graph: states connected
by edges, each edge carrying an input label, an output label, and a weight (a
cost). Composing the four levels bakes "which sounds make which phones make which
words in which likely order" into a single searchable structure. Google has said
these production search graphs ran around 2GB.

### Step three: decoding is a shortest-path search with aggressive pruning

With the graph built, recognizing the audio is a search problem: find the path
through the graph that best explains the sequence of 90,000 feature vectors. Best
means lowest total cost, where cost blends acoustic fit and language likelihood.

This is Viterbi search / token passing over a lattice, and it is the exact same
family of algorithm as the map-matching in the Swiggy and Rapido teardowns, and
the same spirit as the spell-correction and autocomplete teardowns: you cannot
explore every possible path (the number of word sequences is astronomically
large), so you keep only the most promising partial hypotheses at each time step
and throw the rest away. That throwing-away is beam search: at each frame, keep
only hypotheses whose cost is within a "beam" of the current best, prune
everything else. Same "never expand a candidate that cannot win" pruning that
shows up everywhere in this ledger.

Walk the real example. At frame 1,250 (12.5 seconds) the survivors in the beam
might include "...grams of paneer" and "...grams of pioneer." Both fit the sound
okay. The language model cost breaks the tie: "grams of paneer" is a common
cooking phrase, "grams of pioneer" is nonsense, so its path carries a higher
cost and falls out of the beam. The winning path is not just the words. Because
every edge was traversed at a known frame, the path also tells you WHEN each word
happened. "paneer" occupied frames 1,244 to 1,299, which is 12.44s to 12.99s.
That frame alignment is where caption timestamps come from, for free, as a
byproduct of decoding.

The output is not one answer but a lattice: a compact directed acyclic graph of
the top competing hypotheses, which a later pass can rescore with a bigger
language model.

### Step four, the modern shape: collapse the three models into one neural net

The classic pipeline works but it has three separately trained parts and a 2GB
graph. Since the mid 2010s the field (and Google) moved to end-to-end neural
models that learn the whole audio-to-text mapping at once. Two big families:

- Listen, Attend and Spell (LAS): an attention encoder-decoder. The encoder
  ("Listen") turns the feature array into a higher-level representation, the
  attention ("Attend") decides which audio frames matter for the next word, and
  the decoder ("Spell") emits the text. Great accuracy, but attention likes to
  see the whole utterance, which is bad for live captioning.

- RNN-Transducer (RNN-T): an encoder over the audio, a prediction network that
  acts like a built-in language model over the text so far, and a joint network
  that combines them to emit the next symbol (or a "blank," meaning "no new word
  yet, feed me the next audio frame"). RNN-T's alignment is monotonic (it moves
  forward in time and never looks back), which makes it naturally streaming.
  That is exactly what you need for LIVE captions on a livestream, where the
  words must appear a second after they are spoken, not after the stream ends.
  Google shipped an RNN-T on-device recognizer in 2019, quantizing the model
  down to about 80MB from the old multi-gigabyte server graph.

Modern encoders are Conformer or Transformer blocks (attention plus
convolution). On top sit helper models: a neural language model to rescore, a
contextual-biasing step that nudges the recognizer toward words it should expect
(a music video's artist names, a tech talk's product names), a punctuation and
capitalization model, and segmentation/diarization to split lines and tell
speakers apart. The raw auto-caption you see with no punctuation is often the
recognizer output before or without the heavy punctuation model; the nicer
punctuated version is a second pass.

The key thing for this ledger: whether it is a 2GB WFST or an 80MB RNN-T, the
expensive recognition runs offline (or, for live, in a tightly bounded streaming
loop), and the thing the viewer fetches is always the same humble artifact: a
sorted list of timed cues.

### The second feature, auto-alignment, is the same engine with the answer key

Recall the creator who has the words but no timings. Her job is easier and it
reveals something deep. In plain auto-captioning the decoder must search over the
entire vocabulary ("what did she say?"), which is why it makes mistakes and needs
a huge language model and a big beam. In auto-alignment you already KNOW the word
sequence ("add two hundred grams of paneer and stir"), so there is nothing to
guess. The only question is WHEN each known word occurs.

That collapses the problem. Instead of searching the whole HCLG graph, you build
a tiny linear graph from just her known words and run Viterbi to find the single
best time-alignment of that fixed sequence against the acoustic model. This is
forced alignment. It is dramatically cheaper (no vocabulary-wide beam search)
and dramatically more accurate on timing (the words are given, so no word errors
are possible, only timing wobble). Same acoustic model, same Viterbi, constrained
search. Google confirmed both features share the infrastructure and differ in the
data and the task. The lesson to bank: when you already know the answer, do not
run the expensive open search, run the cheap constrained alignment.

### Where the sorting happens

The caption file is sorted by start time when it is written, once, on the server.
The player never sorts. To show the right line at 12.5 seconds the player does a
lookup into an already-sorted array of cues (a binary search finds the cue whose
[start, end] window contains the current time, or the players simply advance a
pointer as playback moves forward). A 15 minute video is a few kilobytes of
WebVTT. It caches trivially and serves from the edge like a thumbnail. The phone
does no speech work and no sorting. All of that is server-side and long done.

### The scale story at three tiers

Tier 1, about 1,000 videos. Just run the recognizer once per video on one
machine as a batch job and write the caption files. Recognition takes some
multiple of real time but nobody is waiting on the live path. At this size you
would not even build a queue. Accuracy (word error rate) is the only thing you
would obsess over, not throughput.

Tier 2, about 100,000 videos. Now you feel the weight. Each recognizer worker
needs the big model (the ~2GB WFST graph in the classic era) resident in memory,
and each video is tens of thousands to hundreds of thousands of frames to decode.
The fix is the same as the Content ID and transcoding teardowns: make caption
generation an offline batch job pulled from a queue by a fleet of workers, fully
decoupled from viewing. The viewing path is untouched by all this; it fetches a
static timed-text file from the CDN. Sorting and decoding are server-side, done
once. This is where the offline-think/online-lookup split earns its keep: the
costly part scales with uploads, the cheap part scales with views, and they never
touch.

Tier 3, 10 million plus, the real YouTube: over 500 hours of video uploaded every
minute. You cannot afford to eagerly caption everything in every language the
instant it lands. So you prioritize. Predicted-demand ranking decides what to
caption first (videos gaining views, popular languages), and the long tail waits
in the queue or is captioned lazily on first demand. Work is sharded by video and
by language, embarrassingly parallel, because captioning one video needs nothing
from any other video. For live streams you switch engines entirely: the streaming
RNN-T runs in a bounded real-time loop so captions trail the speaker by about a
second, because a batch decoder that needs the whole audio is useless on a stream
that has no end yet. The hot serving path stays absurdly cheap at every tier: the
caption artifact is a few kilobytes, so it is a keyed CDN lookup, the same shape
as serving a poster image. The thing that scales up is the offline recognition
fleet, not the thing the viewer touches.

What breaks at each jump: at 1k to 100k the break is throughput, fixed by a
queue and a worker fleet. At 100k to 10M the break is that you literally cannot
caption everything eagerly, fixed by demand-prioritized queuing and lazy
generation, plus a whole separate streaming path for live. The serving side never
breaks, because a sorted array of a few hundred cues is nothing.

## 8. The retention and habit mechanic

Captions do not look like a retention feature. They are one of the strongest in
the product, and they move three different metrics at once.

- Watch-time (retention). A huge share of mobile and feed video is watched with
  the sound off (the widely cited industry figure is that most Facebook and
  social video views happen muted). Without captions, a silent scroller hits a
  wall of soundless talking and leaves. With captions, Rohan on the train keeps
  watching. The caption is what converts a muted impression into completed watch
  time, and watch time is the metric YouTube's whole recommendation engine
  optimizes (see the 2026-06-22 recommendations teardown). Captions feed the
  loop that decides what to show next.

- Reach and activation. Auto-translate rides on the caption track. The moment an
  English tutorial has an English caption, it can be machine-translated into 100
  plus languages, so Sana in Jakarta becomes a viewer of a video she could never
  have understood. One creator's audience quietly expands to the whole planet
  with zero extra work from the creator. That is new activated viewers created
  out of thin air.

- Searchability (a quieter compounding loop). The transcript is indexable text.
  It lets YouTube search find the moment inside a video where "paneer" is said,
  powers "jump to" chapter-like navigation, and gives the recommendation system
  actual words to understand what a video is about, not just its title. The
  transcript makes the video legible to the machine, which makes it easier to
  surface, which brings more views.

Real observed example: creators routinely report that turning on and cleaning up
captions raises their average view duration and their non-English viewership,
which is exactly why YouTube nudges creators to review the auto-captions and why
accessibility advocates pushed for the feature in the first place. The habit is
not a dopamine hook. It is removal of a wall: every video that used to be a
locked box for some viewer is now open, so those viewers keep coming back to a
place where the content actually works for them. That is defensive retention
through inclusion, and it compounds because the library of captioned videos only
ever grows.

## 9. The lesson for Rare.lab

Two lessons, both about the offline-think/online-lookup spine and both directly
about scalability and performance.

First, the caption-track pattern for anything time-varying. YouTube does the
expensive recognition once, offline, and bakes the result into a compact
sorted array of {start, end, payload} cues that the runtime resolves with a cheap
lookup as the clock ticks. Rare.lab should treat any expensive-to-compute,
time-varying effect the same way. If a shader has an animation timeline, a set of
keyframed parameter sweeps, or a precomputed sequence of GPU state changes, do
not evaluate it live on the frame. Precompute it into a compact timeline artifact
keyed by (graph id, version), a sorted array of {t_start, t_end, state}, ship it
next to the compiled shader, and let the embeddable runtime do a binary search
(or a forward-advancing pointer, since playback is monotonic) to find "what
applies at time t." The per-frame cost becomes a lookup into a sorted array, flat
and predictable, instead of recomputation. That is exactly how a few kilobytes of
WebVTT hold up under 10 million concurrent viewers.

Second, the forced-alignment insight, which is the sharper and more valuable one.
Auto-captioning runs an expensive open search over the whole vocabulary because
it does not know the answer. Auto-alignment, given the known words, collapses to
a cheap constrained Viterbi that cannot produce a wrong word and is far faster.
The generalization: when you already know the structure, do not re-run the open
search. In Rare.lab, when only a node's PARAMETERS changed but the graph
STRUCTURE is unchanged, do not recompile and re-search the whole variant space
(the open, expensive path). Recognize that the structure is the known "word
sequence," and run the cheap constrained update: patch the already-compiled
artifact's parameter slots, re-align, done. Reserve the full expensive compile for
when the structure itself changes and you genuinely do not know the answer. Build
the system so it can tell the two cases apart, because the constrained case is the
common one during a live editing session (an artist dragging a slider), and
serving it as a cheap alignment instead of a full recompile is the difference
between a runtime that stutters on every tweak and one that feels instant. Know
when you have the answer key, and never pay the price of searching for what you
already hold.

---

## Sources

- Google Research, "Automatic Captioning in YouTube" (2010): https://research.google/blog/automatic-captioning-in-youtube/ (states auto-captioning and auto-alignment share the Google Voice / Voice Search speech infrastructure, trained on different data)
- Google Research, "An All-Neural On-Device Speech Recognizer" (March 2019): https://research.google/blog/an-all-neural-on-device-speech-recognizer/ (move from ~2GB FST search graph to an RNN-T model quantized to ~80MB; encoder, prediction network, joint network)
- Chan et al., "Listen, Attend and Spell" (2015): https://arxiv.org/abs/1508.01211 (attention encoder-decoder ASR)
- Google announcement and press on the launch of YouTube auto-captions (Nov 2009 limited, March 2010 all English), lead engineer Ken Harrenstien: https://www.npr.org/templates/story/story.php?storyId=124501330 and https://www.cnn.com/2010/TECH/02/08/deaf.internet.captions/index.html
- YouTube Help, "Use automatic captioning": https://support.google.com/youtube/answer/6373554
- Scientific American, "Google Works to Improve YouTube Auto-Captions for the Deaf": https://www.scientificamerican.com/article/google-youtube-auto-caption/
- Survey background on modern ASR architectures (Conformer, Transformer, RNN-T families, WFST decoding, forced alignment via Viterbi): https://arxiv.org/html/2510.12827
