# References: Google Search web crawling and index building

Saved keepers for the 2026-09-08 teardown on how Googlebot crawls the web and how
the Caffeine / Percolator pipeline builds and freshens the inverted index.

## Primary (the foundational papers and Google's own posts)

- **The Anatomy of a Large-Scale Hypertextual Web Search Engine** (Sergey Brin and
  Lawrence Page, Stanford, 1998). Google's original architecture in public: the
  crawler, the repository of raw pages, the lexicon (word to pointer), the "barrels"
  holding posting lists, the forward index, and the sorter that turns the forward
  index into the inverted index by sorting (word, document) pairs. Every later index
  is this shape scaled up.
  http://infolab.stanford.edu/~backrub/google.html

- **Our new search index: Caffeine** (Google Search Central Blog, June 2010). The
  primary source for the batch-to-incremental switch. Key quotes and numbers: the old
  index was built in layers and "to refresh a layer of the old index, we would analyze
  the entire web"; Caffeine "takes up nearly 100 million gigabytes of storage in one
  database and we add new information at a rate of hundreds of thousands of gigabytes
  per day"; results are "50 percent fresher."
  https://developers.google.com/search/blog/2010/06/our-new-search-index-caffeine

- **Large-scale Incremental Processing Using Distributed Transactions and
  Notifications** (Daniel Peng and Frank Dabek, Google, OSDI 2010). The Percolator
  paper under Caffeine. Built on Bigtable; adds cross-row snapshot-isolation
  transactions and observer/notify-column notifications so a crawled document ripples
  only through the rows it affects. Reports the median document moving through the
  pipeline over 100x faster than the old MapReduce system, at the cost of roughly 30x
  more CPU per document (latency bought with throughput).
  https://research.google/pubs/pub36726/

## Primary (Google's crawl-side documentation)

- **Crawling December** series (Google Search Central Blog, December 2024). The
  resources/rendering post especially: rendering with the Web Rendering Service pulls a
  page's sub-resources, which "chips away from the crawl budget of the hostname," so
  WRS "caches everything for up to 30 days" and that cache is "unaffected by HTTP
  caching directives." Also covers scheduling by perceived server load.
  https://developers.google.com/search/blog/2024/12/crawling-december-resources

- **What Crawl Budget Means for Googlebot** (Google Search Central Blog, January 2017).
  Defines crawl budget as crawl rate limit plus crawl demand; crawl rate limit is the
  number of simultaneous parallel connections and the wait between fetches, tuned by
  crawl health (fast responses raise it, slow responses or server errors lower it).
  https://developers.google.com/search/blog/2017/01/what-crawl-budget-means-for-googlebot

## Secondary and classic (for the crawler internals Google does not fully publish)

- **Mercator: A Scalable, Extensible Web Crawler** (Allan Heydon and Marc Najork,
  Compaq/AltaVista, 1999). The classic public blueprint for a web-scale crawl frontier:
  per-host politeness queues, URL deduplication, and an extensible processing pipeline.
  The reference for the "how this class of problem is solved" inference in the teardown.

- **The Register**: "Google Caffeine jolts worldwide search machine" (June 2010) and
  "Google Percolator, global search jolt sans MapReduce comedown" (September 2010).
  Contemporary technical reporting that corroborates the Caffeine and Percolator
  numbers.
  https://www.theregister.com/2010/09/24/google_percolator/

## Why kept

This is the build side of every search teardown in the ledger. Autocomplete, spell
correction, PageRank, Featured Snippets all READ an index; these sources explain who
WRITES it and how it stays fresh. The batch-to-incremental (MapReduce to Percolator)
story is the cleanest real-world example of "make the expensive recompute incremental
by dirty-marking a dependency graph," which is the reusable lesson.
