# Gmail Search: keeper references

Saved for the 2026-10-03 teardown (product-teardowns/2026-10-03-gmail-search.md).

## Primary (Google)

- Google blog, "Gmail's new search update finds relevant emails faster" (Most relevant ranking; signals: recency, frequent contacts, most-clicked; global rollout to personal accounts, March 2025).
  https://blog.google/products-and-platforms/products/gmail/gmail-search-update-relevant-emails/
- Google Workspace Updates, "See the top search results first in Gmail on mobile" (June 2023; machine-ranked top results shown above the chronological list).
  https://workspaceupdates.googleblog.com/2023/06/see-top-search-results-first-in-gmail.html
- Gmail Help, "Refine searches in Gmail" (operator grammar: from:, to:, subject:, has:attachment, filename:, older_than:, larger:; space = AND, leading hyphen = NOT).
  https://support.google.com/mail/answer/7190

## Storage and infrastructure lineage

- Chang et al., "Bigtable: A Distributed Storage System for Structured Data," OSDI 2006 (Google's wide-column NoSQL store; Google states it powers core services including Gmail).
  https://research.google/pubs/pub27898/
- Corbett et al., "Spanner: Google's Globally-Distributed Database," OSDI 2012 (globally distributed SQL store; used by Gmail and Google Photos).
  https://research.google.com/archive/spanner-osdi2012.pdf

## Background

- Wikipedia, "Search engine indexing" (inverted index = key-value map from term to sorted list of documents containing it).
  https://en.wikipedia.org/wiki/Search_engine_indexing
- TechCrunch, "Gmail's new AI search now sorts emails by relevance instead of chronological order" (March 20, 2025; secondary coverage of the Most relevant launch).
  https://techcrunch.com/2025/03/20/gmails-new-ai-search-now-sorts-emails-by-relevance-instead-of-chronological-order

## Numbers used

- Google user milestones: 1B MAU (Feb 2016), 1.5B (April 2019), more than 2.5B (Dec 2024), 3B (Jan 2026). 15 GB free storage shared across Gmail, Drive, and Photos.
  Roundup citing Google's official announcements: https://emailanalytics.com/gmail-statistics/

## The one line

Gmail search is the inverse of web search: billions of tiny per-user inverted indexes instead of one giant shared one, so the hard problem is multi-tenant sharding plus near-real-time incremental indexing rather than candidate generation from billions, and ranking only moved from strict recency to learned personal relevance in March 2025.
