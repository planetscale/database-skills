---
title: Query Optimization
description: ClickHouse query optimization — PREWHERE, avoiding SELECT *, filter pushdown, JOIN strategies, and subquery pitfalls
tags: clickhouse, query-optimization, prewhere, join, filter, performance
---

# Query Optimization

ClickHouse executes queries on a columnar engine with vectorized processing. Query efficiency depends heavily on how much data is read and how filters are applied.

## Read Only Required Columns (Avoid SELECT *)

ClickHouse reads columns independently from disk. `SELECT *` reads every column in every matched granule:

```sql
-- Bad: reads all columns including large payload column
SELECT * FROM events WHERE user_id = 12345;

-- Good: only reads the 3 columns needed
SELECT event_type, event_time, session_id FROM events WHERE user_id = 12345;
```

For tables with many columns or large string/blob columns, the difference in I/O can be 10–100x.

## PREWHERE vs WHERE

`PREWHERE` is a ClickHouse-specific optimization that filters rows **before** reading all selected columns. This reduces I/O when the filter eliminates most rows:

```sql
-- PREWHERE: reads only user_id column first, then reads other columns only for matching rows
SELECT event_type, payload FROM events
PREWHERE user_id = 12345;

-- WHERE: reads all columns for every row in each granule, then filters
SELECT event_type, payload FROM events
WHERE user_id = 12345;
```

**ClickHouse automatically moves eligible WHERE conditions to PREWHERE** in most cases. Explicit `PREWHERE` is rarely needed but can help when:
- You have a highly selective small-type column (e.g., `UInt64 user_id`) + a large column (e.g., `String payload`).
- Auto-promotion heuristics don't apply (e.g., complex expressions).

**PREWHERE limitations:**
- Cannot use `PREWHERE` with `FINAL`.
- Columns used in `PREWHERE` are read fully for matched granules even if not in `SELECT`.

## Filter Placement and Pushdown

Apply the most selective, cheapest filters first:

```sql
-- Good: filter on indexed column first (cheap), then filter on unindexed column (expensive)
SELECT * FROM events
WHERE user_id = 12345          -- hits primary index
  AND has(tags, 'premium');    -- array scan on remaining rows

-- Bad: expensive unindexed scan first, then indexed filter
-- (ClickHouse optimizer usually reorders, but be explicit for clarity)
```

For sub-queries and CTEs, verify that filters are pushed into the inner query and not applied after materialization:

```sql
-- Verify filter pushdown with EXPLAIN PLAN
EXPLAIN PLAN actions = 1
SELECT user_id, total
FROM (
    SELECT user_id, sum(amount) AS total
    FROM orders
    GROUP BY user_id
) subq
WHERE total > 1000;
```

## JOIN Strategies

ClickHouse supports multiple join algorithms. The right choice depends on table sizes:

### Hash Join (default for small right-side tables)

```sql
-- Implicitly uses hash join when right table fits in memory
SELECT l.user_id, r.name
FROM events l
JOIN users r ON l.user_id = r.user_id;
```

- Right table is fully loaded into a hash table in memory.
- Fast when the right table is small (<100M rows, fits in RAM).
- **Always put the smaller table on the RIGHT side of JOIN.**

### Distributed JOIN considerations

In a distributed setup, JOINs on non-sharding-key columns require broadcasting one side or shuffling data:

```sql
-- Use GLOBAL JOIN to broadcast the right-side table to all shards
SELECT l.user_id, r.segment
FROM distributed_events l
GLOBAL JOIN user_segments r ON l.user_id = r.user_id;
```

Without `GLOBAL`, each shard performs a JOIN only against its local data of the right table, which is usually wrong.

### Avoiding Large-to-Large JOINs

ClickHouse is not optimized for large-to-large joins (no hash join on two large tables efficiently):
- Pre-aggregate or pre-filter before joining.
- Consider denormalization: embed frequently-joined fields directly in the fact table.
- Use materialized views to pre-join and aggregate.

## Aggregation Best Practices

### Distinct Counts

Choose the right distinct-count function for your accuracy/speed tradeoff:

| Function | Algorithm | Error | Memory | Use when |
|---|---|---|---|---|
| `uniq()` | HyperLogLog | ~1–5% | Low | Default choice for dashboards |
| `uniqCombined()` | HLL + sampling | ~0.1–3% | Low-medium | Better accuracy than `uniq()`, still fast |
| `uniqExact()` | Hash set | 0% (exact) | High | Exact count required, smaller datasets |
| `COUNT(DISTINCT x)` | Hash set | 0% (exact) | High | Avoid on large tables — same as `uniqExact` but less obvious |

```sql
SELECT uniq(user_id)         AS approx_dau   FROM events WHERE event_date = today();
SELECT uniqCombined(user_id) AS better_approx FROM events WHERE event_date = today();
SELECT uniqExact(user_id)    AS exact_dau     FROM events WHERE event_date = today();
```

### Quantiles / Percentiles

| Function | Notes | Use when |
|---|---|---|
| `quantile(level)(x)` | Single quantile, reservoir sampling | Quick estimates |
| `quantiles(0.5, 0.9, 0.99)(x)` | Multiple quantiles in one pass | Always prefer over multiple `quantile()` calls |
| `quantileTDigest(level)(x)` | T-Digest sketch, better tail accuracy | p99/p999 latency percentiles |
| `quantileExact(level)(x)` | Exact, sorts all values | Small datasets only |

```sql
-- One pass for multiple percentiles (efficient)
SELECT quantiles(0.5, 0.90, 0.99)(response_ms) FROM requests;

-- T-Digest for high-accuracy tail percentiles
SELECT quantileTDigest(0.99)(response_ms) FROM requests;
```

Use `GROUP BY` with `WITH TOTALS` for subtotals without a second query:

```sql
SELECT event_type, count() FROM events GROUP BY event_type WITH TOTALS;
```

## Subquery Pitfalls

**IN with large subqueries:**

```sql
-- Potentially slow: subquery materialized as a set
SELECT * FROM events WHERE user_id IN (SELECT user_id FROM vip_users WHERE tier = 'gold');

-- Better for large sets: JOIN instead
SELECT e.*
FROM events e
JOIN vip_users v ON e.user_id = v.user_id
WHERE v.tier = 'gold';

-- Or use GLOBAL IN for distributed queries
SELECT * FROM distributed_events
WHERE user_id GLOBAL IN (SELECT user_id FROM vip_users WHERE tier = 'gold');
```

**Correlated subqueries** are not well optimized in ClickHouse. Rewrite as JOINs or pre-aggregate with CTEs.

## Pagination

Avoid `OFFSET` for deep pagination — it still scans all skipped rows:

```sql
-- Bad: scans 1,000,100 rows to return 100
SELECT * FROM events ORDER BY event_time LIMIT 100 OFFSET 1000000;

-- Good: cursor-based pagination using last seen value
SELECT * FROM events
WHERE event_time > '2024-01-15 12:34:56'   -- last value from previous page
ORDER BY event_time
LIMIT 100;
```

## Alias Shadowing in WHERE (Silent Bug)

ClickHouse expands column aliases in `WHERE` back to their `SELECT` expressions. If you reuse a column's name as an alias for a transformed version of it, the `WHERE` clause silently uses the transformed expression — not the original column — causing wrong results with no error:

```sql
-- Bug: WHERE becomes bytes2hex(address) = unhex('abc123') — hex vs binary, never matches
SELECT bytes2hex(address) AS address FROM t WHERE address = unhex('abc123');

-- Bug: WHERE becomes price * 100 > 50 — filters on scaled value, not original
SELECT price * 100 AS price FROM t WHERE price > 50;

-- Fix: use a distinct alias name so WHERE sees the original column
SELECT bytes2hex(address) AS address_hex FROM t WHERE address = unhex('abc123');
SELECT price * 100 AS price_scaled FROM t WHERE price > 50;

-- Alternative fix: qualify the column with the table alias
SELECT bytes2hex(t.address) AS address FROM my_table t WHERE t.address = unhex('abc123');
```

**Rule:** never give an alias the same name as the source column when that column also appears in `WHERE`.

## Column Alias Chaining in SELECT

Unlike MySQL and PostgreSQL, ClickHouse allows referencing aliases defined earlier in the same `SELECT` clause:

```sql
-- Valid in ClickHouse (would error in Postgres/MySQL):
SELECT
    balance * price          AS value_usd,
    value_usd / total_value  AS share,      -- references alias from same SELECT
    share * 100              AS share_pct   -- chains from the previous alias
FROM portfolio;
```

This removes the need for nested subqueries just to reuse computed values.

## Arrays and arrayJoin for IN Filters

For filtering with a fixed list of values, define the list as a `WITH` clause array and expand it with `arrayJoin()`. This avoids repeated literal expansion and can improve performance for large lists:

```sql
-- Define once, reference multiple times
WITH ['US', 'CA', 'GB'] AS target_countries
SELECT *
FROM events
WHERE country IN (SELECT arrayJoin(target_countries));

-- Collect a dynamic set from a subquery, then filter with it
WITH (SELECT groupArray(user_id) FROM vip_users WHERE tier = 'gold') AS vip_ids
SELECT *
FROM events
WHERE user_id IN (SELECT arrayJoin(vip_ids));
```

Note: wrap the array in `(SELECT arrayJoin(...))` rather than using it directly in `IN` — direct array use in `IN` has known edge cases in some ClickHouse versions.

## Sampling

For approximate analytics on very large tables, use built-in sampling:

```sql
-- Read 1% of data (sampled by hash of the sampling key)
SELECT count() * 100 AS approx_total FROM events SAMPLE 0.01;

-- Or read a fixed number of rows
SELECT count() FROM events SAMPLE 1000000;
```

Requires `SAMPLE BY` to be defined in the table's `CREATE TABLE`.
