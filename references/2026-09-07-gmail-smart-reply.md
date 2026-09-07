# References: Gmail Smart Reply (2026-09-07 teardown)

Primary sources, worth keeping.

## Papers (primary)

- Kannan, Kurach, Ravi, Kaufmann, Tomkins, Miklos, Corrado, Lukacs, Ganea,
  Young, Ramavajjala. "Smart Reply: Automated Response Suggestion for Email."
  KDD 2016. The first production system: triggering model, seq2seq LSTM response
  selection, beam search over a trie of permitted responses, semi-supervised
  graph label propagation to build the response set, and the diversity rules
  (omit-redundant, force-a-negative via a second LSTM pass).
  - Landing: https://research.google/pubs/smart-reply-automated-response-suggestion-for-email/
  - PDF: https://research.google.com/pubs/archive/45189.pdf

- Henderson, Al-Rfou, Strope, Sung, Lukacs, Guo, Kumar, Miklos, Kurzweil.
  "Efficient Natural Language Response Suggestion for Smart Reply." 2017. The
  rebuild: feedforward dual-encoder, score fit as a dot product S(x,y)=h_x.h_y,
  precompute the response side, serve via approximate nearest-neighbor search.
  Same quality as seq2seq at a fraction of compute and latency.
  - Abstract: https://arxiv.org/abs/1705.00652
  - PDF: https://arxiv.org/pdf/1705.00652

## Blog and secondary

- Google AI Blog. "Efficient Smart Reply, now for Gmail." May 2017. The
  hierarchical feedforward story in plain language, plus the ~12% of mobile
  replies figure.
  http://ai.googleblog.com/2017/05/efficient-smart-reply-now-for-gmail.html

- The Morning Paper (Adrian Colyer). "Smart Reply: Automated response suggestion
  for email." Nov 2016. Clear walkthrough of the 2016 paper.
  https://blog.acolyer.org/2016/11/24/smart-reply-automated-response-suggestion-for-email/

- Shagun Sodhani. Reading notes on the Smart Reply paper (GitHub gist). Compact
  summary of the components and the numbers.
  https://gist.github.com/shagunsodhani/da411f15b71ed6a664f9d5ac46409b42

## Key numbers to remember

- Triggering model fires for ~11% of emails (feedforward MLP, ~1M-word vocab
  embedding + 3 hidden layers).
- Beam search cost O(b*l), l typically 10 to 30 words, independent of catalog
  size, because it walks a trie of permitted responses.
- Response set built from ~100 intent clusters, seeded with 3 to 5 hand-labeled
  examples each, canonicalized with a dependency parser, grown by label
  propagation (5 iterations per phase, sample 100 new candidate clusters, repeat
  to convergence).
- 2017 dual-encoder: response vectors precomputed offline, served via ANN search.
- Impact: ~10% (2016) growing to ~12% (2017) of all mobile replies in Inbox.
