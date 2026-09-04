# Instagram: home-feed fan-out and delivery (how a post you follow actually reaches your feed)

Date: 2026-09-04
Product: Instagram
Feature: Home-feed delivery. Not the ranking of the feed, and not the storage of the photo. The plumbing in between: when someone you follow posts, how does that post end up in your feed, and how does Instagram assemble your feed in a fraction of a second when you pull to refresh.

A note on scope. Earlier teardowns in this ledger covered Instagram's Stories tray ranking, the Explore recommender, photo storage (Haystack and f4), and Reels ranking. None of them touched this: the delivery problem. Who pushes what, to whom, and when. This is the classic "news feed" or "timeline" problem, and it is one of the most-studied problems in large-scale systems. Instagram publishes less about it than Twitter/X or Facebook do, so parts of this report are clearly-labeled inference built on the way this exact class of problem is known to be solved, grounded in the infrastructure Instagram has confirmed it runs.

---

## 1. The user

It is 3 pm. Priya is a 24-year-old in Pune, standing in a metro queue with four minutes to kill. She opens Instagram. She follows 480 accounts: 40 close friends, a dozen meme pages, some food bloggers, three cricketers, and Cristiano Ronaldo (who has more than 600 million followers). She pulls down to refresh. In under a second a feed appears: her friend Rahul's photo from last night, a reel from a food page, a cricketer's story of a match, and further down a post from Ronaldo that went up an hour ago.

She did nothing to build that feed. She did not ask for those 480 people's latest posts to be gathered, sorted, and rendered. It just appeared. That "it just appeared" is the entire feature.

## 2. The real problem

Think about what Priya's phone is really asking for. "Give me the recent posts from the 480 specific accounts I follow, blended together, freshest and most relevant near the top." Sounds easy. It is not.

The naive way: when Priya opens the app, go to the posts table, run "find all posts where author is in (my 480 followed accounts), sorted by time, limit 50." That single query has to scan the posts of 480 people. Now multiply. Instagram has roughly 2 billion monthly users. On a normal day hundreds of millions of them refresh, many of them several times a minute. If every refresh runs a 480-way lookup and a sort, the database is doing billions of these fan-in queries per second. It falls over. Not slowly. Immediately.

And the reverse is just as bad. When Ronaldo posts one photo, more than 600 million people's feeds should now be able to show it. If posting means "immediately write this post into 600 million inboxes," then one tap by one man triggers 600 million writes and the write path chokes for everyone else on the platform for seconds. This is literally called the celebrity problem, and it is not hypothetical. Twitter engineers described Katy Perry, with around 80 million followers at the time, tweeting and the system attempting roughly 80 million cache writes at once, saturating the write path for several seconds and delaying delivery of everyone else's tweets ([techinterview.org write-up](https://www.techinterview.org/post/3233474168/system-design-twitter-news-feed-timeline-fanout-on-write-fanout-on-read-celebrity-problem-ranking-caching/)).

So the real problem is a squeeze. Do the work when someone posts (write-time), and celebrities kill you. Do the work when someone reads (read-time), and normal refreshes kill you. There is no single answer. The feature is the compromise.

## 3. The feature in one sentence

When you open Instagram, your feed is served almost entirely from a pre-built list of post IDs that was assembled ahead of time by pushing each post into your followers' inboxes at post time, except for the handful of very-high-follower accounts whose posts are pulled in and merged at read time.

## 4. Jobs to be done

- "Show me my people's recent stuff without me hunting for it." Priya is hiring the feed to be an always-fresh digest of 480 accounts she chose.
- "Make it instant." A queue-in-the-metro session dies if the feed spins for three seconds. The job is sub-second.
- "Do not miss my close friends." Rahul posting once a week must not get buried under a meme page that posts 20 times a day.
- "Feel alive every time I open it." If the same feed shows twice, Priya closes the app. The job is freshness on every pull.

Notice the tension baked into the jobs: instant (job 2) fights complete and fresh (jobs 1 and 4). Delivery is where that tension is resolved.

## 5. How it works for the user

Priya sees three moments.

First, posting. Rahul takes a photo, adds a caption, taps Share. To Rahul it is done in a second. He does not see that his post is now being copied into hundreds of inboxes.

Second, the pull-to-refresh. Priya swipes down. A spinner flickers. A feed of cards fills the screen: images, a reel, a carousel she can swipe sideways. She never sees a "loading 480 accounts" message because that is not what happens.

Third, the scroll. She scrolls past 10 cards, and when she nears the bottom, more cards load without a tap. That is pagination: the feed is delivered in pages, not all at once.

What she never sees: that Ronaldo's post reached her by a completely different path than Rahul's post did. Both look like identical cards. Under the glass they were delivered by opposite strategies.

## 6. The actual flow, step by step

Post side (Rahul, 300 followers):
1. Rahul taps Share. His photo is already uploaded to blob storage and has a media ID (that is the Haystack/f4 story from an earlier teardown).
2. A new row is written to the posts store: post ID, author ID (Rahul), media ID, caption, timestamp. This is the one true copy of the post.
3. A fan-out job kicks off. It looks up Rahul's follower list (his 300 followers) and writes Rahul's post ID into each of those 300 followers' feed inboxes. Priya is one of them. Her inbox now has a new entry near the top.
4. Rahul's Share is acknowledged. Steps 3 happens asynchronously in the background, so Rahul does not wait for 300 writes.

Post side (Ronaldo, 600M+ followers):
1. Same as above through step 2. The post exists once.
2. The system checks Ronaldo's follower count. It is far above the fan-out threshold. So it does NOT push to 600 million inboxes. It skips fan-out entirely and just marks the post in Ronaldo's own author timeline.

Read side (Priya pulls to refresh):
1. Priya's app calls the feed service: "give me page 1 of Priya's feed."
2. The feed service reads Priya's pre-built inbox: a list of post IDs already sitting there, freshest first. This is a cheap read of one list. Rahul's post is in it.
3. The feed service also checks: which of the accounts Priya follows are "pull" accounts (celebrities exempted from fan-out)? For Priya that is Ronaldo and the three cricketers. It fetches those few authors' recent posts directly and merges them into the candidate list.
4. Now there is a candidate list of maybe 500 post IDs (pushed inbox entries plus pulled celebrity posts). Ranking runs here, ordering by predicted relevance, not just time. (Ranking is its own teardown; here it is one step in the pipeline.)
5. The top ~10 post IDs are hydrated: fetch each post's caption, author name and avatar, like and comment counts, and the CDN URL for the media. The phone gets a compact page, then requests the actual images from the CDN.
6. Priya scrolls near the bottom. The app asks for page 2 with a cursor (a bookmark saying "continue after this post"). Steps 4 and 5 repeat for the next slice.

The key thing to see: on the read path, Priya's phone never runs a 480-way query. It reads one list that was already built for her, plus a tiny pull for a few celebrities. That is the whole trick.

## 7. Under the hood, like the engineer

This section is the heart of it. I will separate confirmed Instagram infrastructure from the well-grounded inference about how they wire it together for the feed.

### The two data shapes

There are two different "inbox" ideas and it helps to name them.

- The author timeline: the list of posts a single account has made. Ronaldo's author timeline is just "all of Ronaldo's posts, newest first." Cheap to store, one per account.
- The home feed inbox (also called a materialized timeline): a per-user list of post IDs that other people pushed to you. Priya's inbox holds Rahul's latest post ID, the food page's post ID, and so on. This is precomputed. It is the thing that makes reads fast.

The home-feed inbox is a classic use of a precomputed list per user. Think of it as a hash map keyed by user ID, where the value is a time-ordered list (newest first) of post IDs. Reading your feed is then "look up my key, read the top N of my list." That is an O(N) read of a short list, not an O(followees) scan of a giant table.

### Push, pull, and the hybrid

There are two strategies, and Instagram (like Twitter/X and Facebook) uses a blend.

Fan-out on write (push). When you post, copy the post ID into every follower's inbox immediately. Reads are then trivially fast because the inbox is already built. Cost: a post by someone with F followers does F writes. Fine when F is 300. A disaster when F is 600 million.

Fan-out on read (pull). Store the post once. When a reader opens their feed, go gather the recent posts of everyone they follow, right then, and merge. Posting is cheap (one write). Reads are expensive because every refresh does a big gather. Fine for a user who follows 10 people. A disaster at 480 followees times hundreds of millions of readers.

The hybrid resolves it by follower count. Push for the many, pull for the few extreme accounts. Concretely, a threshold on follower count decides the path. Accounts under the threshold (the vast majority: Rahul, the food blogger, your cousin) get fan-out on write. Their posts land in follower inboxes at post time. Accounts above the threshold (Ronaldo, big brands, the cricketers) are exempted from fan-out; their posts are pulled in at read time and merged. This is exactly the Twitter/X pattern described publicly: build the home timeline mostly by fan-out on write, but skip fan-out for high-follower accounts and merge their posts in at read time ([Twitter timeline write-up](https://www.techinterview.org/post/3233474168/system-design-twitter-news-feed-timeline-fanout-on-write-fanout-on-read-celebrity-problem-ranking-caching/), [fan-out trade-offs](https://wittycoder.in/courses/news-feed/fan-out-strategies)).

Why the hybrid is correct and not a hack: the right choice depends on the ratio of a poster's followers (write amplification if you push) to how often those followers actually read. A celebrity with 600M followers, most of whom open the app daily anyway, is far cheaper to serve by pull. A friend with 300 followers is far cheaper to serve by push. The threshold is just the crossover point of those two costs.

Note the merge cost for Priya specifically: she follows only a few celebrities, so pulling their recent posts is a handful of small reads. The pull path stays cheap precisely because you only ever pull for the small number of huge accounts.

### What Instagram confirms it runs, and how that maps here

Instagram has publicly confirmed the ingredients this design needs.

- Cassandra for feed-shaped data. Instagram runs Apache Cassandra heavily and even built a custom RocksDB-backed storage engine for it, called Rocksandra, to cut tail latency. In one production cluster the P99 read latency dropped from about 60 ms to about 20 ms and garbage-collection stalls dropped from 2.5% to 0.3%, roughly a 10x tail-latency win ([Instagram Engineering: Open-sourcing a 10x reduction in Apache Cassandra tail latency](https://instagram-engineering.com/open-sourcing-a-10x-reduction-in-apache-cassandra-tail-latency-d64f86b43589)). Why this matters for feeds: a per-user inbox is a wide-row, append-heavy, read-your-recent-slice workload, which is exactly what a log-structured store like Cassandra/RocksDB is good at. You append new post IDs to the top of a user's row and read the newest slice.
- Memcache with leases in front of everything. Instagram's infrastructure talk (Lisa Guo, "Scaling Instagram Infrastructure," QCon 2017) describes using Memcache and specifically memcache leases to stop the thundering-herd problem: when a hot cached value expires and thousands of requests miss at once, a lease lets one request rebuild the value while the others wait, instead of all of them hammering the database. This is the same lease mechanism from Facebook's "Scaling Memcache at Facebook" (Nishtala et al., NSDI 2013). For feeds this is essential: a popular author's post, or a hot user's feed page, must not send a stampede to the store on every cache miss.
- The social graph lives in a graph-aware cache. Follower and following edges are the input to fan-out (to push a post you must read "who follows me"). At Meta this is TAO, a graph-aware cache over sharded MySQL that stores objects (a user, a post) and associations (follows, likes) with a two-tier leader/follower cache design. The 2013 paper reported about a billion reads per second at a 96.4% follower-cache hit rate, and by 2021 Meta cited over ten billion reads per second ([Meta Engineering: TAO](https://engineering.fb.com/2013/06/25/core-infra/tao-the-power-of-the-graph/), [TAO summary](https://www.micahlerner.com/2021/10/13/tao-facebooks-distributed-data-store-for-the-social-graph.html)). Reading a follower list to fan out is a graph edge-list read, which is what TAO is built to serve cheaply and at massive read volume.

Inference, clearly labeled: Instagram has not published a blow-by-blow of its exact feed fan-out service the way it published Rocksandra. The push/pull hybrid, the follower-count threshold, and the per-user inbox stored in Cassandra are the standard, well-documented solution to this exact problem and fit every confirmed piece of Instagram's stack. Treat the specific wiring as informed reconstruction, not a quoted internal design.

### Where sorting happens

Sorting happens on the server, never on Priya's phone. The phone could not sort 500 candidates by a machine-learned relevance score; it does not have the like counts, the model, or the freshness signals, and doing it on-device would need all candidate data shipped down first, which defeats the point. So the feed service builds the candidate list, ranks it server-side, and sends the phone an already-ordered page of ~10. The phone renders in the order it receives. This is the same lesson as the search teardowns in this ledger: matching (gather candidates) and ranking (order them) are two halves, and both live server-side.

Walk the concrete example. Priya refreshes at 3 pm. Matching gathers her candidates: her inbox (pushed posts from ~470 normal accounts she follows, of which maybe 60 posted in the last day) plus a pull of recent posts from Ronaldo and the three cricketers. That is the candidate set, say 500 post IDs. Ranking scores each: Rahul's post scores high (close friend, recent, she usually likes his posts), the food page scores medium, Ronaldo's hour-old post scores lower for her than her friend's fresh one. The server returns the ordered top 10. Rahul is card 2. Ronaldo is card 9. The phone just draws them.

### The scale story at three tiers

Tier 1: 1,000 posts, a few hundred users (a startup clone). Here you do not need any of this. One Postgres table of posts, and the feed query "select posts where author in (my followees) order by time desc limit 50" runs in milliseconds because the whole thing fits in memory. Pure fan-out on read. No inboxes, no push. Building precomputed inboxes at this size would be wasted engineering.

What breaks going to the next tier: as followees per user grow and reads multiply, that fan-in query gets run millions of times per second and repeatedly scans the same rows. The database CPU saturates on read.

Tier 2: 100,000 users, tens of millions of posts. Now you switch to fan-out on write. Build a per-user inbox (a precomputed list of post IDs). Posting does F writes; reading is a cheap single-list read. Put the inboxes in a fast store and cache hot ones in Memcache. Reads are now fast and cheap because they are pre-materialized. This is the sweet spot where push wins big: most accounts have modest follower counts, so write amplification is tolerable and read latency is excellent.

What breaks going to the next tier: a few accounts get huge. When an account with 5 million followers posts, fan-out on write does 5 million inbox writes for one tap. A handful of such accounts posting together can saturate the write path and delay everyone, the celebrity problem. Also the fan-out work becomes bursty and unfair: one celebrity post can starve normal posts of write capacity.

Tier 3: hundreds of millions to billions of users, celebrities with 100M+ followers (real Instagram). Three survival moves.
1. Go hybrid. Set a follower-count threshold. Below it, push (fan-out on write). Above it, do not push at all; pull those authors' posts at read time and merge. This caps write amplification: no single post ever triggers hundreds of millions of writes.
2. Make fan-out a queue-fed background job, not a synchronous step. When Rahul posts, his Share returns immediately; the fan-out to his followers runs on a worker pool fed by a queue (Instagram runs Celery on RabbitMQ for exactly this style of async task). This absorbs bursts and gives fairness: a celebrity's large fan-out job can be sharded and rate-limited so it does not block the queue for everyone else.
3. Cache and shard the reads. Per-user inboxes are sharded across many Cassandra nodes by user ID, so no single node holds all inboxes. Hot inboxes and hot posts sit in Memcache with leases to prevent thundering herds. The follower-graph reads that drive fan-out come from a graph cache (TAO-style) that serves billions of edge reads per second at a very high hit rate, so "who follows Rahul" is a cache hit, not a MySQL scan.

Each tier's fix is a direct response to what broke at the previous one: read-scan saturation pushes you to materialized inboxes; write amplification from celebrities pushes you to the hybrid plus queues plus sharding.

## 8. The retention and habit mechanic

The delivery design is not neutral about habit. It is tuned to make the feed feel fresh every single time Priya opens it, which is the entire engine of the pull-to-refresh loop.

The loop: open app, pull down, get new cards, feel a small reward (a friend's photo, a like-worthy meme), scroll, repeat. This is a variable-reward loop. The reward is variable because the ranked, blended feed is different on almost every pull. Delivery serves this in two ways. First, fan-out on write means new posts are already waiting in your inbox the instant they are posted, so a refresh 30 seconds after a friend posts already shows it: the feed rewards frequent checking. Second, ranking on top of the merged candidates means the order changes between pulls even when the underlying posts are similar, so it rarely feels stale.

Which metric it moves: retention, specifically session frequency and daily returns. A feed that is instant (sub-second, thanks to pre-built inboxes) and always fresh (thanks to push plus ranking) is a feed you can dip into for 90 seconds in a metro queue, which is what turns Instagram into a many-times-a-day habit rather than a once-a-day visit. The pre-materialized inbox is what makes those micro-sessions viable: if each refresh took three seconds, the metro-queue open would not happen.

Real observed example: Instagram's move away from the strict reverse-chronological feed to a ranked feed (announced 2016) was justified publicly by the claim that people were missing about 70% of the posts in their feed, and that ranking surfaced the ones they cared about. Whatever one thinks of that change, it is a delivery-and-ranking decision aimed squarely at making each open feel worth it, and the company reported higher engagement after it. The chronological-vs-ranked debate that followed (and the later option to switch back to Following/Favorites feeds) is the whole industry arguing about exactly this delivery layer.

## 9. The lesson for Rare.lab

Precompute per-consumer, but exempt the fat tail and compute it on read. That single rule is the transferable win here, and it maps cleanly onto a node-based shader editor that compiles to shippable code plus an embeddable runtime.

Concretely. In a shader graph, most nodes are cheap and their outputs are stable across frames for a given input (a color constant, a UV remap, a texture sample from a fixed atlas). A few nodes are the "celebrities": expensive and widely consumed (a blur that many downstream nodes read, a noise field sampled by dozens of branches, a lighting term feeding the whole graph). The naive compiler treats all nodes the same, exactly like the naive feed treats Rahul and Ronaldo the same, and it either recomputes everything every frame (fan-out on read, the read path chokes) or tries to cache everything (fan-out on write, memory and bandwidth choke).

Do what the feed does. Pick a threshold. For a node whose output is stable and consumed by many downstream nodes (high fan-out), precompute it once and materialize it: bake it to a render target or a lookup texture that all consumers read cheaply, the shader-graph equivalent of a pushed inbox. For a node that is cheap or consumed by only one branch (low fan-out), do not materialize it; inline it and recompute at the consumer, the equivalent of pulling at read time. And for the truly expensive-and-widely-read node, materialize it once per frame on a queue-fed pass (a prepass), never recomputed per consumer, exactly as a celebrity post is pulled once and merged rather than pushed to millions.

Two direct payoffs for the runtime:
- Cap the write amplification. Just as no Instagram post ever fans out to hundreds of millions of writes, no node in a compiled Rare.lab shader should be forced to recompute for every one of its many consumers. Materialize the high-fan-out node once; let consumers read the result. Fan-out count of a node is a compile-time signal you already have from the graph, use it as the threshold, the same way follower count is the threshold for push vs pull.
- Keep the hot path a lookup. The reason Priya's feed loads in under a second is that the read is a lookup of a pre-built list, not a live computation. The reason a Rare.lab effect should hit frame budget is the same: the per-frame hot path should read materialized results, not re-derive the expensive subgraph. Move the expensive thinking to a prepass (offline or once-per-frame), and make the per-fragment path a cheap read. This is the same shape as the Discover Weekly lesson in this ledger (expensive thinking offline and static, the live path a cheap memory-mapped lookup), now applied node-by-node inside the compiler.

The threshold is the whole design. Ship a compiler pass that computes each node's fan-out and stability, and route high-fan-out-stable nodes to materialization and low-fan-out nodes to inlining. That is Instagram's push/pull hybrid, expressed as a shader-graph optimization.

---

## Sources

Primary and authoritative:
- Instagram Engineering, "Open-sourcing a 10x reduction in Apache Cassandra tail latency" (Rocksandra; P99 60 ms to 20 ms, GC 2.5% to 0.3%): https://instagram-engineering.com/open-sourcing-a-10x-reduction-in-apache-cassandra-tail-latency-d64f86b43589
- Meta Engineering, "TAO: The power of the graph" (graph-aware cache, leader/follower tiers, ~1B reads/sec, 96.4% hit rate): https://engineering.fb.com/2013/06/25/core-infra/tao-the-power-of-the-graph/
- TAO paper summary and later scale figures (10B+ reads/sec by 2021): https://www.micahlerner.com/2021/10/13/tao-facebooks-distributed-data-store-for-the-social-graph.html
- Lisa Guo, "Scaling Instagram Infrastructure," QCon London 2017 (Memcache leases, thundering herd, Django/Celery/RabbitMQ): https://qconlondon.com/london-2017/london-2017/presentation/scaling-instagram-infrastructure.html
- Nishtala et al., "Scaling Memcache at Facebook," USENIX NSDI 2013 (leases against thundering herd): https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf

Fan-out / timeline delivery (the class-of-problem references, used as clearly-labeled inference):
- Twitter/X news feed, fan-out-on-write vs fan-out-on-read, the celebrity problem: https://www.techinterview.org/post/3233474168/system-design-twitter-news-feed-timeline-fanout-on-write-fanout-on-read-celebrity-problem-ranking-caching/
- "Fan-out on Write vs Fan-out on Read: The Core Trade-off": https://wittycoder.in/courses/news-feed/fan-out-strategies
- ByteByteGo, "How Instagram Scaled Its Infrastructure To Support a Billion Users": https://blog.bytebytego.com/p/how-instagram-scaled-its-infrastructure
- "The Fanout Problem: How Instagram Delivers Your Feed in Under 200ms": https://medium.com/@tahn98/the-fanout-problem-how-instagram-delivers-your-feed-in-under-200ms-67886f1e1ab7
