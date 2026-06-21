# References: Gmail Smart Compose

Saved keepers for the 2026-06-21 teardown.

## Primary

- **Chen, Mia Xu, et al. "Gmail Smart Compose: Real-Time Assisted Writing." KDD 2019.**
  https://arxiv.org/abs/1906.00080 (PDF: https://arxiv.org/pdf/1906.00080)
  The definitive source. Model comparison (seq2seq with attention vs LM-A vs
  LM-B), wordpiece tokenization, beam search with confidence triggering,
  per-user n-gram personalization, TPU serving, latency numbers.

- **Google AI Blog: "Smart Compose: Using Neural Networks to Help Write Emails" (2018).**
  https://research.google/blog/smart-compose-using-neural-networks-to-help-write-emails/
  The product-side explainer. The 100 ms latency requirement, combining a
  bag-of-words model with an RNN-LM for speed over seq2seq, CPU-to-TPU latency
  win (hundreds of ms to tens of ms), TPUv2 Pod training in under a day,
  fairness handling for gendered pronouns.

- **Google Research publication page.**
  https://research.google/pubs/gmail-smart-compose-real-time-assisted-writing/

- **ACM Digital Library entry.**
  https://dl.acm.org/doi/10.1145/3292500.3330723

## Secondary / summaries

- KDD 2019 accepted paper listing:
  https://www.kdd.org/kdd2019/accepted-papers/view/gmail-smart-compose-real-time-assisted-writing
- Paper walkthrough (weak-learner blog):
  https://www.weak-learner.com/blog/2019/11/03/gmail-smart-compose/

## Key facts worth remembering

- Latency budget: aim under ~100 ms per keystroke; 90th-percentile latency ~60 ms.
- They shipped the FASTER model (LM-A, bag-of-words context fed into an RNN-LM),
  not the more accurate seq2seq-with-attention. Speed beat quality on the live path.
- Baseline seq2seq: encoder and decoder each two 1024-dim LSTM layers.
- Wordpiece (subword) vocabulary, multilingual, enables partial-word completion.
- Beam search keeps a heap of m best partial sequences; score is length-normalized
  log probability; a confidence threshold decides whether to show anything at all.
- Per-user n-gram model blended with the global neural model by linear interpolation
  for personalization without per-user neural retraining.
- CPU serving: hundreds of ms average. TPU serving: tens of ms, far more requests
  per machine. Training on a full TPUv2 Pod converged in under a day.
- Scale: Gmail's 1.5 billion users. The confidence gate doubles as a load shedder.
