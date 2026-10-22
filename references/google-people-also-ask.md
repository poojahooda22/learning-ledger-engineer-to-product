# References: Google Search People Also Ask (PAA)

Keeper links for the 2026-10-22 teardown. Primary sources first, then credible
secondary deep-dives. Fact vs inference notes included.

## Primary (patents and Google announcements)

- US Patent US9213748B1, "Generating related questions for search queries,"
  Google LLC, inventors Yossi Matias and Ziv Bar-Yossef.
  https://patents.google.com/patent/US9213748
  Core method CONFIRMED by patent text: a QUESTION GRAPH with one node per
  question in a question database; an EDGE between two questions when they share
  at least a threshold number of common "selected resources" (pages users clicked
  after asking that question). Adjacency is click co-selection, not word overlap.
  (That the live PAA product runs exactly this patent is INFERENCE, well grounded
  by how closely the behaviour matches.)

- US Patent US9679027B1, continuation of the above.
  https://patents.google.com/patent/US9679027

- BERT applied to Google Search and Featured Snippets: rolled out December 2019 in
  70 languages, used for disambiguation and passage understanding in snippets.
  (Google announcement, reported widely; see Search Engine Land link below.)

- Passage ranking / passage indexing: shipped 2021, ranks a single relevant
  passage within an otherwise unfocused page rather than judging the whole page as
  one blob. PAA answers ride the same Featured Snippet extraction path.

## Secondary deep-dives (credible)

- Go Fish Digital, analysis of Google's related-questions and contextual-questions
  patents: https://gofishdigital.com/blog/contextual-questions/

- Search Engine Land, "Could Google passage indexing be leveraging BERT?":
  https://searchengineland.com/could-google-passage-indexing-be-leveraging-bert-342975

- Search Engine Land, "Google People Also Ask boxes not showing as often now?"
  (MozCast share dropping from 93% to 87% of queries, a real observed data point):
  https://searchengineland.com/google-people-also-ask-boxes-not-showing-as-often-now-330321

- Search Engine Roundtable, early PAA coverage and launch timing (2015):
  https://seroundtable.com/people-also-ask-google-26104.html

- Ahrefs blog on People Also Ask (dynamic expansion, occasionally "infinite," most
  chains stop after a few new questions): https://ahrefs.com/blog/people-also-ask

- Semrush study on People Also Ask: https://www.semrush.com/blog/people-also-ask/

- Hallme, "Google's Passage Ranking Launched in February" (passages ranked within
  whole pages, not indexed separately):
  https://www.hallme.com/blog/googles-passage-ranking-launched-in-february-heres-what-that-means/

- Dr. Pete Meyers (Moz / MozCast), tracker of PAA growth across the SERP data set
  (from roughly 40 percent of SERPs early to past 80 percent at peak, then a
  settle-back): https://drpete.co/who-is-dr-pete

## What is confirmed vs inferred

- CONFIRMED: PAA exists since 2015; the US9213748B1 question-graph /
  click-co-selection method; BERT on Featured Snippets (Dec 2019, 70 languages);
  passage ranking (2021); PAA answers share the Featured Snippet extraction path;
  the lazy, grow-on-tap expansion behaviour; MozCast tracking of its SERP share.
- INFERENCE (labelled as such in the report): that the live PAA system runs
  exactly the patent; the inverted-index and sharding specifics; the lazy-fetch
  plumbing; the caching specifics. These are the standard way this class of
  problem is solved web-scale, not a published Google design.
