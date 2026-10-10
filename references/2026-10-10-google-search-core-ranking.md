# References: Google Search core ranking (how the ten blue links get ordered)

Saved for the 2026-10-10 Google Search core-ranking teardown.

## Primary-ish sources (trial, leak, official)

- US v. Google LLC, DOJ antitrust trial (2023 to 2024). Pandu Nayak testimony and trial
  exhibits. The primary public source for live ranking internals.
  Summary and exhibit walk-through: https://searchengineland.com/how-google-search-ranking-works-445141
  Broader trial compendium: https://www.hobo-web.co.uk/google-vs-doj/
  Human-rater / IS score evidence: https://www.hobo-web.co.uk/how-human-quality-raters-are-used-new-evidence-from-doj-v-google-antitrust-trial/

- May 2024 Google Content Warehouse API documentation leak. Primary public analysis by
  Michael King (iPullRank): https://ipullrank.com/google-algo-leak
  ABC-signals framing: https://searchengineland.com/google-abc-ranking-signals-455360
  Neural stack (DeepRank, RankEmbed-BERT): https://www.resoneo.com/?p=21264

- Google, "How Search Works" (official): https://www.google.com/search/howsearchworks/
  Inverted index, index size, retrieve-then-rank.

- RankBrain launch coverage (Bloomberg, Oct 2015, via TheNextWeb / Search Engine Land):
  https://thenextweb.com/google/2015/10/26/how-google-handles-search-queries-its-never-seen-before/

- BERT in Search (Google blog + TechCrunch, Oct 2019):
  https://techcrunch.com/2019/10/25/google-brings-in-bert-to-improve-its-search-results/

## Key facts pulled (fact vs inference)

Fact (official Google):
- Index is organized as an inverted index, "like the index at the back of a book"; index
  size is well over 100,000,000 gigabytes across hundreds of billions of pages.
- RankBrain (2015): embeds queries/words as vectors; handles the ~15% of daily queries never
  seen before; called the third most important ranking signal at launch (Greg Corrado).
- BERT in Search (Oct 2019): ~1 in 10 English US queries at launch, later all languages and
  featured snippets.
- Search Quality Rater Guidelines are public; raters score Needs Met and Page Quality
  (E-E-A-T).

Fact (antitrust trial testimony / exhibits):
- Navboost: click-driven re-ranking over a rolling ~13-month window (cut back from 18 months),
  sliced by country and device. Uses good clicks vs bad clicks (pogo-sticking) and last-longest
  click.
- Glue: sibling of Navboost that assembles the whole SERP (non-web features).
- IS / IS4 (information satisfaction) score from human raters; used to evaluate changes and to
  TRAIN models, not as a direct per-page live signal.
- DeepRank is the internal name for BERT applied to ranking.
- RankEmbed / RankEmbedBERT trained on two sources: search logs and human rater scores;
  improved long-tail / complex query handling.
- Google long denied using click data as a direct ranking signal in public; trial contradicted
  this.
- Court evidence: ~$26.3B paid for default placement in 2021, ~$20B of it to Apple.

Leak-sourced (treat names as leak, not Google-confirmed):
- Ascorer = primary ranking algorithm name.
- Twiddlers = re-ranking functions after Ascorer that adjust IR score or reorder (diversity,
  freshness, demotions).
- ABC signals = Anchors, Body, Clicks, the components of topicality (T*).

Inference / my framing (not quoted from a source):
- The exact ranking math is not public. The durable, defensible claims are the architecture:
  (1) matching (cheap posting-list retrieval on the big set) vs ranking (expensive models on
  the small set) are two halves with different cost budgets; (2) document-sharded index +
  scatter-gather merge is how hundreds of billions of pages are searched in under a second;
  (3) a retrieve-wide-cheap then rank-narrow-expensive cascade; (4) caching + Navboost
  memorization turn the head of the query distribution into a lookup; (5) tiered storage; (6)
  sorting is server-side, never on the client.
- The hand-crafted vs learned "hybrid, keep it debuggable" point is my summary of the spirit of
  the engineers' testimony, not a direct quote.
