# Uber: the car moving on the map (live driver location, map matching, and the smooth marker)

Date: 2026-08-08
Product: Uber
Feature: Real-time driver location on the rider's map. The little car that creeps toward you after you book, snapped to the road, gliding instead of jumping.

A note on scope. Uber's dispatch (who gets matched to whom) and its ETA prediction were both torn down earlier in this ledger (2026-07-02 DISCO, 2026-07-15 DeeprETA). This report is about a different, narrower thing: after the match is made, how does that one specific car end up drawn on your screen, on the correct road, moving smoothly? Two separate jobs live inside that: cleaning a noisy GPS trace onto the road (map matching), and animating a marker between sparse updates (interpolation). They are different halves and I keep them apart below.

---

## 1. The user

It is 9:12 pm on a Tuesday in Koramangala, Bengaluru. Priya just booked an Uber Go home from a friend's place on 80 Feet Road. The screen flips to "Rakesh is on the way" and shows a tiny white car near the 5th Block signal. She is standing on the footpath in the dark, phone in hand, deciding one thing: do I wait inside the gate for two more minutes, or walk out to the corner now?

She is not reading a PM deck. She is watching a dot. She wants that dot to be honest.

## 2. The real problem

Before live tracking, the anxiety was total. You booked a cab, then you stared at a blank phone and a dark street. Is he coming? Did he cancel? Is he stuck at the signal or did he go the wrong way? You called the driver, he did not pick up, you called again. The waiting was pure uncertainty, and uncertainty at night on a footpath feels like risk.

The car on the map removes almost all of that. It answers "is he actually moving toward me" and "roughly how long" without a phone call. But it only works if the car is believable. If the dot sits on top of a building, or teleports across a block every few seconds, or freezes for twenty seconds and then leaps, Priya stops trusting it and picks up the phone again. The whole point is calm, and calm dies the moment the map looks fake.

## 3. The feature in one sentence

Uber takes the driver's noisy GPS pings, snaps them to the actual road, and streams the result to your phone as a marker that slides smoothly along the route toward you.

## 4. Jobs to be done

- "Tell me he is really coming, without me calling him." (Reassurance.)
- "Tell me how close, in a way I can act on right now." (Should I walk out?)
- "Show it to me on the road I know, not floating in a park." (Believability.)
- "Do not make me stare at a frozen screen." (Liveness.)
- For Uber the business: keep the rider in the app and calm during the riskiest window, the pickup wait, so they do not cancel.

## 5. How it works for the user

Priya sees a map. A blue dot is her. A small white car is Rakesh. A line runs from the car to her pin, the planned route. The car edges forward continuously, turns when the road turns, stops at the signal, then moves again. Above the map: "4 min away," Rakesh's name, the WagonR, the plate KA-01-AB-1234, and a call button she does not need to press.

She never sees a raw GPS coordinate. She never sees the car sitting in the middle of a building. She never sees it jump. That smoothness is not the GPS being good. It is a pile of engineering hiding the GPS being bad.

## 6. The actual flow, step by step

1. Priya taps "Confirm Uber Go." Match is made. Rakesh is assigned.
2. Rakesh's driver app starts sending his phone's GPS location upward on a fixed heartbeat. Uber's own driver-location webhook fires every 4 seconds by default, and the app's internal stream runs on a similar few-second cadence.
3. Each ping is a small record: latitude, longitude, timestamp, heading (which way the nose points), speed, and accuracy radius.
4. Uber's backend cleans each ping (snaps it to a road) and figures out the route from Rakesh to Priya.
5. The cleaned position is pushed to Priya's phone over a live connection.
6. Priya's phone does NOT just drop the marker at the new spot. It animates the car from where it was to where it now is, over a few hundred milliseconds, so the eye sees gliding.
7. Between pings, when no new data has arrived, the phone keeps nudging the car forward along the route using the last known heading and speed. This is a small guess that keeps the car alive.
8. When Rakesh actually arrives, the pings cluster at Priya's pin, the ETA hits zero, and the app flips to "Rakesh has arrived."

Concrete moment: at 9:13:04 pm a ping puts Rakesh at a rooftop 18 meters off the road (GPS bounced off a building). The map does not show a car on a roof. By 9:13:04 the backend has already snapped him back onto 80 Feet Road, and Priya sees the car on the road, moving.

---

## 7. Under the hood, like the engineer

There are three problems stacked here. One, get the pings up at scale. Two, clean each ping onto the real road (map matching). Three, make sparse cleaned points look like continuous motion on a phone (interpolation). I will take them in order.

### 7a. Getting the pings up: the ingestion firehose

A single ride is trivial: one phone sends one small message every 4 seconds. The hard part is that Uber runs millions of these at once, and every ping is a write.

The path a ping takes (this is the well-documented shape of Uber-class systems; exact internal names vary):

1. The driver app opens a persistent connection to a gateway and pushes location events on it. Persistent, because opening a fresh HTTPS request every 4 seconds for tens of millions of drivers would drown the system in connection setup.
2. The gateway validates the event (is this a real logged-in driver, are the coordinates sane) and drops it onto a message log, Kafka, partitioned by geography. "Partitioned by geography" matters: all of Koramangala's pings land on the same partition, so the consumers that care about south Bengaluru read one ordered stream instead of scanning the planet. This is the same "turn where into a shard key" idea covered in the surge teardown (H3 hexes) on 2026-06-14.
3. Downstream consumers read that stream independently: the live-tracking service that feeds riders, the analytics pipeline, the fraud checks. Kafka lets one write feed many readers without the driver app knowing any of them exist.
4. The live position of each active driver is kept in a hot, in-memory store keyed by driver id, sharded across many boxes with consistent hashing (Uber's Ringpop is their in-house library for exactly this: spread keys across a ring of nodes, gossip membership, no central coordinator). Priya's tracking session reads Rakesh's latest position from that hot store, not from a cold database.

Why in-memory and last-write-wins? Because for the map, only the newest position matters. Nobody wants the car where it was 12 seconds ago. Old pings can be thrown away the instant a newer one lands. That single fact, "only the latest value counts," is what makes this scale: you are not accumulating state, you are overwriting one cell.

Data structure at the core: a giant hash map from driver id to latest snapped position, sharded across machines. Reads and writes are O(1). The whole design bends around keeping that map hot and close to the readers.

### 7b. Cleaning the ping: map matching (this is the interesting half)

Raw phone GPS in a dense city is bad. In an urban canyon the signal bounces off glass towers, so a ping can land 10 to 50 meters off, on a rooftop, in a park, or on the wrong side of a divided road. If you draw raw pings, the car twitches like a fly.

Map matching fixes this. The job: given a noisy GPS point (or a short sequence of them) and a road network, decide which road segment the car is really on, and where on that segment.

The naive answer is "snap to the nearest road." It fails constantly. Near a flyover the nearest road might be the road underneath, not the flyover you are on. Near two parallel roads a single noisy point can be closer to the wrong one. Snapping each point independently ignores the obvious truth that a car cannot teleport between unconnected roads.

The standard, published solution is a Hidden Markov Model with the Viterbi algorithm. This is the Newson and Krumm method (Microsoft Research, ACM SIGSPATIAL 2009), and Uber built its CatchME map-quality system on exactly this HMM idea. Here is the model in plain terms.

- Hidden states you cannot see directly: which road segment the car is actually on. For each GPS ping, you gather a handful of candidate segments nearby (say the 5 to 10 roads within ~50 meters). Real example: for the 9:13:04 ping near a rooftop, candidates are 80 Feet Road (northbound), 80 Feet Road (southbound), and a small cross lane.
- Observations you can see: the actual GPS points, roof and all.
- Emission probability: how likely is this GPS reading if the car were truly on candidate segment X? Modeled as a Gaussian on the straight-line distance from the ping to the nearest point on X. Closer means more likely. Newson and Krumm measured real GPS noise and used a standard deviation of about 4 meters for this bell curve. So the rooftop ping gives 80 Feet Road a decent score (it is ~18 m away, within a few sigma) and the far cross lane a poor score.
- Transition probability: how likely is it to move from the segment matched at ping t to a candidate segment at ping t+1? Modeled on the difference between two distances: the straight-line distance between the two GPS points, and the actual driving distance along the roads between the two candidate points. If those two distances are close, the move is natural and gets a high score. If the road route is far longer than the straight line (because the two candidates are not really connected, or connecting them means an illegal U-turn), the move is penalized with an exponential drop-off. Widely used implementations default the decay parameter (beta) to around 3.
- Viterbi: instead of picking the best segment for each ping in isolation, Viterbi finds the single most likely whole sequence of segments across all the pings jointly. It walks the trellis of candidates ping by ping, keeping for each candidate the best-scoring path that ends there, and at the end reads back the winning chain. Cost is roughly (number of pings) times (candidates per ping squared), which is tiny per ride because candidates per ping is a single digit.

The payoff: even though the 9:13:04 ping landed on a roof, the sequence of pings before and after it clearly runs up 80 Feet Road, so Viterbi keeps the car on 80 Feet Road and places it at the nearest point on that road. The roof ping is quietly overruled by its neighbors. That is why the car does not jump onto buildings.

One real, published wrinkle from Newson and Krumm: this works well as long as pings are frequent enough. Their measurements showed matching stays accurate up to about a 30-second sampling gap and then degrades fast, because with long gaps too many different routes could explain the two far-apart points. Uber's 4-second cadence sits comfortably inside the good zone, with lots of margin.

Uber's twist with CatchME: they invert the usual assumption. Normally you trust the map and fix the GPS. CatchME treats the aggregated trip GPS as ground truth and, when the matched route keeps disagreeing with the map (drivers repeatedly "drive through" where the map says there is no road, or refuse a turn the map allows), it flags the map as wrong and feeds a correction back into Uber's maps. Same HMM machinery, pointed at map quality instead of live display.

### 7c. Making it move: interpolation on the phone

Even perfectly cleaned points arrive only every ~4 seconds. If the phone teleports the marker to each new point, the car jumps four times a minute. Ugly, and it reads as broken.

So the client does two things:

1. Tweening between known points. When a new snapped position arrives, the phone does not set the marker there instantly. It animates from the old position to the new one over roughly 300 to 500 milliseconds, moving in tiny steps driven by the screen's frame callback (requestAnimationFrame on web, the platform equivalent on native). At 60 frames per second that is 18 to 30 in-between frames, so the eye sees a glide, not a jump. To turn correctly, the tween follows the road polyline between the two points rather than cutting a straight diagonal through a block.
2. Dead reckoning between pings. If the next ping is late, the phone does not freeze. It keeps nudging the car forward along the known route using the last reported heading and speed. A car last seen going 30 km/h north keeps creeping north. When the real ping lands, the marker gently corrects to truth. This is the same "predict, then correct" idea a Kalman filter formalizes, done lightly on the client so the animation never stalls.

The division of labor is the whole trick: the server does the expensive, correctness-critical work (snap to the right road, once, authoritatively), and the client does the cheap, perception-critical work (make 4-second data look like 60-frames-per-second motion). Neither could carry the illusion alone.

### 7d. The scale story at three tiers

Tier 1, about 1,000 concurrent tracked cars (a small city at launch).
One server holds a hash map of driver id to position. Drivers POST location every few seconds. Riders poll "where is my driver" every 3 to 4 seconds over plain HTTP. Map matching can even be skipped or done with cheap nearest-road snapping. Nothing is stressed. Total pings per second: a few hundred. A laptop could run it.

Tier 2, about 100,000 concurrent tracked cars (a large metro at peak).
Now polling hurts. 100,000 riders polling every 3 seconds is ~33,000 read requests per second just to ask "any news," and most answers are "same as last time." The write side is ~25,000 pings per second. Fixes: switch riders from polling to a pushed persistent connection (server sends only when the position actually changes). Put a message log (Kafka) in front of the writes so bursts do not knock over the store. Shard the hot position store by geography or by consistent hashing so no single box holds the metro. Run map matching as a streaming step on each geographic partition. What broke at the jump from tier 1: the single box and the polling model. What survives it: push plus sharding plus a queue to absorb spikes.

Tier 3, 10 million plus concurrent tracked cars (Uber, globally, on a Friday night).
The hard problem becomes the persistent connections themselves: tens of millions of open sockets, each needing a heartbeat and a place to receive pushes. You need a fleet of connection-holding edge servers, geo-distributed so a Bengaluru phone connects to a Bengaluru edge, not a US one. Writes are a firehose of tens of millions of pings per minute, so geographic Kafka partitioning is mandatory and every consumer must be horizontally scaled. The position store is many shards with read replicas, because a popular pickup zone gets many riders reading nearby drivers. Backpressure and last-write-wins keep memory bounded: drop stale pings, never queue them. Reconnection has to be cheap and correct: when Priya goes through a tunnel and her socket dies, on reconnect her phone says "my last event was at 9:13:04," and the server replays only what is newer from a small per-trip buffer, so she does not get a blank map or a rewound car. What broke at the jump from tier 2: holding the connections and the global write volume. What survives it: edge fans of sockets, geo-sharded queues, replicas for hot read zones, and strict staleness dropping.

The elegant part is how little the live path does per rider. Priya's phone holds one connection and receives one small position update every few seconds for one driver. All the heavy thinking (HMM matching, routing, sharding) happens off her critical path. The expensive work is either precomputed (the road network graph, the route) or done once per ping upstream, and her device just paints and interpolates. That is the same lesson as Discover Weekly (2026-06-13): keep the expensive thinking off the live request.

---

## 8. The retention and habit mechanic

The car on the map is not a growth-loop feature like Stories or Discover Weekly. It is a trust-and-anti-cancel feature, and the metric it moves is retention through completion: it keeps the booked ride from falling apart during the most fragile two to five minutes, the pickup wait.

The loop is emotional, not gamified. Book, feel a spike of "is this real," look at the map, see the car actually moving toward you, feel the spike drop. Every glance is a tiny reassurance that pays off. Because it pays off, you keep the app open instead of calling the driver or cancelling and trying a competitor. Over hundreds of rides, "the map is honest, the car really comes" becomes the quiet reason you open Uber without thinking. Trust is the habit.

Real observed example of the mechanic working: when the car freezes on the map (a data gap, a bad reconnect), riders call the driver, and driver-side complaints about "customer keeps calling" spike. When the car glides believably, those calls drop. The smoothness is not vanity. It is directly the thing that keeps the rider calm and in-app until the driver arrives, which is the same interval where cancellations are most expensive to Uber (a cancelled pickup wastes the driver's approach and often the surge match). The animation quality and the cancellation rate are linked.

## 9. The lesson for Rare.lab

Split authoritative truth from perceived smoothness, and put each on the machine that is good at it.

Uber's map is convincing because the server decides the one true thing (which road, exactly where) at a slow, correct cadence of about every 4 seconds, while the client manufactures the illusion of 60-frames-per-second continuity with cheap local interpolation and dead reckoning. The client never waits for the server to look alive, and the server never has to stream 60 updates a second to look correct.

Rare.lab is a node-based shader and visual-effects editor with an embeddable runtime, so the direct application is your parameter and state sync. When a user drags a node value, or when a scene's animated parameters advance, do not push every intermediate value at frame rate across the wire (whether that wire is editor-to-runtime, multiplayer collaborator-to-collaborator, or a live data source into a running effect). Send authoritative keyframes at a modest cadence and let each runtime interpolate between them locally on the GPU-driven render loop. Two concrete wins:

1. Multiplayer editing. If two people tune the same shader, stream each other's control-point changes a few times a second, not every mouse-move frame, and tween the visible value locally. You cut network traffic by an order of magnitude and the other person's edits still look continuous. This is the exact server-slow, client-smooth split.
2. Live-data-driven effects (an embedded runtime reacting to a metric, audio level, or sensor feed). Treat each incoming sample like a GPS ping: it may be sparse, late, or noisy. Interpolate toward it, and dead-reckon (extrapolate along the last trend) when the next sample is late, so the effect never freezes on stage. Snap gently to truth when the real sample lands.

And borrow the "only the latest value counts" discipline for the runtime's state store: for a live parameter, overwrite one cell with last-write-wins instead of queuing a backlog you can never catch up on. It bounds memory, it bounds latency, and it is why Uber can hold ten million moving cars in a hash map without drowning. A dropped stale frame is invisible; a growing queue is a freeze waiting to happen.

---

## Sources

- Paul Newson and John Krumm, "Hidden Markov Map Matching Through Noise and Sparseness," ACM SIGSPATIAL GIS 2009 (Microsoft Research). The canonical HMM plus Viterbi map-matching method, emission and transition probabilities, GPS noise sigma ~4 m, and the finding that accuracy holds up to ~30-second sampling. https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/map-matching-ACM-GIS-camera-ready.pdf
- Uber Engineering, "Improving Uber's Mapping Accuracy with CatchME." Uber's HMM-based snapping of trip GPS traces to detect map errors. https://www.uber.com/en-IN/blog/mapping-accuracy-with-catchme/
- Uber Developer Documentation, driver-location webhook reference. States driver location webhooks are sent at a 4-second interval by default. https://developer.uber.com/docs/guest-rides/references/api/webhooks/driver-location
- Marie Douriez, "A New Real-Time Map-Matching Algorithm at Lyft," Lyft Engineering. Online HMM map matching in production, online vs offline matchers. https://eng.lyft.com/a-new-real-time-map-matching-algorithm-at-lyft-da593ab7b006
- Or Nachmias, "Map Matching: The Viterbi Snapper," Gett Engineering. Viterbi picking the single most likely sequence of road edges for a whole trace. https://medium.com/gett-engineering/map-matching-the-viterbi-snapper-e32d11f0d130
- "Designing Uber," High Scalability. Overview of Uber's real-time dispatch and location architecture, Ringpop, geo-sharding, persistent connections. https://highscalability.com/designing-uber/
- Ndriqim Muhadri, "The Tech Behind Uber's Smooth Real-Time Map Experience," Medium. Client-side interpolation and dead reckoning between sparse GPS pings. https://medium.com/@ndriqim.muhadri99/the-tech-behind-ubers-smooth-real-time-map-experience-55f4cccee2b2
