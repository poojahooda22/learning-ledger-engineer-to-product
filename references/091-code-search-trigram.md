# Day 91 references: code search

- GitHub Engineering, The technology behind GitHub's new code search (Blackbird, Rust, sharded ngram index, Kafka): https://github.blog/engineering/architecture-optimization/the-technology-behind-githubs-new-code-search/
- Russ Cox, Regular Expression Matching with a Trigram Index (how Google Code Search worked): https://swtch.com/~rsc/regexp/regexp4.html
- Zoekt design doc (positional trigrams, mmap shards): https://gerrit.googlesource.com/zoekt/+/master%5E/doc/design.md
- Sourcegraph, shard merging for tiny repos: https://sourcegraph.com/blog/tackling-the-long-tail-of-tiny-repos-with-shard-merging
- Etsy Hound (trigram code search): https://changelog.com/posts/hound-lightning-fast-code-search-tool
- Cursor, fast regex search: https://cursor.com/en-US/blog/fast-regex-search
- PostgreSQL pg_trgm: https://www.postgresql.org/docs/current/pgtrgm.html
Note: pages were not fetchable in the session (proxy), only search excerpts were read.
