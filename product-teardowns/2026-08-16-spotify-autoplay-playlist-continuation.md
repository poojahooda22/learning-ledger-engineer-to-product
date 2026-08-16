# Spotify Autoplay: the music that refuses to stop

Date: 2026-08-16
Product: Spotify
Feature: Autoplay (the endless session that starts the moment your playlist, album, or queue runs out)

Note on scope. This is not Discover Weekly (covered 2026-06-13, a batch job that builds one static playlist every Monday). This is not instant playback (covered 2026-07-07, the cache and CDN that make a track start in under 200ms). This is the thing that happens at the very end of your music: the last song fades, and instead of silence, Spotify keeps going with tracks it picked for this exact moment. Autoplay is real-time session continuation, not a weekly gift. That is what makes it a different animal.

---

## 1. The user

Meet Ananya. It is 7:40 in the morning in Pune. She is making chai and packing her laptop bag before a standup at 9. She opened Spotify, tapped her own playlist "monday reset" (34 songs, about two hours), and set the phone face-down on the kitchen counter. She is not looking at the screen. Her hands are wet. She is thinking about the deploy she has to babysit today, not about music.

Ninety minutes later she is at her desk, headphones on, deep in a code review. The playlist has quietly ended. She has not touched her phone since the kitchen. And the music is still playing. She could not tell you the name of the song that is on right now, but it fits. It sounds like her playlist. She never once had to pick the next thing.

That is the entire point. The user Autoplay serves is a person whose hands and attention are somewhere else.

---

## 2. The real problem

Here is the pain, described like a friend would.

You know that jarring moment when the music just stops? You are in the zone, or driving, or doing dishes, and suddenly the room goes quiet and you realize the playlist ended ten minutes ago and you have been working in silence. Now you have to stop what you are doing, find your phone, unlock it, and go hunting for something else to play. The mood is broken. The flow is broken.

The older fix was worse. You made a giant playlist of 500 songs so it would never end. But then it got stale, you heard the same 500 tracks forever, and it still ended eventually.

The real problem is that a playlist is finite and a listening session is not. A person listens until they decide to stop, not until the playlist decides to stop. There is a gap between "the content I picked ran out" and "I actually want to stop listening," and in that gap sits silence, friction, and a phone you have to go find.

---

## 3. The feature in one sentence

When your chosen music ends, Autoplay keeps the session alive by streaming a never-ending run of tracks chosen to match what you were just listening to and what you personally tend to like.

---

## 4. Jobs to be done

What is Ananya really hiring Autoplay to do?

- "Do not make me touch my phone. My hands are busy and my attention is elsewhere. Keep the sound going without me."
- "Do not break the mood. If I was playing calm morning music, do not throw death metal at me."
- "Play things that feel like me, not generic Top 50 filler. I picked this playlist for a reason; respect that reason."
- "Surface a few new tracks I have not heard, but not so many that it stops feeling like my music."
- "Never, ever go silent while I am still in the room."

Notice the tension baked into these jobs. "Feel like me" pulls toward the familiar. "Surface something new" pulls toward the unfamiliar. Autoplay lives or dies on how it balances those two. Hold that thought; it is the heart of the engineering.

---

## 5. How it works for the user

The visible experience is almost invisible, which is the design goal.

Ananya's "monday reset" playlist reaches its last track. She sees nothing, taps nothing. The last song ends. A new song begins. If she glances at her screen she will see a small line near the player that says something like "Autoplay: based on monday reset" and a queue of upcoming songs she never built. The songs keep coming, one after another, apparently forever, until she pauses or closes the app.

If she does not like a song, she taps skip, and Autoplay quietly takes that as a signal and adjusts. If she lets a song play all the way through, that is a signal too. She never opens a menu. She never picks anything. The feature reads her taps and her silence.

A Premium user gets richer, more varied Autoplay. A free user gets a more limited version with the usual shuffle and ad constraints. Autoplay can be turned off in settings, but the vast majority of people leave it on and never think about it, which is exactly why it works.

---

## 6. The actual flow, step by step

Let us walk it tap by tap with a real example. Ananya is playing the album "Midnights" by Taylor Swift, 13 tracks, about 44 minutes.

1. Track 13, "Mastermind," starts playing. The client knows this is the last item in the current context (the album).
2. As track 13 nears its end, the client has already asked the server: "This session is the Midnights album, listened to by this user, at this time of day. Give me the next batch of tracks." It does this early so there is zero gap.
3. The server returns an ordered list, say 20 to 50 tracks, already ranked. Track number one might be "cardigan" from the album "folklore," because it is the same artist and a similar mood. Track two might be a Lorde song, because listeners of late-night Taylor Swift albums often flow into Lorde.
4. "Mastermind" ends. The client immediately starts the first Autoplay track. No silence.
5. The little label updates to show this is now Autoplay, based on Midnights.
6. Ananya skips the third song (a track she finds too upbeat). The client logs the skip and fires it back to the server. The server uses it to re-rank what is still coming and what to fetch next.
7. When the client has played through most of the batch, it quietly fetches the next batch. This repeats indefinitely.

The key detail: the phone is not choosing songs. The phone is a thin client. It plays what it is given, logs what the user does, and asks for more before it runs dry. All the thinking is on the server.

---

## 7. Under the hood, like the engineer

This is the heart of the report. Autoplay is a recommendation problem, and every recommendation problem splits into two very different halves: matching (cheaply pull a few hundred plausible candidates out of a catalog of over 100 million tracks) and ranking (carefully order that short list for this person, right now). These are two different jobs with two different data structures. Mixing them up is how systems die at scale. Let us take them one at a time.

### 7a. The catalog and why you cannot just sort it

Spotify's catalog is over 100 million tracks (Spotify's own public figure, past 100 million as of 2024). You cannot, for a single Autoplay request, score 100 million songs and sort them. Even at a microsecond per song that is over 100 seconds of compute for one tap, times hundreds of millions of listeners. That is a non-starter. So the first job is to shrink 100 million down to a few hundred without doing 100 million units of work. That is matching.

### 7b. Matching: turn a song into a point in space, then find its neighbors

The trick that makes matching cheap is the embedding. Every track is turned into a vector, a list of numbers, typically 128 or 256 dimensions (commonly cited dimensionality for Spotify track vectors). Two songs that get listened to in the same contexts end up as two points close together in that space. "Mastermind" and "cardigan" sit near each other. A Norwegian death metal track sits very far away.

How are these vectors built? Two grounded sources feed them:

- Collaborative signal. Spotify runs matrix factorization over the giant user-by-track play matrix. If millions of people who play "Mastermind" also play "cardigan," those two tracks drift together in the factorized space. This is the same idea that powers Discover Weekly. It is behavior, not sound.
- Audio signal. Spotify analyzes the raw audio (tempo, key, energy, loudness, valence, danceability, the features that trace back to the Echo Nest acquisition). This is what lets a brand-new track with no play history still land in roughly the right neighborhood. Sound is the cold-start bridge.

Now, given the vector for the song the user just heard (or an average of the session's vectors), matching becomes one clean geometric question: which few hundred tracks are nearest to this point? That is a nearest-neighbor search. Done naively it is still a scan over 100 million points. So Spotify uses approximate nearest neighbor (ANN) search.

This is where a famous piece of real Spotify engineering shows up. Erik Bernhardsson, then at Spotify, built and open-sourced Annoy (Approximate Nearest Neighbors Oh Yeah) around 2013 to 2015. Annoy builds a forest of random-projection trees: pick two random points, split the space by the hyperplane between them, recurse until each leaf is small. To find neighbors of a query point you walk down the trees (cost proportional to tree depth, which is logarithmic in the number of points, not linear) and collect the leaves you land in as candidates. A scan of 100 million becomes a walk of maybe 30 levels across a handful of trees. The index is memory-mapped from disk, so many server processes share one copy in RAM and startup is instant. Annoy powered Home, Discover Weekly, Radio, and the same class of lookup Autoplay needs.

In 2023 Spotify shipped Voyager, a successor built on HNSW (Hierarchical Navigable Small World graphs). HNSW keeps a layered graph where the top layer is a sparse set of long-range links and lower layers get denser, so a search greedily hops from far away to the right neighborhood in a few jumps. Spotify's public position is that Voyager is markedly faster and higher-recall than Annoy with production-ready Java and Python bindings, and is now their recommended ANN library. (Exact speedup figures are on Spotify's engineering blog, which I could not fetch directly for this run; the architectural claim, HNSW replacing random-projection trees, is confirmed by the public repo and announcement titles.)

The point that matters for you: after matching, we are no longer talking about 100 million songs. We are talking about a few hundred. Everything expensive from here on operates on that short list. The candidate-set size is the dial that decouples the cost of ranking from the size of the catalog. This is the same lesson as Amazon search, YouTube, and Canva templates in this ledger.

### 7c. Ranking: the short list is not enough, now order it for this person and this moment

Nearest-neighbor in audio-and-behavior space gives you songs that are similar. Similar is not the same as good for Ananya at 9:10 on a Monday. Ranking is where personalization and context enter.

Spotify's public ranking brain for the Home surface is BaRT, which stands for Bandits for Recommendations as Treatments. The foundational paper is Spotify's 2018 "Explore, Exploit, Explain: Personalizing Explainable Recommendations with Bandits" (McInerney et al., RecSys 2018). Autoplay and Radio are described by Spotify as sharing the same recommendation machinery as Home, tuned with extra session-continuation and audio-similarity signals. So BaRT is the right mental model even where Autoplay's exact production code is not published; I will label the inferred parts.

Why a bandit and not a plain "predict the score, sort by score" model? Because of the tension from the Jobs to be Done. A pure exploit model always plays the safest, most-familiar-feeling track. That is comfortable for a week and then it becomes an echo chamber, the top complaint people have about Autoplay ("it just plays the same songs"). A pure explore model throws novelty at you and feels random. A multi-armed bandit is the formal tool for exactly this trade: each candidate track is an "arm," and the system must decide when to pull the arm it already believes is best (exploit) versus an uncertain arm that might be even better (explore).

BaRT's grounded shape:

- The reward is not a click. The reward Spotify optimizes is satisfaction, operationalized heavily as completed listens versus skips. A track played to the end (or past the ~30-second threshold that counts as a real stream) is a positive reward. A quick skip is a negative reward. This is the same philosophy as YouTube optimizing watch time over clicks: pick the signal that means the user was actually happy, not the one that is easy to count.
- The context is the whole situation, not just the last song: the seed content (the Midnights album), the user's long-term taste vector, time of day, device, and the recent skips inside this very session. Feed the same seed to two different users at two different times and the ranking differs.
- Exploration is tuned to certainty. For a brand-new user with almost no history, BaRT leans toward explore because it does not yet know them. For Ananya, a longtime user with a strong profile, it leans toward exploit, with a controlled trickle of new tracks. This is a real, stated behavior of the system.

The live ranking pass is cheap on purpose. Take the few hundred candidates, score each with the model against the current context, sort descending, and return the top 20 to 50. Sorting a few hundred items is nothing. This sort happens server-side, in the recommendation service, never on the phone. The phone receives an already-ordered queue.

### 7d. Walk one real request end to end

User: Ananya. Seed: the Midnights album just ended. Time: 09:10 Monday.

1. Build the session vector: average the vectors of the Midnights tracks she actually listened through (she skipped none), lightly blended with her long-term taste vector.
2. Matching: ANN query (Voyager/HNSW) for the ~400 nearest tracks to that session vector. Out come "cardigan," "august," several Lorde and Phoebe Bridgers tracks, some Gracie Abrams, a few of her own recent favorites. 100 million became 400 in well under a millisecond of search.
3. Filtering: drop tracks she disliked, tracks already played this session, tracks not licensed in India, podcasts, anything explicit if she has that filter on. 400 becomes maybe 250.
4. Ranking (BaRT-style): score each of the 250 for predicted completed-listen probability given the Monday-morning context and her profile. Mostly exploit (she is a known user), with a few higher-uncertainty tracks nudged up for exploration. Sort.
5. Return the top 30 as the Autoplay queue. "cardigan" is number one.
6. As she listens and skips, those events stream back and re-rank the tail of the queue and seed the next fetch.

Two halves, two data structures: an ANN index for matching, a scored sort for ranking. Everything heavy (training the embeddings, building the ANN index, training the bandit's base model) is done offline and refreshed on a schedule. The live path is one vector lookup plus one small sort. This is the offline-think, online-lookup spine that runs through the whole ledger.

### 7e. The scale story at three tiers

Tier 1, a catalog of 1,000 tracks. You do not need any of this. Score all 1,000 on every request and sort. A single server handles it. Matching and ranking are the same step. A hobby music app lives here happily. Nothing breaks.

Tier 2, 100,000 tracks and, say, 100,000 users. Scoring all 100,000 per request starts to hurt, especially under concurrent load. This is where you split matching from ranking. You precompute embeddings offline, build an ANN index (Annoy-style is plenty here), and at request time pull a few hundred candidates then rank only those. You add a cache for popular seeds: the Autoplay queue seeded by a hugely popular album like Midnights is nearly the same for many similar users, so cache it and serve it warm. What breaks at the edge of this tier is index freshness and memory: the ANN index must be rebuilt as the catalog grows, and it must fit in RAM. Memory-mapping the index (Annoy's design) lets many processes share one on-disk copy.

Tier 3, 100 million plus tracks and hundreds of millions of listeners (Spotify's actual scale, over 100 million tracks and well over half a billion monthly users). Now three things break and each has a real fix.

- The nearest-neighbor search itself gets slow and memory-hungry at 100M points. Fix: better ANN. Random-projection trees (Annoy) gave way to HNSW graphs (Voyager) for higher recall at lower latency. This is a real Spotify migration, not a hypothetical.
- The per-request compute, multiplied by hundreds of millions of sessions, is enormous. Fix: push all the expensive learning offline. Embeddings and the ANN index are rebuilt on a batch cadence. The bandit's heavy model is trained offline; the live path is a scoring pass over a few hundred candidates. The online cost per tap stays roughly constant no matter how big the catalog grows, because the candidate-set size is fixed by choice, not by the catalog.
- The hot-seed problem. When a global album drops (a new Taylor Swift release, a Bad Bunny drop), millions of people finish it within the same few hours and all trigger Autoplay off the same seed. That is a hot key. Fix: cache the candidate set and even a base ranking per popular seed, and personalize with a cheap final re-rank on top. Serve the shared expensive part once, do only the cheap per-user part live. This is the exact hot-key pattern from the Amazon Buy Box and Zepto teardowns.

The through-line: at every tier the survival move is to move work off the live request. Offline builds the index and trains the model. Online does a bounded lookup and a small sort. The catalog can 1000x and the tap still costs the same.

---

## 8. The retention and habit mechanic

Autoplay is a retention feature, and the mechanic is beautifully simple: it removes the natural stopping point.

Every playlist and album has an ending. An ending is a decision moment, and a decision moment is where a user churns out of the session. "The music stopped, do I bother finding more, or do I just close the app and get back to work?" A large fraction of the time, if the music stops, the session ends. Autoplay deletes that moment. There is no ending, so there is no decision, so the session just continues. The metric it moves most directly is session length and time spent listening, which are the core engagement metrics Spotify lives on, and downstream, retention (people who listen more churn less).

Compare the loops in this ledger. Discover Weekly is a scheduled loop: a new playlist every Monday manufactures a weekly reason to come back. Autoplay is the opposite kind of loop, a within-session loop: it does not bring you back next week, it keeps you here now, longer than you planned. Both are habit engines; they just operate on different clocks. Instagram Stories' 24-hour expiry and YouTube autoplay's countdown are the same species as Spotify Autoplay: remove the friction at the boundary so the next unit of content just arrives.

A real observed example of the mechanic working, and its risk: the most common complaint about Spotify Autoplay is that it "plays the same songs" and becomes an echo chamber (a recurring theme in Spotify Community threads). That complaint is the shadow of the retention design. Lean too far toward exploit and the loop is comfortable but goes stale, and staleness is slow churn. That is precisely why the ranking layer is a bandit and not a plain sorter: the exploration term is not a nice-to-have, it is the anti-staleness mechanism that keeps the retention loop alive over months, not just days.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus an embeddable runtime. The Autoplay lesson maps almost one-to-one onto the runtime's hardest job: never let the experience hit a hard stop, and never do the expensive thinking on the hot path.

Concrete lesson: build a "next effect" prefetch-and-continue loop into the embeddable runtime, and split it exactly the way Autoplay splits matching from ranking.

Here is the shape. Say a Rare.lab scene has a sequence of shader effects, or a user is browsing a gallery of effects in an embed. The moment one effect is running, the runtime should already be doing two things in the background, on the same offline-think/online-lookup spine:

- Matching, cheap and precomputed. Give every compiled shader an embedding based on its cost profile and visual family (particle-heavy, post-process bloom, distortion, and so on) and its target hardware tier. Keep a small ANN-style index so that "what could plausibly come next and still run at 60fps on this exact GPU" is a bounded lookup, not a scan over your whole shader library. As your effect catalog grows from 1,000 to 100,000 compiled variants, the candidate-fetch cost stays flat because you cap the candidate set, just like Spotify caps its few hundred.
- Ranking and prefetch, on the short list only. Take those few candidates, rank them by a cheap on-device model (does it fit the current frame budget, does it match the user's recent choices, is the GPU thermally throttled right now), then compile and warm the top one or two before they are needed. When the current effect ends or the user swipes, the next one starts with zero stall, the same way Autoplay's next track starts with zero silence.

The scalability punchline is Autoplay's punchline. The size of your shader catalog must never touch the cost of the live frame. Do the embedding, the index build, and the shader compilation offline or ahead of time; keep the runtime's per-frame decision to a bounded lookup plus a tiny sort. And borrow the bandit's discipline directly: do not always serve the safest, cheapest effect (that is the echo chamber, and your demos will look samey). Reserve a small, uncertainty-driven exploration budget to surface a fresh effect the user has not tried, because in a creative tool, staleness is churn just like it is in music.

---

## Sources

- Spotify Research, "Recsys Challenge 2018: Automatic Music Playlist Continuation" (defines APC as sequential recommendation; 1M playlists, main/creative tracks, 113 teams / 1,228 runs, top R-precision 0.2241): https://research.atspotify.com/publications/recsys-challenge-2018-automatic-music-playlist-continuation
- Spotify Research, "An Analysis of Approaches Taken in the ACM RecSys Challenge 2018 for Automatic Music Playlist Continuation": https://research.atspotify.com/publications/an-analysis-of-approaches-taken-in-the-acm-recsys-challenge-2018-for-automatic-music-playlist-continuation
- ACM RecSys Challenge 2018 dataset and details: https://recsys-challenge.spotify.com/details
- Spotify Million Playlist Dataset (Zenodo record, 1,000,000 playlists 2010 to 2017): https://zenodo.org/records/6425593
- McInerney et al., "Explore, Exploit, Explain: Personalizing Explainable Recommendations with Bandits" (RecSys 2018, the BaRT paper): https://research.atspotify.com/publications/explore-exploit-explain-personalizing-explainable-recommendations-with-bandits
- spotify/annoy, Approximate Nearest Neighbors in C++/Python (random-projection tree forest, memory-mapped): https://github.com/spotify/annoy
- Spotify Engineering, "Introducing Voyager: Spotify's New Nearest-Neighbor Search Library" (HNSW-based successor to Annoy; blog was egress-blocked this run, cited from its public title and repo): https://engineering.atspotify.com/2023/10/introducing-voyager-spotifys-new-nearest-neighbor-search-library
- InfoQ coverage of Voyager (HNSW, production ANN at Spotify): https://www.infoq.com/news/2023/11/spotify-ann-voyager/
- arXiv 1808.04288, "Automatic Playlist Continuation through a Composition of Collaborative Filters": https://arxiv.org/abs/1808.04288
- Background on Spotify audio features / Echo Nest and track vectors: https://python.plainenglish.io/what-spotify-hears-before-you-do-ca7a86be3e20
