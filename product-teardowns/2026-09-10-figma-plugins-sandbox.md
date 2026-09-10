# Figma Plugins: running a stranger's code inside your design file without getting hurt

Date: 2026-09-10
Product: Figma
Feature: Plugins (running untrusted third-party plugin code fast and synchronously in the browser)

A note on what is confirmed and what is inference. The load-bearing facts here come
from Figma's own engineering post "How to build a plugin system on the web and also
sleep well at night," from co-founder Evan Wallace's follow-up "An update on plugin
security," and from Figma's developer docs "How Plugins Run." Where I describe the
memory and CPU guard rails, and the exact reason the code runs on the main thread
instead of a Web Worker, I label it inference and explain why it is the standard way
this class of problem is solved. Sources are listed at the end.

---

## 1. The user

Meet Priya. She is a product designer at a mid-size fintech, and it is 5pm on a
Thursday. She has just finished a 40,000-layer design system file: buttons, cards,
icons, forty screens of a loan-application flow. Before she hands it to engineering
tomorrow morning, two boring things have to happen.

First, every layer needs a clean name. Right now half of them are called "Rectangle
127" and "Group 44" because that is what Figma auto-names them. Engineering cannot read
that. She needs "btn/primary/default," "card/loan/summary," and so on, across tens of
thousands of layers. By hand this is a full evening of clicking.

Second, the twelve placeholder profile cards on the dashboard screen still say "Name
Surname" with a grey circle where the avatar goes. She wants them filled with realistic
names and faces so the mockup looks alive in tomorrow's review.

Priya is not going to do either by hand. She is going to reach for two plugins: "Rename
It" for the first job and "Content Reel" for the second. She opens the right-click menu,
picks the plugin, and thirty seconds later both jobs are done. She never thinks about
what just happened. What just happened is that Figma ran a program written by a stranger,
inside her browser tab, with full read and write access to her most important file of
the quarter, and nothing went wrong. That "nothing went wrong" is the whole feature.

## 2. The real problem

Here is the thing a friend would tell you over coffee. A design tool can never ship
every feature every designer wants. Someone needs to bulk-rename layers to a convention.
Someone needs to pull real photos from Unsplash. Someone needs to check color contrast
for accessibility (the Stark plugin). Someone needs to draw flow arrows between frames
(Autoflow). If Figma tried to build all of these itself it would take a decade and still
miss yours.

So the obvious answer is: let other people write the features. Open a plugin API, let
the community build. Every serious creative tool did this: Photoshop, Sketch, After
Effects, VS Code.

But there is a trap hiding in that obvious answer, and it is nasty. Figma runs in a web
browser tab. That tab already holds your logged-in session, your auth cookies, and, when
a file is open, the entire contents of a document that might be your company's unreleased
product. A plugin is code written by a stranger on the internet. The moment you run that
stranger's code in your tab, you have handed them a loaded gun pointed at your own file
and your own account, unless you do something clever.

Every other app that added plugins had an easier version of this problem, because they
run on the desktop where the operating system gives you process isolation and permission
prompts for free. Figma had the hard version: run untrusted code safely inside a single
browser tab, where by default all JavaScript on a page can touch everything else on that
page. And it had to be fast, because a plugin that renames 40,000 layers cannot take
five minutes. The tension between "safe" and "fast" is the entire engineering story.

## 3. The feature in one sentence

Figma plugins let anyone who can build a website write a small program that reads and
edits your open design file directly, and Figma runs that untrusted program fast and
synchronously inside your browser tab without letting it freeze your canvas, steal your
file, or reach out to the network.

Figma's own stated design principle, from the launch in August 2019, was "if you can
build a website, you should be able to make a plugin."

## 4. Jobs to be done

What is Priya really hiring a plugin to do?

- "Do the boring bulk edit I would never do by hand." Rename 40,000 layers. Recolor 500
  shapes. Translate every text layer to Spanish. The plugin is a macro for repetitive work.
- "Bring outside data into my design." Real photos (Unsplash), real names and avatars
  (Content Reel), real map tiles, real charts from a spreadsheet.
- "Add a capability Figma itself does not have." Accessibility contrast checks (Stark),
  auto-generated user-flow arrows (Autoflow), animation timelines (Figmotion).

And what is Figma, as a company, hiring the plugin system to do? Turn a tool into a
platform, so the reason to stay is not just Figma's own features but the hundreds of
things the community built on top. More on that in section 8.

Underneath all of these is one silent job the user never states but always assumes:
"and do not let this stranger's code hurt me." That silent job is the one the engineers
spent two weeks in a room arguing about.

## 5. How it works for the user

From Priya's side it could not be simpler. She right-clicks on the canvas, hovers
"Plugins," and picks "Content Reel." A small panel slides in from the side. It shows a
searchable list of content packs: names, avatars, job titles, sample copy. She selects
her twelve profile cards on the canvas, clicks "Persona" in the panel, and the twelve
grey circles fill with faces and the twelve "Name Surname" labels become "Aditi Rao,"
"Marcus Chen," and so on. She closes the panel.

Two different things were happening in that panel, and Figma keeps them strictly apart,
though Priya cannot tell. The visual panel she was clicking (the list, the search box,
the buttons) is a normal web page. The part that actually reached into her file and
changed the twelve text layers and twelve image fills is a different, invisible piece of
code with no user interface at all. The panel and the invisible worker talk to each
other by passing messages. We will see why that split exists, and why it is the key to
safety, in section 7.

## 6. The actual flow, step by step

1. Priya selects twelve profile-card frames on the canvas.
2. She right-clicks, hovers "Plugins," clicks "Content Reel."
3. Figma loads the plugin. It has two files: a UI file (an HTML page) and a main file
   (a script). Figma puts the UI file inside a sandboxed `<iframe>` and shows it as the
   side panel. It loads the main script into a separate locked-down sandbox that we will
   describe below.
4. The main script, on startup, asks Figma for the current selection. It gets back a list
   of the twelve frames as objects it can read and write.
5. Priya clicks "Persona" in the panel. The panel (the iframe) cannot touch her file, so
   it cannot do the work itself. Instead it sends a message: `figma.ui.postMessage({ type:
   "fill", pack: "persona" })`.
6. The main script receives that message. Now it does the real work: for each of the
   twelve cards it finds the text layer, sets `textNode.characters = "Aditi Rao"`, finds
   the ellipse, sets its image fill to a fetched avatar. Each of those reads and writes is
   a plain, synchronous property access, like `node.name = "..."`, that returns instantly.
7. Figma takes the batch of changes the script made, applies them to the real document,
   and re-renders the canvas. The twelve cards update.
8. The plugin calls `figma.closePlugin()`. The panel disappears.

The two-file split in step 3, UI in one box, document-editing logic in another box, is
not a UI nicety. It is a security wall, and it is doing more work than any other single
decision in the design.

## 7. Under the hood, like the engineer

This is the heart of the teardown. The problem statement is precise: run a stranger's
JavaScript inside a browser tab so that (a) it can read and write the design document
quickly and with normal, non-async code, but (b) it cannot read your other tabs, steal
your cookies or auth token, reach the network to exfiltrate your file, or reach into the
rest of the Figma app and escape. Let us build up to Figma's answer the way they did,
because the dead ends are the most instructive part.

### First idea: just run the plugin code on the page. Rejected instantly.

If you `eval()` the plugin's code directly on the Figma page, it inherits everything.
It can read `document.cookie`, call `fetch()` to send your file anywhere, and reach any
global variable in the Figma app. This is the loaded gun. No serious tool ships this.

### Second idea: an iframe or a Web Worker with message passing. This is where it gets subtle.

The browser already gives you a sandbox for free: an `<iframe>` from a different origin,
or a Web Worker, runs isolated. It cannot see the parent page's cookies or variables. It
talks to the parent only by `postMessage`, sending copyable data back and forth. Safe.

So the natural design is: put the plugin in an iframe or worker. When it wants to read a
layer's name, it posts a message "what is the name of node 5?", the parent reads the real
document and posts back "Rectangle 127." When it wants to rename, it posts "set node 5
name to btn/primary." Clean and safe.

There are two problems, and Figma hit both.

Problem one is speed. Every message across that boundary is a round trip, and a round
trip is not free. Figma measured the overhead at roughly 0.1 milliseconds per round trip,
which caps you at about 1,000 messages per second. Now walk Priya's rename job. She has
40,000 layers. If reading each layer's current name and writing the new one is two
messages, that is 80,000 messages. At 1,000 per second that is 80 seconds of pure
message-passing overhead, during which the tab feels frozen, before any real work counts.
For a plugin that reads and writes many properties per node the number is far worse. The
boundary that made you safe just made you unusable at scale.

Problem two is worse for a subtle reason: message passing is fundamentally asynchronous.
There is no way in JavaScript to make a blocking, synchronous call to something that
lives across a `postMessage` boundary. So every single read of a property would have to
be written as `await node.getName()`, and every function that calls it would have to be
marked `async`, all the way up. Figma actually tried this in an early alpha, and the very
first piece of feedback from testers was that being forced to sprinkle `async` and
`await` over code that just wants to read a layer's name was miserable. It made the
simplest plugin ugly and hard to write. It broke the "if you can build a website you can
build a plugin" promise.

So Figma made a decision that is the spine of the whole design: do not ask the plugin's
questions across the boundary one at a time. Instead, give the plugin a full copy of the
document up front, so that every read and write is a local, synchronous, instant
operation, and only sync the batch of changes back afterward. Copy once, then all reads
are free. This is the same move the ledger keeps finding, do the expensive coordination
once and up front so the hot path is a cheap local lookup, here applied to inter-process
messaging rather than to search or ranking.

But that decision reopens the security hole. If the plugin has a real, live copy of the
document sitting in the same place its code runs, and if that place is the normal page,
we are back to the loaded gun. We need a sandbox that (a) lets the plugin call into the
document synchronously, like a normal function call, yet (b) still isolates it from the
cookies, the network, and the rest of the app. An iframe cannot do that, because an
iframe can only be reached asynchronously. We need something that runs in the same thread
but is still walled off.

### Third idea: run the plugin in a fresh JavaScript "Realm" in the same VM. Chosen, then broken.

After ruling out the iframe approach, Figma spent about two weeks evaluating options and
picked a technique called the Realms shim. A Realm, in JavaScript terms, is a fresh,
separate global environment: its own `window`-like global object, its own set of built-in
objects, created inside the same JavaScript engine as the host. The idea is that you
create a clean Realm that has no `fetch`, no `document`, no cookies, none of the dangerous
globals, and you run the plugin inside it. Because it is in the same engine and the same
thread, the host can hand the plugin real document objects and the plugin can call methods
on them synchronously. You get the speed of local calls and, in theory, the isolation of
a fresh global scope. Figma chose it because it gave great performance and made plugins
easy to debug with normal browser tools.

Here is the mental model. The host holds the real document. It wraps each document object
in a thin membrane, a proxy, before handing it to the plugin's Realm. Read-only things
get a read-only wrapper. Writable things get a wrapper that funnels writes back into the
host's real object. The membrane is the checkpoint between the two worlds.

And here is where it broke. Independent security researchers found several vulnerabilities
in the Realms shim that would have let code inside the sandbox escape into the host. Figma
audited every published plugin and found no evidence any of these had ever been exploited,
but the class of bug was real and dangerous. The root cause is worth understanding because
it is the deep lesson of this whole teardown.

The Realms shim uses the same JavaScript virtual machine for the code inside the sandbox
and the code outside it. Same engine, same heap, same kind of objects. The only thing
separating "inside" from "outside" is bookkeeping in that shared engine: this object is a
guest object, that one is a host object, this membrane wraps that. When inside and outside
are made of the exact same stuff, an attacker's whole game is to confuse the bookkeeping,
to get the host to treat a guest object as if it were a host object, or slip an unwrapped
host object across the membrane. Do that once and the wall is gone. That is exactly the
class of vulnerability that was found: the shim confusing an object from outside the
sandbox with an object from inside. When your fence is drawn on the same field the
attacker stands on, the fence is only as strong as your ability to never once mislabel a
blade of grass. That is a losing game over time.

### Fourth idea, and the one running today: a whole separate JavaScript engine compiled to WebAssembly.

Figma had built the system to be swappable, so when the Realms vulnerabilities surfaced
they activated a backup plan quickly. That plan, still in place today, is to stop sharing
a VM at all. They took QuickJS, a small, complete JavaScript engine written in C by
Fabrice Bellard, and compiled it to WebAssembly. The plugin's code now runs inside that
QuickJS engine, which is itself running as a WebAssembly module inside the page.

Why this fixes the whole class of bugs, cleanly. A QuickJS value lives inside the
WebAssembly module's own linear memory. It is a C struct in a byte array. A real browser
JavaScript object lives in the browser engine's heap. These two representations are not
just logically separate, they are physically different kinds of things. There is no way
for the plugin to hand the host a "guest object that looks like a host object," because a
guest object is a chunk of bytes in a WASM heap and a host object is a browser engine
object, and nothing can make one masquerade as the other. The confusion attacks that
plagued the Realms shim become impossible by construction, not by careful bookkeeping.
This is the same idea as WhatsApp keeping its backup key inside a tamper-resistant chip
rather than a software counter, from the 09-03 teardown: move the wall from something you
have to police to something the attacker's own materials cannot cross.

You still pay for the document access. The host exposes a small set of functions across
the WASM boundary (get selection, read a property, write a property), and QuickJS calls
them synchronously as if they were normal functions. Because both live in the same thread,
these calls are cheap function calls, not 0.1ms message round trips. So Figma keeps the
synchronous, non-async programming model designers loved, but now the sandbox is a genuine
engine boundary instead of a bookkeeping trick.

### The other half of the split: the UI iframe.

The plugin's document-editing script runs in the QuickJS sandbox on the main thread. That
sandbox is a minimal JavaScript environment: it can touch the Figma scene, but it exposes
no browser APIs at all. No `fetch`. No DOM. No cookies. So how does Content Reel download
an avatar from the internet? It cannot, from the sandbox. That is on purpose.

The plugin's user interface, the side panel Priya clicked, runs in a normal sandboxed
`<iframe>`. The iframe is the opposite: it has full browser APIs, so it can `fetch` an
avatar image, but it has no access to the Figma scene at all. The two halves talk only by
`postMessage`: `figma.ui.postMessage` from sandbox to panel, and the panel posts back.

Look at what this capability split buys you. The code that can see your file cannot reach
the network, so it cannot exfiltrate the file. The code that can reach the network cannot
see your file, so it has nothing worth exfiltrating. To steal your design and send it
away, a malicious plugin would need to move the whole file across the postMessage bridge
and out through the iframe, which is a large, visible, awkward act rather than a one-line
`fetch(yourFile)`. Splitting the two dangerous powers, "sees your data" and "talks to the
internet," so that no single piece of code holds both, is defense in depth. It is the same
instinct as isolating money movement from read traffic in the Stripe rate-limiting
teardown (09-01): separate the powerful thing from the reachable thing.

### Guard rails for a plugin that misbehaves without meaning to (inference, standard practice).

A sandbox stops a plugin from stealing. It does not by itself stop a plugin from hanging.
Because the sandbox runs on the main thread for synchronous speed, a plugin with an
infinite loop, or one that tries to allocate gigabytes, would freeze or crash Priya's tab.
The standard way this class of embedded interpreter is tamed, and what quickjs-emscripten
(a public library built on the same QuickJS-in-WASM idea) exposes directly, is two dials:
an interrupt handler with a deadline (`shouldInterruptAfterDeadline`) that lets the host
forcibly stop the guest after it has run too long, and a hard memory cap
(`memoryLimitBytes`) so a runaway allocation hits a ceiling instead of eating the tab.
Figma has not published its exact limits, so treat the specific numbers as unknown, but a
main-thread interpreter without a "you have run too long, I am stopping you" lever would
be reckless, and this is the well-grounded version of how that lever is built. This is the
frame-budget discipline again: a shared real-time surface cannot let one participant run
unbounded.

### The scale story at three tiers

The thing that grows here is not a catalog of items to search. It is two different axes:
how much of the document a single plugin run touches, and how many untrusted plugins are
installed across how many users. Each tier breaks for a different reason.

Tier one, about 1,000 property operations. Priya runs a plugin that recolors 1,000 shapes
in a small marketing file. Even the naive, rejected design works here: 1,000 async
message round trips at 0.1ms each is 0.1 seconds. Nobody notices. At this size a full
document copy and an embedded JavaScript engine are over-engineering. Nothing breaks. If
this were the whole world, Figma would have shipped the simple iframe and gone home.

Tier two, about 100,000 property operations. Now Priya runs "Rename It" across her 40,000
layer design system, reading and writing several properties per node, easily 100,000-plus
operations. This is where the naive design dies: 100,000 round trips at 0.1ms is 10
seconds of pure overhead with the tab locked, and the forced `async`/`await` on every read
made the plugin painful to write in the first place. This is the exact tier that forced
the real architecture. The fix is the copy-then-sync move plus the synchronous in-process
engine: hand the plugin a local copy of the document so each of those 100,000 operations
is a nanosecond-scale function call inside QuickJS, not a millisecond-scale message, then
apply the batch of changes back to the real document once at the end. The wall at this
tier is latency of cross-boundary access, and the survival trick is to remove the boundary
from the hot path.

Tier three, roughly 10 million and up: thousands of untrusted plugins in the community,
installed by Figma's millions of users, some plugins touching millions of nodes, some
buggy, a few possibly malicious. Two new things break here, neither of which existed at
the smaller tiers. First, trust: at one plugin you can read the code yourself; at thousands
of third-party plugins that auto-update, you cannot, so the architecture itself has to be
the guarantee. That is what the QuickJS-in-WASM engine boundary plus the capability split
(scene access without network, network without scene access) provide, and the swappable
engine is what let Figma hot-swap the whole sandbox fleet-wide the moment the Realms
vulnerabilities were found, without breaking every published plugin. Second, liveness: at
this many plugins and users, some plugin will hang or blow up memory every day, so the
interrupt-deadline and memory-cap guard rails (inference) stop one bad plugin from taking
down one user's tab. The review process on the community marketplace is a further layer of
defense in depth against a plugin shipping a malicious update, but the sandbox is what lets
Figma sleep at night even when review misses something, which is the literal title of their
engineering post.

The through-line: the small tier is a latency problem solved by pre-copying the document,
and the large tier is a trust-and-liveness problem solved by a physical VM boundary plus
per-run resource limits. Different walls, different tools, same discipline of pushing the
expensive or dangerous thing off the hot path.

## 8. The retention and habit mechanic

Plugins are not a per-session habit loop the way an autoplay or a Monday playlist refresh
is. They are a slower, stickier mechanic: an ecosystem flywheel that becomes a switching
cost. The metric they move is retention and expansion, the platform moat, not daily
activation.

The flywheel has two sides. On the user side, every plugin a designer wires into their
daily flow is one more reason the tool is irreplaceable. Priya's team runs Content Reel
for realistic mockups, Unsplash for photos, Stark for accessibility sign-off, and one
custom internal plugin that pushes design tokens to their code repo. That is not four
features, it is four habits, and moving to a competitor means rebuilding all four and
retraining everyone. The lock-in is measured in weeks of lost workflow, which is exactly
the kind of retention that does not show up as a daily-active number but shows up as
almost nobody ever leaving.

On the developer side, the pull is just as strong and it feeds the user side. One
independent developer wrote about shipping a Figma plugin and reaching 4,000 users in under
two weeks. That reach is the incentive that makes developers build the next hundred
plugins, which gives users the next hundred reasons to stay, which grows the audience that
attracts the next developer. A two-sided market where each side makes the other more
valuable is the strongest moat in software, and it is why an app's plugin store, an
operating system's app store, and Figma's community all guard their platforms so carefully.

The habit is quiet but real: the reflex of right-clicking and reaching for a plugin
instead of doing a boring task by hand. Once "I will just run a plugin" replaces "I will
click 40,000 times," the tool that hosts those plugins has become the place the work
lives. And the entire reason a cautious company felt safe opening that door to strangers'
code is the sandbox from section 7. The security work is not separate from the retention
work. It is the precondition for it. Without a sandbox they could sleep behind, Figma
could never have let the ecosystem exist, and the moat would never have formed.

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor that compiles to shippable code, plus an embeddable
runtime. Two parts of that sentence are exactly Figma's plugin problem wearing different
clothes, and both are coming for you whether you plan for them or not.

First, the embeddable runtime. The day your compiled shader effect runs inside a customer's
web page, your runtime is a guest in someone else's tab, sitting next to their auth cookies,
their user data, their DOM. And the effect graph it is running was authored by yet another
party (a Rare.lab user, a marketplace creator). That is the Figma situation exactly:
untrusted authored code, executing in a context full of things it must never touch. If you
ever let users share or sell effects that run in the embeddable runtime, you have inherited
the whole problem, and the four concrete moves from this teardown are your playbook.

1. Do not force async round trips on the hot path. If your runtime evaluates an authored
   graph by messaging across an isolation boundary once per node or once per parameter, you
   will hit Figma's 0.1ms-per-round-trip wall the instant a graph has 100,000 parameter
   reads per frame, and at 60fps you have about 16ms for the whole frame, not 10 seconds.
   Copy the scene or parameter state into the sandbox once per frame and let the authored
   code read it with local, synchronous access. Copy once, then all reads are free, same as
   Figma handing the plugin a document copy.

2. Isolate at a boundary the attacker's materials cannot cross, not at a bookkeeping fence.
   Do not sandbox authored logic by freezing globals or wrapping objects in the same JS VM
   your host uses, because that is the Realms shim, and it leaks through object-identity
   confusion over time. Run authored control code in a separate engine (QuickJS compiled to
   WebAssembly is the proven choice) so guest values are bytes in a WASM heap that cannot
   masquerade as host objects. For the shader code itself, the equivalent is to never hand
   raw GLSL or WGSL straight to the GPU: compile authored graphs to a restricted
   intermediate representation, validate it, and reject anything outside the allowed
   instruction and resource set, so a malicious node cannot smuggle in an infinite loop or
   an out-of-bounds read.

3. Enforce a time budget and a memory cap on every authored run, and make it a hard
   interrupt, not a hope. In a real-time render loop this is not optional politeness, it is
   survival: a single authored effect with a runaway loop must be forcibly stopped at the
   frame deadline (Figma's interrupt-after-deadline) or it freezes every player watching,
   and an effect that allocates without bound must hit a ceiling (memoryLimitBytes) instead
   of crashing the host tab. Tie the budget to your frame time, the same way section 7's
   guard rails tie it to keeping the shared surface live.

4. Keep the sandbox swappable, and split capabilities. Figma survived a live security
   incident because they could hot-swap the entire sandbox engine without breaking published
   plugins. Build your runtime so the isolation layer is one replaceable component, not
   welded through your codebase, so the day a WASM or GPU escape is disclosed you can swap
   it fleet-wide. And copy the capability split: the code that can read the host scene or
   the customer's data should have no network access, and the code that can reach the
   network should have no access to the scene, so no single authored fragment holds both the
   secret and the exit.

The one-sentence version. Letting strangers' code run inside your creative tool is what
turns a tool into a platform and builds a moat almost nobody escapes, but only if the code
runs fast and safe at once. Get speed by copying state into the sandbox once so authored
reads are local and synchronous, get safety by isolating at a real engine boundary
(JavaScript-in-WebAssembly, or a validated shader IR) that an attacker's own objects cannot
cross rather than a same-VM fence you must police, cap every run with a hard time and memory
budget so one bad effect cannot freeze the frame, and keep the whole sandbox swappable and
capability-split so the inevitable escape is a hot-swap and not a catastrophe.

---

## Sources

Confirmed engineering details (execution model, the 0.1ms round-trip and async pain, the
document-copy decision, the Realms shim choice and its object-confusion vulnerabilities,
the switch to QuickJS compiled to WebAssembly, the swappable architecture, the main-thread
sandbox with no browser APIs and the separate UI iframe):

- Figma Engineering, "How to build a plugin system on the web and also sleep well at night": https://www.figma.com/blog/how-we-built-the-figma-plugin-system/
- Evan Wallace, "An update on plugin security": https://madebyevan.com/figma/an-update-on-plugin-security/ (Figma mirror: https://www.figma.com/blog/an-update-on-plugin-security/)
- Figma Developer Docs, "How Plugins Run": https://developers.figma.com/docs/plugins/how-plugins-run/
- Figma Blog, "Plugins are coming to Figma" (Aug 2019 launch, design principles): https://www.figma.com/blog/plugins-are-coming-to-figma/
- Figma Blog, "Introducing Figma Plugins": https://www.figma.com/blog/introducing-figma-plugins/

The QuickJS engine and the public library that demonstrates the same isolation pattern
(memory limit, interrupt deadline, marshalling boundary), used above as the well-grounded
basis for the inferred guard rails:

- QuickJS, by Fabrice Bellard: https://bellard.org/quickjs/
- quickjs-emscripten (QuickJS-in-WebAssembly, `memoryLimitBytes`, `shouldInterruptAfterDeadline`): https://github.com/justjake/quickjs-emscripten

Background on the Realms idea (now the TC39 ShadowRealm proposal) and an outside critique
of the plugin architecture's performance and debuggability:

- TC39 ShadowRealm proposal (successor to the Realms API): https://github.com/tc39/proposal-shadowrealm
- Tom MacWright, "Figma Plugins" (outside critique): https://macwright.com/2024/03/29/figma-plugins

Developer-side ecosystem pull (a plugin reaching 4,000 users in under two weeks):

- Ahmad Al Haddad, "How I created a plugin system for Figma and got 4000 users in under 2 weeks": https://medium.com/@hadd/how-i-created-a-plugin-system-for-figma-and-got-4000-users-in-under-2-weeks-74c94420da6f
