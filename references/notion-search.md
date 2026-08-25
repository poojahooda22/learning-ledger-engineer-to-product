# References: Notion Search (Quick Find + AI Q&A)

Saved 2026-08-25 for the Notion search teardown.

## Notion engineering (primary)
- Rebuilding Notion's lexical search reindexer. https://www.notion.com/blog/rebuilding-notions-lexical-search-reindexer
  - Old ECS/Fargate/Node.js + Snowflake-on-hot-path + separate Kafka catchup pipeline: 2+ weeks full rebuild, ~90% consistency, ~2-day catchup, constant manual intervention.
  - New Apache Spark-native pipeline writing Elasticsearch snapshot format + native primitives: under 2 days rebuild, under 1 hour catchup, 100% consistency, under 2 hours manual, zero on-call pages.
  - Doc-generation logic does "tree traversal and permission data construction for each block" (permission denormalized onto each ES document at index time). Reasons for full rebuilds: new searchable object types (Custom Agents), tokenizer changes (non-Latin scripts), Elasticsearch upgrades, new regions.
- Two years of vector search at Notion: 10x scale, 1/10th cost (published ~Feb 2026). https://www.notion.com/blog/two-years-of-vector-search-at-notion
  - AI Q&A launched Nov 2023; waitlist of millions of workspaces; first arch = pod clusters (storage+compute bundled) sharded by workspace id, hit capacity in ~1 month.
  - Stopgap: generation-id routing, spin up new index clusters and route new workspaces to fresh generations, leave existing workspaces in place.
  - Real fix (late 2024): migrate multi-billion-object workload to Turbopuffer (serverless, object-storage-backed vectors). Embeddings + serving moved to Ray on Anyscale.
  - Page-state caching: chunk page, compare text hash and metadata hash per chunk; if text hash same but metadata (permissions) hash differs, skip re-embed and just PATCH the metadata.
  - Results: cleared multi-million-workspace waitlist, vector DB cost -60%, embeddings infra cost over -90%, query latency 70-100ms to 50-70ms, ~15x active-workspace growth.
- Building and scaling Notion's data lake (Debezium CDC -> Kafka -> Hudi -> Spark -> S3). https://www.notion.com/blog/building-and-scaling-notions-data-lake

## Turbopuffer (vector DB Notion migrated to)
- Architecture: stateless query tier over object storage (S3/GCS/Azure), NVMe + RAM cache, SPFresh centroid-based ANN index, 2-4 round trips per cold query, strong consistency default. https://turbopuffer.com/docs/architecture
- ANN v3: 200ms p99 over 100 billion vectors; sub-10ms p50 warm. https://turbopuffer.com/blog/ann-v3
- fast search on object storage. https://turbopuffer.com/blog/turbopuffer

## Secondary / analysis
- ZenML LLMOps DB, Scaling Vector Search Infrastructure for AI-Powered Workspace Search. https://www.zenml.io/llmops-database/scaling-vector-search-infrastructure-for-ai-powered-workspace-search
- ZenML LLMOps DB, Rebuilding a Production Search Reindexing Pipeline at Scale. https://www.zenml.io/llmops-database/rebuilding-a-production-search-reindexing-pipeline-at-scale
- Zilliz, Notion Vector Search Architecture: What Comes Next. https://zilliz.com/blog/notion-vector-search-next-problem
- Notion Help, Search for pages & content. https://www.notion.com/help/search
- Notion Help, Enterprise Search security & privacy practices. https://www.notion.com/help/enterprise-search-security-and-privacy-practices
- Elastic, Notion connector reference (notes Notion's crawl API does not expose per-document ACLs to external systems). https://www.elastic.co/docs/reference/search-connectors/es-connectors-notion
