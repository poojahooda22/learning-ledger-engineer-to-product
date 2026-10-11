# YouTube Shorts (the vertical swipe feed): how the next video is already playing before your thumb finishes the flick

Date: 2026-10-11
Product: YouTube
Feature: The Shorts feed, the full-screen vertical player you scroll with your thumb. This is a different machine from YouTube's long-form home recommendations (06-22), from "watch next" autoplay, and from the search box (10-02). Search answers "the user told us what they want, go find it." Long-form recommendations answer "pick the next 20 minute watch." Shorts answers a third, harder question: "the user asked for nothing, gave us almost no time to decide, and will swipe away in 1.5 seconds if the video is not already playing." The heart of this teardown is not ranking. It is delivery: how the next clip is buffered and decoded before you even decide to see it.

---

## 1. The user

It is 11:20pm. Aditya, 19, is in bed in Indore. He finished studying, he is not sleepy, and he opens YouTube "for five minutes." His thumb finds the Shorts tab out of habit. A cooking Short starts playing on its own: someone frying bread pakora, sizzling oil, close up. He watches four seconds, flicks up. A cricket highlight. Flick. A dog doing a trick. He watches the whole thing, twice. Flick. A guy explaining a UPI scam in 40 seconds. He watches all of it. Forty minutes later he is still there and genuinely surprised at the clock.

Second user, same feature, different day. Priya, 31, in a Bangalore cab in traffic, patchy 4G, one bar flickering to 3G. She opens Shorts to kill 20 minutes. The hard part for her is not what to show. It is that every swipe must play instantly even while her network drops to almost nothing between flyovers.

Both of them never typed a word. Both expect the next video to be playing the instant their thumb leaves the glass.

## 2. The real problem

Here is the honest version, the way you would tell a friend.

In a search box, the user hands you words. In the Shorts feed, the user hands you nothing but a swipe. You have to guess the next video out of a catalog that is now effectively unbounded. Remember that since October 15, 2024, any upload of three minutes or less counts as a Short (up from the old 60 second cap), so the pool of eligible Shorts is a huge and fast growing slice of YouTube's 2 billion plus video catalog, with 500 plus hours uploaded every minute.

But guessing the right video is only half the pain, and honestly it is the half YouTube already mostly solved years ago for the home feed. The Shorts specific pain is time and waste.

Time: a Shorts viewer is brutal. On the long-form home page a user will tolerate a second or two of spinner before a 15 minute video. On Shorts they will not. The whole interaction is "flick, judge in under two seconds, flick again." If the next video shows a loading spinner, the user does not wait. They swipe past it, and you have lost both that video and some trust. So the next video, and ideally the one after that, must already be downloaded and decoded before the current one ends.

Waste: the obvious fix to the time problem is "just download the next ten videos ahead." But Aditya watched four seconds of the pakora video and swiped. Every byte you downloaded past the first few seconds of a video he skipped is wasted money and wasted battery, and on Priya's flaky 3G it is bandwidth stolen from the video she actually wants. Research on exactly this problem names the trade-off plainly: prefetch too little and you get rebuffering (the spinner), prefetch too much in a fixed way and you get "significant rebuffering and bandwidth wastage" because so many prefetched clips get skipped (Nguyen et al, "Network-aware Prefetching Method for Short-Form Video Streaming," 2022). That tension, instant playback versus wasted bytes, is the entire engineering story of the Shorts feed.

## 3. The feature in one sentence

The Shorts feed is an endless vertical stack of short videos the system picks for you one swipe at a time, engineered so the next clip is already playing the moment you flick, without downloading so far ahead that it wastes the bytes you skip past.

## 4. Jobs to be done

What is Aditya really hiring the Shorts feed to do?

- "Fill this gap in my day with zero effort." He does not want to choose. He wants to be carried.
- "Never make me wait." A spinner between swipes breaks the spell instantly.
- "Read me fast." He gives each video about two seconds to earn the rest. Learn from the swipe.
- "Keep it fresh." If it feels like the same five videos, he is gone.

For Priya the job adds one line: "Work even when my network does not." The feature's real competitor is not TikTok in that moment. It is the lock button on her phone.

## 5. How it works for the user

Aditya taps the Shorts tab. A video is already playing, full screen, vertical, sound on, looping. No play button to press. On the right: like, dislike, comments, share, a spinning record showing the sound. He does one thing, over and over: flick up. Each flick snaps the next video into place, already playing. There is no visible "load." He can double tap to like, hold to pause, swipe up to move on. That is the entire interface. One gesture.

The quiet magic is the thing he never notices: the next video never buffers. Even Priya in the cab, with her bar flickering, mostly sees instant playback, maybe with the picture starting a touch softer and sharpening a second in. That softness is not a bug. It is the system choosing to start playing a lower quality copy immediately rather than make her wait for the crisp one (adaptive bitrate, torn down in full for long-form Netflix, 06-30, but used here with a much tighter deadline).

## 6. The actual flow, step by step

1. Aditya opens the Shorts tab. The app asks the server for a feed: not one video, but a small ordered list of the next several Shorts, each with its metadata and its streaming URLs at several quality levels.
2. The server runs the recommendation pipeline (section 7) and returns, say, the next 5 to 10 recommended Shorts as a list.
3. The phone immediately starts playing video 1. At the same time, quietly, it starts downloading the first chunk of video 2, and maybe video 3, so they are ready. It does NOT download all of video 2. Just enough of the front to start instantly (the research pattern: separate buffer thresholds for "the current video" and "the next videos in the list," Nguyen et al 2022).
4. Aditya watches 4 seconds of video 1 and flicks up. The phone instantly swaps to video 2, which is already buffered and decoded, so it plays with no spinner. The player now promotes video 3 to "next," starts prefetching video 4, and quietly throws away the parts of video 1 it had buffered past where he left.
5. As he nears the bottom of the returned list (say he has swiped through 7 of the 10), the app silently asks the server for the next page of recommendations, so the stack never runs dry. This is pagination, same idea as scrolling a search results page, but prefetched ahead of need.
6. Every judgment he makes, watched to the end, looped twice, liked, skipped at 2 seconds, tapped "not interested," is sent back to the server as a signal. The next page is shaped by what he just did.

The phone does the playback and the prefetch scheduling. The server does the picking and the ranking. Sorting billions of candidates never happens on the phone.

## 7. Under the hood, like the engineer

There are two engines here, and they run at once. Engine A picks the videos (recommendation). Engine B gets the bytes onto the glass before you swipe (delivery). Engine B is what makes Shorts different from everything else YouTube does, so I will go deep there, but let me do A first because A feeds B.

### Engine A: picking the next Shorts (confirmed pipeline, inferred Shorts-specific tuning)

YouTube has told us, in peer reviewed papers, the shape of its recommendation stack. The Shorts feed is not separately documented at the model level, so treat the pipeline as confirmed and the Shorts-specific knobs as clearly-labeled inference.

**Two stages, matching then ranking (confirmed).** YouTube's 2016 paper "Deep Neural Networks for YouTube Recommendations" (Covington, Adams, Sargin) lays out the split that still governs everything: a candidate generation model shrinks the enormous catalog down to hundreds, then a separate, heavier ranking model orders only those hundreds. This is the same matching-then-ranking split torn down for YouTube search (10-02) and Google search (10-10), with one crucial difference: in search the user gave you words to match against. In Shorts there are no words. The "query" is the user themselves: their watch history, the Shorts and channels they enjoyed, the topics they linger on, the sounds they engaged with (these are the exact signals YouTube's own Shorts help page lists).

So candidate generation here is a nearest-neighbor lookup in an embedding space (inference, but well grounded, this is how the 2016 paper frames retrieval and how the industry solves it). Picture every Short as a point in a few-hundred-dimensional space, placed by a learned "video tower" model. Aditya is also a point, placed by a "user tower" from his history. Finding candidates means "find the Shorts whose points sit nearest Aditya's point." The data structure that makes this fast is an approximate nearest neighbor index (ANN), the same family of structure torn down for YouTube search's semantic recall. It lets you find the closest few hundred points out of hundreds of millions without comparing against every point, in sub-linear time. Without ANN you would compute a distance to every Short on every swipe, which at 10 million plus Shorts is hopeless inside a swipe's time budget.

**Ranking is multi-objective (confirmed, and this is the good part).** YouTube's 2019 paper "Recommending What Video to Watch Next: A Multitask Ranking System" (Zhao et al, RecSys 2019) is the real engineering here. The ranker does not predict one number. It predicts many at once: will the user watch it to the end, will they like it, will they share it, will they tap "not interested." These objectives fight each other, so the paper uses a Multi-gate Mixture-of-Experts (MMoE): a shared bottom of neural network "experts," and a separate gate per objective that decides how much to lean on each expert. The final score is a weighted blend of the predicted objectives, with weights tuned by hand (the paper says exactly this).

For Shorts, the dominant objective is almost certainly completion and loop (inference): did you watch the whole thing, did it loop. That is why a 20 second video that gets watched twice beats a 2 minute 50 second video watched halfway. It matches what YouTube's Shorts help page says in plain English: it looks at whether people watched or skipped, average percentage viewed, and average view duration.

**The selection-bias trick (confirmed, and worth stealing).** The 2019 paper adds a "shallow tower" to correct for position and selection bias. The problem: a video shown in slot 1 gets watched more just because it was first, not because it was better. If you train naively on that, you teach the model "slot 1 is good," which is circular. The shallow tower learns the bias from features like which position the video was shown in, then subtracts it out, so the main model learns true quality. In a Shorts feed, where the video plays automatically and the top of the stack has an enormous positional advantage, this correction matters even more (inference).

### Engine B: getting the bytes there before the swipe (the Shorts-specific heart)

This is where Shorts stops looking like the home feed and starts looking like its own hard systems problem. The enemy is a human thumb that moves in under two seconds and a decision that the next frame must already be on screen.

**The data structure is a sliding window, not a list.** Think of the returned feed as a queue of upcoming videos: [current, next, next+1, next+2, ...]. The player keeps a small window of "hot" videos around the current position: the one playing, plus a few ahead that are partly buffered, plus maybe one behind in case you swipe back. As you flick, the window slides forward by one: the video you left gets its buffered bytes discarded, a new video at the far edge of the window starts prefetching. This is a classic sliding-window-over-a-queue, the same shape as a ring buffer. The window is deliberately small because the cost of being wrong (prefetching a video the user skips) is real money and real battery.

**The core move: prefetch the front of the next video, not the whole thing.** A Short might be 30 seconds. You do not download 30 seconds of video 2 while video 1 plays. You download just the first chunk, enough to start playback instantly, then pause. If the user actually swipes to it, you resume downloading the rest while they watch the opening. The research names this directly: use separate buffer thresholds for the current video versus the next videos in the recommended list (Nguyen et al 2022). The current video you fill generously because it is definitely being watched. The next videos you fill only to a shallow "can start instantly" threshold, because most of them will be skipped. A measurement study quoted in that literature found that the naive commercial baseline just downloads videos sequentially to the device, which either wastes bytes or rebuffers; the whole research direction is about being smarter than that.

**Adaptive, network-aware prefetch (confirmed research pattern).** How deep to prefetch is not fixed. It flexes with the network and with behavior. On Aditya's home wifi the player can afford to prefetch two or three videos a little deeper. On Priya's flickering 3G it must prefetch shallower and pick lower bitrates, or it will starve the video she is actually watching. The 2022 network-aware method and the 2023 "Digital Twin-Assisted Adaptive Preloading" work both size the prefetch to live throughput and viewing habits; the digital-twin paper reports cutting startup latency and data waste substantially versus fixed baselines (figures in that class of work run from double-digit to very large percentage reductions in waste, so treat the exact number as indicative, not a YouTube-published figure). The honest label: YouTube has not published its exact Shorts prefetch depth or thresholds, so the specific numbers are inference from this research class, not confirmed internal values. What IS confirmed is the shape: instant start by prefetching the front of upcoming clips, sized to the network.

**Why skips are the hidden tax.** Every time Aditya swipes away at 2 seconds, the bytes prefetched for the part of that video he never saw are pure waste. At 200 billion views a day (section 8), a tiny per-swipe waste multiplies into a staggering amount of pointless bandwidth. So "prefetch less, but smarter" is not a nicety. It is a direct line on YouTube's CDN bill and on the user's data plan. This is the single sharpest lesson for any infinite feed: the cost of being wrong about the next item is paid in bytes, and it scales with your most engaged users.

### The scale story, three tiers

Tier 1, about 1,000 Shorts (a tiny new app). You do not need any of this. Keep all video metadata in memory, pick the next video with a simple score, let the phone download each video just in time. A sliding window of one (just prefetch the single next video) is plenty. Nothing breaks. You could even skip ANN and brute-force the nearest-neighbor pick over 1,000 items on every swipe.

Tier 2, about 100,000 Shorts (a growing regional app). Brute-force scoring over the whole catalog on every swipe now costs too much per request, so candidate generation becomes real: precompute video embeddings offline, build an ANN index, and at request time pull a few hundred candidates near the user, then rank only those. Delivery: one video of prefetch is now risky on mobile networks, so you introduce the shallow "front chunk" prefetch for the next one or two videos and start serving multiple bitrates. What broke at the jump from tier 1: you could no longer touch the whole catalog per swipe, and you could no longer assume good network. What you add: the ANN index and the front-chunk prefetch window.

Tier 3, 10 million plus Shorts and hundreds of millions of viewers (actual YouTube Shorts). Everything is pushed to its limit. The catalog is sharded across many machines and the ANN index is distributed; no single box holds it all. The ranking model is the MMoE multi-objective network, served on fleets of machines, scoring a few hundred candidates per request in milliseconds. Video bytes do not come from origin at all: they come from a global CDN sitting close to the user (torn down for long-form as Netflix Open Connect, 07-13, and YouTube's own edge caches). The prefetch window is adaptive and network-aware because you cannot assume anything about a user's connection across the planet. Pagination of the feed is itself prefetched, so the stack never empties. What breaks if you try to run tier 2's design here: a central index and origin fetches would fall over under hundreds of millions of concurrent swipes, and fixed prefetch would either spinner on bad networks or bankrupt you on bytes. What survives the tier: sharding plus ANN for the pick, CDN edges plus adaptive front-chunk prefetch for the delivery.

A real walked swipe. Aditya's phone holds [dog-trick (playing), UPI-scam (buffered 3s), cricket-six (front chunk fetching)]. He watches the dog twice (strong completion-and-loop signal, logged). He flicks. UPI-scam snaps in with zero spinner because its first 3 seconds were already on the device. The window slides: dog-trick's buffer is discarded, cricket-six is promoted and deepened, a new candidate (a pakora recipe, because he lingered on cooking earlier) begins its front-chunk fetch. On the server, his "watched dog twice" event nudges the next page toward more animal clips. None of this is visible to him. He just sees video, video, video.

## 8. The retention and habit mechanic

The loop is the swipe itself, and it is close to the strongest habit loop in consumer software. It is a variable-reward slot machine: each flick might be the great video or a dud, and you cannot know without flicking. Because the cost of one more flick is near zero (no choosing, no waiting, instant playback), the user keeps pulling the lever. The instant-playback engineering is not a separate thing from the habit. It IS the habit. Add half a second of spinner per swipe and the whole loop weakens, because now each pull of the lever has a cost.

Which metric does it move: all three, but retention and time-spent most of all. YouTube reported Shorts going from about 70 billion daily views in 2023 to over 200 billion average daily views by mid 2025 (Neal Mohan, YouTube CEO, Cannes Lions, June 2025; repeated in YouTube's August 2026 Partner Program update). A real, concrete, and slightly sneaky example of how the metric is kept growing: on March 31, 2025, YouTube changed how a Short's views are counted, so that a view now counts when the Short starts to play or replays, rather than only after a watch threshold. That counting change means part of the jump from 70 billion to 200 billion is definitional, not purely organic, which is exactly why you should never compare the two numbers as if they measured the same thing. It is a clean example of how a metric definition quietly shapes the story a product tells about itself.

The second retention mechanic is the sound. The spinning record and the "use this sound" affordance turn one trending audio clip into thousands of Shorts, so a sound you heard yesterday reappears under new creators today. That recurrence makes the feed feel alive and current, the same "keep it feeling fresh so they open it again" instinct as Swiggy and Zomato rotating festival animations on the home screen (noted in earlier teardowns). A trending sound is a retention loop disguised as a creative tool.

## 9. The lesson for Rare.lab

The sharpest lesson from Shorts is about prefetch economics, and it maps straight onto a node-based shader editor with a live preview.

In Rare.lab, the expensive moment is recompiling and re-rendering the preview when the user tweaks a node. The naive design waits for the user to finish, then compiles, then renders, and the user sees a spinner on the canvas. That spinner is your "buffering between swipes." It breaks flow in exactly the way a Shorts spinner breaks the feed.

Do what the Shorts player does: prefetch the front of the likely-next state, sized to your budget, and throw away what you do not use. Concretely:

- When a user hovers or selects a node and starts to drag a parameter, speculatively compile and warm the shader for the small neighborhood of values around where their slider is, in the background, before they let go. When they release, the result is already warm and the preview updates instantly. This is the "front chunk of the next video" move: you are not pre-rendering every possible value (that is the fixed-prefetch waste trap the 2022 paper warns about), only the shallow, likely-next ones.
- Make the prefetch depth adaptive, exactly like network-aware preloading. On a strong GPU, speculatively compile two or three likely next edits. On a weak integrated GPU, compile only the single most likely one, or you will starve the preview the user is actually watching. Sense the device's real throughput and size the speculation to it, the same way the Shorts player sizes prefetch to the network.
- Treat a wasted speculative compile as a real cost, measured, not ignored. At your scale (an embeddable runtime on many pages at once), a tiny per-edit waste multiplies the way a per-swipe waste does at 200 billion views. Track your speculative-compile hit rate. If most speculations are thrown away, you are prefetching too aggressively; pull the window in.
- Keep the sort and the heavy compile off the embed and on your own machines where the equivalent of candidate generation lives. The embeddable runtime should be the thumb-flick-fast playback layer: it draws and it prefetches, it does not do the heavy picking. The phone in Shorts never ranks billions; your embed should never do the heavy compile it can get warmed and handed.

One concrete number to anchor the goal: the Shorts player aims to make the next video start with effectively zero perceived delay on a swipe that takes well under two seconds. Set the same bar for Rare.lab's preview on parameter release: the warm result should be there inside the time it takes the user's hand to come off the control. If your preview update is slower than the user's own motion, the flow breaks, and flow is the whole product.

---

## Sources

- Covington, Adams, Sargin, "Deep Neural Networks for YouTube Recommendations," RecSys 2016. https://research.google/pubs/pub45530/
- Zhao et al, "Recommending What Video to Watch Next: A Multitask Ranking System," RecSys 2019 (MMoE, shallow tower for selection bias). https://research.google/pubs/pub49380/
- Nguyen et al, "Network-aware Prefetching Method for Short-Form Video Streaming," 2022 (separate buffer thresholds for current vs next videos; rebuffering vs bandwidth-waste trade-off). https://arxiv.org/abs/2209.02927
- "Bandwidth-Efficient Multi-video Prefetching for Short Video Streaming," 2022. https://arxiv.org/pdf/2206.09839
- "Digital Twin-Assisted Adaptive Preloading for Short Video Streaming," 2023 (adaptive preload sizing; startup-latency and data-waste reductions). https://arxiv.org/pdf/2307.07836
- YouTube Help, "Search & discovery on Shorts" (official signals: watch/skip/not interested, average view duration, average percentage viewed, likes, surveys, past Shorts and channels, trending sounds). https://support.google.com/youtube/answer/11914225
- Neal Mohan remarks on 200 billion daily Shorts views, Cannes Lions, June 2025, as reported across YouTube statistics roundups; 70 billion daily views figure from 2023. https://blog.youtube/
- YouTube Shorts length expanded from 60 seconds to 3 minutes, effective October 15, 2024. https://www.socialmediatoday.com/news/youtube-expands-3-minute-shorts-to-all-users/736967/
- YouTube Shorts view-counting change (a start/replay counts as a view), effective March 31, 2025. https://support.google.com/youtube/
- Cross-references within this ledger: YouTube recommendations (2026-06-22), YouTube search ranking (2026-10-02), Netflix adaptive bitrate streaming (2026-06-30), Netflix Open Connect CDN (2026-07-13), Google Search core ranking (2026-10-10).
