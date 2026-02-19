---
title: Materialized Views
description: ClickHouse incremental materialized view pattern, AggregatingMergeTree with -State/-Merge combiners, and common MV gotchas
tags: clickhouse, materialized-views, aggregatingmergetree, incremental, pre-aggregation
---

# Materialized Views

ClickHouse materialized views (MVs) are **incremental triggers**, not cached query results. They fire on every INSERT into the source table and write transformed/aggregated rows to a target table.

## How ClickHouse MVs Work

```
INSERT → source_table → MV trigger runs SELECT on new block → writes to target_table
```

- MVs process only **newly inserted data**, not existing data (they are not refreshed like in other databases).
- They do not re-read the source table on query — the target table is queried directly.
- Multiple MVs can write to the same target table.

## Basic MV Pattern

```sql
-- Source table
CREATE TABLE page_views (
    event_time  DateTime,
    page        LowCardinality(String),
    user_id     UInt64
)
ENGINE = MergeTree()
ORDER BY (page, event_time);

-- Target table (pre-aggregated)
CREATE TABLE page_views_hourly (
    hour        DateTime,
    page        LowCardinality(String),
    views       UInt64,
    unique_users UInt64
)
ENGINE = SummingMergeTree((views, unique_users))
ORDER BY (page, hour);

-- Materialized view (the trigger)
CREATE MATERIALIZED VIEW page_views_mv TO page_views_hourly AS
SELECT
    toStartOfHour(event_time) AS hour,
    page,
    count()                   AS views,
    uniqExact(user_id)        AS unique_users
FROM page_views
GROUP BY page, hour;
```

**Note:** `SummingMergeTree` works here but `unique_users` will be summed (not deduplicated) across merges. For accurate distinct counts, use `AggregatingMergeTree` with `-State` functions.

## AggregatingMergeTree + -State/-Merge Pattern

For accurate aggregations that survive merges, use `AggregateFunction` types with `-State` and `-Merge` combiners:

```sql
-- Target table with AggregateFunction columns
CREATE TABLE page_views_hourly_agg (
    hour         DateTime,
    page         LowCardinality(String),
    views        AggregateFunction(count, UInt64),
    unique_users AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (page, hour);

-- MV using -State combiner to write aggregate states
CREATE MATERIALIZED VIEW page_views_agg_mv TO page_views_hourly_agg AS
SELECT
    toStartOfHour(event_time)  AS hour,
    page,
    countState()               AS views,
    uniqState(user_id)         AS unique_users
FROM page_views
GROUP BY page, hour;

-- Query using -Merge combiner to read/combine states
SELECT
    page,
    hour,
    countMerge(views)       AS total_views,
    uniqMerge(unique_users) AS distinct_users
FROM page_views_hourly_agg
WHERE hour >= now() - INTERVAL 24 HOUR
GROUP BY page, hour
ORDER BY hour, page;
```

**Why this works:** Each insert block writes a partial aggregate state to the target table. `AggregatingMergeTree` merges these states during background merges. `-Merge` functions at query time combine any remaining unmerged states. The result is always correct regardless of merge status.

## Common -State/-Merge Combiner Pairs

| Aggregate | State function | Merge function |
|---|---|---|
| `count()` | `countState()` | `countMerge(col)` |
| `sum(x)` | `sumState(x)` | `sumMerge(col)` |
| `avg(x)` | `avgState(x)` | `avgMerge(col)` |
| `uniq(x)` | `uniqState(x)` | `uniqMerge(col)` |
| `uniqExact(x)` | `uniqExactState(x)` | `uniqExactMerge(col)` |
| `max(x)` | `maxState(x)` | `maxMerge(col)` |
| `min(x)` | `minState(x)` | `minMerge(col)` |
| `quantile(0.95)(x)` | `quantileState(0.95)(x)` | `quantileMerge(0.95)(col)` |

## Populating Historical Data

MVs do not backfill existing data. To populate the target table with historical data:

```sql
-- One-time backfill insert
INSERT INTO page_views_hourly_agg
SELECT
    toStartOfHour(event_time) AS hour,
    page,
    countState()              AS views,
    uniqState(user_id)        AS unique_users
FROM page_views
GROUP BY page, hour;
```

Run this before enabling the MV (or after, with care for deduplication) to ensure the target table has historical data.

## MV Gotchas

**1. MVs process each inserted block independently**

If you insert 1000 rows in one batch, the MV sees all 1000 rows together. If you insert 1 row at a time, the MV fires 1000 times (each with a single row). Batch inserts are critical for MV efficiency.

**2. Async insert deduplication works with dependent MVs (v26.1+)**

When using async inserts with `async_insert_deduplicate = 1`, deduplication is now propagated to dependent materialized views — duplicate blocks are filtered from MV writes as well as the source table.

**3. MV failures do not roll back the source INSERT**

If the MV query or the target table INSERT fails, the source data is still written. You may end up with inconsistent source and target tables. Monitor MV errors in `system.query_log`.

**4. Schema changes require MV rebuild**

Altering the source table schema (adding/removing columns) does not automatically update the MV. You may need to:
1. Drop and recreate the MV with the new SELECT.
2. Backfill the target table.

**5. Chaining MVs is fragile**

MV-on-MV (where the source of MV B is the target of MV A) can cause hard-to-debug ordering issues and inconsistent states. Prefer MVs directly on the original source table when possible.

**6. POPULATE keyword**

```sql
CREATE MATERIALIZED VIEW my_mv TO target AS SELECT ... FROM source;
-- WITH POPULATE: backfills existing data AND starts listening to new inserts
-- RACE CONDITION: inserts during POPULATE may be missed or double-counted
```

Avoid `WITH POPULATE` in production — do a manual backfill INSERT after creating the MV instead.
