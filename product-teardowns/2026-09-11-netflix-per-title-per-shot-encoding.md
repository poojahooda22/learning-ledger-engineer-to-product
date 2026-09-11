# Netflix: per-title and per-shot video encoding (the bitrate ladder, the convex hull, and the Dynamic Optimizer)

Date: 2026-09-11
Product: Netflix
Feature: How Netflix decides, offline and per video, the exact set of quality versions of a movie to store and stream, so a cartoon does not waste 5,800 kbps and a grainy film does not look blocky.

A note on where this sits. The ledger already has three Netflix streaming teardowns and they are all downstream of this one. Adaptive bitrate streaming (2026-06-30) is the player on your TV switching between versions as your wifi wobbles. Open Connect (2026-07-13) is the fridge-sized server in your ISP that holds those versions close to you. This teardown is the step before both: how those versions get made in the first place, and why choosing them well is the single biggest lever Netflix has over its bandwidth bill and your picture quality. It is a pure offline compile-and-optimize pipeline with a perceptual quality target, which is why it maps almost one to one onto Rare.lab.

---

## 1. The user

Meet Anjali. It is 11pm on a Tuesday in Pune. She is on the couch with her phone hotspotting her laptop because the building fiber is down again, so she has maybe 2 Mbps and it dips. She puts on "Roma," the Alfonso Cuaron film, black and white, slow, quiet. Long static shots of a tiled floor. Then, forty minutes in, the beach rescue: waves, a crowd, motion everywhere.

She never thinks about encoding. She thinks two thoughts, and only when they go wrong. One: "why does this look like mud." Two: "why did it drop to potato resolution when nothing is even happening on screen." Both of those thoughts are decided months earlier, by a pipeline she will never see, in a datacenter, before she ever pressed play.

The other user here is Netflix itself, specifically the finance and infrastructure side. Netflix streams to hundreds of millions of members. Video is the overwhelming majority of internet traffic it pushes. Every percent of bitrate it can shave at the same picture quality is a direct, permanent cut to CDN and storage cost and a direct improvement for every Anjali on a weak connection. A few percent is real money at that scale, and it compounds forever because the encode is done once and served billions of times.

## 2. The real problem

Told like a friend would tell it: for years, every movie on Netflix got the same menu of quality versions, no matter what the movie was.

That menu is called the bitrate ladder. It was a fixed list. The public numbers from Netflix's own 2015 write-up: the bottom rung was 235 kbps at 320x240, and the top rung was 5,800 kbps at 1920x1080, with about a dozen rungs in between. Same ladder for a Pixar-flat cartoon and for a noisy, grainy war film.

That one-size-fits-all ladder is wrong in both directions at once.

For a simple cartoon like a flat-shaded animated show, 5,800 kbps is wild overkill. The picture is flat color regions. It would look perfect at a third of that. So Netflix was shipping huge files and eating huge bandwidth to deliver quality nobody could see. Worse for the viewer: because the fixed ladder said "1080p only lives at 5,800 kbps," a member throttled to 1,750 kbps got served standard definition for a cartoon that would have looked flawless in full HD at that same 1,750 kbps. The ladder cost her resolution she could have had for free.

For a hard title, grainy film or heavy camera noise, the same 5,800 kbps top rung was not enough. The noisy regions still showed blocking artifacts. The ladder promised "this is our best" and the best was not good enough for that content.

And here is the deeper problem that the fix in 2018 attacked: even within one movie, difficulty is not constant. "Roma" is the perfect example. The static shot of a tiled floor and the churning beach rescue are in the same film, but one of them is nearly free to encode and the other is expensive. A single number for the whole title is still an average that overpays for the floor and underpays for the beach.

So the problem is: content complexity varies enormously, across titles and even across shots inside one title, and a fixed ladder ignores all of it. You are simultaneously wasting bandwidth on easy content and starving quality on hard content.

## 3. The feature in one sentence

Instead of one fixed bitrate ladder for everything, Netflix runs an expensive offline analysis on each video (later, on each individual shot within the video) to compute the smallest set of resolution-and-bitrate versions that hit a target perceived quality, so easy content ships small and hard content gets the bits it needs.

## 4. Jobs to be done

What is the viewer really hiring this feature to do:

- "Give me the best-looking picture my connection can carry, right now, without me touching anything."
- "Do not make me watch standard definition when full HD would fit in the same pipe."
- "Do not fall apart into blocky mush on the hard scenes."

What is Netflix the company hiring it to do:

- "Cut our bandwidth and storage bill without lowering quality, permanently, because we pay the encode cost once and the delivery cost every single stream."
- "Let more members on weak or expensive connections (a metered mobile plan in a small town) watch in better quality, which widens the market."

## 5. How it works for the user

Nothing visible. That is the whole point, and it is the same invisible-craft pattern this ledger keeps finding (Spotify loudness normalization on 2026-08-28, Stripe rate limiting on 2026-09-01): the feature succeeds by never being noticed.

Anjali presses play on "Roma." Her player asks Netflix for the manifest, a small file that lists the available versions of this specific title (the tailored ladder). The player measures her bandwidth, picks the highest rung that fits, and starts pulling chunks. As her hotspot dips, the player steps down a rung; as it recovers, it steps up. That switching is the adaptive bitrate story from 2026-06-30. What is new here is that the rungs on the menu were chosen for "Roma" specifically, not handed down from a fixed list.

The visible result, if you could measure it: on a cartoon, she gets HD at a bandwidth where the old ladder would have given her SD. On "Roma," the bits are concentrated where the picture is actually hard, so the beach scene holds together instead of blocking up, and no bits are wasted on the still shots.

## 6. The actual flow, step by step

Two flows. The offline one (the encode, done once, months before anyone watches) is where all the intelligence lives. The online one (the stream, done every play) is deliberately dumb and cheap.

Offline, per title, this is the per-title method (2015):

1. A new title arrives as a high-quality master (the source).
2. Netflix runs a batch of test encodes. It takes the source and encodes it at many combinations of resolution and quality setting. Concretely: a grid of several resolutions (for example 1920x1080, 1280x720, 960x540, 640x360) crossed with a range of quality settings (the encoder's QP, the quantization knob, from high quality to low). That is dozens to hundreds of trial encodes for one title.
3. For each trial encode, measure two numbers: the bitrate it came out at, and the picture quality. Early on quality was measured with PSNR; Netflix later switched the objective to VMAF, its own perceptual metric (section 7).
4. Plot every trial as a dot: bitrate on one axis, quality on the other. Each resolution traces its own curve. Low resolutions win at low bitrates (a 540p encode looks better than a starved 1080p encode when bits are scarce); high resolutions win once there are enough bits.
5. Trace the upper-left envelope over all those curves. That envelope is the convex hull: for every bitrate, the single resolution-and-setting that gives the most quality. Everything below the hull is a version that is strictly worse than something else at the same bitrate, so it is thrown away.
6. Pick the ladder rungs off that hull. That tailored ladder is what goes into the manifest for this title.

Offline, per shot, this is the Dynamic Optimizer (2018), the big upgrade:

1. Cut the title into shots. A shot is a continuous run of frames between two hard cuts (the camera cut from the floor to the beach in "Roma" is a shot boundary). Detect these boundaries automatically.
2. Run the same grid of trial encodes, but per shot. Every shot gets its own set of resolution-and-QP trials and its own convex hull.
3. Now the clever part: stitch one operating point from each shot's hull into a single encode for the whole movie, under one global bandwidth budget. This is a trellis optimization solved by the constant-slope rule (section 7).
4. The output is a stream where the still floor shot rides a cheap operating point and the beach shot rides an expensive one, and the two trade bits so the whole movie hits a target average quality for the fewest total bytes.

Online, per play, unchanged and cheap:

1. Player fetches the manifest (the tailored ladder).
2. Player estimates bandwidth, picks a rung, pulls chunks, switches rungs as the network moves.

All the expensive thinking is offline. The live path is a lookup and a comparison. This is the offline-think / online-lookup spine that runs through Discover Weekly (2026-06-13), YouTube (2026-06-22), and search ranking (2026-06-23): do the costly optimization ahead of time, serve a prebuilt answer live.

## 7. Under the hood, like the engineer

This is the heart. Three engineering ideas stacked: the convex hull, the perceptual metric (VMAF), and the constant-slope trellis. Then the scale story.

### 7a. The convex hull, and why that is the right structure

Start with the core question for one title: out of a huge space of possible encodes, which handful do we keep.

Think of every possible encode as a point in a 2D plane. X axis is bitrate (bytes per second, the cost). Y axis is quality (the benefit). You run trial encodes and get a cloud of points, grouped into curves, one curve per resolution.

Two facts shape the cloud. First, within one resolution, spending more bits buys more quality, but with diminishing returns: the curve rises then flattens. A 1080p encode at 8,000 kbps is barely better than at 6,000 kbps, because the source only has so much detail. Second, resolution curves cross. At a low bitrate, a 540p encode beats a 1080p encode, because 1080p starved of bits spends them all just drawing blocky macroblocks, while 540p at the same bitrate is clean and the player upscales it. At a high bitrate, 1080p pulls ahead because now there are enough bits to actually resolve the extra pixels.

The set of points you want is the upper-left frontier of this cloud: for each bitrate, the highest quality achievable by any resolution-and-setting. That frontier is a convex hull (specifically its upper-left boundary, the Pareto frontier of quality versus bitrate). Any encode not on the hull is dominated: there exists another encode with equal or higher quality at equal or lower bitrate, so keeping it is pure waste.

Why a convex hull and not just "sort by bitrate and pick some": because the hull is exactly the set of non-dominated tradeoffs, and it is convex because of those diminishing returns (the quality-versus-bitrate function is concave for real encoders in the region that matters). Convexity is not a cosmetic detail. It is what makes the per-shot stitching in 7c solvable with a simple slope rule instead of an exponential search.

Concrete example, "Roma." For the still tiled-floor shot, the hull is boring: quality maxes out at a very low bitrate, because there is almost nothing to encode, so even the top rung on that shot's hull is cheap. For the beach rescue shot, the hull climbs slowly and keeps climbing well past 5,800 kbps, because the water and crowd carry real detail and motion. Same movie, two completely different hulls. The old fixed ladder pretended they were the same.

Data structures in play: this is not a fancy tree or graph problem. It is a set of (bitrate, quality) points per resolution, and computing the upper convex hull of a set of 2D points is a classic O(n log n) sort-and-scan (Andrew's monotone chain, Graham scan). The heavy cost is not the hull computation. It is generating the points, because every point is a real video encode of real frames. The algorithmic cleverness is cheap; the compute is in the trial encodes.

### 7b. VMAF: making "quality" a number a machine can optimize

You cannot optimize toward "looks good" until "looks good" is a number. The naive number is PSNR, which measures pixel-by-pixel error against the source. PSNR is easy and Netflix used it first, but it disagrees with human eyes: it punishes film grain and camera noise (which humans read as texture, not as damage) and it under-punishes blocking in smooth areas (which humans hate). Optimizing PSNR can make choices a viewer would call worse.

So in 2016 Netflix open-sourced VMAF, Video Multimethod Assessment Fusion (BSD licensed, shipped as the C library libvmaf and as an FFmpeg filter, and it later won a Technology and Engineering Emmy). VMAF is a learned perceptual quality metric. It works like this:

- Take the reference (the pristine source) and the distorted encode.
- Extract a few elementary features that each capture one axis of human-perceived quality: VIF (visual information fidelity, at multiple scales), DLM / ADM (detail loss measure, how much fine detail the encode destroyed), and a motion / temporal feature (how much the frame is changing, because the eye tolerates more error in fast motion).
- Fuse those features into one score with a machine-learning model, a support vector machine regressor, trained on subjective scores. Netflix showed real clips to human panels, collected their opinion scores (DMOS), and fit the model so VMAF predicts what humans said.
- Output a single number from 0 to 100, where roughly 100 means "indistinguishable from the reference" on a standard viewing setup.

The reason VMAF matters to this teardown: it is the objective function. The Dynamic Optimizer literally chooses encode settings to maximize VMAF at a given bitrate. That is a big claim to sit on, so note the honest caveat: VMAF is a model of human opinion, trained on specific content and viewing conditions, and it can be gamed (encoders that tune for VMAF can score well while a human sees something slightly off). Netflix knows this and keeps retraining and adding datasets. But having any consistent, machine-computable perceptual number is what turned encoding from an art into an optimization problem. Before VMAF you argued about picture quality; after it you could put it on the Y axis and take a convex hull of it.

Concrete example: on the grainy "Roma" beach shot, PSNR would scream at the film grain and push the encoder to smear it away to reduce pixel error, which a human would see as plasticky and worse. VMAF, trained on human opinion, rewards keeping the grain-like texture, so the optimizer spends bits to preserve what the eye actually wants.

### 7c. The constant-slope trellis: stitching one point per shot under a global budget

Per-title (7a) gives one hull for the whole movie. Per-shot gives one hull per shot, and now you have a combinatorial problem: for each of, say, 1,000 shots, pick one operating point from its hull, such that the whole movie hits a target average quality for the minimum total bitrate (or maximum quality for a fixed total budget). If each shot has 40 candidate operating points on its hull, brute force is 40 to the power 1,000. Impossible.

This is a classic constrained optimization, and it has a beautiful, cheap solution because every shot's curve is a convex hull. The method Netflix describes is the constant-slope (Lagrangian) rule solved as a trellis:

- On each shot's quality-versus-bitrate hull, every point has a local slope: how much quality you buy per extra bit right there. On the cheap end of the hull the slope is steep (bits are very productive); on the expensive end the slope is shallow (bits barely help, diminishing returns).
- The optimal global solution picks, on every shot, the point where the slope equals one single shared value, call it lambda, the same lambda for every shot in the movie.

The intuition is an economics argument, and it is worth saying plainly. Lambda is the exchange rate of "quality per bit" for the whole movie. If one shot were sitting at a steeper slope than another (its bits are buying more quality per bit than the other shot's), you could move bits from the shallow-slope shot to the steep-slope shot and gain quality for free at the same total bitrate. You keep doing that until every shot is at the same slope. At that point no reallocation helps, so you are optimal. That single shared slope is exactly why the still floor shot ends up cheap (its hull goes flat almost immediately, so matching the shared slope lands on a low-bitrate point) and the beach shot ends up expensive (its hull is still climbing at that slope, so matching it lands on a high-bitrate point). The bits flow to where the eye gets the most for them.

Mechanically you sweep lambda. Pick a lambda, and for each shot independently pick the point on its hull with that slope (a fast local choice, no coordination between shots), then add up total bitrate and total quality. High lambda gives a cheap, low-quality whole-movie encode; low lambda gives an expensive, high-quality one. Sweeping lambda traces out the movie's overall quality-versus-bitrate curve, and you read off the lambda that hits your target. Netflix frames the stitching as building end-to-end paths in a trellis (shots are the stages, hull points are the states) under the constant-slope principle, and reports it comes within about 1 percent of the true optimum at a fraction of the brute-force cost.

Reported payoff, at equal VMAF, per-shot Dynamic Optimizer versus fixed-quality encoding: about 28 percent bitrate reduction for x264 (H.264), about 34 percent for x265 (HEVC), and about 38 percent for VP9. Read that again in Anjali terms: roughly a third fewer bytes for the same measured picture quality, forever, on every stream of that title. On a service where video is the dominant cost, a third is enormous.

Inference, clearly labeled: Netflix has not published the exact grid size (how many resolutions and QP steps per shot), the exact shot-boundary detector, or the exact lambda-search implementation in production. The shapes above (a resolution-by-QP grid, automatic hard-cut detection, a Lagrangian slope sweep over convex hulls) are what its papers and blog posts describe and are the standard way this class of rate-distortion-optimization problem is solved in the video-coding literature. The specific 28/34/38 numbers and the "within 1 percent" and "constant-slope trellis" framing are Netflix's own published figures.

### 7d. The scale story: 1,000, 100,000, 10 million-plus

The thing that grows here is not a catalog you search. It is the number of encodes you must produce and store. Watch it explode across three tiers.

Tier 1, a small library, roughly 1,000 titles. One fixed ladder for everything is genuinely fine here, and even per-title is cheap. A dozen trial encodes per title times 1,000 titles is a weekend of compute. Storage of a dozen renditions per title is trivial. At this tier the Dynamic Optimizer would be over-engineering: the compute to run per-shot analysis on every title costs more than the bandwidth it saves, because your bandwidth bill is small. A startup streamer should ship the fixed ladder and go home. Nothing breaks.

Tier 2, a real catalog with real traffic, on the order of 100,000 encodes. Now per-title earns its place, because bandwidth is now your biggest bill and a 20 percent cut pays for the encode farm many times over. But the per-title method needs a burst of trial encodes per title, and doing that on one machine is hopeless. What breaks is compute throughput. The fix is the cloud burst: Netflix chops each encode into chunks and encodes chunks in parallel across a large fleet (its parallel, chunked encoding pipeline), so a two-hour movie is not encoded start to finish on one core; it is sliced and fanned out. Storage also grows, because each title now carries a custom set of renditions rather than a shared template, but it is still linear in titles.

Tier 3, the real Netflix, 10 million-plus encode units and climbing. This is where per-shot bites hard, and it bites on the production side, not the viewer side. Multiply it out. A title is not one encode. It is (many shots, hundreds to a couple thousand per film) times (a grid of resolutions and QP settings per shot, dozens of trial encodes) times (multiple codecs: H.264 for old devices, HEVC and VP9, AV1 for new ones) times (HDR and SDR variants) times (audio and subtitle and language packaging). Per-shot took the count of encode units and multiplied it by roughly the number of shots. Netflix has said plainly that moving to the Dynamic Optimizer forced it to retrofit its parallel encoding pipeline to process dramatically more encode units than before. What breaks at this tier is the encoding pipeline's ability to schedule, run, track, and store an explosion of tiny encode jobs. The survival kit:

- Massive parallelism. Every trial encode of every shot at every setting is independent, so the whole thing is embarrassingly parallel and fans out across a huge cloud fleet. This is the same "the work shards itself so parallelism is free" property the ledger found in Notion-by-workspace (2026-06-25) and Stripe-by-account (2026-09-01), applied to encode jobs instead of user requests.
- Prioritization. You do not give every title the full per-shot treatment on day one. The most-watched titles, where saved bytes multiply across the most streams, get the expensive optimization first; a long-tail documentary nobody streams can wait or ride a cheaper method. The optimization budget follows the viewership, because the payoff is (bytes saved per stream) times (number of streams).
- The counterweight the whole system optimizes against is storage and CDN cost. More renditions and more codecs mean more files sitting on Open Connect boxes worldwide. The per-shot savings on bandwidth have to beat the extra storage and extra encode compute, and at Netflix's stream volume they do, easily, because the encode and storage cost is paid once and the bandwidth saving is collected on every one of billions of plays.

And notice what does not change across all three tiers: the viewer's live path. Whether the ladder came from a fixed template, a per-title hull, or a per-shot trellis, Anjali's player still just fetches a manifest and picks a rung. All the tier-3 pain is offline. The online path stayed a cheap lookup. That is the discipline: push the expensive, exploding work off the live path and spread it across a parallel farm ahead of time.

## 8. The retention and habit mechanic

This feature does not have a daily loop like a home feed. Its retention mechanic is trust through the absence of a bad moment, and it moves two metrics at once.

The metric it moves most directly is retention through quality of experience, and the sharp version of that is the play that does not fall apart. Netflix has published for years that streaming quality, specifically rebuffering and picture quality on constrained connections, correlates with engagement and retention. The mechanism: the head-turn moment where a viewer half-watching sees the picture drop to a blocky mush or slam down to potato resolution is exactly where a fragile session dies (the same fragility Spotify's loudness normalization and shuffle protect against). Per-title and per-shot encoding deletes those moments on the margin. A member on a weak connection who would have been pushed to ugly SD now gets a clean HD-ish stream at the same bandwidth, so she keeps watching, so she keeps her subscription. Nobody ever thinks "thank you, convex hull." They think "Netflix just always looks good, even on my bad wifi," and that reflex is the retention.

The second metric is revenue, indirectly but hugely, through cost and reach. A third fewer bytes at the same quality is a third off the marginal cost of the most expensive thing Netflix does. Lower delivery cost per stream widens the margin on every existing member and, more strategically, makes it viable to serve members on slow or metered or expensive mobile connections in price-sensitive markets. Better quality inside a small data budget is directly what lets Netflix grow in places where bandwidth is the binding constraint. The encoding team's work shows up on the finance statements and in the addressable market, not just in a quality dashboard.

Real observed example of the loop closing: Netflix's move to mobile-optimized and per-shot AV1 encodes (announced through 2019 and 2020) was pitched explicitly around members on cellular data, giving them watchable video inside a tight data cap and letting them download more per gigabyte. The retention loop there is concrete: a member who can actually finish a show on her mobile plan without blowing her data budget is a member who renews.

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor that compiles visual effects to shippable code, plus an embeddable runtime. Netflix's encoding pipeline is, structurally, exactly what Rare.lab's compiler should be: an expensive offline optimizer that turns one authored source into the smallest set of shippable artifacts that hit a perceptual quality target, so the runtime path stays a cheap lookup. Four concrete lessons, biased to scalability and performance.

1. Do not ship one fixed "quality ladder" for every effect. Netflix's whole origin story is that a fixed ladder overpays for easy content and starves hard content. The Rare.lab equivalent: do not compile every shader graph to one fixed set of quality or LOD (level-of-detail) tiers. A flat two-node color effect and a 300-node volumetric fog have wildly different complexity, exactly like a cartoon versus "Roma." Compile a per-effect ladder: analyze each graph's cost and emit the LOD tiers and precision variants that actually make sense for it, so a cheap effect does not carry the machinery of an expensive one and an expensive effect gets the tiers it needs to degrade gracefully on a weak GPU.

2. Go per-shot, meaning per-region and per-frame-budget, not per-effect-average. The single biggest jump in this teardown was per-title to per-shot: complexity varies inside one title, so one number for the whole thing is an average that wastes bits on the easy parts. Inside one Rare.lab effect, cost varies by region and by frame: the still background is the tiled floor, the particle burst is the beach rescue. A runtime that renders every part at a uniform quality is the fixed ladder. Allocate the frame's GPU budget across regions and passes the way the Dynamic Optimizer allocates bits across shots: measure each part's quality-per-millisecond curve and spend the frame budget where the eye gets the most, using the constant-slope rule (bring every pass to the same quality-per-ms slope, move budget from shallow-slope passes to steep-slope ones until they equalize). That is dynamic resolution and adaptive quality done right, as a budget allocation over convex hulls, not as a global on/off toggle.

3. Optimize toward a perceptual metric, not a pixel-error metric. VMAF is the load-bearing idea: Netflix could not optimize until "looks good" was a machine-computable number that matched human eyes, and PSNR (raw pixel error) actively made wrong choices (smearing grain the eye wanted). Rare.lab should bake a perceptual quality metric into the compiler so it can auto-tune LOD, shader precision (fp16 vs fp32), sample counts, and texture resolution against "does a human see the difference" rather than "is the pixel identical." Naive pixel-diff will happily throw away dithering, grain, and blue-noise texture a human wants (the exact Spotify-shuffle blue-noise lesson from 2026-09-05), and will waste budget matching pixels nobody can distinguish. If you cannot ship a full learned metric, even a cheap perceptual proxy (SSIM-class, or a small learned model on your own effect corpus) beats pixel error, and it is what lets the compiler make the aggressive cuts that keep frame time down.

4. Keep the expensive optimization offline and the runtime a flat lookup, and make the offline farm shard-and-prioritize. The entire pipeline works because the convex-hull search and trellis run once, offline, on a massive parallel farm, and the player just reads a manifest and picks a rung. Rare.lab's compile (graph to shippable artifact plus its LOD ladder) is the offline encode; the embeddable runtime must never re-derive that at 60fps, it should read the precomputed ladder and pick a tier for the current GPU and frame budget, a lookup, not a solve. And when the effect library gets large (thousands of published effects times device targets times quality tiers), copy Netflix's tier-3 survival kit: every compile job is independent so fan it out in parallel, and prioritize the compile budget by usage (the most-embedded effects get the full per-region optimization first, the long tail rides a cheaper default), because the payoff is bytes-or-milliseconds saved per play times number of plays, exactly Netflix's math.

One line: Netflix's encoding pipeline is a per-title then per-shot offline optimizer that spends huge compute to build the smallest set of quality versions hitting a perceptual (VMAF) target via convex hulls and a constant-slope trellis, cutting about a third of the bytes at equal quality forever, so Rare.lab should make its compiler the same thing (a perceptually-scored, per-region budget optimizer that runs offline and sharded) and keep the runtime a flat ladder lookup.

---

## Sources

- Netflix Technology Blog, "Per-Title Encode Optimization" (2015), Anne Aaron, Jan De Cock, David Ronca, et al. https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2
- Netflix Technology Blog, "Dynamic Optimizer, a perceptual video encoding optimization framework" (March 2018). https://netflixtechblog.com/dynamic-optimizer-a-perceptual-video-encoding-optimization-framework-e19f1e3a277f
- Netflix Technology Blog, "Optimized shot-based encodes: Now Streaming!" (2018). https://medium.com/netflix-techblog/optimized-shot-based-encodes-now-streaming-4b9464204830
- I. Katsavounidis and L. Guo, "Video codec comparison using the dynamic optimizer framework" (SPIE, 2018). https://www.spiedigitallibrary.org/conference-proceedings-of-spie/10752/107520Q/Video-codec-comparison-using-the-dynamic-optimizer-framework/10.1117/12.2322118.short
- Netflix Technology Blog, "Toward A Practical Perceptual Video Quality Metric" (VMAF, 2016). https://netflixtechblog.com/toward-a-practical-perceptual-video-quality-metric-653f208b9652
- Netflix/vmaf, open-source VMAF library. https://github.com/Netflix/vmaf
- Wikipedia, "Video Multimethod Assessment Fusion." https://en.wikipedia.org/wiki/Video_Multimethod_Assessment_Fusion
- Jan Ozer, "How Netflix Pioneered Per-Title Video Encoding Optimization," Streaming Media / Streaming Learning Center. https://streaminglearningcenter.com/encoding/how-netflix-pioneered-per-title-video-encoding-optimization.html
- Fora Soft Learn, "Building a Bitrate Ladder: Classic Netflix Ladder, Per-Title, Per-Shot." https://www.forasoft.com/learn/video-streaming/articles-streaming/bitrate-ladder-per-title-per-shot
- Hackaday, "Decoding The Netflix Announcement: Explaining Optimized Shot-Based Encoding For 4K" (2020). https://hackaday.com/2020/09/16/decoding-the-netflix-announcement-explaining-optimized-shot-based-encoding-for-4k/
