# References: Google Search Knowledge Panel and the Knowledge Graph

Saved 2026-08-05.

## Primary (Google)
- "Introducing the Knowledge Graph: things, not strings" (Google blog, 16 May 2012): https://blog.google/products/search/introducing-knowledge-graph-things-not/
  - Launch. ~500 million objects, more than 3.5 billion facts at launch. Sources include Freebase, Wikipedia, CIA World Factbook.
  - Core slogan: the node is a THING (an ID), the query text is a STRING; entity linking closes the gap.
- "How Google's Knowledge Graph works" / knowledge panels explainer (Google Search blog, 2020): https://blog.google/products-and-platforms/products/search/about-knowledge-graph-and-knowledge-panels/
  - More than 500 billion facts about 5 billion entities (May 2020). Panels auto generated, not hand written.

## The graph's origin and lineage
- Google acquires Metaweb / Freebase (announced 16 July 2010): https://www.seobythesea.com/2010/07/google-gets-smarter-with-named-entities-acquires-metaweb/
- Freebase (Wikipedia): final data dump ~1.9 billion RDF triples; shutdown announced 16 Dec 2014, service closed 2 May 2016, data migrated toward Wikidata: https://en.wikipedia.org/wiki/Freebase_(database)
- Wikidata, the open successor knowledge base (entity IDs like Q243 = Eiffel Tower): https://www.wikidata.org/

## Automatic fact harvesting and confidence
- Dong et al., "Knowledge Vault: A Web-Scale Approach to Probabilistic Knowledge Fusion" (Google, KDD 2014): https://dl.acm.org/doi/10.1145/2623330.2623623
  - Extractors over free text, HTML DOM, HTML tables, human annotations propose candidate triples.
  - Graph based priors (Path Ranking Algorithm) fuse with extractors to compute a CALIBRATED probability per fact.
  - A shaky fact with low probability is never surfaced (asymmetric caution, same as the panel confidence gate).

## Entity linking (the matching half)
- "Entity linking" (Wikipedia): two step process, candidate generation then disambiguation/ranking: https://en.wikipedia.org/wiki/Entity_linking
  - "jaguar" candidate set: animal (Panthera onca), Jaguar Cars, Jacksonville Jaguars, Atari Jaguar, Mac OS X Jaguar.
  - Disambiguation signals: prior popularity, rest of query, user context/location, news spikes; below threshold = no panel.

## Retention / zero click context
- SparkToro / Rand Fishkin zero click search studies (third party estimates, NOT Google confirmed): https://sparktoro.com/blog/

## Key technical spine (for reuse)
- Atom = triple (subject, predicate, object) = a labeled directed edge. Node = a stable entity ID, not text.
- Whole KG = one huge directed labeled graph. Neighbours (related entities) are cheap edge follows, not SQL joins.
- Storage: entity ID -> precomputed property bag (adjacency list of predicate/object + top related nodes). Live path = one keyed O(1) read.
- Two halves: matching = entity linking (alias dictionary = inverted index string->entity IDs, then disambiguate); assembly = read property bag + pick facts + render.
- Entity resolution (dedup) merges the same real thing from many sources into one node. Split = half empty panels, over merge = nonsense panel.
- Scale: 1k = one table/map, no ambiguity; 100k = ambiguity + duplicates break, need alias dict + entity resolution; 5B entities / 500B facts = shard by entity ID, replicate hot shards, cache celebrity entities (hot key), precompute property bags so NO live cross shard graph walk, run Knowledge Vault fusion offline. Offline-think/online-lookup.
- Confidence gate: show a panel only above a threshold, else fall back to plain links (same bouncer as Smart Compose / "Did you mean").
