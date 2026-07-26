# References: WhatsApp voice and video calling

Keeper links for the 2026-07-26 teardown on WhatsApp calling (call setup, relay,
encrypted media, codec).

## Primary sources (Meta / WhatsApp / At Scale)

- Meta Engineering, "MLow: Meta's low bitrate audio codec" (June 13, 2024).
  ~2x Opus quality at 6 kbps (POLQA MOS 3.9 vs 1.89), ~10% lower complexity,
  built for ARMv7 and 10-year-old phones, better FEC packing.
  https://engineering.fb.com/2024/06/13/web/mlow-metas-low-bitrate-audio-codec/
- Meta Engineering, "Enhancing the security of WhatsApp calls" (Nov 8, 2023).
  "Protect IP address in calls" relays every call to hide the IP; calls stay
  E2E encrypted; relay never sees media.
  https://engineering.fb.com/2023/11/08/security/whatsapp-calls-enhancing-security/
- At Scale Conference, "Calling Relay Infrastructure at WhatsApp Scale."
  Relay can't decrypt/transcode; video simulcast (send high+low, relay selects);
  New Year's Eve extreme spike; simplicity/resilience philosophy.
  https://atscaleconference.com/calling-relay-infrastructure-at-whatsapp-scale/

## Technical background (WebRTC media plane)

- webrtcHacks, "What's up with WhatsApp and WebRTC?" Media as SRTP over UDP to
  relay servers ("conf bridge"); relay-first then P2P upgrade; ICE/STUN/TURN.
  https://webrtchacks.com/whats-up-with-whatsapp-and-webrtc/
- webrtcHacks, "How WebRTC's NetEQ Jitter Buffer Provides Smooth Audio."
  https://webrtchacks.com/how-webrtcs-neteq-jitter-buffer-provides-smooth-audio/
- WebRTC NetEQ design docs. InsertPacket/GetAudio, packet buffer, target level,
  accelerate / preemptive-expand / expand (PLC).
  https://github.com/zhaoxiu-zeng/webrtc/blob/master/modules/audio_coding/neteq/g3doc/index.md
- GetStream, "Media Resilience in WebRTC." FEC, NACK/RTX, PLC, RED.
  https://getstream.io/resources/projects/webrtc/advanced/media-resilience/
- Google Research, Holmer/Shemer/Paniconi, "Handling Packet Loss in WebRTC."
  https://research.google.com/pubs/archive/41611.pdf

## Facts, numbers, feature history

- Business of Apps, WhatsApp Revenue and Usage Statistics (2026).
  https://www.businessofapps.com/data/whatsapp-statistics/
- SQ Magazine, WhatsApp Statistics 2026. 5.5B voice + 2.4B video calls/month,
  2B+ call minutes/day, avg voice call ~9.7 min.
  https://sqmagazine.co.uk/whatsapp-statistics/
- The Hacker News, "WhatsApp Introduces New Privacy Feature to Protect IP
  Address in Calls" (Nov 2023).
  https://thehackernews.com/2023/11/whatsapp-introduces-new-privacy-feature.html
- Deccan Herald, group video call limit increased to 32 (late 2022).
  https://www.deccanherald.com/technology/whatsapp-gets-communities-group-video-call-limit-increased-to-32-1158991.html
- WhatsApp Blog, "Group Video and Voice Calls Now Support 8 Participants" (2020).
  https://blog.whatsapp.com/group-video-and-voice-calls-now-support-8-participants

## Notes on fact vs inference

- Confirmed: SRTP-over-UDP media, relay/"conf bridge," relay-first-then-P2P,
  simulcast for encrypted group calls, MLow numbers, IP-protection relay, 32-cap,
  NYE spike shape.
- Inference (clearly labeled in the report): exact WhatsApp group-call key
  management internals are not fully public; the "encrypt once, server fans out"
  model is grounded in WhatsApp's published group-messaging design (Sender Keys)
  and general E2E group-call practice, not a published WhatsApp group-call spec.
