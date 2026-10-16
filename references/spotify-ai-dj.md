# Spotify AI DJ (the synthetic radio host)

Keeper links for the 2026-10-16 teardown on how Spotify's AI DJ picks your
music, writes a personalized spoken intro, and speaks it in a human voice.
Distinct from the song-picking teardowns already in this ledger: Discover
Weekly (2026-06-13), autoplay (2026-08-16), personalized homepage (2026-08-06),
instant playback (2026-07-07). This one is about the VOICE and the generative
commentary pipeline, not the recommender itself.

## Primary sources (Spotify's own)

- Spotify Newsroom, "Spotify Debuts a New AI DJ, Right in Your Pocket"
  (2026-02-22 launch)
  https://newsroom.spotify.com/2023-02-22/spotify-debuts-a-new-ai-dj-right-in-your-pocket/
  The three technologies stated in Spotify's own words: (1) personalization,
  (2) "Generative AI through the use of OpenAI technology. We put this in the
  hands of our music editors," (3) "A dynamic AI voice platform from our Sonantic
  acquisition that brings to life stunningly realistic voices from text." The DJ
  voice model was built with Xavier "X" Jernigan, Head of Cultural Partnerships
  and former host of Spotify's morning show The Get Up. Launched beta US and
  Canada, English only.

- Spotify Newsroom / Engadget + TheNextWeb coverage of the 2026 expansion
  https://thenextweb.com/news/spotifys-ai-dj-now-speaks-4-new-languages
  https://www.engadget.com/spotify-ai-powered-personal-dj-expands-to-more-languages-and-countries.html
  Multilingual growth: Spanish first, then (May 2026) French (DJ Maïa), German
  (DJ Ben), Italian (DJ Alex), Brazilian Portuguese (DJ Dani); 75+ markets. Voice
  requests arrived in English (2025), then text requests and Spanish. Each new
  language needs its own voice model plus localized phrase cache and editorial.

## Key secondary sources (quotes and mechanism)

- PopSci, "Meet Spotify's new AI DJ"
  https://www.popsci.com/technology/spotify-ai-dj/
  Three components spelled out. The weekly "writer's room" of human music editors
  that curates the base commentary; generative AI scales that base script and
  tailors it to the individual listener. The leash on the LLM is the story.

- TechCrunch, "Spotify's new DJ feature is the first step into the streamer's
  AI-powered future" (2023-03-13)
  https://techcrunch.com/2023/03/13/spotifys-new-dj-feature-is-the-first-step-into-the-streamers-ai-powered-future/
  Uses OpenAI's model for the DJ; Spotify may move to in-house models later.

- Voicebot.ai + Tubefilter on the Sonantic acquisition (June 2022)
  https://voicebot.ai/2022/06/13/spotify-to-acquire-ai-voice-startup-sonantic/
  https://www.tubefilter.com/2022/06/13/spotify-just-acquired-a-company-that-can-make-anyone-say-anything/
  Sonantic = emotional TTS. Learns a voice from recorded audio clips; the model
  then reads any text. Two breakthroughs pitched: subtle emotional expression and
  realistic non-speech sounds (laughter, breathing). API + GUI to "sculpt and
  direct" synthetic voices like human actors. Recreated Val Kilmer's voice for
  Top Gun: Maverick (2022). Ziad Sultan (VP Personalization) quote on bringing it
  onto Spotify.

- Interesting Engineering, "Spotify unveils an AI-powered personal DJ"
  https://interestingengineering.com/innovation/spotify-ai-personal-dj
  Ziad Sultan: realism "is about the emotion in the voice," pauses for breathing,
  varied intonation. Spotify runs a large research team on LLMs, generative voice,
  and personalization, though the DJ itself used OpenAI's model at launch.

- Dynamoi, "Spotify algorithm glossary" / BaRT explainer
  https://dynamoi.com/learn/spotify-algorithm/spotify-algorithm-glossary
  BaRT (Bandits for Recommendations as Treatments) = the multi-armed bandit that
  governs Spotify Home recommendations, balancing exploitation and exploration.
  CONFIRMED that BaRT powers Home/recs; the DJ reusing this exact stack is
  well-grounded INFERENCE (Spotify says the DJ is built on "your
  personalization").

## Scale figures

- Billboard, Spotify Q4 2025 earnings
  https://billboard.com/pro/spotify-q4-2025-earnings-subscribers-total-revenue/
  751 million monthly active users, 290 million Premium subscribers at end of
  2025. AI DJ is a Premium feature.

## Fact vs inference summary

- CONFIRMED: three technologies (personalization + OpenAI generative AI, leashed
  by editors + Sonantic TTS voice built with X Jernigan); Sonantic acquisition
  June 2022; emotional TTS; multilingual 2026 expansion; BaRT powers Spotify
  recommendations; Premium-only; 751M MAU / 290M subs.
- INFERENCE (labeled as such in the report): the exact serving architecture, the
  template-plus-slot prompt design for the LLM, the precompute/cache of shared
  phrases, the splice of cached frame + fresh personalized slice, prefetch during
  the current song, the two-stage neural TTS (text -> mel-spectrogram -> vocoder)
  internals of Sonantic's model. These follow standard practice for the problem
  class and match the constraints Spotify has described, but are not published.
