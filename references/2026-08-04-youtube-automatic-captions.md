# References: YouTube automatic captions (ASR pipeline)

Saved 2026-08-04.

## Primary (Google)
- Automatic Captioning in YouTube (Google Research, 2010): https://research.google/blog/automatic-captioning-in-youtube/
  - Auto-captioning AND auto-alignment use the same speech infra behind Google Voice / Voice Search, trained on different data.
  - Auto-alignment = forced alignment: creator supplies the words, ASR supplies the timings.
- An All-Neural On-Device Speech Recognizer (Google Research, March 2019): https://research.google/blog/an-all-neural-on-device-speech-recognizer/
  - Classic pipeline = acoustic model + pronunciation model + language model composed into one WFST search graph (~2GB in production).
  - Moved to RNN-T (encoder + prediction network + joint network), quantized to ~80MB, streaming/monotonic alignment.

## Architecture background
- Listen, Attend and Spell (Chan et al., 2015): https://arxiv.org/abs/1508.01211
- Modern ASR survey (Conformer/Transformer/RNN-T, WFST decoding, Viterbi forced alignment): https://arxiv.org/html/2510.12827

## History / launch
- Launched Nov 2009 (limited), all English videos March 2010. Lead engineer Ken Harrenstien (deaf).
- NPR (2010): https://www.npr.org/templates/story/story.php?storyId=124501330
- CNN (2010): https://www.cnn.com/2010/TECH/02/08/deaf.internet.captions/index.html
- Scientific American (improving accuracy): https://www.scientificamerican.com/article/google-youtube-auto-caption/

## Product docs
- Use automatic captioning: https://support.google.com/youtube/answer/6373554

## Key technical spine (for reuse)
- Audio -> 25ms window / 10ms hop -> feature vectors (~100/sec). ~90k vectors for a 15-min video.
- Acoustic model (DNN/LSTM) -> phone-state probs; lexicon FST words->phonemes; n-gram LM for context.
- HCLG WFST composes all four; decode = Viterbi/beam search over a lattice; frame alignment gives caption timestamps for free.
- Forced alignment (known words) = cheap constrained Viterbi, no vocabulary search, no word errors possible.
- Serving artifact = sorted WebVTT array of {start,end,text}, few KB, CDN keyed lookup. Offline-think/online-lookup.
- Scale: 1k = batch on one box; 100k = queue + worker fleet (offline); 10M+ / 500 hrs uploaded per minute = demand-prioritized queue + lazy caption + separate streaming RNN-T path for live.
