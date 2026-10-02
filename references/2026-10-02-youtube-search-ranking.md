# References: YouTube Search (2026-10-02 teardown)

Keeper links for the YouTube Search teardown. Primary sources first.

## Primary / confirmed

- **YouTube Help, "How YouTube search works"** (official). The three pillars: relevance, engagement, quality. Confirms YouTube uses aggregate watch time for a video on a particular query as a relevance signal, and leans on expertise/authoritativeness/trust for sensitive topics.
  https://support.google.com/youtube/answer/16090438
  (Note: support.google.com was blocked by the egress proxy during this run; the content above is from Google's indexed summary of the page. Re-verify the live page when possible.)

- **Covington, Adams, Sargin, "Deep Neural Networks for YouTube Recommendations," RecSys 2016.** The published anchor for the two-stage funnel (candidate generation shrinks billions to hundreds, ranking scores only those hundreds), approximate-nearest-neighbor serving of learned embeddings, the watch-time (not clicks) objective, and the example-age freshness feature. Applied here to search by analogy, clearly labeled.
  https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45530.pdf
  Mirror: https://cseweb.ucsd.edu/classes/fa17/cse291-b/reading/p191-covington.pdf

## Grounded class-level references (how this class of problem is solved)

- **VdoCipher, "YouTube Tech Stack: How 2.5B Users Stream Video at Scale."** Public writeup describing an Elasticsearch-style inverted index over titles, descriptions, transcripts, and auto-captions, with BM25 scoring, autocomplete, spell correction, and personalized ranking. Treat as grounded inference, not YouTube-confirmed internals.
  https://www.vdocipher.com/blog/youtube-tech-stack-architecture/

- **iPullRank, "Relevance Engineering for YouTube Video and AI Search."** How title/description/transcript semantic relevance interacts with engagement signals in observed ranking behavior.
  https://ipullrank.com/video-youtube-relevance-engineering

## Scale figures

- **Wikipedia, "YouTube."** ~2.7 to 2.9 billion monthly logged-in users, 500+ hours uploaded per minute, 2 billion+ video catalog, 1 billion+ hours watched per day.
  https://en.wikipedia.org/wiki/YouTube

- **Tubular Labs, "500 Hours of Video Uploaded To YouTube Every Minute."** Upload rate (500 hours/minute, ~30,000 hours/hour, ~720,000 hours/day).
  https://tubularlabs.com/blog/hours-minute-uploaded-youtube/

- **Oklahoma State University Libraries, "YouTube: The World's Second Largest Search Engine."** Search-volume framing; estimate of billions of searches a month. Exact daily counts vary by source, so the robust claim is "second-largest search engine," not any single number.
  https://open.library.okstate.edu/introtosocialmedia/chapter/youtube-the-worlds-second-largest-search-engine/

## Cross-links in this ledger

- Matching-then-ranking + inverted index: Amazon product search ranking (2026-06-23), Google Search crawling/indexing (2026-09-08).
- Two-tower + approximate nearest neighbor retrieval: YouTube recommendations (2026-06-22), Netflix artwork personalization (2026-06-19).
- Autocomplete / typeahead (the dropdown before enter): Google Search autocomplete (2026-06-16).
- Spell correction on the query: Google Search spell correction (2026-07-05).
- Spatial cull before expensive work (the Rare.lab per-frame lesson): Swiggy serviceability (2026-09-28).
- Signed per-device performance quote baked into ranking: Uber upfront fares (2026-09-13).
- Invisible-craft trust retention: Spotify loudness (2026-08-28), Netflix picture quality (2026-09-11).
