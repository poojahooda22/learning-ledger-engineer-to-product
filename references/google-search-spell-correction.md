# References: Google Search spell correction

Saved for the 2026-07-05 teardown on query spell correction ("Did you mean" and
"Showing results for").

## Primary engineering and research

- Whitelaw, Hutchinson, Chung, Ellis. "Using the Web for Language Independent
  Spellchecking and Autocorrection." EMNLP 2009. https://aclanthology.org/D09-1093/
  The canonical Google approach: no hand-built dictionary, vocabulary and error
  model harvested from web text, n-gram language model for context, confidence
  classifier tuned on injected typos, language-independent, beats dictionary
  baselines on English and German real typos.
- Kernighan, Church, Gale. "A Spelling Correction Program Based on a Noisy
  Channel Model." COLING 1990. https://aclanthology.org/C90-2036.pdf
  Origin of noisy channel plus del/ins/sub/trans confusion matrices.
- Brill, Moore. "An Improved Error Model for Noisy Channel Spelling Correction."
  ACL 2000. Substring-to-substring edit model.
- Damerau. "A Technique for Computer Detection and Correction of Spelling
  Errors." CACM 1964. ~80% of errors within a single edit; adds transposition.
- Schulz, Mihov. "Fast String Correction with Levenshtein Automata." IJDAR 2002.
  DFA accepting exactly the strings within edit distance k.

## Implementations and deep dives

- Peter Norvig. "How to Write a Spelling Corrector."
  https://norvig.com/spell-correct.html
  21-line noisy-channel corrector. Generate edit-distance-1 and 2 candidates,
  filter to real words, rank by P(w|c)P(c).
- Wolf Garbe. SymSpell, symmetric delete algorithm.
  https://wolfgarbe.medium.com/1000x-faster-spelling-correction-algorithm-2012-8701fcd87a5f
  https://github.com/wolfgarbe/SymSpell
  Deletes only, on both dictionary and query side. 5-letter word, edit distance
  3: ~3 million possible errors covered by only 25 delete-variants. Up to a
  million times faster than generate-and-check; trades memory for speed.
- Mike McCandless. "Lucene's FuzzyQuery is 100 times faster in 4.0."
  https://blog.mikemccandless.com/2011/03/lucenes-fuzzyquery-is-100-times-faster.html
  Robert Muir replaced the brute-force scan of every unique term with a
  Levenshtein automaton (DFA) intersected with the term dictionary FST. ~100x.
- Elastic. "Fuzzy search: the story behind the scenes."
  https://www.elastic.co/blog/found-fuzzy-search
  Damerau-Levenshtein (optimal string alignment), edit distance limits, runtime
  grows with number of unique terms.

## Data

- Google Web 1T 5-gram dataset. N-gram counts over ~1 trillion tokens of web
  text, up to 5-grams. Raw material for context-sensitive correction (choosing
  "defiantly" vs "definitely" by the neighbors).

## The through-line for the ledger

Matching then ranking, again. Candidate generation (edit-distance neighbors via
automaton / BK-tree / delete-index) is the matching half; noisy-channel scoring
(error model times language model, plus context n-grams) is the ranking half.
Offline-think, online-lookup, again: harvest the web and query logs offline,
precompute correction tables and n-gram counts, cache the head misspellings, so
the live query is a keyed lookup plus a tiny sort. Confidence threshold splits
silent autocorrect from polite suggestion from silence, the same three-way gate
seen in Gmail Smart Compose and Stripe Radar.
