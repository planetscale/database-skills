---
title: Table Engines
description: Overview of ClickHouse MergeTree family engines and when to use each variant
tags: clickhouse, table-engines, mergetree, replication, aggregation
---

# Table Engines

ClickHouse has many table engines, but the **MergeTree family** handles almost all production OLAP workloads.

## MergeTree (Base Engine)

The default choice for append-only analytics tables:

```sql
CREATE TABLE events (
    event_date  Date,
    user_id     UInt64,
    event_type  LowCardinality(String),
    payload     String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_type, user_id, event_date);
```

- Data is written as immutable **parts** and merged in the background.
- `ORDER BY` defines the physical sort order and the sparse primary index.
- Use when: simple append-only writes, no deduplication needed.

## ReplacingMergeTree

Deduplicates rows with the same `ORDER BY` key during merges:

```sql
ENGINE = ReplacingMergeTree(version_column)
ORDER BY (user_id, record_id);
```

- Deduplication is **not synchronous** — duplicates may exist until a merge occurs.
- `version_column` (UInt, Date, or DateTime): the row with the highest version is kept.
- Use `FINAL` modifier or `argMax` in queries if you need deduplicated reads before merge completes.
- Use when: CDC pipelines, upsert-like patterns, slowly changing dimensions.

**Gotcha:** `SELECT ... FINAL` forces a merge at read time — expensive on large tables.

## AggregatingMergeTree

Stores aggregate function states instead of raw values; merges combine states:

```sql
CREATE TABLE page_views_agg (
    date        Date,
    page        LowCardinality(String),
    views       AggregateFunction(count, UInt64),
    uniq_users  AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (date, page);
```

- Always used as the backing table for materialized views with aggregate queries.
- Query with `-Merge` combiners: `countMerge(views)`, `uniqMerge(uniq_users)`.
- Use when: pre-aggregated rollup tables, materialized views over high-volume streams.

## CollapsingMergeTree

Handles row "cancellation" using a `sign` column (+1 = add, -1 = cancel):

```sql
ENGINE = CollapsingMergeTree(sign)
ORDER BY (session_id, user_id);
```

- Merges collapse pairs of +1/−1 rows with the same sort key.
- Before merge, queries must account for intermediate duplicates (use `sign` in queries).
- Use when: event sourcing patterns, session tracking with cancellations.

## VersionedCollapsingMergeTree

Like `CollapsingMergeTree` but also takes a `version` column — handles out-of-order cancellation events:

```sql
ENGINE = VersionedCollapsingMergeTree(sign, version)
ORDER BY (entity_id);
```

## SummingMergeTree

Sums numeric columns during merges for rows with the same `ORDER BY` key:

```sql
ENGINE = SummingMergeTree((amount, count))
ORDER BY (date, category);
```

- Only the listed columns are summed; other columns take the value from an arbitrary kept row.
- Use `sum()` in queries regardless (some duplicates may exist pre-merge).
- Use when: simple counter tables, pre-aggregated totals without complex aggregates.

## Engine Selection Guide

| Use Case | Engine |
|---|---|
| Append-only analytics | `MergeTree` |
| Upserts / CDC dedup | `ReplacingMergeTree` |
| Pre-aggregated rollups | `AggregatingMergeTree` |
| Event cancellation patterns | `CollapsingMergeTree` |
| Simple numeric counters | `SummingMergeTree` |
| HA replication | `Replicated*MergeTree` |

## Non-MergeTree Engines (Special Cases)

- **Buffer**: buffers writes in RAM before flushing to a target MergeTree table. Useful for absorbing bursts of small inserts.
- **Distributed**: a virtual engine that distributes queries across shards. Always backed by real MergeTree tables.
- **Memory**: stores data in RAM only. Use for temp tables or testing — data lost on restart.
- **Null**: discards all writes. Useful as a source for materialized views that only need the MV's target table.
