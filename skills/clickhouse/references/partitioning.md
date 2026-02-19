---
title: Partitioning
description: ClickHouse time-based partitioning, partition pruning, managing partitions, and avoiding over-partitioning
tags: clickhouse, partitioning, partition-pruning, time-series, mergetree
---

# Partitioning

Partitioning in ClickHouse divides table data into separate directories on disk by a partition key expression. Unlike some databases, ClickHouse partitioning is primarily a data management and pruning tool — not an indexing mechanism.

## Defining a Partition Key

```sql
CREATE TABLE events (
    event_date  Date,
    user_id     UInt64,
    event_type  LowCardinality(String),
    ...
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_type, user_id, event_date);
```

Common partition expressions:
- `toYYYYMM(date_col)` — monthly partitions (most common for time-series)
- `toYYYYMMDD(date_col)` — daily partitions (use only for high-volume tables)
- `toYear(date_col)` — annual partitions (for lower-volume historical data)
- `(tenant_id, toYYYYMM(date_col))` — multi-column partition for multi-tenant data

## Partition Pruning

When a query includes a filter on the partition key column, ClickHouse reads **only the matching partitions**:

```sql
-- Partition pruning: only reads 2024-01 partition
SELECT count() FROM events WHERE event_date >= '2024-01-01' AND event_date < '2024-02-01';

-- No partition pruning: scans all partitions
SELECT count() FROM events WHERE user_id = 12345;
```

**Important:** Pruning requires the filter to be on the partition key expression, not just a related column. Wrapping the column in a function that doesn't match the partition expression will defeat pruning.

```sql
-- PARTITION BY toYYYYMM(event_date)

-- Good: direct filter on partition column
WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31'

-- Also good: ClickHouse can evaluate toYYYYMM for pruning
WHERE toYYYYMM(event_date) = 202401

-- Bad: non-partition function prevents pruning
WHERE formatDateTime(event_date, '%Y-%m') = '2024-01'
```

## Avoiding Over-Partitioning

**This is the most common ClickHouse partitioning mistake.** Each insert creates at least one new part per partition touched. Too many partitions causes:
- Excessive part count → slow merges, high memory usage, merge pressure warnings.
- `Too many parts` errors when background merges can't keep up with writes.
- Slow `SHOW PARTITIONS`, `system.parts` queries.

Guidelines:
- **Monthly partitions** are suitable for most time-series tables up to billions of rows/day.
- **Daily partitions** only if you need per-day data management (drops, moves) and write volume is high enough that daily parts remain large (>1M rows/day).
- **Never** partition by a high-cardinality column like `user_id` or a UUID.
- The rule of thumb: aim for **<1000 active partitions** per table per node.

## Partition Management Operations

```sql
-- List partitions
SELECT partition, sum(rows), formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE table = 'events' AND database = 'default' AND active
GROUP BY partition
ORDER BY partition;

-- Drop an old partition (instant, no merge needed)
ALTER TABLE events DROP PARTITION '202301';

-- Detach a partition (moves to detached/, recoverable)
ALTER TABLE events DETACH PARTITION '202301';

-- Attach a detached partition back
ALTER TABLE events ATTACH PARTITION '202301';

-- Move partition between tables (same structure, same engine)
ALTER TABLE events MOVE PARTITION '202301' TO TABLE events_archive;

-- Freeze a partition (backup snapshot)
ALTER TABLE events FREEZE PARTITION '202301';
```

`DROP PARTITION` is the key advantage of partitioning for time-series retention — instant deletion without a heavy mutation.

## Part vs Partition

- **Partition**: logical grouping of data (e.g., `202401`). A partition contains one or more **parts**.
- **Part**: a physical directory on disk containing column files, index, and metadata. Created on each insert, merged in the background.

Monitoring part health:
```sql
-- Check for "too many parts" situations
SELECT table, partition, count() AS part_count
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table, partition
HAVING part_count > 100
ORDER BY part_count DESC;
```

ClickHouse will warn at 300+ parts and error at 1000+ parts per partition (configurable via `parts_to_delay_insert`, `parts_to_throw_insert`).

## Combining Partitioning with TTL

Partitioning and TTL complement each other:
- **Partitioning**: for bulk drops of entire time ranges.
- **TTL**: for fine-grained row-level expiry or storage tiering within partitions.

For large-scale time-series tables, use monthly partitions for bulk drops (e.g., drop partitions older than 12 months) combined with TTL for tiered storage moves within recent partitions.
