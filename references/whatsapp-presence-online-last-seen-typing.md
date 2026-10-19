# References: WhatsApp Presence (online / typing / last seen)

Saved for the 2026-10-19 teardown (the live status line, fan-out, subscribe-on-demand,
push-vs-pull, Erlang/BEAM/FreeBSD scale).

## Primary sources (WhatsApp scale and stack)

- WhatsApp engineering blog, "1 million is so 2011" (2012). A single FreeBSD 8.2 box
  pushed past 2 million simultaneous TCP connections on a 24-core Xeon X5675, with CPU
  and memory to spare. The canonical "we hold millions of idle sockets cheaply" result
  that presence rides on top of.
  https://blog.whatsapp.com/1-million-is-so-2011

- Rick Reed, "That's 'Billion' with a 'B': Scaling to the Next Level at WhatsApp,"
  Erlang Factory SF Bay 2014. 8,000+ cores, 70M+ Erlang messages/sec, Erlang R16B01 with
  WhatsApp's own BEAM patches, FreeBSD 9.2, Mnesia for in-memory state. Post-$19B
  acquisition.
  https://www.erlang-factory.com/sfbay2014/rick-reed

- Rick Reed, "Scaling to Millions of Simultaneous Connections," Erlang Factory SF 2012.
  Started around 200k connections/server, goal 1M; the work was tuning and patching the
  BEAM emulator and the FreeBSD kernel to remove bottlenecks.
  https://www.erlang-factory.com/conference/SFBay2012/speakers/RickReed

- FreeBSD Foundation testimonial, WhatsApp (December 2015). By then, 2 to 3 million
  concurrent TCP connections per single FreeBSD server.
  https://freebsdfoundation.org/testimonial/whatsapp/

## Secondary (reputable write-up of the 2014 talk)

- High Scalability, "How WhatsApp Grew to Nearly 500 Million Users, 11,000 cores, and
  70 Million Messages a Second." ~342k inbound / 712k outbound messages/sec, ~150 chat
  servers each holding ~1M phones, Mnesia routing. Good for the fleet shape.
  https://highscalability.com/how-whatsapp-grew-to-nearly-500-million-users-11000-cores-an/

## Protocol lineage (XMPP, used as the class-of-problem model; WhatsApp internals are not public)

- XEP-0085: Chat State Notifications. The "composing" typing-indicator state and its
  paused / active / inactive / gone siblings. WhatsApp's "typing..." mirrors this.
  https://xmpp.org/extensions/xep-0085.html

- XEP-0318: Last User Interaction in Presence. The "last seen" style last-presence-change
  timestamp carried in presence stanzas; also advises clients not to probe every contact
  on login.
  https://xmpp.org/extensions/xep-0318.pdf

## Fan-out math and subscribe-on-demand (system-design consensus, treat as inference)

- TechInterview, "User Presence System Low-Level Design: Online Status, Heartbeat, and
  Real-Time Fan-out." The 100k users x 50 contacts x 20 state-changes/hour = 100M
  events/hour arithmetic, and the fix: subscribe narrowly to the open conversation rather
  than the full contact graph. This is the exact behavior WhatsApp is reported to use
  (push presence only to those who have your chat open).
  https://www.techinterview.org/post/3233468381/lld-user-presence/

## Note on confirmed vs inference

Confirmed: the Erlang/BEAM/FreeBSD stack, the millions-of-connections-per-box numbers, the
fleet throughput, and Mnesia for in-memory routing (primary sources above). Inference,
clearly labeled in the report: that WhatsApp's wire protocol is a modified binary XMPP
(FunXMPP, from community reverse-engineering), and that presence is subscribe-on-demand,
push-for-online/typing, pull-for-last-seen. WhatsApp has not published presence internals,
so the XMPP standards and system-design literature stand in as the well-grounded
class-of-problem model.
