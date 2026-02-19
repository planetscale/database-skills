---
name: clickhouse
description: Plan and review ClickHouse schema, table engines, indexing, query tuning, materialized views, and operations. Use when creating or modifying ClickHouse tables or queries; diagnosing slow queries; planning data ingestion pipelines; or working with replication, partitioning, and TTL policies.
---

# ClickHouse

Use this skill to make safe, measurable ClickHouse changes.

## Workflow
1. Define workload and constraints (query patterns, data volume, write rate, ClickHouse version, cluster topology).
2. Read only the relevant reference files linked in each section below.
3. Propose the smallest change that can solve the problem, including trade-offs.
4. Validate with evidence (`EXPLAIN`, `EXPLAIN PIPELINE`, system tables, and production-safe rollout steps).
5. For production changes, include rollback and post-deploy verification.

## Schema Design
- Choose the right MergeTree variant for your workload: plain `MergeTree` for append-only, `ReplacingMergeTree` for deduplication, `AggregatingMergeTree` for pre-aggregation.
- Order columns by cardinality (low → high) in `ORDER BY` to maximize compression and index selectivity.
- Avoid `Nullable` columns unless truly needed — they add overhead and complicate aggregation.
- Prefer `LowCardinality(String)` over plain `String` for columns with few distinct values.

References:
- [table-engines](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/table-engines.md)
- [schema-design](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/schema-design.md)

## Indexing
- The primary index in ClickHouse is sparse — it indexes granules, not individual rows. Design `ORDER BY` with this in mind.
- `ORDER BY` and `PRIMARY KEY` serve different purposes: `ORDER BY` determines physical sort order and data locality; `PRIMARY KEY` controls what gets into the sparse index.
- Add skipping indexes only after confirming they provide measurable benefit — they add write and merge overhead.

References:
- [primary-key-index](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/primary-key-index.md)
- [skipping-indexes](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/skipping-indexes.md)

## Partitioning & TTL
- Partition by time (e.g., `toYYYYMM(event_date)`) for time-series data to enable efficient partition pruning and drops.
- Avoid over-partitioning: thousands of partitions slow down merges and increase memory usage.
- Use TTL to automatically expire or move old data to cheaper storage tiers.

References:
- [partitioning](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/partitioning.md)
- [ttl](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/ttl.md)

## Query Optimization
- Use `PREWHERE` instead of `WHERE` to filter rows before reading full column data — ClickHouse often does this automatically, but explicit `PREWHERE` can help.
- Avoid `SELECT *` — ClickHouse is columnar; only read columns you need.
- Prefer `JOIN` with small tables on the right side; large-to-large joins are expensive without careful planning.
- Use materialized views to pre-aggregate or pre-filter frequently queried data.

References:
- [explain-analysis](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/explain-analysis.md)
- [query-optimization](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/query-optimization.md)
- [materialized-views](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/materialized-views.md)

## Data Ingestion
- Batch inserts aggressively: aim for ≥1000 rows (ideally tens of thousands) per INSERT to avoid excessive part creation.
- Use async inserts or a buffer layer (Kafka, ClickHouse Buffer engine) for high-frequency small writes.
- Never insert one row at a time in a loop from application code against MergeTree tables.

References:
- [data-ingestion](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/data-ingestion.md)

## Replication & Clustering
- Use `ReplicatedMergeTree` (or its variants) with ClickHouse Keeper or ZooKeeper for HA replication.
- Use the `Distributed` table engine as the query entry point; underlying tables are `Replicated*MergeTree`.
- Choose sharding keys carefully — poor key selection leads to data skew and hot shards.

References:
- [replication](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/replication.md)

## Operations
- `ALTER TABLE ... UPDATE/DELETE` are heavy mutations — they rewrite entire parts. Use sparingly and monitor with `system.mutations`.
- Prefer lightweight deletes (`DELETE FROM`) on recent ClickHouse versions when available.
- Avoid frequent `OPTIMIZE TABLE` in production; let the background merge process run naturally.

References:
- [mutations](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/clickhouse/references/mutations.md)

## Guardrails
- Prefer measured evidence over blanket rules of thumb.
- Note ClickHouse-version-specific behavior when giving advice (features vary significantly between versions).
- Ask for explicit human approval before destructive data operations (drops, mutations, partition detaches/drops).
