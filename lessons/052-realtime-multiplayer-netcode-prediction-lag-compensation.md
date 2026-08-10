# Day 52 — How does a shot in a competitive shooter feel instant when the server is 50 to 150 milliseconds away?

**Date:** 2026-08-10
**Difficulty:** Expert
**Topic:** Real-time competitive multiplayer netcode: why a purely server-authoritative design feels laggy no matter how fast the server is, and how client-side prediction, entity interpolation, server reconciliation, and lag compensation (rewinding server-side history to the instant the shooter actually saw) combine to make a networked game feel local, via Valve's 2001 lag-compensation paper (the industry's founding primary source on this problem) and Blizzard's public 2017 GDC talk on Overwatch's netcode.
**Stack relevance:** Rare.lab's runtime keeps one shared WebGL context per session. The moment that runtime supports more than one live participant, a collaborator scrubbing a shared preview, a co-editor tweaking a shader parameter while someone else watches, or a "replay this exact frame" debug view, it inherits this exact problem: the person doing the interacting needs their own change to feel instant, everyone else watching needs smooth motion instead of teleporting snapshots, and any feature that says "show me what this looked like a moment ago" is structurally the same rewind-and-evaluate operation lag compensation performs, just applied to shader state instead of player position.

---

## 1. The company and the breaking number

**Valve**, in Yahn Bernier's 2001 paper *"Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization,"* presented at the Game Developers Conference and later folded into the Valve Developer Community documentation for the Source engine (the engine under Half-Life, Counter-Strike, and Team Fortress). This paper is the field's founding primary source: it is the first widely circulated, vendor-published account of *lag compensation*, rewinding the server's copy of the world back to the exact moment a shot was fired, from the shooter's point of view, before deciding whether it hit. The Source engine's default configuration keeps **up to one full second of every player's position and animation history** on the server (the `sv_maxunlag` setting) specifically so this rewind has somewhere to rewind to.

**Blizzard**, in Timothy Ford's March 2017 GDC talk *"'Overwatch' Gameplay Architecture and Netcode,"* gave the modern, quantified version of the same problem. The number that breaks a naive design here is not exotic, it is almost embarrassingly simple: **before a 2017 patch, Overwatch's PC client received a fresh copy of the world only 20 to 21 times a second, a new packet roughly every 50 milliseconds, even though the server itself was already simulating at somewhere around 62.5 to 64 ticks a second (about 15 to 16 milliseconds per tick).** The server was doing the work nearly four times faster than the network was willing to report it. That 50-millisecond gap was a hard floor on how stale every other player's position looked on your screen, and it existed before counting a single millisecond of actual network latency. Ford's own account of the fix (a 2017 "high bandwidth" PC option raising the client update rate to 60Hz) states it shaved roughly 20 milliseconds off interpolation delay and roughly another 20 milliseconds off processing time, about 40 milliseconds combined, for a player with an otherwise good connection.

Stack that architectural floor on top of ordinary network reality and the number that actually breaks things comes into focus. Round-trip time to a nearby matchmaking server for a healthy home connection commonly sits in the 30 to 80 millisecond range; human perception starts registering delayed feedback as "not instant" once total round-trip response time crosses roughly 100 milliseconds. A design that waits for a full server round trip before showing the player anything, the naive version, adds that 30-to-80-millisecond RTT directly on top of render latency and input latency, before a single dropped packet or a single player with worse-than-average internet is even in the picture. The naive design is already over budget on a good day.

## 2. Why the naive (demo) design dies

**Version one: the client sends every keypress and mouse click to the server, waits for the server's authoritative reply, and only then renders anything.** This is the version anyone builds first because it is obviously correct: the server is the single source of truth, so wait for it. It fails on three separate axes.

**Every input has the full round trip baked into it as visible lag.** Press the trigger, wait 30 to 150 milliseconds (one way to the server, simulate, one way back) before the gun even appears to fire on your own screen. In a genre where reaction windows are measured in tens of milliseconds, that delay is not a minor rough edge, it is the whole game feeling broken, and it gets strictly worse for every player who is not sitting next to the datacenter.

**Rendering other players straight off arriving network packets is jittery, not just slow.** Packets do not arrive on a metronome; network jitter means the gap between one update and the next varies. Draw every other player exactly where their latest packet says, and instead of smooth motion you get a stutter-step: stand still, teleport forward, stand still, teleport forward, because the renderer has no idea when the *next* packet is coming and has nothing to interpolate toward in the meantime.

**Checking a hit against the target's live position is unfair in a way players can feel.** If the shooter's screen is rendering the target where they were a render-frame and an interpolation-delay ago, and the server checks the shot against the target's actual current position, a shot that looked dead-on to the shooter reports as a miss, because in the time the shot's packet was in flight the target kept moving on the server's clock. This is not a rare edge case, it is what happens on essentially every shot against a moving target, all the time, and it is the single biggest source of "that should have hit" complaints in every shooter that has ever shipped without solving it.

**Broadcasting full world state to every client every tick does not scale with entity count.** If the server's answer to "keep everyone in sync" is "send everyone a complete copy of every object's position every tick," bandwidth grows with player count times entity count times tick rate. That is fine at Overwatch's roughly 6-to-12-entity match scale; it collapses immediately in a title with hundreds of simultaneous entities (a battle royale, an RTS) unless something smarter than "send everything" is done.

## 3. The architecture

```
[Client: input capture — mouse, keyboard, or a UI drag, stamped with a
 monotonically increasing input sequence number]
   analogy: numbering every order slip the instant it's written, before
   it ever leaves the counter
   |
   v
[Client-side prediction: apply the input to a LOCAL copy of the
 simulation immediately, render the result before the server has
 even seen the packet]
   analogy: an autocomplete that shows your keystroke on screen the
   instant you type it, not after a spellchecker server confirms it
   |
   v
[Network: input sent to server tagged with its sequence number; NOT
 waited on before rendering — the send is fire-and-forget from the
 renderer's point of view]
   |
   v
[Server: fixed-tick authoritative simulation (e.g. ~64Hz), applies
 every client's inputs in sequence-number order for that tick — the
 ONLY copy of the world anyone is allowed to disagree with]
   analogy: the scoreboard official who alone decides what counted,
   no matter what any player's own stopwatch says
   |
   v
[Lag-compensated hit resolution: for a hit-scan action, rewind OTHER
 entities' positions back to how they looked at the shooter's own
 timestamp (bounded, e.g. up to Source's 1-second cap), resolve the
 hit against that rewound snapshot, then discard the rewind]
   analogy: reviewing a play by rolling the game-film back to the
   instant the whistle should have blown, judging it there, then
   returning to live play
   |
   v
[Snapshot broadcast: server sends the resulting state, delta-
 compressed against the last snapshot that client acknowledged, not
 a full copy of the world]
   analogy: mailing only the changed lines of a form, not retyping
   and resending the whole document every time
   |
   v
[Other clients — entity interpolation: buffer the last two-plus
 snapshots and render every OTHER player deliberately slightly in
 the past, smoothly interpolating between known points instead of
 snapping to each new packet]
   analogy: a subtitle track that trails the audio on purpose, by a
   fixed small delay, so the words never have to jump or overlap
   |
   v
[Owning client — server reconciliation: when the server's ack for
 YOUR OWN input sequence number arrives, compare it to what the
 local prediction guessed; if they match, discard the old predicted
 state; if they diverge, snap to the server's answer and REPLAY every
 locally-predicted input newer than that ack on top of it]
   analogy: a bank statement arriving and you replaying every purchase
   you made since the last statement on top of the corrected balance,
   instead of assuming your checkbook was right
```

Two structural choices carry the real weight here.

**Authority and experience are deliberately split across two different clocks.** The server's clock is authoritative and always in the present. The owning client's clock is allowed to run *ahead* of the server (predicting a result before the server has confirmed it), and every other client's clock is deliberately running *behind* the server on purpose (interpolation delay). Nobody, including the server's own hit-resolution step, is looking at "now" from the same point in time. That sounds like it should be chaotic; it works because each of the three clocks is solving a different problem: the owning client's clock optimizes for feeling instant, the other clients' delayed clock optimizes for smoothness, and the server's rewound clock optimizes for fairness to whoever pulled the trigger.

**Prediction is corrected by replay, not by teleporting.** A naive correction would just snap the client to whatever the server said. Ford's talk (and Gambetta's public write-up of the same pattern) both describe replaying the unacknowledged inputs on top of the corrected state instead: the client keeps a small buffer of "things I've sent but haven't heard back about yet," and after a correction, re-simulates just those, so the player ends up in the position their more-recent inputs actually justify, not one input-sequence-number behind. This is the same shape as replaying a write-ahead log's tail after restoring from a checkpoint: take the last known-good state, then deterministically re-apply everything since.

## 4. The transferable mechanisms

**Client-side prediction, optimistic UI applied to physics.** Show the effect of the user's own action before the authoritative system has confirmed it, on the assumption that "confirmed" and "predicted" will usually agree. This is the exact same trust decision an optimistic UI checkout button makes (show "order placed" before the payment has actually cleared) applied to movement and aim instead of a purchase. The generalizable rule: predict locally only for the actor whose own action it is, they have the context (their own intent) to predict correctly; never predict on someone else's behalf, because you're guessing, not reading their mind.

**Server reconciliation by replay, not by snapping.** Keep a small ordered buffer of "sent but not yet acknowledged" actions, and on a correction, re-derive state by replaying that buffer on top of the authoritative baseline instead of discarding local progress. Any client that speculatively acts ahead of a slower source of truth (an offline-first note-taking app reconciling with a sync server, a form that submits optimistically and later gets a validation error) can use this same buffer-and-replay shape instead of a jarring full reset.

**Entity interpolation, a deliberate, constant, small delay bought on purpose to buy smoothness.** Rendering other players slightly in the past, instead of snapping to each new packet, is the visual equivalent of a jitter buffer in VoIP or video calling: you are trading a small, fixed, and controllable amount of latency for the elimination of a much worse, unpredictable amount of visible stutter. The general lesson: when raw data arrives unevenly, buffering a *little* on purpose is often a better trade than reacting to every packet the instant it lands.

**Lag compensation, a bounded rewind for authoritative reads.** The server keeps a short rolling history and, for one specific class of decision (did this hit land), reads that history at a timestamp in the recent past instead of reading current state. This generalizes to any "was this valid at the moment it was attempted" check: time-travel debugging, "show me this record as of the timestamp on the request" audit reads, or a shared canvas answering "what did this look like when they clicked." The bound matters as much as the mechanism, Source caps the rewind at one second precisely so the cost and the fairness window both stay finite.

**Fixed tick, sequence-numbered input as the ordering key.** The server does not process inputs whenever they happen to arrive, it buckets them into a fixed-rate tick and applies them in sequence-number order, so two clients' inputs that both claim to have happened "now" get a deterministic, reproducible resolution instead of a race. This is the same job an idempotency key or a Kafka partition offset does elsewhere in this ledger: give every event a position in one agreed total order before anyone acts on it.

**Delta-compressed, relevance-filtered snapshots instead of full-state broadcast.** Sending only what changed since the client's last acknowledged snapshot, and only for entities that actually matter to that client, keeps bandwidth from scaling linearly with total world size. This is the same instinct as sending a diff instead of a full file, or paginating a feed instead of shipping the whole timeline on every request.

## 5. The trade-offs

**The player's own movement and aim: availability over consistency, resolved locally first.** The owning client renders its own predicted outcome immediately and only reconciles later if the server disagrees. This is a deliberate choice to feel responsive at the cost of occasionally being visibly wrong for a frame or two (a small correction snap), the same shape as accepting a cart addition immediately and squaring up inventory truth afterward.

**Hit registration and damage: never trusted from the client, always the server's call.** Unlike movement, the server is the sole source of truth here, and the client's prediction of "I hit them" is treated as a guess to be checked, not a fact to be believed. This is as much a security boundary as a consistency one: a client that could unilaterally decide "that was a hit" is a client that can cheat. Lag compensation exists specifically so the server can be both the sole authority on hits *and* fair to a shooter whose view of the world was, correctly, a little old.

**Cost vs. latency: rewinding history and raising tick rate both cost real server resources, spent unevenly on purpose.** Keeping a second of position and animation history per entity is memory and CPU that has to be paid on every tick, for every player, whether or not a shot is ever fired that needs it. Raising client update rate from 20Hz to 60Hz roughly triples outbound bandwidth per connection. Neither is free, and neither is spent uniformly: Overwatch's own tournament configuration reportedly runs a tighter tick (community accounts citing Ford's talk put it near a 7-millisecond tick, well under half the live 15-to-16-millisecond tick) specifically for competitive play, where the cost of extra precision is worth paying, while ordinary matchmaking runs the cheaper default. Spend the expensive version only where the stakes justify it.

**"Favor the shooter" is a product decision wearing a technical costume.** Lag-compensated hit resolution is symmetric by construction, every player's shots get rewound the same way, but it is *felt* asymmetrically: the shooter experiences "my shot landed, and it should have," while the target sometimes experiences "I died after I was already behind cover," because the hit was resolved against where they were, not where they currently are. Both experiences are the correct output of the same rule. Choosing that rule at all, rather than something friendlier to the target, is a stated design preference (favor the person taking the action, not the person reacting to it), not a technical inevitability, and it is worth naming as a choice rather than treating it as physics.

## 6. The systems-thinking lens

The feedback loop worth naming here is **misprediction begets worse misprediction, a rubber-banding spiral**, not a thundering herd.

Picture the mechanism: a player's connection has a rough patch, a few packets are late or lost. Their client's predicted state and the server's authoritative state start to diverge more than usual, which means reconciliation corrections get bigger and more frequent, which shows up on screen as visible snapping, "rubber-banding." A player who sees that instinctively reacts, jiggling input, mashing a movement key, sometimes even their client-side network stack retrying more aggressively, and all of that adds more queued, not-yet-acknowledged inputs to the replay buffer described in section 3. A longer unacknowledged-input buffer means the *next* correction, when it comes, has more inputs to replay on top of it, which takes longer to converge and is more likely to diverge again before it settles. The rough patch that started the spiral is long over; the correction mechanism itself is now what's sustaining the visible chaos.

Buying a faster server does not touch this loop, the divergence is happening on the client's own predicted timeline relative to a server that may be performing fine. The senior fix breaks the correlation between "a correction happened" and "the next correction gets worse":

- **Smooth corrections over several frames instead of a single instant snap.** Closing the gap between predicted and authoritative state gradually, rather than teleporting to the correct answer in one frame, keeps a correction from itself becoming a jarring new input the player reacts to.
- **A jitter buffer sized to absorb short gaps without triggering a reaction.** Entity interpolation already buffers a little on purpose (section 4); sizing that buffer to comfortably cover ordinary jitter means a brief rough patch doesn't visibly manifest at all, so there's nothing for the player, or an overeager client retry loop, to react to.
- **Bound the replay buffer and the rewind window, not just the retry rate.** Capping how many unacknowledged inputs will be replayed in one reconciliation, the same instinct as Source's fixed one-second lag-compensation cap, keeps a bad stretch of connectivity from producing an unbounded amount of catch-up work on the very tick where the connection is already struggling most.

The general principle again: when the mechanism built to recover from a problem is itself capable of producing a bigger version of the same problem, correction feeding future correction, the fix is to dampen and bound that mechanism, not to add capacity to the systems around it. A faster server does not stop a rubber-banding client from rubber-banding harder.

---

## Sources

- [Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization, Yahn Bernier, Valve/GDC 2001 (PDF mirror)](http://web.cs.wpi.edu/~claypool/courses/4513-B03/papers/games/bernier.pdf)
- [Lag Compensation, Valve Developer Community wiki](https://developer.valvesoftware.com/wiki/Lag_Compensation)
- ['Overwatch' Gameplay Architecture and Netcode, Timothy Ford, GDC 2017 (GDC Vault listing)](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)
- [Overwatch gets 'high bandwidth' patch for PC, tick-rate discussion, VentureBeat/GamesBeat](https://gamesbeat.com/overwatch-netcode-tick-rate-60hz/)
- [Overwatch's Low Tick Rate Can Be A Problem, Kotaku](https://kotaku.com/overwatchs-low-tick-rate-can-be-a-problem-1778708854)
- [Fast-Paced Multiplayer: Client-Side Prediction and Server Reconciliation, Gabriel Gambetta](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
- [Fast-Paced Multiplayer: Entity Interpolation, Gabriel Gambetta](http://www.gabrielgambetta.com/fpm3.html)
- [What Every Programmer Needs To Know About Game Networking, Glenn Fiedler, Gaffer On Games](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/)

---

*Inference vs. fact, stated plainly: Valve's one-second `sv_maxunlag` rewind window and Bernier's original 2001 paper are drawn directly from Valve's own published documentation. Overwatch's roughly 62.5-to-64Hz server tick, the pre-2017 20-to-21Hz PC client update rate, and the roughly 40-millisecond combined interpolation-plus-processing improvement from the 2017 high-bandwidth patch are drawn from secondary reporting (GamesBeat/VentureBeat, Kotaku, and community accounts quoting Timothy Ford's GDC 2017 talk) rather than a directly-viewed primary transcript, since the GDC Vault video itself was not accessible while researching this lesson; the tournament-configuration tick figure is sourced the same secondary way and is the softest number in this lesson. The rubber-banding feedback loop and the specific claim that it, rather than a thundering herd, is the dominant systemic failure mode here are this lesson's own reasoned synthesis of how the documented prediction-and-reconciliation mechanism would behave under degraded network conditions, not a quoted incident report from either company.*
