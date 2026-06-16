# Reference: Google Search autocomplete / typeahead

Keeper links for the 2026-06-16 teardown on Google Search autocomplete (query
suggestions / typeahead).

## Primary (Google's own statements)

- How Google autocomplete predictions work (Search Help). Predictions "reflect
  real searches that have been done on Google"; signals named: language, location,
  freshness/trending; whole-query vs partial-query for long tail; policy-based
  removals (violent, hateful, explicit, dangerous, personal info).
  https://support.google.com/websearch/answer/7368877
- How Google autocomplete works in Search (Google blog, official explainer).
  https://blog.google/products/search/how-google-autocomplete-works-search/

## Engineering (the trie + top-k + scale machinery)

- PruningRadixTrie, Wolf Garbe (GitHub). Stores max rank per node, prunes branches
  that cannot beat current top-k, early termination. Up to 1000x faster than an
  ordinary radix trie. Benchmark: 6M unigrams+bigrams from English Wikipedia.
  Key quote: "37 ms for an autocomplete might seem fast enough for a single user,
  but it becomes insufficient when we have to serve thousands of users in parallel."
  https://github.com/wolfgarbe/PruningRadixTrie
- Designing a Search Autocomplete System (system design notes). Trie, precomputed
  top-k per node, shard by first character with uneven ranges to balance load,
  caching for O(1)-ish response.
  https://torontostudygroup.github.io/study-notes/notes/system-design-interview/ch13/
- Implementation: Autocomplete System Design for Large Scale, Pedro Lopes (Byte Tank).
  https://lopespm.com/2020/08/03/implementation-autocomplete-system-design.html
- Count-Min Sketch (Redis blog). Probabilistic frequency counting in fixed memory
  with bounded over-count error; used for the long tail of rare queries.
  https://redis.io/blog/count-min-sketch-the-art-and-science-of-estimating-stuff/
- Typeahead (Autocomplete) System Design, EnjoyAlgorithms. Trie + cache, low-latency
  per-keystroke requirement, ~5-10 suggestions is enough so k is small.
  https://www.enjoyalgorithms.com/blog/design-typeahead-system/

## Core ideas worth remembering

- Two halves: matching (prefix walk in a trie) then ranking (score the ~10
  survivors by popularity, location, freshness, personalization). Different jobs.
- Precompute top-k per node offline so the live keystroke is a lookup, not a sort
  over the subtree. Move the thinking off the hot path.
- Sharding by prefix; many read replicas because reads >> writes; cache hot prefixes.
- Every keystroke is a separate server round trip; a 10-letter query = up to 10
  suggest requests, so suggest QPS is several times human search QPS. Debounce on
  client, keep payload tiny.
- Fact vs inference: Google confirms "real searches + language/location/freshness +
  policy filtering." The exact trie variant, counters, and refresh cadence are not
  published; the trie/top-k/Count-Min-Sketch picture is standard-practice inference.
