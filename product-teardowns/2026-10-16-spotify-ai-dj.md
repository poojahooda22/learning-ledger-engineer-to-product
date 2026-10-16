# Spotify AI DJ: the robot radio host that knows your name

Date: 2026-10-16
Product: Spotify
Feature: AI DJ (the personalized voice that picks your music, announces it, and talks between songs)

This teardown is about the voice. Not the playlist. Spotify has many ways to
pick songs for you, and we have torn down several already: Discover Weekly
(2026-06-13), autoplay (2026-08-16), the personalized homepage (2026-08-06),
and instant playback (2026-07-07). AI DJ is different in one specific way. It
does not just hand you a list. It talks. A voice says your name, tells you what
is coming up, drops a fact about the artist, then plays the song. It feels like
a radio host who somehow knows exactly what you want.

That talking part is the whole engineering story. Picking songs, Spotify
already knew how to do. Making a human sounding voice say a personalized,
factual, brand safe sentence to 290 million paying listeners, fast enough that
it feels live, without melting a datacenter full of GPUs, is the new problem.
This report is about how that voice gets made.

A note on scope. AI DJ is a Spotify Premium feature. It launched in beta in the
US and Canada on 2026-02-22 (English only), and by 2026 it had grown to more
than 75 markets with voices in English, Spanish, French, German, Italian, and
Brazilian Portuguese. Where this report reasons about the exact serving
architecture rather than confirmed facts, it says so in plain words and labels
the guess as INFERENCE.

---

## 1. The user

Meet Aditi. She is 26, lives in Bengaluru, works in product marketing. It is
Tuesday, 9:10pm. She just got home, kicked off her shoes, and dropped onto the
sofa. She wants music. She does not want to choose music.

This is the exact gap. She has the energy to listen but not the energy to
decide. Her own library has 2,000 saved songs. Her Liked Songs playlist is a
wall. Discover Weekly sits there, but picking it still means picking. Opening
the search bar means knowing what she is in the mood for, and right now she does
not. She just wants someone to put something good on and tell her a little about
it, the way the radio did when she was a kid in the car with her dad.

She opens Spotify. On the home screen, near the top, there is a tile that just
says DJ with a little blue cartoon face. She taps it.

## 2. The real problem

Told the way a friend would tell it.

The hardest moment in any music app is the blank screen. You open it, and now
you have to be the DJ for yourself. That sounds like freedom. In practice it is
work, and it is work at exactly the moment you have the least energy for it,
which is why you so often just replay the same five albums.

Old radio solved this beautifully, but with one flaw. A radio host picks the
songs and does the talking, so you lean back and let it flow. The flaw is that
the host picks for everyone at once. The song after this one is chosen for the
whole city, not for you. So radio is lean back but not personal, and a Spotify
playlist is personal but not lean back. You still have to drive it.

The real problem is to get both at the same time. A voice that leans you back
like radio, but a playlist that is tuned to you like Discover Weekly. And the
voice cannot be a robot reading a stock sentence, because the second it sounds
canned, the magic dies and Aditi goes back to replaying her five albums. It has
to say her name, name her artists, sound like it means it, and do that for
hundreds of millions of different people on hundreds of millions of different
evenings.

## 3. The feature in one sentence

AI DJ is a synthetic radio host that uses Spotify's recommendation engine to
build you a personalized set of songs, writes a short spoken intro for each
part of the set using a large language model fed with editorial facts, and
speaks it all in a realistic human voice, so you get radio style lean back
listening that is tuned to you personally.

## 4. Jobs to be done

What is Aditi really hiring the DJ to do?

Job one, kill the blank screen. She is hiring it to make the first decision for
her so she never faces the empty "what now" moment. A real sign of this job:
she taps DJ instead of scrolling, the way you flip on the radio instead of
choosing a CD.

Job two, make it feel alive and for her. A silent playlist feels like a vending
machine. She is hiring the DJ to feel like a person who gets her. The voice
saying "Aditi, I've got something for you" does a job no shuffle button can do.

Job three, tell her why. When the DJ says "this next artist blew up after a tiny
show in Mumbai," she learns something, and the song means more. She is hiring it
for context, not just sound. That context is what old radio gave and what a
playlist never did.

Job four, let her nudge without taking over. If the vibe is wrong, she wants to
say "something more chill" and have it adjust, without going back to being the
DJ herself. She is hiring it to stay in charge while still listening to her.

## 5. How it works for the user

Aditi taps the DJ tile. A short beat, then the voice comes in. It is warm,
casual, clearly a real sounding person, not a flat text to speech robot. It says
something like: "What's good, Aditi. Let's ease into your Tuesday. Starting you
off with some of that indie you've been looping lately." Then a song plays.
Prateek Kuhad, "Kasoor," which she has played a lot this month.

Two or three songs go by. Then the voice cuts back in between tracks, for maybe
ten seconds. It names the next artist, gives a small fact, maybe ties it to
something she has listened to before, and rolls into the next song. The music
never really stops. The talk sits in the gaps.

At the bottom right there is a DJ button. If she is not feeling this run of
songs, she taps it. The DJ says something like "alright, switching it up," and
the whole direction changes to a different mood or genre. On newer versions she
can press and speak, or type, a request like "play me something upbeat for
cooking," and the DJ answers in its own voice and rebuilds the set around that.

That is the whole visible experience. A voice, a set of songs, a button to
change the vibe. Simple on the surface. Three heavy machines underneath.

## 6. The actual flow, step by step

Tap by tap, from Aditi's finger to sound in her ears.

1. Aditi taps the DJ tile on home.
2. The app asks Spotify's servers for a DJ session for her account.
3. On the server, the recommendation engine builds a lineup. Not one song, a
   sequenced run of songs grouped into small themed sets. For Aditi tonight:
   a "wind down indie" set of four songs, then a "songs you forgot you loved"
   set, and so on.
4. For the top of each set, the server assembles a short script. It takes an
   editorial template written by a human (for example, "Kicking things off with
   some [genre] you've been into") and fills the blanks with Aditi's own data
   (genre = indie, artist = Prateek Kuhad) plus a fact an editor wrote about the
   artist.
5. That finished sentence of text is sent to the voice system, which turns it
   into an audio clip in the DJ's voice.
6. The audio clip and the first songs are delivered to Aditi's phone.
7. The phone plays the intro clip, then the songs.
8. While those songs play, the server is already preparing the next commentary
   clip and the next set. The length of the current songs is the time budget to
   get the next spoken bit ready. A three minute song buys about three minutes.
9. When the set ends, the next clip is already there, so the voice comes in with
   no awkward wait.
10. If Aditi taps the DJ button, steps 3 through 9 restart with a new direction.
    If she speaks a request, the request is turned into text, understood as an
    intent ("more upbeat"), and fed back into step 3 as a new seed.

The thing to notice: the expensive work (choosing songs, writing the sentence,
making the voice audio) happens on Spotify's servers, ahead of the moment you
hear it, during the cover of the current song. The phone just plays files. Hold
that thought. It is the key to the scale story.

## 7. Under the hood, like the engineer

The DJ is three machines in a line: a music picker, a script writer, and a voice
maker. Each one is expensive in a different way. The art is in making them cheap
enough to run for 290 million people.

### Machine one: the music picker (mostly reused, not new)

This is the part Spotify already had. The DJ does not invent a new recommender.
It leans on the same recommendation stack that feeds the home screen, which
Spotify runs on a system it has described publicly as BaRT, short for Bandits
for Recommendations as Treatments. (CONFIRMED that BaRT powers Spotify Home and
recommendations; INFERENCE that the DJ reuses this same stack, though Spotify
describes the DJ as built on "your personalization," so the lean is well
grounded.)

BaRT is a multi armed bandit. Picture a row of slot machines. Each lever is a
candidate song or candidate theme. Pulling a lever (playing a song) returns a
reward (you listen, you save, you skip). A bandit balances two urges:
exploitation (play the Prateek Kuhad she clearly loves) and exploration (slip in
one new artist to learn something). This explore and exploit balance is why the
DJ feels both familiar and fresh.

Two halves run here, the same two halves as every search and feed teardown in
this ledger. First MATCHING: pull a candidate pool from Aditi's history (she
streamed Prateek Kuhad 12 times this month, saved "Kasoor," skipped three EDM
tracks). This is fast lookups into per user data, keyed by her account id,
backed by precomputed song vectors like the ones we saw with the Annoy index in
Discover Weekly (2026-06-13). Second RANKING: order the survivors by predicted
reward and group them into labeled sets. The new twist specific to the DJ is the
grouping and SEQUENCING. The output is not a flat list. It is an ordered list of
small sets, each set with a label the script writer can talk about ("wind down
indie"). The right data structure is exactly that: an array of sets, each set a
short queue of track ids plus a theme tag.

So machine one is old news dressed for a new job. The DJ's real novelty starts
at machine two.

### Machine two: the script writer (the LLM on a tight leash)

Now the server has a set labeled "wind down indie" with Prateek Kuhad on top. It
needs a sentence to say. This is where Spotify uses generative AI. Spotify's own
newsroom confirms two things in plain words: the commentary uses "generative AI
through the use of OpenAI technology," and Spotify put that technology "in the
hands of our music editors."

That second phrase is the whole engineering decision, so read it slowly. The LLM
is not left to free write about the song. If it were, it would hallucinate wrong
facts ("this artist won a Grammy" when they did not), drift off brand, or say
something unsafe, and it would do that to millions of people. Spotify instead
runs what PopSci described as a weekly writer's room, where human music editors
write the real facts and the base scripts. The generative model's job is to take
that editorial base and scale it, bending each line to the individual listener.

Think of it as fill in the blanks, not free writing. (The exact prompt design is
not public, so the mechanism below is INFERENCE, but it matches what Spotify has
said and how this class of problem is solved.) An editor writes a template and a
pool of vetted facts:

- Template: "Kicking off with some {genre} you've been into. Here's {artist}."
- Vetted fact about Prateek Kuhad, written and checked by a human.

The server fills the slots from Aditi's data (genre = indie, artist = Prateek
Kuhad), hands the model the vetted fact, and asks it to write one smooth, short,
on brand line. The model makes it sound natural and personal. It is not allowed
to supply the facts. The facts come from the editor. This is the same trick as a
good autocomplete: constrain the powerful model so its output is safe and
correct by construction. The data structure under this is a template with slots
plus a per user fact lookup, a hash map from account id to "recent artists,
recent genres, time of day."

Why constrain it so hard? Because the cost of a wrong or weird sentence is not
one annoyed user. It is one annoyed user times hundreds of millions, plus a
headline. The leash is not timidity. It is what makes a generative feature
shippable at this scale.

### Machine three: the voice maker (the expensive one)

Now the server has a finished sentence of text: "Kicking off with some indie
you've been into. Here's Prateek Kuhad." It has to become audio, in a specific
human voice, that sounds like it means it.

The voice comes from Sonantic, a London text to speech company Spotify acquired
in June 2022. Sonantic's whole pitch was emotional speech: not just reading
words, but adding the pauses, breaths, and changes in tone that make a voice
sound alive. Its technology was good enough to recreate Val Kilmer's voice for
the 2022 Top Gun sequel after illness had damaged his real one. The English DJ
voice was built with a real person, Xavier "X" Jernigan, Spotify's Head of
Cultural Partnerships and a former host of Spotify's morning show. He recorded
many hours of speech, and the model learned to speak new text in his voice.

How neural text to speech works, in plain terms (the general shape is CONFIRMED
for this class of system; Sonantic's exact model is not public, so specifics are
INFERENCE). It runs in two stages. Stage one takes the text and predicts the
sound shape over time: the pitch, the timing, where to pause, where to push.
That intermediate is usually a mel spectrogram, basically a picture of the sound
across frequencies and time. Stage two is a vocoder that turns that picture into
an actual audio waveform you can hear. Emotion lives mostly in stage one: a
slightly longer pause before "Prateek," a lift in pitch on "you've been into,"
the little breath that a flat robot never takes. That is what Spotify's Head of
Personalization Ziad Sultan meant when he said the realism "is about the emotion
in the voice."

This machine is the costly one. Running a neural vocoder is real compute. You
cannot casually synthesize a fresh audio clip for every utterance for every one
of hundreds of millions of sessions as if it were free. So the engineering
question becomes: how do you avoid doing this work over and over?

### Where the heavy work happens, and when

Here is the spine of the whole feature, the same spine as instant playback
(2026-07-07) and Netflix downloads (2026-10-08): do the expensive thinking
ahead of time and off the hot path, so the live moment is cheap.

Three moves make the voice affordable (the moves are INFERENCE in their exact
form, but they are how any sane team would build this, and they match the
constraints Spotify has described):

1. Precompute and cache the shared parts. A lot of what the DJ says is not
   unique to you. "Alright, let's switch it up." "Coming up next." These stock
   phrases are the same for every English listener. Synthesize each one ONCE,
   store the audio clip, and reuse it for everybody. This turns a synthesis job
   into a cache lookup keyed by (phrase id, language, voice). It is the same
   edge cache idea as serving a song file (Open Connect, 2026-07-13): make it
   once, serve it a million times.

2. Only synthesize the small personalized slot. The unique bit is short: a name,
   an artist, a genre, one fact. Synthesize just those pieces fresh, then splice
   the fresh slot into the cached frame. Most of the audio you hear was made long
   ago. Only a sliver is new.

3. Prefetch during the cover of the current song. You do not need the next
   commentary clip the instant the current one ends. You need it by the time the
   current set of songs finishes. A three to four minute song is a generous
   deadline. So the server starts preparing the next clip while you are still
   listening, and it is ready and waiting when the moment comes. The song itself
   is your latency budget. This is why the DJ never makes you sit through a
   loading spinner before it speaks.

Put together: the live path for Aditi's tap is cheap. Pull her cached candidate
pool, fill a template, look up the shared audio frame from cache, synthesize one
short personalized slice, splice, send. The genuinely expensive pieces, training
the voice model, writing the editorial facts, synthesizing the common phrases,
were all done earlier and are shared across everyone. Offline think, online
lookup, one more time.

### The scale story, three tiers

Walk the same feature at three sizes, with real numbers.

Tier one, 1,000 users. Just run everything live. Call the LLM per intro, call the
voice model per sentence, no cache, no prefetch. The GPU bill is a rounding
error at this size. Building a cache or a prefetch pipeline here would be
over engineering. Ship the simple thing. Spotify at a startup size would have
done exactly this.

Tier two, 100,000 users. Now the voice synthesis starts to hurt. If each person
hears, say, 15 spoken bits an hour and you synthesize every one from scratch,
your GPU fleet is doing a huge amount of repeated work, because a big share of
those bits are the same stock phrases. This is the tier where you start caching
common phrases, deduplicating identical utterances, and batching synthesis jobs
so the GPUs stay busy instead of idling between requests. You also lock the LLM
down to templates, because at 100,000 people a one in a thousand weird sentence
is now 100 weird sentences a session, and some of them will be screenshots on
social media.

Tier three, hundreds of millions (Spotify reported 751 million monthly users and
290 million Premium subscribers at the end of 2025). Live per utterance synthesis
for everyone is simply off the table. Four things would break, and here is what
you do about each:

- Voice synthesis cost explodes. Survive it by precomputing the shared audio
  (stock phrases, intros, transitions) once and serving from a cache and CDN, so
  the live work shrinks to one short personalized slice per spoken bit.
- The music picker cannot be rebuilt for the DJ. Survive it by reusing the
  existing recommendation infrastructure (BaRT, the candidate stores, the song
  vectors). The DJ adds sequencing on top, not a new recommender.
- Per user state is enormous. Survive it by sharding by account id, the same
  shard by key move as Notion by workspace (2026-06-25) and Gmail by mailbox
  (2026-10-03). Aditi's history and session live on specific shards, reached by
  keyed lookups, never a scan across everyone.
- Each new language multiplies everything. This is the sneaky one. When Spotify
  added Spanish, then French (DJ Maïa), German (DJ Ben), Italian (DJ Alex), and
  Brazilian Portuguese (DJ Dani), reaching 75 plus markets by 2026, every new
  language needed its own trained voice model, its own cache of synthesized stock
  phrases, and its own localized editorial facts and templates. So the thing that
  grows is not songs. It is languages times the shared phrase library times the
  small personalized slot. You survive it by keeping the shared parts shared and
  the fresh parts tiny, per language.

The one line version of the scale story: the catalog of songs is not what
strains the DJ, because song picking was already solved. What strains it is
turning text into a human voice, over and over, in many languages. The answer is
to make most of that audio once and reuse it, and synthesize only the thin
personalized slice live, prefetched inside the cover of the song that is already
playing.

## 8. The retention and habit mechanic

The DJ is built to win one specific moment: the blank screen, the lean back
moment when you want music but not decisions. That is the highest churn risk
moment in a music app, because it is when you might just close Spotify and open
YouTube instead.

The loop is clean. Cue: Aditi gets home, wants music, does not want to choose.
Routine: she taps DJ. Reward: a voice says her name and puts on exactly the right
indie, and tells her something she did not know. Repeat it enough evenings and
tapping DJ becomes the default the way flipping on the radio was. The decision
cost goes to zero, which is the entire point.

Two design choices sharpen the hook. First, the name and the voice. "What's
good, Aditi" does something a silent playlist physically cannot. It creates a
small parasocial, companion feeling, the same pull that made people loyal to a
specific morning radio host. A playlist is a vending machine. A voice that knows
you is a friend who is good at music. Second, the fact between songs. When the
DJ tells you why the next artist matters, you learn, the song lands harder, and
you trust the DJ's next pick more. Discovery plus trust is what keeps you from
reaching for the skip button and the exit.

Which metric does it move? Mainly engagement and retention, not revenue
directly. It is aimed at session length (you let a narrated set run longer than
you would babysit a playlist) and at frequency (you open Spotify more often
because there is now a zero effort front door). A concrete real world signal:
Spotify positioned the DJ squarely against choice overload and leaned into the
voice request feature (2026) so you can steer without leaving the lean back
state, which is a direct bet that reducing friction in that moment keeps people
listening. (The oft repeated "save rate lift" numbers floating around come from
vendor marketing pages, not Spotify, so this report does not cite them.)

## 9. The lesson for Rare.lab

Rare.lab is your node based shader and visual effects editor that compiles to
shippable code, plus an embeddable runtime. The day you add an AI assist, say
"suggest the next node," or "describe the look and I'll generate a graph," or a
voice that narrates what the effect is doing, you are standing exactly where
Spotify stood with the DJ. You are bolting an expensive generative machine onto
a product that has to stay fast. The DJ gives you a four part playbook, and it is
all about scale and performance.

1. Never run the heavy model on the hot path. The DJ does not synthesize voice
   while you wait, and it does not run the LLM per keystroke. Your editor renders
   at 60fps. So an AI node suggestion must never be a model call that blocks a
   frame. Make the live action a cheap lookup (a cached co occurrence index of
   "what node usually follows a Noise node") plus a light sort, and keep any
   heavy generation off the interaction thread entirely.

2. Put the generative model on a tight leash, like the editors' templates. The
   DJ's LLM is not allowed to invent facts; it fills editorial slots. Your code
   generator must not be allowed to invent a node graph that will not compile.
   Constrain it to your validated node schema so every graph it emits is valid by
   construction. A wrong sentence annoys one listener; a shader graph that fails
   to compile breaks someone's build. The leash is what makes the feature
   shippable.

3. Precompute and cache the shared parts, synthesize only the thin unique slice.
   Most of what the DJ says is shared stock audio made once; only a sliver is
   personal. For Rare.lab, precompile and cache the common shader variants and
   bundles that everyone uses, and generate fresh only the small custom piece a
   given user needs. Your compression and caching of these bundles is your egress
   bill and your cold start time, exactly as the DJ's phrase cache is its GPU
   bill.

4. Use the user's own thinking time as your latency budget, and prefetch into it.
   The DJ prepares the next clip during the cover of the current song. Your user
   pauses between dragging nodes. That pause is your three minute song. Prefetch
   and precompile the next likely bundle during idle moments so the switch is
   instant and never stalls a frame on a compile.

And the sharp, specific one, the way the DJ's multilingual growth taught it: the
thing that will blow up your cost is not the size of your shader catalog, it is
the number of targets you compile to. Each new backend (WebGPU, Metal, Vulkan,
a console) is the DJ adding another language: a whole new voice model and a whole
new phrase cache. Budget for that multiplication from day one. Keep the shared,
device independent work shared and computed once, and keep the per target fresh
work as thin as you possibly can. One line: build the AI assist as a splice of
cheap cached parts plus one small freshly generated slot, prepared ahead of the
moment it is needed, never synchronously, and never per target from scratch.

## Sources

- Spotify Newsroom, "Spotify Debuts a New AI DJ, Right in Your Pocket"
  (2026-02-22 launch announcement; the three technologies, OpenAI generative AI
  "in the hands of our music editors," the Sonantic voice platform, and the voice
  model built with Xavier "X" Jernigan).
  https://newsroom.spotify.com/2023-02-22/spotify-debuts-a-new-ai-dj-right-in-your-pocket/
- PopSci, "Meet Spotify's new AI DJ" (the three components, the weekly writer's
  room of editors, generative AI scaling the base script to individuals).
  https://www.popsci.com/technology/spotify-ai-dj/
- TechCrunch, "Spotify's new DJ feature is the first step into the streamer's
  AI-powered future" (uses OpenAI's model, may move to in-house models later).
  https://techcrunch.com/2023/03/13/spotifys-new-dj-feature-is-the-first-step-into-the-streamers-ai-powered-future/
- Voicebot.ai, "Spotify Acquires Synthetic Voice Startup Sonantic" and coverage
  of the June 2022 acquisition (emotional TTS, non-speech sounds, API plus GUI to
  "sculpt and direct" voices, Ziad Sultan quote).
  https://voicebot.ai/2022/06/13/spotify-to-acquire-ai-voice-startup-sonantic/
- Tubefilter, "Spotify just acquired a company that can make anyone say anything"
  (Sonantic learns from audio clips; the Val Kilmer / Top Gun voice).
  https://www.tubefilter.com/2022/06/13/spotify-just-acquired-a-company-that-can-make-anyone-say-anything/
- Interesting Engineering, "Spotify unveils an AI-powered personal DJ"
  (Ziad Sultan on emotion in the voice; the research team on LLMs and generative
  voice).
  https://interestingengineering.com/innovation/spotify-ai-personal-dj
- TheNextWeb, "Spotify's AI DJ now speaks French, German, Italian, and Brazilian
  Portuguese" (the 2026 multilingual expansion: DJ Maïa, Ben, Alex, Dani; 75 plus
  markets).
  https://thenextweb.com/news/spotifys-ai-dj-now-speaks-4-new-languages
- Engadget, "Spotify's AI-powered personal DJ expands to more languages and
  countries" (voice and text requests; market expansion).
  https://www.engadget.com/spotify-ai-powered-personal-dj-expands-to-more-languages-and-countries.html
- Billboard, "Spotify Q4 2025 earnings" (751 million monthly active users,
  290 million Premium subscribers at the end of 2025).
  https://billboard.com/pro/spotify-q4-2025-earnings-subscribers-total-revenue/
- Dynamoi, "Spotify algorithm glossary" and BaRT explainer (BaRT multi-armed
  bandit powering Home and recommendations; explore versus exploit).
  https://dynamoi.com/learn/spotify-algorithm/spotify-algorithm-glossary
