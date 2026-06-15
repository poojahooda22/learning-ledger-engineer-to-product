# References: WhatsApp delivery receipts (the ticks)

Keeper links for the 2026-06-15 teardown.

## Architecture and scale
- GetStream, "How WhatsApp Works: Architecture Deep Dive on 100 Billion Messages":
  https://getstream.io/blog/whatsapp-works/
- ByteByteGo, "How WhatsApp Handles 40 Billion Messages Per Day":
  https://blog.bytebytego.com/p/how-whatsapp-handles-40-billion-messages
- High Scalability, "Designing WhatsApp":
  https://highscalability.com/designing-whatsapp/
- Software Patterns Lexicon, "Erlang in Large-Scale Messaging Systems (WhatsApp)":
  https://softwarepatternslexicon.com/erlang/case-studies-and-practical-applications/erlang-in-messaging-systems-e-g-whatsapp/
- scalewithchintan, "WhatsApp Erlang Architecture, Scaling to 2 Billion Users":
  https://scalewithchintan.com/blog/whatsapp-erlang-architecture-2-billion-users

## Protocol
- XEP-0184: Message Delivery Receipts (XMPP standard the tick scheme resembles):
  https://xmpp.org/extensions/xep-0184.html
- FunXMPP in early decompiled WhatsApp Android client (community reverse engineering):
  https://github.com/JaapSuter/Niets/blob/master/external/decompiled/android/WhatsApp_2_0_7.src_fun_xmpp/WhatsApp_2_0_7.src/com/whatsapp/client/FunXMPP.java

## What the ticks mean
- WhatsApp Help Center, read receipts: https://faq.whatsapp.com/665923838265756
- Blueticks, "WhatsApp Read Receipts Explained": https://blueticks.co/blog/whatsapp-read-receipts-explained

## Key facts pinned down
- Built on Erlang/OTP, on a modified ejabberd XMPP server; custom compressed binary
  protocol nicknamed FunXMPP. Session/routing state and offline queues in Mnesia.
- Rick Reed (WhatsApp) Erlang Factory 2012 talks describe FreeBSD + Erlang tuning to
  hold ~2 million+ simultaneous TCP connections per server.
- Store-and-forward: server holds the message, retries delivery, deletes after the
  recipient device acknowledges. Undelivered messages kept up to ~30 days.
- Three states: one grey tick = sent to server; two grey = delivered to device;
  two blue = opened/read. Group rule: grey-double when delivered to all, blue when
  read by all.
- Receipts are metadata (envelope), so they keep working under Signal end-to-end
  encryption (since 2016) even though the server cannot read message content.
- Inference (not fully public): exact offline-queue layout and group receipt
  aggregation; described as the standard solution given the known stack.
