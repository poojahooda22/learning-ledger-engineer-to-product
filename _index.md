# Index of Teardowns

All product teardowns, newest first.

| Date | Product | Feature | Report |
|------|---------|---------|--------|
| 2026-08-22 | YouTube | View count (the number under every video, and the two famous failures that exposed the machine: counting is not "add one" once the number must be fraud-defensible, instant to read for hundreds of millions, and never stops changing globally; a naive `UPDATE views=views+1` dies on the single hot row and on having no memory of what it counted, so plays live in an append-only event log and the count is derived; a two-clock lambda design runs a speed layer over sharded counters for the number you watch move, a batch layer that re-aggregates with fraud filters for the verified truth, and Procella as the serving layer that answers the public read as a cheap cached O(1) lookup; the "301+" freeze 2012-2015 was the seam between speed and batch made visible, holding rather than clawing back; the Gangnam Style overflow Dec 2014 hit 2,147,483,647 = signed 32-bit INT_MAX and forced a 64-bit widening; fraud rules 30s + viewer-initiated + ~4-5 replays/day + invalid-view removal up to 30 days) | [report](product-teardowns/2026-08-22-youtube-view-count.md) |
| 2026-08-21 | Zomato | Movie and event ticket booking: seat selection and the seat lock under a flash sale (District, the Paytm Insider ticketing business Zomato bought in 2024: a seat exists exactly once so booking is a contested-resource problem inside the 90-second gap between tapping seat H14 and the payment clearing; the double-booking guard is a two-layer lock, a fast in-memory Redis hold via one atomic SET NX PX with a self-expiring 5-minute TTL that auto-releases abandoned holds, backed by the database as final judge via an optimistic conditional UPDATE ... WHERE status='AVAILABLE'; seat-viewing traffic is ~100x seat-booking so cache the read path and keep the tiny atomic write path sacred; at the Coldplay tier of 13 million people chasing 178,000 tickets on one map, sharding by show fails against a single hot partition, so a virtual waiting room throttles the origin and randomized queue positions kill the millisecond bot lottery) | [report](product-teardowns/2026-08-21-zomato-district-ticketing-seat-lock.md) |
| 2026-08-20 | Google Search | Featured Snippets and passage ranking (the boxed answer at "position zero" above the ten blue links: three stacked problems, understand the question with BERT so tiny words like "no" survive and served on Cloud TPUs, find-and-rank with the match-then-rank spine, inverted index plus dense bi-encoder retrieval for candidates then an expensive cross-encoder BERT re-ranking a bounded shortlist, the 2020 passage-ranking upgrade that surfaces a paragraph buried deep in a page, and extractive span selection to fill the box, all server-side; a zero-click trust reflex that also starves the publishers it quotes) | [report](product-teardowns/2026-08-20-google-search-featured-snippets.md) |
| 2026-08-19 | Figma | Components, instances, and library updates (one edit to a master component flows to thousands of instances, and a published team library pushes it across files: an instance is a recipe not a deep copy, a pointer to a master plus a tiny override map, with children materialized on demand via the Materializer; a master edit invalidates dependents in a dependency graph and re-materializes only those, so cost scales with affected instances not document size; cross-file the link is a versioned snapshot with opt-in per-file pull, publish/subscribe as load-shedding) | [report](product-teardowns/2026-08-19-figma-components-instances-library-updates.md) |
| 2026-08-18 | Canva | Magic Resize (take a finished design and re-lay it out into any set of new dimensions in one click: a design is a scene graph of positioned elements with proportions and anchors, not a flat picture, so retargeting is a coordinate-system change over a tree, uniform per-element scaling to avoid the funhouse-mirror stretch, anchor-based repositioning so the logo stays bottom-right, and a local overflow solve, all run server-side for font/asset consistency; the paywall sits at the moment of maximum motivation and has gated revenue since the 2015 Canva for Work launch) | [report](product-teardowns/2026-08-18-canva-magic-resize.md) |
| 2026-08-17 | Netflix | Scrubbing preview thumbnails (the little filmstrip above the seek bar: trickplay images, a separate pre-made pack of tiny JPEGs indexed by time, packed into sprite sheets / BIF, generated once offline on the Cosmos pipeline and served from Open Connect, so every drag is a local O(1) crop with zero server work while the real video seek fires exactly once on release) | [report](product-teardowns/2026-08-17-netflix-scrubbing-preview-thumbnails.md) |
| 2026-08-16 | Spotify | Autoplay (the endless session that kicks in when your playlist, album, or queue runs out: real-time session continuation, not the weekly-batch Discover Weekly; ANN candidate fetch over 100M+ track embeddings via Annoy then Voyager/HNSW, BaRT bandit ranking that optimizes completed listens over clicks and balances exploit-the-familiar against explore-the-new to beat the echo chamber, and the offline-think/online-lookup path that keeps the per-tap cost flat as the catalog grows) | [report](product-teardowns/2026-08-16-spotify-autoplay-playlist-continuation.md) |
| 2026-08-15 | Gmail | Undo Send (the "Message sent. Undo" toast and the 5-30 second window: it never recalls mail because SMTP has no un-send verb once the receiving server returns 250 OK; instead the send is deferred a few seconds behind a timing-wheel/delay-queue and Undo is an O(1) cancel of a scheduled job that never fired) | [report](product-teardowns/2026-08-15-gmail-undo-send.md) |
| 2026-08-14 | Netflix | Skip Intro (the one-tap button past the theme song, plus Skip Recap / Skip Credits / Next Episode: intros repeat across a season, so find the repetition with Shazam-style audio fingerprinting, verify with computer vision + shot boundaries, do it all offline on Archer/Cosmos, and ship the player two timestamps) | [report](product-teardowns/2026-08-14-netflix-skip-intro.md) |
| 2026-08-13 | Razorpay | Magic Checkout (one-click checkout: the shared 100M-shopper network address prefill keyed off a verified phone identity, and the real-time COD Return-to-Origin risk model, grown from the acquired Thirdwatch engine, that decides whether Cash on Delivery is offered, nudged to prepaid, or blocked) | [report](product-teardowns/2026-08-13-razorpay-magic-checkout-rto.md) |
| 2026-08-12 | Amazon | Buy Box / Featured Offer (the box that decides which of many competing sellers sits behind "Buy Now": the eligibility gate, the landed-price + delivery-speed + seller-metric scoring, the 2025 fulfillment-neutral shift, per-ASIN precompute-and-cache, and rotation among near-ties) | [report](product-teardowns/2026-08-12-amazon-buy-box-featured-offer.md) |
| 2026-08-11 | WhatsApp | Multi-device (companion devices with their own Signal identity, client-fanout encryption to N devices, Automatic Device Verification, and App State Sync via the LTHash homomorphic hash) | [report](product-teardowns/2026-08-11-whatsapp-multi-device.md) |
| 2026-08-10 | Gmail | Conversation threading (how loose emails become one conversation: Message-ID/In-Reply-To/References headers, the JWZ Container forest built through one id_table hash map, dummy containers for out-of-order/missing mail, loop guard, pruning, subject merge, and Gmail's write-time threadId assignment) | [report](product-teardowns/2026-08-10-gmail-conversation-threading.md) |
| 2026-08-09 | Stripe | Checkout (the hosted pay page: which payment methods appear and in what order, the PaymentIntent state machine, and 3D Secure authentication underneath) | [report](product-teardowns/2026-08-09-stripe-checkout.md) |
| 2026-08-08 | Uber | Live driver location on the map (the moving car after you book: driver GPS pings, HMM+Viterbi map matching to snap to the road, and client-side interpolation + dead reckoning for the smooth marker) | [report](product-teardowns/2026-08-08-uber-live-location-map-matching.md) |
| 2026-08-07 | Instagram | Reels recommendation and ranking (the vertical video feed of strangers: retrieval funnel, two-tower ANN, first-stage distillation, second-stage MTML, the value model, watch time over likes) | [report](product-teardowns/2026-08-07-instagram-reels-recommendation-ranking.md) |
| 2026-08-06 | Netflix | Personalized homepage (the grid of rows: which titles fill each row, the order within a row, and the order of the rows themselves; PVR, Top-N, Trending, BYW, page generation, GenPage) | [report](product-teardowns/2026-08-06-netflix-homepage-rows.md) |
| 2026-08-05 | Google Search | Knowledge Panel and the Knowledge Graph behind it (things not strings: the entity graph, entity linking, Knowledge Vault) | [report](product-teardowns/2026-08-05-google-search-knowledge-panel.md) |
| 2026-08-04 | YouTube | Automatic captions (the ASR pipeline: speech to timed text, WFST decoding + RNN-T, plus auto-alignment) | [report](product-teardowns/2026-08-04-youtube-automatic-captions.md) |
| 2026-08-03 | Spotify | Spotify Connect (device handoff: tap a speaker and playback follows you, control plane split from data plane) | [report](product-teardowns/2026-08-03-spotify-connect-device-handoff.md) |
| 2026-08-02 | Notion | Databases and database views (table/board/calendar, filters/sorts, the collection data model + Postgres sharding) | [report](product-teardowns/2026-08-02-notion-databases-and-views.md) |
| 2026-08-01 | Razorpay | UPI payments (the checkout method: intent vs collect, the async switch, deemed success, reconciliation) | [report](product-teardowns/2026-08-01-razorpay-upi-payments.md) |
| 2026-07-31 | Stripe | Webhooks (the event delivery system: the Event object, the signed POST, retries with backoff, at-least-once delivery) | [report](product-teardowns/2026-07-31-stripe-webhooks-event-delivery.md) |
| 2026-07-30 | Canva | Background Remover (the one-click "BG Remover" that cuts the subject out of a photo) | [report](product-teardowns/2026-07-30-canva-background-remover.md) |
| 2026-07-29 | Figma | Auto Layout (the frame that stacks, spaces, and resizes its children automatically) | [report](product-teardowns/2026-07-29-figma-auto-layout.md) |
| 2026-07-28 | Amazon | Shopping cart (the "Add to Cart" button and the always-available Dynamo store underneath) | [report](product-teardowns/2026-07-28-amazon-shopping-cart-dynamo.md) |
| 2026-07-27 | Netflix | Continue Watching row and cross-device resume (viewing-history + bookmark engine) | [report](product-teardowns/2026-07-27-netflix-continue-watching.md) |
| 2026-07-26 | WhatsApp | Voice and video calling (call setup, the relay, encrypted media, the codec) | [report](product-teardowns/2026-07-26-whatsapp-voice-video-calling.md) |
| 2026-07-25 | Amazon | Delivery date promise (the "Get it by" date, countdown timer, Promised Delivery Date) | [report](product-teardowns/2026-07-25-amazon-delivery-date-promise.md) |
| 2026-07-24 | Gmail | Priority Inbox (automatic importance ranking, the yellow marker) | [report](product-teardowns/2026-07-24-gmail-priority-inbox.md) |
| 2026-07-23 | Stripe | Balance, funds availability, and payouts (the money-movement ledger underneath) | [report](product-teardowns/2026-07-23-stripe-balance-payouts-ledger.md) |
| 2026-07-22 | Zomato | Order batching and delivery-partner assignment (the dispatch engine that clubs orders) | [report](product-teardowns/2026-07-22-zomato-order-batching-dispatch.md) |
| 2026-07-21 | Spotify | Wrapped (annual year-in-music recap + the batch data pipeline) | [report](product-teardowns/2026-07-21-spotify-wrapped-data-pipeline.md) |
| 2026-07-20 | Instagram | Explore page recommender (candidate generation + multi-stage ranking) | [report](product-teardowns/2026-07-20-instagram-explore-recommendations.md) |
| 2026-07-19 | Swiggy | Search and ranking (the search box: autocomplete, retrieval, ranking) | [report](product-teardowns/2026-07-19-swiggy-search-ranking.md) |
| 2026-07-18 | Google Search | PageRank and the web link graph (query-independent quality score) | [report](product-teardowns/2026-07-18-google-search-pagerank.md) |
| 2026-07-17 | YouTube | Upload transcoding pipeline (chunked parallel encoding + the Argos VCU chip) | [report](product-teardowns/2026-07-17-youtube-video-transcoding-pipeline.md) |
| 2026-07-16 | Notion | Offline mode and the sync/conflict engine ("Available offline" + CRDT data model) | [report](product-teardowns/2026-07-16-notion-offline-sync-crdt.md) |
| 2026-07-15 | Uber | Trip and pickup ETA prediction (routing engine + DeeprETA correction) | [report](product-teardowns/2026-07-15-uber-eta-prediction-deepreta.md) |
| 2026-07-13 | Netflix | Open Connect (private CDN, proactive fill, play-time steering) | [report](product-teardowns/2026-07-13-netflix-open-connect-cdn.md) |
| 2026-07-12 | Amazon | Item-to-item collaborative filtering ("Customers who bought this also bought") | [report](product-teardowns/2026-07-12-amazon-item-to-item-collaborative-filtering.md) |
| 2026-07-11 | Instagram | Photo storage and delivery (Haystack + f4 + CDN) | [report](product-teardowns/2026-07-11-instagram-photo-storage-haystack-f4.md) |
| 2026-07-10 | Gmail | Spam filtering (Spam folder, reputation, ML classifier + RETVec) | [report](product-teardowns/2026-07-10-gmail-spam-filtering.md) |
| 2026-07-09 | Figma | Canvas rendering engine (WebGL/WebGPU tile renderer, C++ to WebAssembly) | [report](product-teardowns/2026-07-09-figma-canvas-rendering-engine.md) |
| 2026-07-08 | YouTube | Content ID (copyright fingerprint matching on every upload) | [report](product-teardowns/2026-07-08-youtube-content-id.md) |
| 2026-07-07 | Spotify | Instant playback and streaming (cache, P2P overlay, CDN) | [report](product-teardowns/2026-07-07-spotify-instant-playback-streaming.md) |
| 2026-07-06 | Amazon | 1-Click ordering (Buy Now, no cart, no checkout) | [report](product-teardowns/2026-07-06-amazon-1-click-ordering.md) |
| 2026-07-05 | Google Search | Spell correction ("Did you mean" / "Showing results for") | [report](product-teardowns/2026-07-05-google-search-spell-correction.md) |
| 2026-07-04 | Stripe | Radar real-time card-fraud risk scoring (during a charge) | [report](product-teardowns/2026-07-04-stripe-radar-fraud-scoring.md) |
| 2026-07-03 | Rapido | Turn-by-turn navigation for the captain (the routing engine) | [report](product-teardowns/2026-07-03-rapido-turn-by-turn-navigation.md) |
| 2026-07-02 | Uber | Batched matching / dispatch (DISCO) | [report](product-teardowns/2026-07-02-uber-batched-matching-dispatch.md) |
| 2026-07-01 | WhatsApp | End-to-end encryption (Signal Protocol: X3DH, Double Ratchet, Sender Keys) | [report](product-teardowns/2026-07-01-whatsapp-end-to-end-encryption.md) |
| 2026-06-30 | Netflix | Adaptive bitrate streaming (per-title/per-shot encoding + client ABR) | [report](product-teardowns/2026-06-30-netflix-adaptive-bitrate-streaming.md) |
| 2026-06-29 | Zomato | Food Preparation Time (KPT/FPT) prediction | [report](product-teardowns/2026-06-29-zomato-food-prep-time.md) |
| 2026-06-28 | Canva | Template search and ranking (the search box) | [report](product-teardowns/2026-06-28-canva-templates.md) |
| 2026-06-27 | Instagram | Stories tray ranking and 24-hour expiry | [report](product-teardowns/2026-06-27-instagram-stories-tray-ranking.md) |
| 2026-06-26 | Razorpay | Smart payment routing (Optimizer) | [report](product-teardowns/2026-06-26-razorpay-smart-routing.md) |
| 2026-06-25 | Notion | The "/" slash command (and the block model) | [report](product-teardowns/2026-06-25-notion-slash-command-block-model.md) |
| 2026-06-24 | Swiggy | Live order tracking (the moving rider and the ETA timer) | [report](product-teardowns/2026-06-24-swiggy-live-order-tracking.md) |
| 2026-06-23 | Amazon | Product search ranking (the results page) | [report](product-teardowns/2026-06-23-amazon-product-search-ranking.md) |
| 2026-06-22 | YouTube | Recommendations (home feed and "watch next") | [report](product-teardowns/2026-06-22-youtube-recommendations.md) |
| 2026-06-21 | Gmail | Smart Compose (inline sentence completion) | [report](product-teardowns/2026-06-21-gmail-smart-compose.md) |
| 2026-06-20 | Stripe | Idempotency keys | [report](product-teardowns/2026-06-20-stripe-idempotency-keys.md) |
| 2026-06-19 | Netflix | Personalized artwork (per-member thumbnails) | [report](product-teardowns/2026-06-19-netflix-artwork-personalization.md) |
| 2026-06-18 | Figma | Multiplayer cursors and live presence | [report](product-teardowns/2026-06-18-figma-multiplayer-cursors.md) |
| 2026-06-17 | Zepto | Dark-store inventory and order routing | [report](product-teardowns/2026-06-17-zepto-dark-store-inventory-routing.md) |
| 2026-06-16 | Google Search | Autocomplete (query suggestions / typeahead) | [report](product-teardowns/2026-06-16-google-search-autocomplete.md) |
| 2026-06-15 | WhatsApp | Delivery receipts (the ticks) | [report](product-teardowns/2026-06-15-whatsapp-delivery-receipts.md) |
| 2026-06-14 | Uber | Surge pricing | [report](product-teardowns/2026-06-14-uber-surge-pricing.md) |
| 2026-06-13 | Spotify | Discover Weekly | [report](reports/2026-06-13-spotify-discover-weekly.md) |

---

# System Design Lessons

Daily system-design lessons, newest first.

| Day | Date | Topic | Lesson |
|-----|------|-------|--------|
| 60 | 2026-08-21 | Shader and pipeline-state-object compilation caching: why a GPU shader is source code plus a specific render-state combination that the driver only compiles into real machine code the first time it's drawn with, Unreal Engine's own telemetry showing a single uncached compile can cost up to 117ms against a 16.6ms frame budget at 60fps, Valve's Fossilize precaching system shipping Overwatch 2's shader cache to every Steam player ahead of time (a ~7.7-7.8GB precache download, a build pass spiking to 22-23GB RAM), and the real incident where Nvidia's own 1GB default driver cache limit silently evicted that precached cache mid-session, a borrowed-cache metastable failure | [lesson](lessons/060-shader-compilation-caching-pso-precaching.md) |
| 59 | 2026-08-20 | Event sourcing and CQRS: LMAX's Business Logic Processor settling 6,000,000 trades/sec on a single lock-free thread by treating the append-only order log as the only written fact and deriving all state by replay, why that dies twice over (a nanosecond budget that a single uncontended lock already blows through, and an `UPDATE` statement destroying the very history an auditor needs), CQRS's split of a strongly-consistent write log from denormalized, laggable read projections, snapshotting to bound replay time, and the replay-storm feedback loop that turns a routine projection rebuild into an 18-hour outage | [lesson](lessons/059-event-sourcing-cqrs.md) |
| 58 | 2026-08-19 | Online, zero-downtime schema migrations on tables too large to lock: GitHub's 2016 motivation for building gh-ost (a naive `ALTER TABLE` on the billion-row `pull_requests` table would lock it for over an hour), the shadow-table pattern both gh-ost and Percona's pt-online-schema-change use (chunked backfill + async convergence + atomic rename cut-over), the triggerless binlog-based change capture that separates gh-ost from trigger-based tools, replication-lag-driven throttling, and GitHub's own November 2021 incident where that same design's final sub-second rename step still deadlocked a significant portion of MySQL read replicas, a lock-convoy / metastable-failure story | [lesson](lessons/058-online-schema-migrations-zero-downtime-ddl.md) |
| 57 | 2026-08-18 | Distributed cron and exactly-once scheduling: CircleCI's December 2025 race condition (33,135 duplicate workflow runs across 38 projects from one duplicated completion event), why a single crontab is a silent single point of failure and running it on every replica turns into N duplicate firings instead, and the real fix (leader election / distributed lock via Quartz's JDBC row-lock or ShedLock, an idempotency key scoped to job+scheduled-time, decoupling the firing decision from the work queue, and a dead man's switch that alerts on absence, not just errors) versus Kubernetes CronJob's own documented "at least once" guarantee and 100-missed-schedule catch-up cap | [lesson](lessons/057-distributed-cron-exactly-once-scheduling.md) |
| 56 | 2026-08-17 | Real-time bidding and ad exchange auctions at scale: OpenRTB's tmax deadline, header bidding's parallel fan-out replacing the sequential waterfall, second-price vs first-price mechanism design, precomputed O(1) feature lookups on the hot path, and the hidden timeout-budget cascade that silently kills a bidder's fill rate when deadlines aren't propagated hop to hop | [lesson](lessons/056-realtime-bidding-ad-auctions-at-scale.md) |
| 55 | 2026-08-16 | Anycast and BGP-based global routing: how Cloudflare answers the same IP address from 337 buildings on 6 continents at once, why no system ever decides which data center handles a given request, and how that same property turned a 31.4 Tbps DDoS flood into hundreds of locally survivable ones automatically | [lesson](lessons/055-anycast-global-server-load-balancing.md) |
| 54 | 2026-08-15 | Real-time leaderboard ranking at scale: why sorting a live, constantly-written table on every read collapses (Dream11's up to 40M fantasy-team entries and 666,667 recomputed rows/sec across a match), the precompute-and-serve fix (Spark batch recompute -> Aerospike key-value serving store, AP over CP, sharded by contest_id, bounded-staleness refresh cycle instead of live sort), and the read-write coupling feedback loop it breaks | [lesson](lessons/054-realtime-leaderboard-precompute-rerank-at-scale.md) |
| 53 | 2026-08-14 | Flash-broadcast push notification fan-out: Duolingo's Super Bowl LVIII ad delivering 6M+ notifications within 5 seconds, why an exactly-once SQS FIFO queue caps out at 300-3,000 msg/sec and can't also carry the volume, splitting a correctness-only dedup queue from a high-throughput fan-out queue, pre-warming a 5,000-worker fleet ahead of a known deadline instead of reactive autoscaling, and prefetching the recipient list before the burst instead of querying live at zero-hour | [lesson](lessons/053-push-notification-fanout-flash-broadcast.md) |
| 52 | 2026-08-10 | Real-time competitive multiplayer netcode: client-side prediction, entity interpolation, server reconciliation by input replay, and lag-compensated hit resolution (bounded server-side rewind), via Valve's foundational 2001 lag-compensation paper and Blizzard's 2017 GDC talk on Overwatch's netcode | [lesson](lessons/052-realtime-multiplayer-netcode-prediction-lag-compensation.md) |
| 51 | 2026-08-08 | Store-and-forward messaging at planetary scale: WhatsApp's multi-device fan-out (up to five independent per-device encrypted copies), durability-before-delivery, per-copy acknowledgment, and Signal Protocol Sender Keys for groups | [lesson](lessons/051-messaging-exactly-once-multidevice-fanout.md) |
| 50 | 2026-08-07 | Sandboxed multi-tenant code execution: why running untrusted code in the API process or in a full VM both fail, AWS Firecracker's stripped-down microVM (sub-125ms boot, under 5MiB overhead, thousands per host) versus the namespace+cgroup+seccomp jail (`isolate`/Judge0) an online judge like LeetCode uses instead, and enforcing wall-clock and memory limits from outside the sandboxed process | [lesson](lessons/050-sandboxed-multi-tenant-code-execution.md) |
| 49 | 2026-08-06 | Real-time audio and video at scale: why WebRTC full-mesh calls collapse around 4 to 6 participants (K*(N-1) bandwidth and N-1 simultaneous encoders), Discord's SFU architecture serving 2.6M+ concurrent voice users off 850+ servers in 13 regions, simulcast, receiver-driven "Media Sink Wants" stream selection, and closed-loop congestion control instead of TCP-style retransmission | [lesson](lessons/049-webrtc-sfu-realtime-video-at-scale.md) |
| 48 | 2026-08-05 | Geospatial indexing and real-time ride matching: Uber's H3 hexagonal hierarchical grid (successor to Google's S2), the DISCO dispatch system, Ringpop's consistent-hash-ring-plus-gossip sharding for a stateful service, and why lat/lng B-tree columns can't answer "who is near me" fast enough once the map gets crowded | [lesson](lessons/048-geospatial-indexing-ride-matching.md) |
| 47 | 2026-08-04 | Quicksilver, Cloudflare's config-distribution key-value store: how a DNS record, WAF rule, or Workers script edit reaches every one of 300+ edge data centers in seconds, and why the request-time read hits a local memory-mapped file, not a database call | [lesson](lessons/047-cloudflare-quicksilver-config-distribution.md) |
| 46 | 2026-08-03 | Distributed SQL joins across shards: Google F1's hierarchical schema over Spanner for the 100+TB AdWords database, Vitess/PlanetScale's vtgate nested-loop join engine (one query per left-row by default), Citus colocated vs repartition joins, and CockroachDB's DistSQL hash/merge/lookup join strategies | [lesson](lessons/046-distributed-joins-across-shards.md) |
| 45 | 2026-08-02 | Global and secondary indexes in sharded databases: Uber Schemaless's 4,096-shard trip datastore, DynamoDB GSI hot-partition backpressure onto the base table, Vitess consistent lookup Vindexes (SELECT FOR UPDATE instead of 2PC), and Spanner's interleaved indexes | [lesson](lessons/045-secondary-indexes-sharded-databases.md) |
| 44 | 2026-07-31 | LLM inference serving at scale: Character.AI's ~20,000 QPS chat stack, vLLM's PagedAttention cutting KV-cache waste from 60-80% to under 4%, and continuous/iteration-level batching | [lesson](lessons/044-llm-inference-serving-continuous-batching.md) |
| 43 | 2026-07-29 | Real-time OLAP at scale: how LinkedIn's Pinot (born from "Who's Viewed Your Profile") and Uber's Neutrino answer 250,000+ QPS / 500M+ queries a day over billions of rows via columnar segments, the star-tree pre-aggregation index, and replica groups that bound scatter-gather tail latency | [lesson](lessons/043-realtime-olap-pinot-star-tree.md) |
| 42 | 2026-07-28 | Kafka's partitioned commit log at LinkedIn scale: 7 trillion messages/day across ~7 million partitions, leader/ISR replication, consumer-group offset ownership, log compaction, and the rebalancing-storm feedback loop | [lesson](lessons/042-kafka-partitioned-commit-log.md) |
| 41 | 2026-07-27 | Distributed file systems: how Google split GFS's single metadata master into Colossus's sharded curators (backed by Bigtable) to scale storage 100x past the largest GFS clusters, chunk leases, D servers, custodians, and flash/disk tiering | [lesson](lessons/041-distributed-file-systems-gfs-colossus.md) |
| 40 | 2026-07-24 | Stream processing exactly-once semantics: Alibaba's Flink dashboards at 583,000 orders/sec, Chandy-Lamport checkpoint barriers, watermarks for out-of-order event time, and why aligned checkpoints stall under backpressure | [lesson](lessons/040-stream-processing-exactly-once-watermarks.md) |
| 39 | 2026-07-23 | Distributed tracing and sampling at scale: Dapper's adaptive rate-based sampling, Jaeger's context propagation and tail-based sampling, why the observability path must never block the request it's observing | [lesson](lessons/039-distributed-tracing-sampling-at-scale.md) |
| 38 | 2026-07-22 | Distributed schedulers: how Borg and Kubernetes bin-pack hundreds of thousands of jobs onto tens of thousands of machines, filter/score, equivalence-class caching, priority preemption, oversubscription | [lesson](lessons/038-distributed-schedulers-borg-kubernetes.md) |
| 37 | 2026-07-20 | Durable execution and workflow orchestration: how Uber Cadence/Temporal keep a multi-day business process correct across guaranteed host crashes via event-sourced replay and per-shard serialization | [lesson](lessons/037-durable-execution-workflow-orchestration.md) |
| 36 | 2026-07-19 | Facebook TAO: the social graph store built because a key-value cache can't represent a growing edge list without invalidating the whole thing on every write | [lesson](lessons/036-tao-facebook-social-graph-store.md) |
| 35 | 2026-07-18 | Zanzibar: Google's global ReBAC authorization system, answering 10M+ permission checks/sec over 2T+ stored edges at sub-10ms p95 | [lesson](lessons/035-zanzibar-global-authorization.md) |
| 34 | 2026-07-17 | Cell-based architecture and shuffle sharding: bounding blast radius so one bad thread, customer, or deploy can't take everyone down at once | [lesson](lessons/034-cell-based-architecture-shuffle-sharding.md) |
| 33 | 2026-07-15 | Vector search at scale: approximate nearest neighbor search over a billion embeddings via IVF, product quantization, and HNSW graphs | [lesson](lessons/033-vector-search-ann-at-scale.md) |
| 32 | 2026-07-14 | Erasure coding for durable storage: Reed-Solomon striping vs 3x replication, how Facebook stores tens of petabytes of photos cheaply | [lesson](lessons/032-erasure-coding-durable-storage.md) |
| 31 | 2026-07-13 | Session guarantees and causal consistency: read-your-writes and monotonic reads when the next read can land on a replica that hasn't heard about the write yet | [lesson](lessons/031-session-guarantees-causal-consistency.md) |
| 30 | 2026-07-12 | Byzantine fault tolerance: PBFT's O(n^2) message wall and HotStuff's linear, chained-signature fix | [lesson](lessons/030-byzantine-fault-tolerance-pbft-hotstuff.md) |
| 29 | 2026-07-11 | Write skew and Serializable Snapshot Isolation: why write-write conflict checks miss invariants that span two rows, via Berenson et al.'s 1995 paper, PostgreSQL 9.1's SSI, and CockroachDB's on-call-doctors example | [lesson](lessons/029-write-skew-serializable-snapshot-isolation.md) |
| 28 | 2026-07-10 | Probabilistic data structures at scale: Bloom filters (Google Safe Browsing), HyperLogLog (cardinality estimation), Count-Min Sketch (streaming heavy hitters) | [lesson](lessons/028-probabilistic-data-structures-bloom-hyperloglog-cms.md) |
| 27 | 2026-07-09 | External consistency in global databases: Spanner's TrueTime and commit-wait vs CockroachDB's Hybrid Logical Clocks | [lesson](lessons/027-truetime-spanner-external-consistency.md) |
| 26 | 2026-07-08 | Distributed unique ID generation: Twitter Snowflake vs Instagram's in-database composite IDs, vs UUIDv4 and auto-increment | [lesson](lessons/026-distributed-id-generation-snowflake.md) |
| 25 | 2026-07-07 | Gossip protocols and failure detection: SWIM, phi-accrual, epidemic dissemination at 10,000+ nodes | [lesson](lessons/025-gossip-protocols-failure-detection.md) |
| 24 | 2026-07-06 | Distributed locks, leases, and fencing tokens: Chubby, ZooKeeper/HBase, and the Redlock debate | [lesson](lessons/024-distributed-locks-fencing-tokens.md) |
| 23 | 2026-07-06 | Content-addressed storage: Git's object model, Merkle DAGs, dedup, garbage collection by reachability | [lesson](lessons/023-content-addressed-storage-merkle-dags.md) |
| 22 | 2026-07-04 | Leaderless replication: quorums, vector clocks, sibling reconciliation, Merkle-tree anti-entropy | [lesson](lessons/022-leaderless-replication-quorums-vector-clocks.md) |
| 21 | 2026-07-02 | Write-optimized storage engines: LSM-trees, memtables, SSTables, compaction | [lesson](lessons/021-lsm-trees-write-optimized-storage.md) |
| 20 | 2026-07-01 | Distributed transactions across shards: 2PC vs sagas vs TrueTime | [lesson](lessons/020-distributed-transactions-2pc-vs-sagas.md) |
| 19 | 2026-06-30 | Caching strategies and invalidation (stampede prevention) | [lesson](lessons/019-caching-strategies-invalidation.md) |
| 18 | 2026-06-29 | Search ranking internals | [lesson](lessons/018-search-ranking-internals.md) |
| 17 | 2026-06-28 | Write-ahead logs and change data capture | [lesson](lessons/017-wal-and-change-data-capture.md) |
| 16 | 2026-06-27 | The hot-key / celebrity problem | [lesson](lessons/016-hot-key-celebrity-problem.md) |
| 15 | 2026-06-26 | CRDTs for collaborative editing | [lesson](lessons/015-crdts-collaborative-editing.md) |
| 14 | 2026-06-25 | Multi-region active-active: writes from every continent | [lesson](lessons/014-multi-region-active-active.md) |
| 13 | 2026-06-24 | Backpressure and load shedding | [lesson](lessons/013-backpressure-load-shedding.md) |
| 12 | 2026-06-23 | Idempotency and exactly-once delivery | [lesson](lessons/012-idempotency-exactly-once.md) |
| 11 | 2026-06-22 | Consensus and Raft | [lesson](lessons/011-consensus-raft.md) |
| 10 | 2026-06-21 | Consistent hashing and sharding | [lesson](lessons/010-consistent-hashing-sharding.md) |
| 9 | 2026-06-20 | Queue as shock absorber | [lesson](lessons/009-queue-as-shock-absorber.md) |
| 8 | 2026-06-19 | Rate limiting | [lesson](lessons/008-rate-limiting.md) |
| 7 | 2026-06-18 | Feed ranking at scale | [lesson](lessons/007-feed-ranking-at-scale.md) |
| 6 | 2026-06-17 | Stripe correctness under load | [lesson](lessons/006-stripe-correctness-under-load.md) |
| 5 | 2026-06-16 | YouTube video storage and streaming | [lesson](lessons/005-youtube-video-storage-streaming.md) |
| 4 | 2026-06-15 | CDN zero origin egress | [lesson](lessons/004-cdn-zero-origin-egress.md) |
| 3 | 2026-06-14 | Figma real-time multiplayer | [lesson](lessons/003-figma-realtime-multiplayer.md) |

---

# Daily Viral Tech Reports

Engineering intelligence briefings, newest first.

| Date | Report |
|------|--------|
| 2026-08-21 | [Daily Viral Tech Report](daily-tech-reports/2026-08-21-daily-viral-tech-report.md) |
| 2026-08-20 | [Daily Viral Tech Report](daily-tech-reports/2026-08-20-daily-viral-tech-report.md) |
| 2026-08-19 | [Daily Viral Tech Report](daily-tech-reports/2026-08-19-daily-viral-tech-report.md) |
| 2026-08-18 | [Daily Viral Tech Report](daily-tech-reports/2026-08-18-daily-viral-tech-report.md) |
| 2026-08-17 | [Daily Viral Tech Report](daily-tech-reports/2026-08-17-daily-viral-tech-report.md) |
| 2026-08-16 | [Daily Viral Tech Report](daily-tech-reports/2026-08-16-daily-viral-tech-report.md) |
| 2026-08-15 | [Daily Viral Tech Report](daily-tech-reports/2026-08-15-daily-viral-tech-report.md) |
| 2026-08-14 | [Daily Viral Tech Report](daily-tech-reports/2026-08-14-daily-viral-tech-report.md) |
| 2026-08-13 | [Daily Viral Tech Report](daily-tech-reports/2026-08-13-daily-viral-tech-report.md) |
| 2026-08-12 | [Daily Viral Tech Report](daily-tech-reports/2026-08-12-daily-viral-tech-report.md) |
| 2026-08-11 | [Daily Viral Tech Report](daily-tech-reports/2026-08-11-daily-viral-tech-report.md) |
| 2026-08-10 | [Daily Viral Tech Report](daily-tech-reports/2026-08-10-daily-viral-tech-report.md) |
| 2026-08-09 | [Daily Viral Tech Report](daily-tech-reports/2026-08-09-daily-viral-tech-report.md) |
| 2026-08-08 | [Daily Viral Tech Report](daily-tech-reports/2026-08-08-daily-viral-tech-report.md) |
| 2026-08-07 | [Daily Viral Tech Report](daily-tech-reports/2026-08-07-daily-viral-tech-report.md) |
| 2026-08-06 | [Daily Viral Tech Report](daily-tech-reports/2026-08-06-daily-viral-tech-report.md) |
| 2026-08-05 | [Daily Viral Tech Report](daily-tech-reports/2026-08-05-daily-viral-tech-report.md) |
| 2026-08-04 | [Daily Viral Tech Report](daily-tech-reports/2026-08-04-daily-viral-tech-report.md) |
| 2026-08-03 | [Daily Viral Tech Report](daily-tech-reports/2026-08-03-daily-viral-tech-report.md) |
| 2026-08-02 | [Daily Viral Tech Report](daily-tech-reports/2026-08-02-daily-viral-tech-report.md) |
| 2026-08-01 | [Daily Viral Tech Report](daily-tech-reports/2026-08-01-daily-viral-tech-report.md) |
| 2026-07-31 | [Daily Viral Tech Report](daily-tech-reports/2026-07-31-daily-viral-tech-report.md) |
| 2026-07-30 | [Daily Viral Tech Report](daily-tech-reports/2026-07-30-daily-viral-tech-report.md) |
| 2026-07-29 | [Daily Viral Tech Report](daily-tech-reports/2026-07-29-daily-viral-tech-report.md) |
| 2026-07-28 | [Daily Viral Tech Report](daily-tech-reports/2026-07-28-daily-viral-tech-report.md) |
| 2026-07-27 | [Daily Viral Tech Report](daily-tech-reports/2026-07-27-daily-viral-tech-report.md) |
| 2026-07-26 | [Daily Viral Tech Report](daily-tech-reports/2026-07-26-daily-viral-tech-report.md) |
| 2026-07-25 | [Daily Viral Tech Report](daily-tech-reports/2026-07-25-daily-viral-tech-report.md) |
| 2026-07-24 | [Daily Viral Tech Report](daily-tech-reports/2026-07-24-daily-viral-tech-report.md) |
| 2026-07-23 | [Daily Viral Tech Report](daily-tech-reports/2026-07-23-daily-viral-tech-report.md) |
| 2026-07-22 | [Daily Viral Tech Report](daily-tech-reports/2026-07-22-daily-viral-tech-report.md) |
| 2026-07-21 | [Daily Viral Tech Report](daily-tech-reports/2026-07-21-daily-viral-tech-report.md) |
| 2026-07-20 | [Daily Viral Tech Report](daily-tech-reports/2026-07-20-daily-viral-tech-report.md) |
| 2026-07-19 | [Daily Viral Tech Report](daily-tech-reports/2026-07-19-daily-viral-tech-report.md) |
| 2026-07-18 | [Daily Viral Tech Report](daily-tech-reports/2026-07-18-daily-viral-tech-report.md) |
| 2026-07-17 | [Daily Viral Tech Report](daily-tech-reports/2026-07-17-daily-viral-tech-report.md) |
| 2026-07-15 | [Daily Viral Tech Report](daily-tech-reports/2026-07-15-daily-viral-tech-report.md) |
| 2026-07-14 | [Daily Viral Tech Report](daily-tech-reports/2026-07-14-daily-viral-tech-report.md) |
| 2026-07-13 | [Daily Viral Tech Report](daily-tech-reports/2026-07-13-daily-viral-tech-report.md) |
| 2026-07-12 | [Daily Viral Tech Report](daily-tech-reports/2026-07-12-daily-viral-tech-report.md) |
| 2026-07-11 | [Daily Viral Tech Report](daily-tech-reports/2026-07-11-daily-viral-tech-report.md) |
| 2026-07-10 | [Daily Viral Tech Report](daily-tech-reports/2026-07-10-daily-viral-tech-report.md) |
| 2026-07-09 | [Daily Viral Tech Report](daily-tech-reports/2026-07-09-daily-viral-tech-report.md) |
| 2026-07-08 | [Daily Viral Tech Report](daily-tech-reports/2026-07-08-daily-viral-tech-report.md) |
| 2026-07-07 | [Daily Viral Tech Report](daily-tech-reports/2026-07-07-daily-viral-tech-report.md) |
| 2026-07-06 | [Daily Viral Tech Report](daily-tech-reports/2026-07-06-daily-viral-tech-report.md) |
| 2026-07-05 | [Daily Viral Tech Report](daily-tech-reports/2026-07-05-daily-viral-tech-report.md) |
| 2026-07-04 | [Daily Viral Tech Report](daily-tech-reports/2026-07-04-daily-viral-tech-report.md) |
| 2026-07-03 | [Daily Viral Tech Report](daily-tech-reports/2026-07-03-daily-viral-tech-report.md) |
| 2026-07-02 | [Daily Viral Tech Report](daily-tech-reports/2026-07-02-daily-viral-tech-report.md) |
| 2026-07-01 | [Daily Viral Tech Report](daily-tech-reports/2026-07-01-daily-viral-tech-report.md) |
| 2026-06-30 | [Daily Viral Tech Report](daily-tech-reports/2026-06-30-daily-viral-tech-report.md) |
| 2026-06-29 | [Daily Viral Tech Report](daily-tech-reports/2026-06-29-daily-viral-tech-report.md) |
| 2026-06-28 | [Daily Viral Tech Report](daily-tech-reports/2026-06-28-daily-viral-tech-report.md) |
| 2026-06-27 | [Daily Viral Tech Report](daily-tech-reports/2026-06-27-daily-viral-tech-report.md) |
| 2026-06-26 | [Daily Viral Tech Report](daily-tech-reports/2026-06-26-daily-viral-tech-report.md) |
| 2026-06-25 | [Daily Viral Tech Report](daily-tech-reports/2026-06-25-daily-viral-tech-report.md) |
| 2026-06-24 | [Daily Viral Tech Report](daily-tech-reports/2026-06-24-daily-viral-tech-report.md) |
| 2026-06-23 | [Daily Viral Tech Report](daily-tech-reports/2026-06-23-daily-viral-tech-report.md) |
| 2026-06-22 | [Daily Viral Tech Report](daily-tech-reports/2026-06-22-daily-viral-tech-report.md) |
| 2026-06-21 | [Daily Viral Tech Report](daily-tech-reports/2026-06-21-daily-viral-tech-report.md) |
| 2026-06-20 | [Daily Viral Tech Report](daily-tech-reports/2026-06-20-daily-viral-tech-report.md) |
| 2026-06-19 | [Daily Viral Tech Report](daily-tech-reports/2026-06-19-daily-viral-tech-report.md) |
| 2026-06-18 | [Daily Viral Tech Report](daily-tech-reports/2026-06-18-daily-viral-tech-report.md) |
| 2026-06-17 | [Daily Viral Tech Report](daily-tech-reports/2026-06-17-daily-viral-tech-report.md) |
| 2026-06-16 | [Daily Viral Tech Report](daily-tech-reports/2026-06-16-daily-viral-tech-report.md) |
| 2026-06-15 | [Daily Viral Tech Report](daily-tech-reports/2026-06-15-daily-viral-tech-report.md) |
| 2026-06-14 | [Daily Viral Tech Report](daily-tech-reports/2026-06-14-daily-viral-tech-report.md) |
