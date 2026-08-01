# Day 45 — How does a shooter let you hit a target that, on the server, has already moved?

*2026-08-01*

---

## 1. The company and the number that breaks a naive design

**Riot Games, Valorant, 2020.** Valorant is a tactical shooter where a single bullet decides a round, so "did that shot land" has to be right, not just fast. Riot's developers publicly framed the whole netcode problem around one physical constant they cannot engineer away: **light in fiber travels at about two-thirds the speed of light in a vacuum, so a round trip between a player in Chicago and a data center in Virginia is already 20-30ms before any software touches the packet, and a transcontinental or cross-ocean round trip is 60-150ms.** In that 100ms window, an enemy strafing at normal movement speed has moved roughly a full body-width sideways. A server that just asks "is my shot touching the enemy's hitbox right now" will miss constantly, because "right now" on the server is not "right now" on your screen, it is 50-150ms in your screen's past.

Riot's answer was to run its authoritative servers at **128 ticks per second instead of the industry-standard 64**, meaning the server samples and resolves the world state every **7.8 milliseconds instead of every 15.6 milliseconds.** That single change doubles the CPU cost of every match (Riot's engineers said CPU, not memory or bandwidth, was the hard constraint on hosting 128-tick servers at scale) purely to shrink the "peeker's advantage" window, the extra reaction time a player gets by being the one to round a corner, since a lower tick rate lets a peeking player's position update land before the corner-holder's server-side check catches up (Riot Games, "VALORANT's 128-Tick Servers"; GameSpot, "Riot Details Valorant Netcode").

The same physical constraint breaks a completely different naive design just as hard. **Age of Empires (1997)** shipped multiplayer over 28.8k modems using a lockstep model: every player's machine runs an identical simulation, and to guarantee everyone sees the same game, every machine must wait to receive every other machine's commands before it is allowed to advance to the next turn. The breaking number there, from the team's own postmortem: **one player with a slow or lagging connection stalls the entire game for every other player, every single turn**, because lockstep has no concept of "proceed without you" (Bettner and Terrano, "1500 Archers on a 28.8: Network Programming in Age of Empires and Beyond," GDC 2001).

## 2. Why the naive (demo) design dies

The demo version of "multiplayer game" picks one of three naive models, and each one dies a different way.

**a. Pure client-authoritative (trust what the client reports).** The simplest possible multiplayer game just has each client send its own position and "I hit you" events, and every other client trusts them. This dies immediately to cheating: nothing stops a modified client from reporting a position it never occupied, or claiming a hit that never happened. There is no ground truth anywhere in the system, so there is nothing to check a claim against.

**b. Pure server-authoritative with no prediction (wait for the round trip).** The obvious fix is to make the server the only source of truth: the client sends inputs, waits for the server to simulate them, and only then draws the result. This is honest, but it feels terrible. At 100ms round-trip latency, every single keypress takes 100ms to visibly do anything, so movement feels like wading through syrup. This is not a bug that better code fixes, it is the speed-of-light number from section 1 showing up directly in how the game feels to a human hand on a mouse.

**c. Naive lockstep (wait for everyone, every turn).** Age of Empires' fix for a different problem, bandwidth on modems, was to sync commands instead of positions: send "unit 4, move to (x,y)" once, and let every machine simulate the move identically, rather than streaming positions every frame. This works for bandwidth, but it creates a new failure mode: since every machine must have every other machine's command before advancing, **the whole match runs at the speed of its single slowest or laggiest player.** One person's packet loss becomes everyone's stutter. The team's own fix was to buffer input by roughly two turns (about 200ms) so that normal jitter didn't cause a visible stall, but a genuinely dropped or late player still froze the room, the same way a single slow cashier backs up an entire single-file checkout line no matter how fast everyone behind them can shop.

## 3. The architecture, drawn top to bottom

This is the client-server shooter model (Valorant, Overwatch, Counter-Strike), which is the dominant approach when the game needs a server nobody can lie to. A peer-to-peer alternative (rollback) is described separately in section 4, since it solves the same problem with no authoritative server at all.

```
CLIENT
   captures input every frame, immediately simulates it locally
   (CLIENT-SIDE PREDICTION) so movement feels instant, and sends
   the input, tagged with a sequence number, to the server
   analogy: you start walking the instant you decide to, you
   don't wait for a friend on the phone to confirm each step
   |
   v
RIOT DIRECT / PRIVATE BACKBONE (or, generically, anycast + GSLB)
   routes the player to the physically nearest data center,
   because no software trick reduces speed-of-light latency,
   only shorter physical distance does
   analogy: the private backbone is a dedicated highway lane
   that skips the public road's traffic lights, it can't beat
   the distance, only the delay added on top of it
   |
   v
AUTHORITATIVE GAME SERVER, fixed tick loop (128Hz = every 7.8ms)
   the single source of truth: applies every player's inputs in
   the order their timestamps say they happened, runs physics
   and hit detection, and nobody's local view overrides this
   analogy: the referee's whistle, not any one player's opinion,
   decides whether the ball crossed the line
   |
   v
LAG-COMPENSATION HISTORY BUFFER (a ring buffer of past ticks,
capped at about 1 second, e.g. Source engine's sv_maxunlag)
   when a shot arrives, the server rewinds every OTHER player's
   hitboxes back to the exact timestamp the shooter says they
   saw, checks the hit against that rewound position, then
   resumes the present
   analogy: a security camera's instant-replay button, rewind
   to the moment in question, check what actually happened
   there, then let time resume from now
   |
   v
DELTA-COMPRESSED SNAPSHOTS over UDP, not TCP
   the server sends only what changed since the last snapshot
   the client acknowledged, over a channel that tolerates lost
   packets without stalling everything behind them (unlike TCP,
   which blocks all later data until a lost packet is resent)
   analogy: a UDP packet that goes missing is a postcard that
   never arrives, annoying but the next postcard still gets
   through; TCP is a numbered set of boxes that must be opened
   in order, so one missing box jams the whole delivery truck
   |
   v
CLIENT RECONCILIATION
   the client's incoming snapshot may not match what it already
   predicted; the client discards its predicted state, applies
   the server's authoritative snapshot, then REPLAYS every input
   it sent that the server hasn't acknowledged yet, so the
   correction is invisible instead of a visible snap
   |
   v
ENTITY INTERPOLATION for every OTHER player on screen
   rendered deliberately about 100ms in the past, so there are
   always two real snapshots to smoothly draw between, instead
   of guessing forward from just one (extrapolation) and risking
   a visible correction when the guess is wrong
```

## 4. The transferable mechanisms

- **Client-side prediction: act now, correct later.** The local client simulates its own input the instant it happens instead of waiting for a round trip, trading a small chance of being wrong for the removal of all perceived latency on your own actions. This generalizes to any interface with a slow authority behind it: an optimistic UI update in a web app is the same trick, show the "liked" heart immediately, reconcile with the server's real answer a moment later.

- **Server reconciliation: replay unacknowledged work on top of the corrected state.** When the authoritative answer arrives and disagrees with what was predicted, don't just snap to the new value, discard the stale prediction, apply the authoritative state, then re-run every action that hasn't been confirmed yet on top of it. This is the same idea as replaying a database's write-ahead log on top of a restored snapshot: the snapshot is truth, the log is the work that hasn't been durably confirmed, and replaying it is cheaper and less jarring than losing it.

- **Lag compensation: rewind a short history buffer to the requester's timestamp.** Rather than forcing every check to happen against "now," keep a bounded ring buffer of recent past states and let a request specify which past moment it's asking about. This only works because the buffer is short and bounded (about a second here), an unbounded "ask about any point in history" would be a different, far more expensive system.

- **Interpolation over extrapolation for remote state.** Deliberately render other players slightly in the past (about 100ms) so you're always interpolating between two known-real points, instead of extrapolating forward from one point and guessing. This trades a small, constant, predictable added latency for removing the occasional large, unpredictable correction that a wrong guess produces. The same trade appears anywhere a system must display something whose true current value isn't known yet: show a slightly-stale, smooth number over a jumpy, right-now-but-often-wrong one.

- **UDP plus your own reliability layer, not TCP, when order-blocking is worse than occasional loss.** TCP guarantees every byte arrives in order, which means one lost packet blocks everything sent after it (head-of-line blocking) until it's resent. For a stream of "here's where everyone is right now" updates, a single stale, dropped update simply doesn't matter, since a fresher one is already on the way. Choosing the transport's failure mode to match what the data actually needs is the general lesson: don't pay for a stronger guarantee than the data requires.

- **Rollback and resimulation, the peer-to-peer alternative when there's no authoritative server.** GGPO (2009), used across modern fighting games, has no central server: each peer predicts what the other player's next input will be (usually "the same as their last known input"), keeps simulating forward, and only rolls back and deterministically resimulates a few frames when the real remote input arrives and turns out to have been different. This buys the same "act now, correct later" property as client prediction, but achieved through deterministic lockstep simulation plus rollback instead of a server plus reconciliation, useful anywhere a central authority isn't available or affordable (GGPO SDK documentation; "Delta Rollback: New optimizations for Rollback Netcode").

## 5. The trade-offs

- **Consistency vs. availability, and it's different per data type, in the same system.** Your own movement favors immediate local availability: it's shown instantly, might be wrong, and gets silently corrected, because responsiveness matters more than momentary accuracy for your own hands. Whether a shot actually killed someone favors strict server-side consistency: only the authoritative, rewound server check counts, because ranked outcomes and reported stats can't be "eventually consistent." Other players' positions on your screen favor smoothness over immediacy: deliberately rendered 100ms stale so the motion looks clean rather than jumpy. One system, three different consistency choices, each matched to what that specific data is for.

- **Cost vs. latency, spent on physical infrastructure, not just code.** Riot built and operates its own private backbone (Riot Direct) specifically to shave milliseconds off the trip between a player's ISP and its data centers, because past a certain point, no clever software fixes physical distance, only a shorter or less congested physical path does. Doubling tick rate from 64 to 128 roughly doubles per-match CPU cost for a proportionally smaller gain in fairness. Both are cases of spending real, ongoing infrastructure money to buy single-digit milliseconds, a trade that only makes sense because the product is latency-sensitive competitive play, not every product needs to make it.

- **The lag-compensation fairness paradox: it favors the shooter over the target, on purpose.** Rewinding a target's hitbox to the shooter's timestamp means you can be killed by someone you had already, on your own screen, ducked behind cover from. This feels unfair to the target and is a constant source of player complaints, but the alternative (checking hits against the target's current, non-rewound position) makes it nearly impossible to hit anyone who is strafing at realistic latency, which is worse for the game as a whole. Valve and Riot both accept the "shot around a corner" complaint as the cost of hit registration actually working (Valve Developer Community, "Lag Compensation").

## 6. The systems-thinking lens

The feedback loop here is a **metastable failure driven by rollback cost creeping past the frame budget.** Under normal jitter, a rollback-based system mispredicts a few frames, resimulates them cheaply, and nobody notices. But if network jitter or packet loss rises, more predictions turn out wrong, so more frames need resimulating. If that resimulation work is expensive (physics, many entities), the extra work can push a single frame's total compute past its deadline (for example, past 16.6ms at 60fps). A missed deadline delays the next frame's input processing too, which grows the queue of unconfirmed predictions, which means the next real input arrival is even more likely to trigger yet another, now-deeper rollback. Once the resimulation backlog exceeds the frame budget, the system does not recover just because the network jitter that triggered it goes away, the backlog is now self-sustaining. That is the same shape as Day 44's inference-serving metastable failure: a trigger that comes and goes, but an internal backlog that persists after it.

The senior fix does not mean "make resimulation faster and hope," it means bounding the loop so it structurally cannot run away:

- **Cap rollback depth.** GGPO and its successors impose a maximum number of frames that can ever be rolled back at once; past that cap, the system accepts a visible correction rather than letting resimulation cost grow unbounded, the same principle as Day 13's backpressure, refuse to accept unbounded work rather than accept it and stall everyone.
- **Adaptive input delay.** When measured jitter or packet loss rises, increase the number of frames of input buffered before it's applied, which lowers the chance of a misprediction in the first place, trading a small amount of extra fixed latency for a large reduction in expensive rollback events.
- **Keep the resimulation path deterministic and cheap, separate from rendering.** Game logic runs on fixed-point or otherwise deterministic math specifically so that resimulating N frames costs a fixed, predictable amount of CPU, decoupled from whatever the rendering pipeline is doing that frame, so a spike in rollback frequency can't also spike frame-render time.

The general lesson, the same one Day 13 teaches from overload and Day 44 teaches from GPU memory: whenever a system's response to more error (more mispredictions, more retries, more resimulation) is to do more work per unit of error, and that work competes for the same fixed resource the rest of the system needs, a small trigger can produce a backlog that outlives the trigger. The fix is always a cap plus a way to shed or delay gracefully, not raw speed.

---

## Sources

- Riot Games, ["VALORANT's 128-Tick Servers"](https://www.riotgames.com/en/news/valorants-128-tick-servers)
- GameSpot, ["Riot Details Valorant Netcode: 128-Tick Servers And Their Own ISP"](https://www.gamespot.com/articles/riot-details-valorant-netcode-128-tick-servers-and/1100-6475960/)
- Bettner, Paul and Terrano, Mark, ["1500 Archers on a 28.8: Network Programming in Age of Empires and Beyond"](https://zoo.cs.yale.edu/classes/cs538/readings/papers/terrano_1500arch.pdf) (GDC 2001)
- Valve Developer Community, ["Source Multiplayer Networking"](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- Valve Developer Community, ["Lag Compensation"](https://developer.valvesoftware.com/wiki/Lag_Compensation)
- Fiedler, Glenn, ["Networked Physics (2004)"](https://gafferongames.com/post/networked_physics_2004/) and ["Network Neutrality Considered Harmful"](https://gafferongames.com/post/network_neutrality_considered_harmful/), Gaffer On Games
- Gambetta, Gabriel, ["Client-Side Prediction and Server Reconciliation"](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
- GDC Vault, ["'Overwatch' Gameplay Architecture and Netcode"](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and) (Tim Ford, GDC 2017)
- GGPO, [Rollback Networking SDK documentation](https://www.ggpo.net/)
- de Haene, David, ["Delta Rollback: New optimizations for Rollback Netcode"](https://medium.com/@david.dehaene/delta-rollback-new-optimizations-for-rollback-netcode-7d283d56e54b)
