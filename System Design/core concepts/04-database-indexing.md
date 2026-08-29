# Database Indexing in Production Systems

## Problem & Context

As tables grow past a few hundred thousand rows, queries that scan every row become the dominant source of latency and database load. Indexing is the primary lever for making reads fast without changing your data model — but every index also adds write overhead and storage, so indexing is a trade-off decision, not a free performance win.

## Core Concept

An index is a secondary data structure that maps column values to row locations, letting the database avoid a full table scan. The default structure is a **B-tree**, which supports both exact-match and range queries in O(log n). Specialized indexes (hash, full-text, geospatial, vector) trade generality for performance on specific query shapes.

## Production Architecture

| Component | Responsibility |
|---|---|
| Primary DB (Postgres/MySQL) | B-tree indexes on hot query columns |
| Full-text Search Engine (Elasticsearch/OpenSearch) | Inverted-index-based text search |
| Geospatial Extension (PostGIS) | R-tree/GiST indexes for location queries |
| CDC Pipeline | Sync primary DB changes to search/derived indexes |
| Query Planner / Observability | `EXPLAIN ANALYZE`, slow query logs |

The primary database owns transactional correctness and simple indexed lookups; specialized engines own query shapes the primary database can't index efficiently (full-text relevance ranking, geospatial radius search, vector similarity).

## Architecture / Data Flow

```mermaid
flowchart LR
    Write[Write Path] --> DB[(Postgres)]
    DB -->|B-tree index| ReadFast[Indexed lookup: O(log n)]
    DB -->|CDC| Kafka[(Kafka)]
    Kafka --> ES[(Elasticsearch)]
    Search[Full-text search query] --> ES
    Geo[Radius search] --> PostGIS[(PostGIS index)]
```

**Flow:**
1. A write inserts/updates a row; every index on that table is updated synchronously, adding write latency proportional to index count.
2. A query filtering on an indexed column uses the B-tree to jump directly to matching rows instead of scanning the table.
3. For full-text search, CDC streams changes to Elasticsearch, which maintains its own inverted index asynchronously.
4. Geospatial queries use PostGIS's spatial index (GiST) directly in the primary database when latency requirements allow synchronous queries.

## Concrete Example

An auth service indexes `email` (unique B-tree index) for login lookups, and a compound index on `(user_id, created_at)` on the `sessions` table to serve "get active sessions for user X, most recent first" without a sort step. Adding a third, rarely-used index on `last_login_at` for an internal admin report is deferred to a read replica to avoid slowing down the write-heavy primary.

## AI & Cloud Architecture

Vector databases (Pinecone, pgvector, Milvus) use **approximate nearest neighbor (ANN)** indexes — HNSW or IVF — instead of B-trees, trading exact recall for sub-linear search time over millions of embeddings. In a RAG pipeline, this index sits alongside a metadata filter (e.g., tenant ID, document date) that must be applied efficiently; poorly designed vector indexes without metadata pre-filtering force a full scan of irrelevant tenants' data, both a performance and a data-isolation risk.

## Technology Choices

- **B-tree** (default in Postgres/MySQL): general purpose, supports range queries — use for the vast majority of columns.
- **Hash index**: faster exact-match lookups but no range queries — rarely worth it over B-tree in practice.
- **GIN/GiST** (Postgres): full-text search and geospatial queries within the primary database, avoiding a separate search cluster for moderate scale.
- **Elasticsearch**: when search volume or relevance-ranking needs exceed what Postgres full-text search can deliver.
- **HNSW/IVF (vector)**: for semantic similarity search at scale in AI retrieval systems.

## Scalability

- Every additional index slows down writes — measure write throughput impact before adding indexes speculatively.
- Composite indexes should match the query's filter and sort order exactly (leftmost-prefix rule); a mismatched compound index won't be used.
- Move analytical/report queries to read replicas so their required indexes don't compete with OLTP write load on the primary.
- For search at scale, shard the Elasticsearch index by tenant or time range to keep individual shards queryable within SLA.

## Reliability

- Adding an index on a large table can lock it (in some databases); use `CREATE INDEX CONCURRENTLY` (Postgres) to avoid write downtime.
- CDC-fed search indexes lag the primary — design UX and API contracts to tolerate a few seconds of staleness for search results.
- Monitor for index bloat (especially after heavy delete/update churn) and schedule periodic `VACUUM`/reindexing.

## Security & Observability

- Full-text indexes can inadvertently expose sensitive fields in search results — apply the same field-level access controls to the index as to the source table.
- Track slow query logs and `EXPLAIN ANALYZE` output as part of routine review, not only during incidents.
- Alert on unindexed queries exceeding a latency threshold in production traffic (many databases expose this via query statistics extensions like `pg_stat_statements`).

## Trade-offs

| Choice | When it wins | When it loses |
|---|---|---|
| More indexes | Read-heavy workload, few writes | Write-heavy workload — each index adds write cost |
| In-DB full-text (GIN) | Moderate search volume, want to avoid a new system | High-volume search with complex relevance ranking |
| External search engine | Complex ranking, faceting, high query volume | Adds infra, eventual consistency with primary |

## Common Pitfalls & Best Practices

- **Pitfall**: indexing every column "just in case," degrading write throughput without measurable read benefit.
- **Pitfall**: a compound index built in the wrong column order, silently unused by the query planner.
- **Pitfall**: full table scans in production going unnoticed because response times are still "acceptable" at current scale, until a 10x traffic spike.
- **Best practice**: index based on `EXPLAIN`-verified query plans, not intuition; revisit indexes quarterly as query patterns evolve.

## Staff Engineer Perspective

Indexing decisions are individually small but collectively define your database's ceiling. The judgment call is knowing when a query pattern justifies a new index (a slow, high-frequency query) versus when it's a one-off report that belongs on a replica or warehouse instead. At 10x scale, the failure mode is almost always a missing index on a newly hot query path that wasn't hot when the schema was designed — build in a review cadence, not just a one-time indexing pass.

## When to Use / Avoid

- **Use** B-tree indexes on every column used in `WHERE`, `JOIN`, or `ORDER BY` clauses of frequent queries.
- **Use** a dedicated search engine once relevance ranking, faceting, or query volume exceeds what in-database full-text search comfortably handles.
- **Avoid** adding indexes to write-heavy tables without profiling — the write cost may outweigh the read benefit for rarely-run queries.
