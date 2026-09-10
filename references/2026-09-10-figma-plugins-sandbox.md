# References: Figma Plugins sandbox (untrusted code isolation)

Saved 2026-09-10 for the Figma plugin-system teardown.

## Primary (confirmed engineering)

- Figma Engineering, "How to build a plugin system on the web and also sleep well at night"
  https://www.figma.com/blog/how-we-built-the-figma-plugin-system/
  Key facts used: message-passing overhead ~0.1ms per round trip (~1,000 msgs/sec ceiling);
  they give the plugin a copy of the document instead of message-passing every property
  read/write; alpha testers rejected forced async/await from the iframe model; ~two weeks
  re-evaluating; chose the Realms shim for performance and debuggability.

- Evan Wallace, "An update on plugin security"
  https://madebyevan.com/figma/an-update-on-plugin-security/
  (Figma mirror: https://www.figma.com/blog/an-update-on-plugin-security/)
  Key facts used: independent researchers found several Realms-shim vulnerabilities allowing
  sandbox escape; audit found no evidence of exploitation; root cause was the shim using the
  SAME JS VM for inside and outside code, so objects could be confused across the membrane;
  fix was switching to QuickJS (Fabrice Bellard) cross-compiled to WebAssembly, whose object
  representations are too different to confuse; swappable architecture let them switch fast.

- Figma Developer Docs, "How Plugins Run"
  https://developers.figma.com/docs/plugins/how-plugins-run/
  Key facts used: sandbox (main plugin code) runs on the main thread in a minimal JS env with
  no browser APIs but access to the Figma scene; UI runs in a sandboxed iframe with browser
  APIs but no scene access; the two communicate via postMessage / figma.ui.postMessage.

- Figma Blog, "Plugins are coming to Figma" (Aug 2019 launch, "if you can build a website,
  you should be able to make a plugin"): https://www.figma.com/blog/plugins-are-coming-to-figma/
- Figma Blog, "Introducing Figma Plugins": https://www.figma.com/blog/introducing-figma-plugins/

## Supporting (the isolation pattern, used as the grounded basis for inferred guard rails)

- QuickJS, by Fabrice Bellard: https://bellard.org/quickjs/
- quickjs-emscripten (QuickJS-in-WASM; exposes memoryLimitBytes and
  shouldInterruptAfterDeadline, plus an explicit host/guest marshalling boundary):
  https://github.com/justjake/quickjs-emscripten

## Background and outside view

- TC39 ShadowRealm proposal (successor to the Realms API):
  https://github.com/tc39/proposal-shadowrealm
- Tom MacWright, "Figma Plugins" (outside critique of performance and debuggability):
  https://macwright.com/2024/03/29/figma-plugins
- Ahmad Al Haddad, "How I created a plugin system for Figma and got 4000 users in under 2
  weeks" (developer-side ecosystem pull):
  https://medium.com/@hadd/how-i-created-a-plugin-system-for-figma-and-got-4000-users-in-under-2-weeks-74c94420da6f

## Confirmed vs inference (for this teardown)

- CONFIRMED: launch date and principle; the 0.1ms/round-trip figure; the document-copy
  decision; the async/await rejection; the Realms shim choice and its object-confusion
  vulnerabilities; the QuickJS-in-WASM fix and why it closes the bug class; the swappable
  architecture; the main-thread-sandbox / UI-iframe split and postMessage channel.
- INFERENCE (labeled as such in the report): the exact reason main thread over Web Worker
  is synchronous host-function access; the CPU interrupt-deadline and memory-cap guard rails
  and their exact limits (Figma has not published numbers; quickjs-emscripten shows the
  standard mechanism); the read-only vs writable membrane wrapping details.
