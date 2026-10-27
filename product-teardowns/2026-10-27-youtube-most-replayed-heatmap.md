# YouTube, the "Most replayed" heatmap

Date: 2026-10-27
Product: YouTube
Feature: The "Most replayed" graph, the little gray mountain range that appears above the
red seek bar when you scrub a video, with the tallest peak labeled "Most replayed."

A note on scope so this does not collide with earlier YouTube teardowns. On 2026-06-22 I
tore down home-feed and watch-next recommendations (which video to show you next). On
2026-08-22 I did the view count (the single number under the video and why it freezes at
301). On 2026-10-02 I did Search (the results page). On 2026-10-11 I did Shorts (the
vertical swipe feed). This is none of those. This is one tiny widget that lives inside the
player of a single already-open video: the aggregated rewatch curve drawn on the scrubber.
It is not ranking, not counting lifetime views, not search. It is a different machine: take
the per-second playback behavior of every person who ever watched this one video, squash it
into about one hundred numbers between 0 and 1, and draw a curve. It earns its own day
because it is the cleanest possible example of the pattern this whole ledger keeps circling:
do the expensive aggregation offline, serve the live path as a tiny fixed-size lookup.

---

## 1. The user, in their day

It is 11:40pm. Rohan is lying in bed with his phone, watching a 22-minute YouTube video
titled "How to fix a slow Postgres query." He does not want the whole 22 minutes. He wants
the one part where the person actually runs EXPLAIN ANALYZE and reads the plan. He starts
dragging his thumb along the seek bar, and as he drags, a gray graph appears above the bar.
It is mostly flat, then there is one clear hill about two thirds of the way in. A small
thumbnail preview over that hill says "Most replayed." Rohan lifts his thumb right there.
The video is now at 14:30, exactly on the EXPLAIN ANALYZE walkthrough. He watched for two
minutes and closed the app, question answered.

That is the whole moment. A tired person, a long video, and one curve that answers the only
question that matters at 11:40pm: where is the good part. He did not read the description,
did not scroll comments for a timestamp, did not guess. The crowd that watched before him
left a trail, and he followed it.

---

## 2. The real problem, like a friend would say it

Long videos bury the good part. You click a 22-minute tutorial for a 90-second answer, or a
4-minute music video for one 20-second drop, or a 40-minute podcast for one story someone
quoted to you. The old ways of finding the part you want are all bad. Scrubbing blind means
you overshoot and undershoot, dragging back and forth like you are tuning an old radio.
Reading the comments for "the good bit is at 14:30" works only if some kind stranger typed
it. Chapters help only if the creator bothered to add them, and even then a chapter is a
whole section, not the one moment.

The deeper pain is that you are alone with the timeline. Thousands of people watched this
exact video before you and most of them had the same instinct you do, to skip to the part
that matters. All of that knowledge evaporated. The problem is not "play the video." The
problem is "tell me what everyone who came before me already figured out about where this
video is worth watching," and show it to me in the half second I spend dragging my thumb.

---

## 3. The feature in one sentence

"Most replayed" draws a crowd-sourced heatmap over the seek bar where the height at each
point means how often viewers rewatched that exact moment, so you can jump straight to the
part people kept going back to.

---

## 4. Jobs to be done

What Rohan is really hiring this little curve to do:

- "Get me to the useful part without watching the boring parts." Cut 22 minutes down to the
  two that matter.
- "Tell me if this long video is even worth it." A video with one sharp spike and flat
  everywhere else is often one good moment padded with filler. A video with a broad, high
  plateau is good most of the way through. The shape is a quality signal before he commits.
- "Confirm I am in the right spot." When the peak sits exactly where he expected the answer,
  that is social proof that he guessed right.
- "Find the moment someone told me about." A friend says "watch the part where the drop
  hits." The peak on a music video sits almost exactly on the chorus or the drop, so he does
  not even need the timestamp.

And for the creator, a second, quieter job: "Tell me where my audience actually leans in and
where they bail," which is the same raw data seen from the other side (YouTube Studio's
audience retention report).

---

## 5. How it works for the user, the visible experience

On desktop you hover over the progress bar and a gray graph fades in above it, following
your cursor with a thumbnail preview. On mobile you press and drag the seek bar and the
same graph appears. The curve is normalized: the tallest point is always drawn at full
height no matter whether this video got a thousand rewatches there or a million, because the
curve shows the shape of attention within this one video, not an absolute count. The single
highest peak gets a label, "Most replayed," and the preview thumbnail snaps to it so you can
release your finger right on it.

It appears only on videos with enough watch data. A video posted an hour ago with 40 views
has no graph. A video with a few hundred thousand views almost always does. It is a
long-form player feature: Shorts, which have their own vertical player and no scrubbable
seek bar in the same way, do not show it (CONFIRMED by the product behavior and reporting).

Concrete shape examples you can go verify right now:
- On most official music videos the "Most replayed" peak sits on the chorus or the beat drop,
  because that is the part people loop. The verses are the valleys.
- On a tutorial the peak sits on the one step people kept scrubbing back to redo, like the
  exact command being typed.
- On a long interview the peaks cluster on the three or four quotable moments and the intro
  small-talk is a flat valley.

---

## 6. The actual flow, tap by tap

1. Rohan opens the 22-minute Postgres video. The player loads. Behind the scenes the page
   response already carried a small block of data describing the heatmap (more on its exact
   shape below). No extra network round trip is needed when he starts to scrub; it was
   bundled with the player data.
2. He puts his thumb on the seek bar and starts dragging. The player's UI layer reads the
   heatmap data it already has and renders it as an SVG curve above the bar.
3. As his thumb moves, a preview thumbnail follows. When it crosses the tallest peak, the
   label "Most replayed" shows.
4. He lifts his thumb at the peak. The player seeks to that timestamp (around 14:30) and
   resumes playback from there.
5. Nothing he just did changed the heatmap. His own scrubbing is a client-side read. If his
   watch session later gets logged and aggregated, it will nudge this curve for the next
   person, but not in this moment and not visibly.

The key thing in the flow: step 2 touches zero servers for the graph. The curve is a tiny
precomputed artifact that rode along with the player. That is the entire performance trick,
and section 7 is about why it has to be that way.

---

## 7. Under the hood, like the engineer

This is the heart of the report. The feature looks like "draw a line." It is really three
separate problems stacked on top of each other: what gets measured, how billions of raw
events become one hundred numbers, and how those numbers reach the phone for free. Matching
and ranking are the two halves of a search feature; here the two halves are **aggregate**
(the expensive offline job) and **serve** (the cheap online lookup). Keep them separate in
your head and the whole thing makes sense.

### 7a. What is actually public, and what is inference

Let me be strict about fact versus inference, because the framework demands it and because
YouTube has never published the internal pipeline.

CONFIRMED (from YouTube's own product and from tooling that reads the public response):
- The feature launched to all users on 18 May 2022, after a period as a YouTube Premium
  experiment at youtube.com/new, and was previewed in late 2021 (TechCrunch, 9to5Google,
  MobileSyrup, all 18 May 2022).
- YouTube's own description: the graph is drawn over the seek bar and "if the graph is high,
  it means that part of the video has been frequently re-watched."
- The data that drives the client graph is delivered in the player response as a structured
  object. Open-source tooling that parses YouTube's internal API ("InnerTube") reads it at a
  known path: `playerOverlays` then `decoratedPlayerBarRenderer` then
  `multiMarkersPlayerBarRenderer` then `markersMap`, where one entry has the key `heatmap`
  and holds a `heatmapRenderer` with a list of `heatMarkers`. Each marker is a
  `heatMarkerRenderer` carrying three fields: `timeRangeStartMillis` (where this segment
  starts), `markerDurationMillis` (how long the segment is), and
  `heatMarkerIntensityScoreNormalized` (a float from 0 to 1). This is confirmed by the
  `yt-dlp` heatmap extractor and by the `youtubei.js` library's `HeatMarker` class.
- The number of markers is about 100 per video. The `yt-dlp` test fixture asserts
  `heatmap: count:100`. So a 248-second video gets segments roughly 2.48 seconds long, and a
  25-minute video gets segments roughly 15 seconds long. The segment count is fixed, the
  segment duration scales with video length.
- In the browser the curve is painted as an SVG element (`svg.ytp-heat-map-svg`, drawn in a
  1000-by-100 coordinate box) whose shape is a path made of cubic Bezier curves. The Bezier
  curve is the smoothing: 100 discrete points become one soft continuous line.
- The creator side of the same data lives in YouTube Studio's audience retention report,
  which shows absolute retention (what fraction is still watching at each moment) and
  relative retention (how this video holds attention versus similar videos), and where rewind
  spikes can push a moment's "watching" figure above 100 percent because people watched it
  more than once.

INFERENCE (clearly labeled, grounded in how this class of problem is always solved):
- The raw input is per-session playback telemetry. As you watch, the client periodically
  reports its playhead position and state (playing, paused, seeked). YouTube already collects
  this for analytics and for the retention report, so the heatmap is a second product built
  on an input that already exists.
- The aggregation buckets that telemetry into the fixed number of normalized time bins per
  video, counts how many times viewers were present or re-entered each bin, and divides every
  bin by the maximum bin so the peak is always 1.0. That last step is why the public field is
  called `IntensityScoreNormalized` and never a raw count, and why the tallest point is always
  full height.
- This runs as a batch job over watch logs, not live per request, and the result is cached and
  attached to the player response. The threshold ("enough views") exists because with few
  viewers the curve is noise: one person scrubbing twice would create a fake mountain.

I will keep the inference label attached whenever I lean on it below.

### 7b. The data structure in play, and why that one

The served artifact is the simplest useful structure there is: a **fixed-length array of about
100 floats in [0, 1]**, each tagged with a start time and a duration. For the Postgres video
it is roughly `[0.08, 0.05, 0.04, ... 0.31, 1.00, 0.44, ...]` where that `1.00` is the
EXPLAIN ANALYZE moment at bin 59 (around 14:30 of 22 minutes). That is the whole thing.

Why an array and not something cleverer:
- The length is fixed at about 100 no matter if the video is 30 seconds or 3 hours. That is a
  deliberate choice. A pixel-perfect curve at one point per second would make a 3-hour video
  carry 10,800 numbers and a 30-second clip carry 30, so the payload would swing wildly and
  the client rendering code would need to handle both. One hundred bins means the artifact is
  always the same tiny size, the seek bar always has the same resolution of detail, and the
  SVG path always has about the same number of control points. Fixed size is a feature.
- The client turns the 100 points into a smooth line using cubic Bezier curves (CONFIRMED by
  the SVG path). Bezier smoothing hides the bucket edges so the curve looks continuous even
  though the data is chunky. A real example: the EXPLAIN ANALYZE spike at bin 59 and the
  slightly-lower bin 60 get joined into one rounded hill instead of two sharp bars.

On the aggregation side (INFERENCE), the natural structure is a **histogram keyed by (video_id,
bin_index)**: a counter per time bucket per video. You stream watch events in and increment
the right bucket. This is the same shape as a word-count or a view-by-hour rollup, which is
why it maps so cleanly onto batch data tools. It is not a tree, not a graph, not an inverted
index. The problem has no ranking across videos and no text matching, so it stays a plain
counting-into-buckets job, which is the cheapest thing a data pipeline can do.

### 7c. Where the work happens, aggregate versus serve

The expensive half is offline. Imagine the Postgres video has 900,000 views. Each view is a
stream of playhead samples. To build the curve you (INFERENCE) group all those samples by the
video, drop them into the 100 bins, count per bin, find the max bin, divide everything by it,
and write out 100 floats. That is a big MapReduce-style or stream-aggregation job, and it runs
on a schedule over the day's logs, far away from any user's live request. It can take minutes
and nobody cares, because nobody is waiting on it.

The cheap half is online. When Rohan opens the video, the server attaches the already-computed
100-float array to the player response. When he scrubs, the client reads the array it already
has and paints the SVG. **Zero database work happens during the scrub.** This is the exact same
split the ledger keeps finding: Discover Weekly precomputes the playlist and serves a lookup;
Netflix Top 10 counts the firehose offline and writes ten IDs; here the heatmap aggregates
offline and serves 100 floats. The live path is always made trivial by paying the cost
somewhere the user cannot feel it.

### 7d. The scale story at three tiers

Tier one, about 1,000 views. The aggregation is trivial, you could do it in memory in a
single process. But the feature deliberately does NOT show here. With 1,000 views and far
fewer rewatches, a single enthusiastic viewer who loops one moment five times can create a
peak that means nothing. The curve would be noise dressed as signal. So the real move at this
tier is restraint: cross a data threshold before you publish a curve at all. What breaks at
the next tier is not compute, it is trust, and the fix is the threshold.

Tier two, about 100,000 views. Now the aggregate is statistically stable; the chorus really is
the peak because tens of thousands of people really did rewind to it. The batch job is still
easy, a few million playhead samples is nothing for a cluster. The thing you must get right
here is the read path, not the write path. This video might be on someone's screen a few
hundred thousand times a day, and you cannot recompute the curve on each open. So you compute
once per batch cycle and cache the 100-float array, and every player load reads the cache.
What breaks at the next tier is the sheer volume of raw events flowing into the job and the
number of distinct videos you must keep curves for.

Tier three, 10 million plus views, and in aggregate billions of watch sessions a day across
the whole catalog. You can no longer scan raw events on demand for anything. The raw playhead
telemetry is a firehose, so it (INFERENCE) lands in a partitioned log sharded by video_id, and
the aggregation is a scheduled batch or streaming job that reads each shard and emits that
video's 100 floats. The output is tiny (100 floats per video, call it under a kilobyte) even
though the input is enormous, which is the whole point: the artifact size is decoupled from the
audience size. A video with 10 million views and a video with 10,000 both end up as the same
100-float array. That is what lets the serve path stay a flat, cheap lookup while the input
scales without limit. What would break if you ignored this is obvious: compute the curve live
from raw events for a video on the home page during a traffic spike and you would try to scan
hundreds of millions of rows per page load. The survival move is precompute plus fixed-size
output plus cache, the same three moves every time.

### 7e. One subtle correctness trap

Rewatch is not enjoyment. The curve measures where people went back, and people go back for
two opposite reasons: the part was so good they looped it, or the part was so confusing they
replayed it to understand it. A mumbled word in a lecture can become the "Most replayed" peak
because fifty people rewound to catch it. YouTube's own help text is careful to say "re-watched,"
not "most liked." The engineering is honest about this; it reports a behavior, not a verdict.
Good lesson in not overclaiming what your metric means.

---

## 8. The retention and habit mechanic

The loop is a time-to-value loop, and it feeds the one metric YouTube cares about most: watch
time, specifically satisfied watch time rather than raw minutes.

Here is the mechanic. A long video has a hidden tax: the risk that you click, cannot find the
good part, get bored, and leave. Every abandoned long video is watch time YouTube did not get
and a small ding to how much you trust long videos in the future. "Most replayed" lowers that
tax. It turns "I might waste 20 minutes" into "I can get to the good 2 minutes in one drag,"
so you are more willing to click the long video in the first place, and more likely to come
away feeling the click paid off. A viewer who keeps feeling their clicks pay off keeps clicking.
That is the habit: the feature quietly makes long-form feel low-risk.

It also runs a second loop on the creator side, through the same data. Creators stare at the
audience retention graph in YouTube Studio, see exactly where viewers lean in and where they
bail, and rebuild their videos around it: front-load the hook, cut the slow intro, put the
payoff where the crowd clearly wants it. Better-shaped videos hold attention better, which
produces cleaner heatmaps, which help the next viewer, which lifts watch time again. The
feature improves its own input over time.

A real observed example of the viewer loop: on music videos the "Most replayed" peak reliably
lands on the chorus or the drop. That is not a guess, you can open almost any popular music
video and watch the peak sit on the hook. The consequence is that millions of casual listeners
use YouTube like a jump-to-the-chorus button, which keeps them on YouTube instead of opening a
music app, which is watch time retained inside the product. The metric moved is retention and,
downstream, the ad-supported watch time that is YouTube's revenue.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code plus
an embeddable runtime. The exact move here transfers cleanly, and it is a scalability move, so
it fits the bias.

Build a **frame-cost heatmap over the effect timeline**, aggregated from the embedded runtime,
and serve it to the creator as a precomputed fixed-size array, never computed live.

Concretely. Your embeddable runtime is already running shaders on real devices in real
end-user pages. Have it sample, cheaply and at a low rate, the GPU frame time at each point
along the effect's timeline, plus which node in the graph was the hot one. That telemetry is a
firehose exactly like watch events. Do not touch it on the live path. Offline, bucket it into a
fixed number of time bins per effect (steal the 100-bin choice: fixed size no matter if the
effect is a 2-second transition or a 2-minute ambient loop), take a robust statistic per bin
like the median or the 95th percentile frame time, normalize, and write out one tiny array per
effect per device-class. Then in the editor, draw that array as a heatmap right under the
effect's timeline, with the worst bin labeled the way YouTube labels "Most replayed." The
creator drags along the timeline and sees, in half a second, "frame time spikes right here, on
the blur node, on low-end Android." They jump to that node and optimize the one moment that
actually costs, instead of guessing.

Three reasons this is the right shape for you:
- **Fixed-size artifact decoupled from audience size.** An effect embedded on a page with 10
  million views and one embedded on a page with 10,000 both compress to the same small array.
  Your serve path stays flat and cheap while telemetry scales without limit. That is the whole
  lesson from section 7d, applied to your product.
- **Expensive thinking stays off the hot path.** Profiling every frame on every device in real
  time would wreck the very performance you are selling. Sample low, aggregate offline, serve a
  lookup. Never aggregate on the 60fps path and never recompute the heatmap when the editor
  opens; read the cached array.
- **It improves its own input, like the creator loop in section 8.** Creators optimize the hot
  bin, the effect gets faster, the next heatmap is flatter and cheaper to run, which is exactly
  the outcome you want to sell: effects that stay smooth on low-end hardware.

One honest caveat, borrowed from section 7e: a hot bin is not always a bug, sometimes the
expensive moment is the one that matters and is worth the cost. Report the behavior, let the
creator make the call, and do not auto-"fix" the peak. Measure, show, let a human decide.

---

## Sources

- TechCrunch, "YouTube's player gains new features including Most Replayed, video chapters,
  single loop and more" (18 May 2022):
  https://techcrunch.com/2022/05/18/youtubes-player-gains-new-features-including-most-replayed-video-chapters-single-loop-and-more
- 9to5Google, "YouTube 'Most replayed' rolling out" (18 May 2022):
  https://9to5google.com/2022/05/18/youtube-most-replayed/
- MobileSyrup, "YouTube's 'Most replayed' experimental feature is now rolling out for all users"
  (18 May 2022):
  https://mobilesyrup.com/2022/05/18/youtubes-most-replayed-experimental-feature-is-now-rolling-out-for-all-users/
- iMore, "YouTube is highlighting popular parts of videos with a new 'Most Replayed' feature":
  https://www.imore.com/youtube-highlighting-popular-parts-videos-new-most-replayed-feature
- yt-dlp source (heatmap extractor commit, defines the InnerTube path and the count:100 test):
  https://github.com/yt-dlp/yt-dlp
- youtubei.js, `HeatMarker` class (fields: intensity score normalized, marker duration millis,
  time range start millis): https://ytjs.dev/api/namespaces/YTNodes/classes/HeatMarker
- youtubei.js, `MacroMarkersListEntity` class (marker type "heatmap", audience engagement data):
  https://ytjs.dev/api/namespaces/YTNodes/classes/MacroMarkersListEntity
- vidIQ, "How to See the Most Replayed Parts of a YouTube Video" (feature behavior, creator
  retention data when public graph is hidden, Shorts has no heatmap):
  https://vidiq.com/blog/post/most-replayed-scenes-youtube-video
- Google / YouTube Help, audience retention report (absolute vs relative retention, rewatch
  spikes): https://support.google.com/youtube/answer/9314415
