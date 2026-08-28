# Spotify: Loudness normalization and crossfade (even volume across every track, seamless transitions between them)

Date: 2026-08-28
Product: Spotify
Feature: Loudness normalization (the "every song is the same volume" magic) plus crossfade and gapless playback (the seamless join between songs)

---

## 1. The user

It is 8pm on a Friday. Priya has friends over. She opens Spotify, hits her "House Party" playlist, and drops her phone on the kitchen counter. The playlist is a mess in the best way: a quiet, breathy Billie Eilish track sits right next to a wall-of-sound EDM drop from Martin Garrix, then an old ABBA song ripped from a 1976 master, then a brand-new 2026 pop single mastered to be as loud as physically possible.

She is not going to babysit the volume. Her hands are full of samosas. She wants to set the volume once and forget it for the next three hours.

The second user is Arjun on his evening run. He has a "Deep Focus / Running" playlist of ambient and post-rock. He does not want a hard silence and a jarring click every time one track ends and the next begins. He wants the music to flow like one long piece, so his rhythm never breaks.

Both of them are asking Spotify for the same invisible thing: make the sound behave itself so I never have to think about it.

## 2. The real problem

Here is the honest version, the way you would explain it to a friend.

Every song on Spotify was made by different people, in different studios, in different decades, mastered to a different volume. This is not a small difference. A modern pop single might be mastered at -6 LUFS (very loud, squashed, almost no dynamic range). A classical piano recording might sit at -23 LUFS (quiet, wide dynamic range). That is a 17 dB gap. To your ear, that is roughly the difference between a whisper and a shout.

So without help, Priya's party is a disaster. The Billie Eilish track plays at a polite living-room level. Then the Martin Garrix drop hits and it is suddenly twice as loud, and everyone flinches and someone spills a drink. Then the 1976 ABBA track is so quiet nobody can hear it over the conversation, so Priya lunges for the volume knob, cranks it up, and gets blasted again on the next modern track. She spends the whole night as a human volume compressor.

And Arjun's problem: most audio files have a tiny bit of silence baked into the start and end, and the naive way to play a playlist is "finish file A, then start file B." That leaves a small but real gap, sometimes a click, right at the emotional handoff between two songs. For a DJ set, a live album, or a continuous ambient mix, that gap is the difference between "a playlist" and "an experience."

The problem is not that these are hard sounds to play. The problem is consistency across a catalog of 100 million tracks that Spotify did not create and cannot re-record.

## 3. The feature in one sentence

Spotify measures how loud every track really is, once, and then quietly nudges each one up or down at playback so they all sit at the same perceived loudness, and optionally overlaps the end of one song with the start of the next using an equal-power curve so the transition never dips or jumps.

## 4. Jobs to be done

- "Let me set the volume once and never touch it again for the whole session." (Loudness normalization.)
- "Do not make me flinch when a loud song comes on." (Peak and limiter protection.)
- "Keep the quiet, intimate songs quiet the way the artist intended, do not flatten everything." (Album normalization plus dynamic-range respect.)
- "Make my playlist feel like one continuous flow, not a list of separate files." (Crossfade and gapless.)
- "Do all of this without me learning what LUFS means." (Invisible by default.)

## 5. How it works for the user

For loudness, the user does almost nothing. Normalization is ON by default. A Premium user who digs into Settings finds one control, "Volume level," with three choices: Loud (-11 LUFS), Normal (-14 LUFS, the default), and Quiet (-19 LUFS). Quiet is for a late-night listen where you want to protect the dynamics. Loud is for a noisy environment like a gym. Most people never open this menu at all, which is exactly the point.

For crossfade, the user goes to Settings, finds "Crossfade," and drags a slider from 0 to 12 seconds. Set it to 6 seconds and every song now bleeds into the next over a 6 second overlap. There is a separate "Gapless playback" toggle for the case where you want zero silence but no blending (important for live albums and DJ mixes where the tracks were designed to touch).

That is the entire visible surface. Three radio buttons and two sliders. Everything hard happens underneath.

## 6. The actual flow, step by step

Priya's party, one real transition, the Billie Eilish track ending and the Martin Garrix track starting:

1. Priya taps "House Party" and hits play. Her phone volume is at 60 percent.
2. Track 1 (Billie Eilish, mastered around -14 LUFS already) plays. Spotify reads a stored number for this track and applies almost no gain change. It sounds normal.
3. Track 2 in the queue is the Martin Garrix drop, mastered at -6 LUFS (very loud). Spotify has a stored number for this track too: it needs -8 dB of gain to bring it down to -14 LUFS.
4. As track 1 nears its end, Spotify (crossfade on, 6 seconds) starts decoding track 2 in parallel while track 1 is still playing.
5. For the 6 second overlap, track 1's samples are multiplied by a falling curve and track 2's samples by a rising curve. Both curves are shaped so their combined power stays constant, no dip in the middle.
6. Crucially, track 2's samples are also multiplied by its normalization gain (-8 dB) the whole time. So the loud EDM track arrives already tamed. Nobody flinches. Nobody reaches for the phone.
7. Track 2's true peak is checked. If bringing a quiet track up would push its peaks past the safe ceiling, a limiter catches the overshoots so there is no digital clipping.

The whole dance happens on Priya's phone in real time, using two small numbers that Spotify computed long before she ever pressed play.

## 7. Under the hood, like the engineer

This is the heart of the report. The big idea is the same one that runs through half this ledger: do the expensive thinking once, offline, and store a tiny answer, so the live path is a cheap lookup and a cheap multiply.

### The two halves: measure once (offline), apply forever (online)

Loudness normalization splits into a measure half and an apply half, and they run at completely different times.

The measure half runs once, at ingest, when a track is first uploaded to Spotify. The apply half runs every single time anyone anywhere presses play. These are decoupled on purpose. If you tried to measure loudness live, you could not: measuring "integrated loudness" requires seeing the whole song start to finish, and you are trying to play the first second right now. You cannot measure the average loudness of a song you have not finished hearing. So measurement must be offline.

### The measure half: what "how loud is this song" actually means

You cannot just take the peak sample value. A track can have one loud spike and be quiet everywhere else. You cannot just average the raw sample amplitudes either, because the human ear does not hear all frequencies equally. A booming 40 Hz bass note and a piercing 3 kHz whistle at the same physical energy do not sound equally loud to a person.

The industry solved this with a standard, ITU-R BS.1770 (the same math behind EBU R128 broadcast loudness). Spotify measures against it. Here is what the algorithm does to one track, for example the ABBA "Dancing Queen" 1976 master:

1. K-weighting filter. Run the audio through a filter that boosts the highs a little and cuts the lows, shaped to match how the ear actually perceives loudness. Now a 40 Hz rumble counts for less than a 3 kHz vocal, the way it does to a real listener.
2. Chop into gating blocks. Slice the song into 400 millisecond windows, each overlapping the next by 75 percent (so the windows slide 100 ms at a time). For each window, compute the mean square power (average of the sample values squared, which is energy, not amplitude).
3. Gate out the silence. This is the clever part. Throw away the quiet windows so they do not drag the average down. Two thresholds: an absolute gate at -70 LUFS (kill dead silence and near silence), then a relative gate 10 dB below the loudness of what survived the first gate (kill the quiet passages relative to the song's own body). The soft intro and the fade-out do not get to make "Dancing Queen" look quieter than it feels when it is actually playing.
4. Average what is left. The mean of the surviving blocks, converted to the log scale, is the integrated loudness in LUFS. One number for the whole song.

For "Dancing Queen" that number might be around -13 LUFS. For the Martin Garrix track, -6 LUFS. For the classical piano piece, -23 LUFS.

### The apply half: one subtraction, one multiply

Once you have the measured loudness, the gain is trivial arithmetic. Target is -14 LUFS (Normal). 

- Martin Garrix at -6 LUFS needs -6 minus -14 = -8 dB. Apply -8 dB.
- Classical piano at -23 LUFS needs -23 minus -14 = +9 dB. Apply +9 dB.
- "Dancing Queen" at -13 needs +1 dB.

That gain is stored as one small number attached to the track's metadata. This is the ReplayGain idea (a tag that says "play me this many dB louder or quieter"), applied at streaming scale. The audio file itself is never changed. Spotify does not re-encode your master. It stores a gain value and multiplies your samples by it at playback. Turning normalization off just means "ignore the stored gain," and the untouched original plays.

Applying it live is the cheapest thing in the whole system. For each audio sample flowing out of the decoder, multiply by a constant gain factor (a -8 dB gain is a multiply by about 0.398). That is one floating-point multiply per sample. At 44,100 samples per second per channel, that is a rounding error of CPU. A phone from 2014 does it without noticing.

### The two traps, and how they are handled

Trap one: positive gain can clip. Turning the -23 LUFS piano piece up by +9 dB could push its loudest peaks past the maximum a digital signal can represent (0 dBFS), which causes ugly distortion. Spotify guards this two ways. First, for quiet-but-peaky tracks it refuses to lift all the way. The rule Spotify publishes: if a track is -20 LUFS but its true peak is already at -5 dBFS, Spotify only lifts it to -16 LUFS, not the full -14, leaving about 1 dB of headroom for lossy encoding. It accepts "slightly too quiet" over "clipped and distorted." Second, when positive gain is applied, a limiter sits on the output set to engage at -1 dB (sample values), with a 5 ms attack and a 100 ms decay, catching the occasional peak that would otherwise clip. Negative gain (turning loud tracks down) can never clip, so it is applied freely with no distortion.

Trap two: normalizing every track individually would flatten an album. Think of an album where the artist deliberately put a whisper-quiet interlude before an explosive finale. If Spotify measured each track alone and shoved both to -14 LUFS, the interlude would get loud and the finale would get quiet, destroying the artist's intended contrast. So when you play a full album in order, Spotify computes one album gain (based on the album's overall loudness) and applies that same gain to every track. The relative loudness the artist designed is preserved: the interlude stays soft, the finale stays huge. Individual track gain is used for shuffle and playlists (where tracks come from different albums); album gain is used for album playthrough. Same measured data, two ways to group it.

### Crossfade: the equal-power curve

Crossfade is a separate, smaller piece of DSP, and it hides a real trap of its own.

Naive crossfade is linear: fade track A's volume down from 1.0 to 0.0 on a straight line, fade track B up from 0.0 to 1.0 on a straight line, over 6 seconds. This sounds broken. At the midpoint both tracks are at 0.5 (which is -6 dB). For two uncorrelated songs, perceived loudness follows the power (the square of the amplitude), and 0.5 squared plus 0.5 squared is 0.5, so the combined power drops to half. You hear a distinct dip, a hole in the middle of every transition. Priya's party would sag every six-second crossfade.

The fix is the equal-power (constant-power) crossfade. Instead of straight lines, use:

- gainA = cos(t times pi/2)
- gainB = sin(t times pi/2)

where t runs from 0 to 1 across the fade. At the midpoint (t = 0.5) both gains are 0.707 (which is -3 dB), and here is the magic: sin squared plus cos squared always equals 1, at every point in the fade. So the combined power stays constant the whole way across. No dip, no hole. The transition sounds like one continuous wall of sound handing off to another, which is exactly what a good DJ does by ear.

Gapless playback is a different problem again: it is about removing silence, not blending. Most encoded audio (AAC, Ogg Vorbis, MP3) has a few milliseconds of encoder delay and padding baked into the start and end of each file. Gapless playback reads the encoder-delay and padding metadata, trims those samples, and starts decoding track B early so its first real sample lands exactly when track A's last real sample ends. Zero inserted silence, no click, but no overlap either. This is what a live album like Pink Floyd's "The Dark Side of the Moon" needs, where one track flows straight into the next by design.

### The scale story, three tiers

The "catalog" here is the number of tracks. The cost that matters is: where does the loudness measurement run, and where does the gain get applied.

Tier 1, about 1,000 tracks (an indie service, a hackathon clone). You could honestly measure loudness on the fly with a small buffer, or precompute lazily the first time each track is played and cache the result. Nothing breaks. Do not over-engineer. Store the gain in a column next to the track row and move on.

Tier 2, about 100,000 tracks (a mid-size library). Measuring at playback now wastes real CPU, because popular tracks get played millions of times and you would re-measure the same song over and over. The move: measure once at ingest, in a batch job, and store the gain as metadata. The measurement is embarrassingly parallel (every track is independent, no track's loudness depends on another's), so you fan it out across a worker pool. The live path becomes a metadata read plus a per-sample multiply, which costs effectively nothing. This is where the offline-think / online-lookup split becomes mandatory rather than optional.

Tier 3, 100 million plus tracks (real Spotify). The measurement is still embarrassingly parallel, so the ingest pipeline just scales horizontally: throw more workers at the queue, each measures a track independently, writes back one float (track gain) plus one float per album (album gain). The stored data is tiny, a few bytes per track, so it rides along with the track's existing metadata and gets cached at the CDN edge right next to the audio the client is already fetching. The live per-sample multiply happens on the listener's own device, so the serving cost of applying normalization is zero to Spotify: it is paid by 600 million phones, each doing one cheap multiply. Nothing in the loudness path scales with catalog size on the hot path. The only thing that grows is the one-time ingest cost, which is linear in new uploads per day (tens of thousands of new tracks daily), and that is a background batch, not a latency-critical service. Crossfade and gapless are pure client-side DSP, so they cost Spotify nothing at any scale: the phone decodes two streams for a few seconds during the overlap, which is the one moment the client does slightly more work.

What breaks at the next tier and what saves you: the thing that would break if you did it naively is measuring or normalizing on the server at play time. That would make loudness a per-play server cost, 100 billion plays a year each doing a full-song analysis, which is absurd. The save is the same pattern every time: precompute the small answer offline, store it as metadata, apply the cheap transform on the client.

### Confirmed vs inferred

Confirmed from Spotify's own artist documentation and the ITU standard: the -14 LUFS Normal target, the -11 Loud and -19 Quiet listener settings, ITU-R BS.1770 as the measurement standard, gain applied at playback rather than re-encoding the file, album normalization for album playthrough, the limiter at -1 dB with 5 ms attack and 100 ms decay, the 1 dB headroom rule and the -20 LUFS / -5 dBTP to -16 LUFS example, and the 0 to 12 second crossfade range. Confirmed from the ITU-R BS.1770 recommendation: K-weighting, 400 ms gating blocks at 75 percent overlap, the -70 LUFS absolute gate and the -10 dB relative gate, mean-square power. The equal-power cos/sin crossfade curve is the standard, well documented DSP solution and is inference on Spotify's exact implementation (the audible result matches equal-power, but Spotify has not published its curve). The internal storage-as-metadata, CDN-edge caching, and ingest fan-out details are clearly labeled inference on how this class of problem is solved at scale, grounded in the ReplayGain design and Spotify's known offline-then-serve architecture seen elsewhere in this ledger.

## 8. The retention and habit mechanic

This feature is a pure friction remover, and friction removers move retention by protecting session length, not by adding a dopamine hit.

The mechanic is the absence of a bad moment. Every time Priya's party would have made her reach for the volume, and does not, that is a session that keeps going instead of a session that gets interrupted. Interruptions are where sessions die: the flinch, the lunge for the phone, the "ugh, this playlist is all over the place," the decision to just put on the radio instead. Loudness normalization removes the single most common reason a long listening session gets broken, which is a jarring volume change. Crossfade does the same for the gap between songs: a party or a workout playlist with 6-second crossfades feels like a professional DJ set, so it holds the mood and the room, and the listener never gets the little "song's over, what now" cognitive gap where they might stop.

Which metric does it move: retention (session length and days-active), through fewer skips and fewer session-ending interruptions. There is a real, observed behavioral link here: when a track jumps out at unexpected loudness, listeners skip it, and skips are one of Spotify's clearest negative signals (BaRT, the ranking system behind Autoplay covered earlier in this ledger, optimizes for completed listens over skips). Normalization removes a whole class of "that was too loud, skip" reactions that have nothing to do with the song and everything to do with mastering. A concrete example: an old, quiet 1970s remaster dropped into a modern playlist would get skipped as "boring / low energy" simply because it plays quiet, even if the listener would love it at matched volume. Normalization gives that track a fair hearing, which keeps more of the catalog in play and keeps the session alive.

It is worth naming why this is a retention feature and not a revenue feature: the deeper "Volume level" control (Loud / Normal / Quiet) is Premium-only, so there is a thin revenue edge, but the default normalization is on for everyone, free and paid. The point is not to sell it. The point is that nobody notices it, and "nobody notices it" is the highest compliment an infrastructure feature can earn.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code plus an embeddable runtime. The transferable lesson here is two-part, and both parts are about doing analysis once and applying a cheap transform forever.

Part one, normalize per asset offline, apply a scalar at runtime. Spotify's whole trick is: measure each track's loudness once at ingest, store one number, multiply at playback. Do the exact same thing for visual assets and effects. Every shader, texture, or effect node a user drops into a Rare.lab scene has an intrinsic "loudness": its average brightness, its peak intensity, its bloom energy, its GPU cost. Today a scene that composites a subtle fog effect next to a screaming neon glow effect has the same problem Priya's party had: the two do not sit at a consistent level, and the user has to hand-tune every effect's intensity by eye every time. Instead, analyze each effect once at compile or import time (measure its output brightness and peak the way BS.1770 measures loudness, gated so a single bright spark does not define the whole effect), store one normalization scalar in the compiled artifact's metadata, and apply it as a single cheap multiply in the runtime. A user can then drop any effect into any scene and have it arrive already leveled to the scene's target intensity, with a per-scene "intensity level" control (your Loud / Normal / Quiet) that is one uniform, not a re-analysis. The analysis is embarrassingly parallel across assets, so it fans out at build time and never touches the hot per-frame path. And keep a true-peak guard exactly like Spotify's limiter: when you scale an effect up, clamp or soft-knee the output so a boosted effect never blows past the display's white point and clips, the visual equivalent of digital distortion.

Part two, use equal-power blending, never linear, when you cross-dissolve between two shader states. This is the sharpest, most concrete transfer. When Rare.lab transitions between two visual states (a scene morph, a material cross-dissolve, blending two shader outputs over time), the naive move is a linear lerp: output = (1 - t) times A + t times B. That is the exact linear crossfade that gives Spotify a hole in the middle, and it gives you the same thing visually: at t = 0.5 both contributions are at half, and for two uncorrelated images the perceived brightness and energy sag in the middle of the transition, a visible dip that reads as a flicker or a wash-out. Use the equal-power curve instead: weight A by cos(t times pi/2) and B by sin(t times pi/2), so the summed energy stays constant across the whole dissolve (cos squared plus sin squared equals 1). The transition holds its intensity from start to finish, no mid-point dip. This is a one-line change in your blend node's weighting function that makes every cross-dissolve in the product look professionally even instead of subtly broken, and almost nobody implementing a blend node thinks to do it. Ship equal-power blending as the default in the compiler, and let the sag-prone linear blend be the opt-in special case, not the reverse.

The one line: measure each asset's intensity once offline and store a normalization scalar you apply as a cheap runtime multiply (with a peak-limiter guard), and blend between visual states on an equal-power curve so transitions never dip, because the cheapest way to make a whole system feel consistent is to precompute one small number per asset and apply it live.

---

## Sources

- Spotify for Artists, "Loudness normalization on Spotify" (official): https://support.spotify.com/us/artists/article/loudness-normalization/
- Spotify support, "Tracks transitions" (crossfade and gapless settings): https://support.spotify.com/us/article/tracks-transitions/
- Recommendation ITU-R BS.1770-5 (11/2023), "Algorithms to measure audio programme loudness and true-peak audio level" (PDF): https://www.itu.int/dms_pubrec/itu-r/rec/bs/R-REC-BS.1770-5-202311-I!!PDF-E.pdf
- Wikipedia, "LUFS" (K-weighting, gating, LKFS vs LUFS): https://en.wikipedia.org/wiki/LUFS
- Fora Soft, "Loudness normalization: EBU R128, ITU-R BS.1770, ATSC A/85" (gating blocks, thresholds explained): https://www.forasoft.com/learn/audio-for-video/articles-audio/loudness-normalization-ebu-r128-bs1770-atsc-a85
- Sage Audio, "Mastering for Streaming: Platform Loudness and Normalization Explained" (per-platform LUFS targets, listener volume levels): https://www.sageaudio.com/articles/mastering-for-streaming-platform-loudness-and-normalization-explained
- Audacity Manual, "Fade and Crossfade" (equal-power vs linear crossfade, -3 dB midpoint): https://manual.audacityteam.org/man/fade_and_crossfade.html
- Signalsmith Audio, "A cheap energy-preserving-ish crossfade" (constant-power crossfade math): https://signalsmith-audio.co.uk/writing/2021/cheap-energy-crossfade/
- ReplayGain specification (the store-a-gain-tag idea): https://en.wikipedia.org/wiki/ReplayGain
