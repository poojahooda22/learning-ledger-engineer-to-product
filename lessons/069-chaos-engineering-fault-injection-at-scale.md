# Day 69 — If your failover code has never actually run, how do you know it works?

**Date:** 2026-09-02
**Difficulty:** Expert
**Topic:** Chaos engineering: the discipline of deliberately, continuously breaking small, controlled pieces of production to prove a resilience assumption true instead of hoping it is. This ledger has built most of the pieces this lesson assembles into a single practice. Day 68 built a guardrail metric wired straight to a kill switch, for a rollout a human starts. Day 13 named the metastable failure loop and the rule that adding capacity cannot fix a feedback loop. Day 14 covered multi-region active-active as the way to survive losing a whole region. Day 34 covered cell-based architecture and shuffle sharding as the way to cap how much of a fleet one failure can reach. Chaos engineering is the practice that asks, on a schedule, before an incident forces the question: does all of that actually work, right now, on this system, under real traffic, or does it just look like it should on a whiteboard?
**Stack relevance:** Rare.lab's embeddable runtime shares one WebGL context per page across everything rendered on it. That single shared context is the blast-radius risk this lesson is built to find, before a customer's GPU driver finds it first: if a scene node or a compiled shader misbehaves badly enough to trip a WEBGL_lose_context event, or a real GPU driver timeout does it involuntarily, does Rare.lab's recovery code actually rebuild every shader program, texture, and buffer bound to that context, correctly, live, on someone else's website, with no rebuild and no redeploy? Today the honest answer is "we believe so, because we wrote the recovery path." The next ceiling is turning that belief into a measurement: deliberately firing simulated context-loss events, on a small percentage of live embeds, wired to Day 68's guardrail metrics (context loss rate, frame time regression, compile failures), and finding out on a Tuesday afternoon with an engineer watching, not during a customer's product launch with nobody watching at all.

---

## 1. The company and the breaking number

**The number hiding inside any large fleet's own SLA.** AWS's own Compute Service Level Agreement states two different numbers: a single EC2 instance is committed to 99.5 percent uptime, while a fleet spread across multiple availability zones in a region is committed to 99.99 percent. Take the honest, unglamorous 99.5 percent single-instance number and ask a simple question: at a given moment, across a fleet of independently-failing instances, what is the probability that every single one of them is up at once? At 100 instances, that probability is 0.995^100, about 60.6 percent, meaning close to 4 in 10 moments something in that fleet of 100 is already down or degraded. At 1,000 instances, 0.995^1000 is about 0.66 percent, meaning 99.3 percent of the time, right now, something in that fleet is broken. At 10,000 instances, the probability every instance is healthy simultaneously rounds to zero. This is not a rare edge case a team might get unlucky and hit. At real fleet scale, "something is currently broken" is not the exception, it is the ambient, permanent condition of the system, and the only real question is whether the rest of the system was built assuming that or built hoping around it.

**Netflix, 2010, and the migration that made hoping impossible.** Netflix began moving its systems onto AWS starting in 2010, and in that environment, unlike Netflix's own datacenters, hosts could be terminated and replaced by the cloud provider at any time, with no warning and no negotiation. Netflix engineer Greg Orzell built Chaos Monkey that same year: a tool with the single job of randomly terminating production EC2 instances during business hours, on purpose, so that any assumption of "that server is always there" would be forced to break in a controlled way, with engineers watching, instead of breaking on its own, unwatched, during an actual incident. On July 19, 2011, Netflix publicly announced the Simian Army, a family of tools built on the same idea aimed at different failure types (a Latency Monkey that injected artificial delays, a Conformity Monkey that flagged instances not following best practices, and more). Yury Izrailevsky, Netflix's former VP of Cloud and Platform Engineering, summarized the philosophy in one line that the whole practice is built around: "The best way to avoid failure is to fail constantly."

**The proof this was not theater: a hand-patched "snowflake" server, found by the tool built to find exactly this.** IEEE Spectrum's retrospective on the practice describes a real incident Chaos Monkey surfaced early on: a manually, hand-patched server was quietly responsible for synchronizing DNS configuration, a piece of undocumented, un-automated infrastructure nobody had flagged as special. When Chaos Monkey terminated it, as it terminates any instance, AWS replaced it with a fresh instance that did not carry the hand-applied settings, and the gap became visible immediately, during business hours, with the team that owned it already looking. That is the entire value proposition of the practice in one incident: a fragile, undocumented dependency that would otherwise have been discovered for the first time during a real 3 a.m. failure was instead discovered on a Tuesday afternoon, by the team that could fix it, at the moment they chose.

**The sobering counter-example: chaos engineering existing did not save Christmas Eve 2012.** Netflix's own multi-hour streaming outage on December 24, 2012 was caused by an AWS engineer's maintenance process accidentally running against, and deleting, the production state data behind AWS's own Elastic Load Balancers in the US East region. Netflix's TechBlog account of the incident is direct about what this exposed: Chaos Monkey and the Simian Army of the time tested instance-level and service-level failure, not the failure of a shared control-plane dependency like ELB state itself, and not the failure of an entire region. The outage lasted roughly 20 hours before full resolution. It is the reason Netflix went on to invest heavily in the multi-region active-active architecture this ledger covered on Day 14, and the direct ancestor of Chaos Kong, described below, which tests exactly the failure category Christmas Eve 2012 exposed: not "can this instance disappear," but "can this entire region disappear," on purpose, before it does so by accident.

---

## 2. Why the naive (demo) design dies

**The obvious version:** write resilience into the code (retry logic, a failover path, a circuit breaker, a documented runbook), then verify it the way most teams verify anything: read the code, maybe write a unit test against a mocked failure, and schedule a disaster-recovery drill once a year, in a maintenance window, on staging, with everyone told in advance exactly what is about to happen and when.

**Death one: an unexercised failure path rots silently, and nothing tells you it happened.** Retry logic gets refactored six months later by someone who does not know it is load-bearing. A dependency gets upgraded and its timeout behavior changes. A runbook references a tool that was renamed or deprecated. None of this shows up in any dashboard, because the path is never actually walked, so there is nothing to alert on. The first time that path is genuinely exercised is the real incident, at real scale, with the team discovering in the middle of the outage that the thing they were counting on quietly stopped working months earlier.

**Death two: no blast radius means the first real test is the worst possible test.** Meta's own engineering account of its October 4, 2021 outage is a clean case study in this exact failure. A routine maintenance command intended to assess the capacity of Facebook's global backbone network contained an error, and the audit tool whose specific job was to catch that kind of error, before it could take effect, had a bug of its own that let the erroneous command through. The result, per Meta's own post: the entire global backbone was withdrawn at once, Facebook's DNS servers correctly judged their own network unreachable and withdrew their BGP advertisements in response, and Facebook, Instagram, and WhatsApp went fully offline worldwide for more than seven hours, from roughly 15:40 to 22:45 UTC. Worse, the outage reached deep enough into Meta's own internal tooling and physical building access systems that engineers were reportedly slowed down getting into the datacenters that needed hands-on recovery. Nothing about this command had ever been exercised at a small, scoped blast radius first; when it broke, by construction, it broke everywhere, including the tools meant to fix it, simultaneously.

**Death three: a once-a-year drill on staging tests a system that no longer exists by the time it matters.** Real production traffic patterns, real data skew, real dependency graphs, and real configuration all drift continuously, week to week. A game day scheduled a year in advance, run against synthetic load in an environment that approximates production, verifies that last year's mental model of the system survives a failure. It says nothing about whether this week's system, with three dependency upgrades and a schema migration since the drill, still does. Confidence built this way is confidence in a snapshot, decaying from the moment the drill ends, with no signal telling anyone how far it has decayed by the time it is actually needed.

---

## 3. The architecture

```
Steady-state hypothesis engine
  - job: define "normal" as a measurable business metric (Netflix's own
    examples: stream starts per second, checkout success rate), and state
    a falsifiable hypothesis, that metric will hold steady in an
    experimental group the same way it holds in a control group
  - analogy: taking a patient's baseline vitals before deliberately
    stress-testing them on a treadmill, so "worse than baseline" has a
    number attached instead of a feeling

        |
        v
Experiment scheduler + blast-radius scoping
  - job: pick a small, bounded slice of real production traffic to run
    the experiment against, the same percentage-bucketing-by-hashed-key
    mechanism Day 68 uses for a canary rollout, aimed here at "who gets
    hit with the failure" instead of "who gets the new feature"; Netflix's
    ChAP platform caps this hard, no combination of concurrently running
    experiments may affect more than 5 percent of total traffic in any
    one of Netflix's three geographic regions
  - analogy: running a fire drill on one specific floor of one specific
    building, not pulling the alarm for the whole city

        |
        v
Fault injection layer (proxy/sidecar or SDK level)
  - job: actually cause the failure being tested, an instance
    termination, injected network latency, a dropped dependency call, a
    forced GPU context loss, on the real, scoped slice of traffic, not a
    mocked stand-in for it; Netflix's own principle is to vary
    real-world events specifically because a mocked failure only proves
    the mock was handled correctly
  - analogy: a stunt coordinator who can cut one specific actor's mic on
    cue, not someone who pulls the building's main power breaker

        |
        v
Automated abort trip-wire (reused from Day 68's guardrail-to-kill-switch)
  - job: watch the real customer-facing metric the experiment could hurt,
    and stop the experiment immediately, automatically, the instant that
    metric crosses a line, with no human required to notice first;
    ChAP's own account is explicit that it halts an experiment early the
    moment it detects excessive customer impact
  - analogy: a dead-man's switch on the treadmill, not a technician who
    has to be watching the heart-rate monitor at the exact right second

        |
        v
Control-vs-experiment comparison
  - job: compare the scoped experimental slice against an equivalent
    control slice receiving the same real traffic without the injected
    fault, so "did this actually hurt anyone" is a measured difference,
    not an impression
  - analogy: an A/B test, but the variant being tested is "what happens
    when this breaks," not a new button color

        |
        v
Continuous, automated re-scheduling
  - job: run this again, automatically, regularly, forever, not once a
    year in a calendar-blocked drill, so bit rot in a failure path is
    caught within days of being introduced, not years after
  - analogy: a fire drill on a random floor every week, not a single
    company-wide drill announced a month in advance once a year
```

---

## 4. The transferable mechanisms

- **State a steady-state hypothesis before you break anything.** "Normal" has to be a number (a request success rate, a stream-start rate, a frame time budget) before an experiment can say whether it was violated. Without this, "did that hurt" is a matter of opinion after the fact instead of a measurement during the test.

- **Scope the blast radius, and make the scoping mechanism the same one you already trust for rollouts.** Netflix's own ChAP platform reuses the exact idea Day 68 covers for feature-flag percentage rollouts, bucket real traffic by a hashed key into a small experimental slice, capped hard (5 percent of a region's total traffic across every concurrently running experiment). The mechanism that makes a rollout safe to widen gradually is the same mechanism that makes a deliberate failure safe to inject at all.

- **Inject the real event, not a mock of it.** A unit test against a mocked timeout proves the code path that handles a mocked timeout works. It proves nothing about what happens when a real GPU driver actually times out, a real instance is actually terminated mid-request, or a real BGP route is actually withdrawn. Netflix's own principle is explicit: vary real-world events. The value of the practice comes entirely from testing the thing that can really happen, not a stand-in for it.

- **Wire the abort to the metric automatically, not to a human noticing.** The same structural move Day 68 makes for a bad feature rollout applies here: an automated trip-wire tied directly to a real guardrail metric reacts in the time it takes to evaluate a threshold, not the time it takes a human to notice a dashboard, get paged, and act.

- **Run it continuously and automatically, in production, forever.** A resilience claim that was true last year, verified once, in staging, says nothing about whether it is still true this week, in production, after three dependency upgrades nobody thought to re-test against. Netflix's own fourth principle of chaos engineering is exactly this: automate experiments to run continuously. Continuous, automated exercise is what turns "we believe this works" into "we know this worked as of this morning."

- **Cap blast radius structurally, at the architecture level, as the backstop underneath the experiment.** Chaos Kong, sitting above Chaos Monkey in Netflix's own tooling hierarchy, tests the failure of an entire AWS region by evacuating real traffic out of it on purpose. Running that experiment repeatedly, ahead of any real regional outage, is exactly what let Netflix confirm its Day 14 multi-region active-active architecture could actually absorb a whole region disappearing, before a real one did, and Day 34's cell-based blast-radius containment is the same structural idea one level down, inside a single region.

---

## 5. The trade-offs

**A small, controlled outage now, on purpose, versus a large, uncontrolled one later, by accident.** Chaos experiments run during business hours, deliberately, specifically so that if something genuinely goes wrong, the engineers who understand the system are already awake, already at their desks, and already watching. ChAP's own operational constraint bears this out directly: Netflix restricts its automated chaos experiments to weekdays, 9 a.m. to 5 p.m. The cost is accepting some nonzero number of small, real, customer-visible degradations that would not have happened at all if nobody had run the experiment. The trade is against the alternative: the first real exercise of that same failure path happening at 2 a.m., unscoped, with nobody watching, and no control group to compare against.

**A narrow blast radius is safer per experiment, but a system tested only at 5 percent of traffic has not proven anything about the other 95 percent.** Netflix's answer to this is not to choose one scope and stop: it runs frequent, narrow, automated ChAP-style experiments continuously for everyday resilience claims, and periodically runs Chaos Kong at full region scale specifically because some failure modes, real cascading load, real capacity limits, genuinely only appear near real total load, and no 5-percent-scoped experiment will ever surface them.

**Building and staffing this is a real, visible cost, against a benefit that is invisible until the day it matters.** Fault-injection tooling, an on-call culture willing to treat "the chaos experiment tripped a guardrail" as useful information instead of an embarrassment, and engineering time spent writing and maintaining experiments, all cost real money and show up on no uptime SLA line item, right up until the day a would-be Christmas-Eve-2012 or October-2021-scale incident is instead a five-minute blip an engineer watched happen at 2 p.m. on a Tuesday.

**Confidence is not the same axis as correctness, and chaos engineering exists to keep those two from drifting apart.** A team's confidence in its own resilience tends to rise the longer nothing visibly breaks. Actual resilience, absent any exercise of the failure paths that back that confidence, tends to fall over the same period, as dependencies drift and refactors quietly remove guarantees nobody re-checks. The trade this practice makes is spending real engineering effort keeping those two lines close together, instead of letting confidence run ahead of reality until an incident forces a correction.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is different in shape from Day 13's metastable failure loop, but just as dangerous: it is a loop in what a team believes, not in what a system's resources are doing. It runs like this: nothing visibly breaks, so confidence in the system's resilience rises; rising confidence makes it feel less urgent to exercise the failure paths that confidence is actually resting on; the less those paths are exercised, the more they silently drift out of date, dependencies upgrade, retry logic gets refactored, runbooks reference tools that no longer exist, safety checks accumulate their own unexamined bugs; and none of that drift produces any signal, because the path that would reveal it is exactly the path nobody is exercising. Confidence keeps climbing while true readiness keeps falling, and the gap between the two is invisible by construction, right up until an incident forces the untested path to run for real. Meta's own account of its October 2021 outage names this loop almost exactly: the tool built to prevent that specific class of error existed, was trusted, and had a bug in it that nobody had found, because nothing had forced that specific tool to actually catch a real erroneous command in a long time.

The naive fix, "review the runbook again" or "double-check the failover code by reading it," does not break this loop, because a read-through produces the same false confidence a stale runbook already had: it looks correct, on paper, to the person reading it, which is precisely the condition that was already true right before both Christmas Eve 2012 and October 4, 2021. The senior fix is structural, in the same shape Day 13 insists a retry storm has to be broken structurally rather than argued down: stop treating "we have not seen it break lately" as evidence that it works, and replace it with a mechanism that forces the failure path to run, for real, on a schedule, at a small controlled blast radius, with an automated trip-wire watching. That converts an assumption that can silently rot for years into a claim that gets re-verified continuously, so if it stops being true, the gap between belief and reality is discovered within days, by an engineer watching a scoped 5-percent experiment on a Tuesday afternoon, instead of within seven hours, by the entire world, during the real thing.

---

## Sources

- [Chaos Monkey at Netflix: the Origin of Chaos Engineering, gremlin.com](https://www.gremlin.com/chaos-monkey/the-origin-of-chaos-monkey): source for Netflix's 2010 AWS migration as the trigger for Chaos Monkey, Greg Orzell's creation of the tool, the July 19, 2011 public announcement of the Simian Army, and Yury Izrailevsky's "the best way to avoid failure is to fail constantly" quote.
- [What Is Chaos Monkey? A Complete Guide, gremlin.com](https://www.gremlin.com/chaos-monkey): general background on Chaos Monkey's mechanism (random production instance termination during business hours) and its role as the origin point of the broader chaos engineering practice.
- [Chaos Engineering Saved Your Netflix, IEEE Spectrum](https://spectrum.ieee.org/chaos-engineering-saved-your-netflix): source for the hand-patched "snowflake" DNS-configuration server incident, surfaced by Chaos Monkey terminating it and AWS replacing it with an unpatched instance, as a concrete early proof of the practice's value.
- [A Closer Look at the Christmas Eve Outage, Netflix TechBlog](https://netflixtechblog.com/a-closer-look-at-the-christmas-eve-outage-d7b409a529ee): primary Netflix account of the December 24, 2012 outage, its root cause (a maintenance process that deleted production AWS ELB state data), and the roughly 20-hour resolution window; source for the framing that this incident exposed a failure category (control-plane and regional failure) that instance-level chaos testing of the time did not cover.
- [Updated: Netflix Crippled On Christmas Eve By AWS Outages, TechCrunch](https://techcrunch.com/2012/12/24/netflix-crippled-on-christmas-eve-by-aws-outages/): independent contemporaneous corroboration of the Christmas Eve 2012 outage timeline and scope.
- [Principles of Chaos Engineering, principlesofchaos.org](https://principlesofchaos.org/): primary source for the four core principles cited (build a hypothesis around steady-state behavior, vary real-world events, run experiments in production, automate experiments to run continuously) and the blast-radius-control practice of starting small and expanding only after confirming safety controls work.
- [Chaos Engineering (IEEE Software, 2017, arXiv preprint), arxiv.org](https://arxiv.org/pdf/1702.05843): the Netflix-authored paper formalizing chaos engineering as a discipline; source for the steady-state hypothesis framing used throughout this lesson.
- [ChAP: Chaos Automation Platform, Netflix TechBlog (Medium)](https://medium.com/netflix-techblog/chap-chaos-automation-platform-53e6d528371f): primary source for ChAP's control-versus-experimental-group traffic split, the hard cap of no more than 5 percent of total traffic in any one of Netflix's three regions across all concurrently running experiments, and automatic early termination of an experiment on detected customer impact.
- [Automating chaos experiments in production, the morning paper (blog.acolyer.org)](https://blog.acolyer.org/2019/07/05/automating-chaos-experiments-in-production/): secondary summary of the ChAP paper corroborating the 5-percent regional traffic cap and the 9 a.m.-to-5 p.m. weekday operating window for automated experiments.
- [Amazon Compute Service Level Agreement, aws.amazon.com](https://aws.amazon.com/compute/sla/): primary source for the 99.5 percent single-EC2-instance uptime commitment and the 99.99 percent multi-AZ region-level commitment used to derive this lesson's breaking-number math.
- [Update about the October 4th outage, Meta Engineering](https://engineering.fb.com/2021/10/04/networking-traffic/outage/): primary Meta account of the October 4, 2021 outage's proximate cause, a maintenance command auditing global backbone capacity that an audit tool failed to correctly stop.
- [More details about the October 4 outage, Meta Engineering](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/): primary Meta follow-up with the full mechanism, the BGP route withdrawal, the DNS self-withdrawal cascade, and the resulting difficulty accessing physical datacenter systems during recovery.
- [2021 Facebook outage, Wikipedia](https://en.wikipedia.org/wiki/2021_Facebook_outage): cross-check source for the outage duration, approximately 15:40 to 22:45 UTC, more than seven hours.
- [WEBGL_lose_context extension, MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/WEBGL_lose_context): source for the real, standard browser API (`loseContext()`, `restoreContext()`, the `webglcontextlost`/`webglcontextrestored` events) this lesson's Rare.lab application proposes using to deliberately, safely simulate GPU context loss on a scoped slice of production embeds.
- [Resilience Engineering: Learning to Embrace Failure, ACM Queue](https://queue.acm.org/detail.cfm?id=2371297): source for Amazon's own GameDay practice, created in the early 2000s by Jesse Robbins, deliberately injecting major failures into production systems on a schedule to find flaws before an incident does.
- Day 13 (this ledger, backpressure and load shedding, and the metastable failure loop), Day 14 (multi-region active-active), Day 34 (cell-based architecture and shuffle sharding), Day 47 (Cloudflare Quicksilver and push-based config distribution), Day 68 (feature flags, the guardrail-to-kill-switch mechanism, and percentage-bucketed rollout by hashed key): the ledger's own prior lessons this one builds directly on, reused here for blast-radius scoping, automated abort trip-wires, and the regional and cell-level containment chaos experiments are ultimately built to verify.

**A note on sourcing for this lesson:** the Christmas Eve 2012, ChAP traffic-cap, and October 2021 Meta figures are each corroborated across a primary company account (Netflix TechBlog, Netflix TechBlog again, and Meta Engineering respectively) and at least one independent secondary source, and are treated as solid. The AWS EC2 SLA percentages are taken directly from AWS's own current published SLA page and the derived probability math (99.3 percent chance at least one instance in a 1,000-instance fleet is down at any given moment) is this lesson's own calculation from that published figure, not a number AWS itself publishes, and should be read as an illustrative lower bound rather than an exact operational statistic, since real instance failures are not perfectly independent events.

---
