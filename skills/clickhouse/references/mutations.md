---
title: Mutations and Data Modifications
description: ClickHouse ALTER TABLE UPDATE/DELETE mutations, lightweight deletes, OPTIMIZE TABLE, and monitoring merges
tags: clickhouse, mutations, delete, update, alter, optimize, merges
---

# Mutations and Data Modifications

ClickHouse is optimized for append-only workloads. Data modification operations (UPDATE, DELETE) are implemented as **mutations** — heavy background operations that rewrite affected parts.

## What Are Mutations?

A mutation is a background operation that:
1. Reads each affected part.
2. Applies the modification.
3. Writes a new version of the part.
4. Marks the old part for deletion after the new part is ready.

```sql
-- These are mutations (heavy, async):
ALTER TABLE events UPDATE status = 'archived' WHERE event_date < '2023-01-01';
ALTER TABLE events DELETE WHERE event_date < '2022-01-01';
```

Mutations are **asynchronous** — they return immediately and execute in the background. The data is not modified until the mutation completes.

## Monitoring Mutations

```sql
-- Check active and completed mutations
SELECT
    table,
    command,
    create_time,
    is_done,
    latest_failed_part,
    latest_fail_reason,
    parts_to_do,
    parts_to_do_names
FROM system.mutations
WHERE database = 'default'
ORDER BY create_time DESC;

-- Count of pending mutations per table
SELECT table, count() AS pending_mutations
FROM system.mutations
WHERE NOT is_done
GROUP BY table;
```

A mutation stuck with `is_done = 0` and non-empty `latest_fail_reason` requires investigation — often caused by schema mismatches or disk space issues.

## Mutation Costs and Limitations

- **Full part rewrite**: mutations rewrite every part that contains matching rows, even if only 1 row in a 10M-row part matches.
- **I/O intensive**: doubles the disk I/O temporarily (old part + new part coexist during the operation).
- **Concurrent mutations**: by default, ClickHouse limits concurrent mutations. Multiple ALTER UPDATE/DELETE statements queue up.
- **No transactions**: mutations are not atomic across partitions.

**Avoid mutations for:**
- Frequent per-row updates (e.g., updating `last_login` for every user on login).
- High-volume deletes of scattered individual rows.
- Time-sensitive operations where data must be modified within seconds.

**Use mutations for:**
- Bulk updates of large historical data ranges.
- GDPR/compliance deletion of a user's data (rare, infrequent).
- Schema-level data corrections.

## Lightweight Deletes (DELETE FROM)

```sql
-- Lightweight delete — marks rows as deleted without immediate part rewrite
DELETE FROM events WHERE user_id = 12345;
```

How it works:
- Sets a hidden `_row_exists` bitmask column — much cheaper than rewriting parts.
- Deleted rows are filtered at query time until the next merge removes them.
- Parts are eventually rewritten during background merges.

**Limitations:**
- Only available for MergeTree family engines.
- Does not work for tables with projections by default (enable via `lightweight_mutation_projection_mode` setting).
- Deleted rows still consume storage until the next merge — not suitable for immediate space reclamation.
- Performance degrades with complex `WHERE` conditions, full mutation queues, or tables with many small parts.

## OPTIMIZE TABLE

Forces immediate merges of parts:

```sql
-- Merge parts in a specific partition
OPTIMIZE TABLE events PARTITION '202401';

-- Merge all parts into a single part (very expensive!)
OPTIMIZE TABLE events FINAL;
```

**When to use:**
- After loading historical data via INSERT to compact many small parts into fewer large parts.
- After applying mutations to force immediate rewrite of affected parts.

**When NOT to use:**
- Routinely on large production tables — `OPTIMIZE FINAL` on a large table can run for hours and block merges.
- As a workaround for a high part count from small inserts — fix the ingestion pattern instead.

## Altering Table Schema

Schema changes via `ALTER TABLE` are generally instant metadata operations:

```sql
-- Add column (instant — no data rewrite)
ALTER TABLE events ADD COLUMN user_agent String DEFAULT '';

-- Rename column (instant)
ALTER TABLE events RENAME COLUMN user_agent TO ua;

-- Drop column (instant — column files removed lazily)
ALTER TABLE events DROP COLUMN ua;

-- Change column type (requires mutation — rewrites parts)
ALTER TABLE events MODIFY COLUMN user_id UInt64;
-- Avoid if the column is very large or the table is huge
```

Changing a column's type triggers a full mutation if the type is not trivially compatible. Test on a replica or non-production table first.

## Part and Merge Monitoring

```sql
-- Active merges
SELECT
    table,
    partition,
    elapsed,
    progress,
    num_parts,
    source_part_names,
    result_part_name,
    total_size_bytes_compressed,
    bytes_read_uncompressed,
    rows_read
FROM system.merges
ORDER BY elapsed DESC;

-- Part health overview
SELECT
    table,
    partition,
    count()                                      AS parts,
    sum(rows)                                    AS total_rows,
    formatReadableSize(sum(bytes_on_disk))        AS disk_size
FROM system.parts
WHERE database = 'default' AND active
GROUP BY table, partition
ORDER BY parts DESC
LIMIT 20;

-- Find tables with too many parts (warning threshold)
SELECT table, partition, count() AS parts
FROM system.parts
WHERE active
GROUP BY table, partition
HAVING parts > 50
ORDER BY parts DESC;
```

If parts per partition regularly exceeds 100–200, the ingestion rate is too high for the merge throughput — increase batch sizes or reduce insert frequency.
