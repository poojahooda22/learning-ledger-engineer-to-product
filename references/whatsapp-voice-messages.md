# References: WhatsApp voice messages (teardown 2026-10-05)

Keeper links on how WhatsApp records, compresses, encrypts, ships, and plays back the
press-and-hold voice note.

## Codec and format
- Opus audio format (RFC 6716), bitrate 6 to 510 kbit/s, 8 to 48 kHz, beats MP3/AAC at
  low bitrate: https://en.wikipedia.org/wiki/Opus_(audio_format)
- Meta / WhatsApp developer docs, Audio messages: media must be `.ogg` with the Opus
  codec:
  https://developers.facebook.com/documentation/business-messaging/whatsapp/messages/audio-messages

## Encryption and media delivery
- WhatsApp Encryption Overview whitepaper: media attachments use an ephemeral 32-byte
  AES-256 key (CBC, random IV) plus a 32-byte HMAC-SHA256 key; encrypted blob uploaded
  to a blob store; URL plus keys plus SHA-256 sent over the E2E message channel:
  https://md.teyit.org/file/1586701275510-whatsapp-security-whitepaper.pdf

## Features and scale
- WhatsApp blog, "We are making voice messages even better" (March 2022 bundle: faster
  playback 1.5x/2x, out-of-chat playback, pause/resume recording, draft preview,
  remember playback): https://blog.whatsapp.com/making-voice-messages-better
- Adweek: Meta figure of 7 billion voice messages sent per day on WhatsApp (March
  2022): https://www.adweek.com/media/7b-voice-messages-are-being-sent-via-whatsapp-every-day/
- MacRumors: on-device, end-to-end encrypted voice message transcripts (November
  2024): https://www.macrumors.com/2024/11/21/whatsapp-voice-message-transcripts/
- 9to5Mac: transcripts rolling out to all users (November 2024):
  https://9to5mac.com/2024/11/21/whatsapp-now-rolling-out-voice-message-transcripts-to-all-users/

## Key numbers
- Raw 40s stereo CD-style audio: about 7 MB. Same clip in Opus at ~24 kbit/s mono:
  about 120 KB. Roughly a sixtyfold reduction.
- 7 billion voice messages/day; 100 billion+ total messages/day; 2 billion+ users.
