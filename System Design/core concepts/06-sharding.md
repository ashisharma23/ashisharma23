# Database Sharding in Production Systems

## Problem & Context

A single database instance has finite limits on storage (a few TB comfortably), write throughput (tens of thousands of transactions per second), and connection count. Read replicas scale reads, but writes are still bound to a single primary. Sharding splits data across multiple independent database instances so write throughput and storage scale horizontally — at the cost of losing single-instance transactions and joins across the whole dataset.

## Core Concept

Sharding partitions a dataset by a **shard key** so each shard holds a disjoint subset of rows, and each shard runs as an independent database with its own compute and storage. The choice of shard key determines which queries stay fast (single-shard) and which become expensive (scatter-gather across all shards).

## Production Architecture

| Component | Responsibility |
|---|---|
| Shard Router / Proxy | Maps a request's shard key to the correct shard |
| Shard (Postgres/MySQL instance) | Owns a partition of the data, independent scaling |
| Config Service | Tracks shard-to-key-range mapping |
| Cross-shard Aggregator | Executes and merges scatter-gather queries |
| Resharding Pipeline | Moves data when adding/removing shards |

The router owns request-to-shard mapping and must be kept in sync with the config service; each shard owns its own data and indexes independently — shards should never directly query each other synchronously.

## Architecture / Data Flow

```mermaid
flowchart TD
    App[Application] --> Router[Shard Router]
    Router -->|hash(user_id) % N| S1[(Shard 1: users 0-999)]
    Router -->|hash(user_id) % N| S2[(Shard 2: users 1000-1999)]
    Router -->|hash(user_id) % N| S3[(Shard 3: users 2000-2999)]
    Config[(Shard Config Service)] --> Router
```

**Flow:**
1. A write or read request includes the shard key (e.g., `user_id`).
2. The router hashes the key and consults the config service to determine which shard owns that key range.
3. The request is forwarded to the owning shard only — no cross-shard coordination needed for single-user queries.
4. For queries spanning users (e.g., admin reports), a scatter-gather aggregator fans out to every shard and merges results, which is inherently slower and should be rare.

## Concrete Example

A social platform shards its `posts` table by `user_id`, so fetching "all posts by user X" (the dominant query pattern) hits exactly one shard. A secondary use case — "trending posts across all users" — is deliberately not served from the sharded OLTP store; instead it's computed via a separate streaming aggregation pipeline (Kafka + Flink) that doesn't require scatter-gather across shards on every request.

## AI & Cloud Architecture

Vector databases used in RAG systems are commonly sharded by **tenant ID** in multi-tenant SaaS deployments, ensuring one customer's embedding volume and query load can't degrade another's search latency, and simplifying data deletion/GDPR compliance (drop a tenant's shard entirely rather than filtering rows). Managed services like DynamoDB and Cassandra handle sharding (partitioning) internally via consistent hashing, so architects choose the partition key but don't operate the resharding pipeline themselves.

## Technology Choices

- **Hash-based sharding**: default choice — even distribution, avoids hot spots, but makes range queries and resharding harder.
- **Range-based sharding**: works when access patterns naturally align with ranges (e.g., multi-tenant SaaS, one tenant per shard); risks hot spots if one range gets disproportionate traffic.
- **Directory-based sharding**: most flexible, but the lookup adds latency and a dependency to every query — rarely justified outside very large, evolving systems.
- **Managed sharded databases (DynamoDB, Cassandra, Vitess, CockroachDB)**: preferred over hand-rolled sharding when available, since resharding and rebalancing are handled by the platform.

## Scalability

- Do the capacity math before sharding: if you're at 10K writes/sec and 100GB, a single well-tuned instance with read replicas is likely sufficient.
- Choose a shard key that matches your dominant query pattern to keep the vast majority of queries single-shard.
- Plan for resharding from day one — even if you don't need it immediately, using consistent hashing (see the dedicated article) makes adding shards later far less disruptive.

## Reliability

- Cross-shard transactions are effectively unavailable in most sharded relational setups — design shard boundaries to avoid needing them, or use the saga pattern for cross-shard workflows that require compensating actions on failure.
- Hot spots (a single shard receiving disproportionate load) need monitoring and, potentially, a "hot key" mitigation like sub-sharding a single popular key.
- Resharding must be done as an online, incremental migration (dual-write or CDC-based backfill) to avoid downtime; a big-bang cutover is a common source of outages.

## Security & Observability

- Per-shard access controls and encryption keys where regulatory requirements (e.g., data residency) dictate that certain tenants' shards must live in specific regions.
- Track per-shard latency and load as separate SLIs — an aggregate p99 can hide one hot, struggling shard.
- Audit cross-shard aggregator queries specifically, since they're the most likely source of timeouts and resource exhaustion under load.

## Trade-offs

| Choice | When it wins | When it loses |
|---|---|---|
| Hash-based | Even load distribution | Range queries and resharding are harder |
| Range-based | Natural query alignment (tenant, geography) | Hot spots on unevenly-loaded ranges |
| Directory-based | Maximum flexibility | Added latency, single point of failure risk |

## Common Pitfalls & Best Practices

- **Pitfall**: sharding prematurely, before actual write throughput or storage justifies the operational complexity.
- **Pitfall**: choosing a shard key that doesn't match the dominant access pattern, turning every query into scatter-gather.
- **Pitfall**: no resharding plan, forcing a painful manual migration when a shard becomes too hot.
- **Best practice**: state the shard key and its trade-off explicitly ("fast for per-user queries, slow for cross-user aggregation") as part of the design, not as an afterthought.

## Staff Engineer Perspective

Sharding is one of the most expensive architectural decisions to reverse — once data and application logic assume a shard key, changing it requires a full data migration under live traffic. The judgment call is resisting the urge to shard as a default scaling strategy and instead exhausting vertical scaling and read replicas first. At 10x scale, the real risk isn't lacking shards — it's having chosen the wrong shard key early, which forces a resharding project under production pressure rather than as a planned initiative.

## When to Use / Avoid

- **Use** sharding once write throughput, storage, or connection limits on a single well-tuned instance are demonstrably insufficient.
- **Use** managed sharded databases when available instead of building custom shard routing.
- **Avoid** sharding as a first response to "the database is slow" — check indexing, caching, and read replicas first.
