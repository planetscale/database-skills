---
title: Primary Key and Sparse Index
description: How ClickHouse's sparse primary index works, ORDER BY vs PRIMARY KEY, granule sizing, and selectivity tradeoffs
tags: clickhouse, primary-key, sparse-index, order-by, granule, index
---

# Primary Key and Sparse Index

ClickHouse's primary index is fundamentally different from row-oriented databases like MySQL. Understanding it is essential for designing efficient schemas.

## Sparse Index Basics

ClickHouse does **not** index every row. Instead, it indexes every **granule** — a block of consecutive rows (default: 8192 rows per granule):

```
Row 0       → index entry 0
Row 8192    → index entry 1
Row 16384   → index entry 2
...
```

The index stores the first `PRIMARY KEY` value of each granule. At query time, ClickHouse uses binary search on the index to find which granules **might** contain matching rows, then reads only those granules.

**Implication:** The primary index narrows down reads to granules, not individual rows. Highly selective point lookups are less efficient than in B-tree databases unless the key uniquely identifies a very small number of granules.

## ORDER BY vs PRIMARY KEY

These are separate clauses that serve different purposes:

- **`ORDER BY`**: defines the physical sort order of rows on disk and the data locality for compression. Also implicitly defines the primary index if `PRIMARY KEY` is not specified.
- **`PRIMARY KEY`**: defines which columns go into the sparse index. Must be a prefix of `ORDER BY`.

```sql
CREATE TABLE user_events (
    user_id   UInt64,
    event_date Date,
    event_type LowCardinality(String),
    ...
)
ENGINE = MergeTree()
ORDER BY (user_id, event_date, event_type)
PRIMARY KEY (user_id, event_date);
-- event_type is in ORDER BY (for sort/compression) but not in PRIMARY KEY (not indexed)
```

**When to split them:** When the last `ORDER BY` column(s) are needed for compression or physical locality but add overhead to the index without improving granule selectivity.

## Designing ORDER BY / PRIMARY KEY

Rules of thumb:

1. **Columns you filter on most** should lead the key.
2. **Low-cardinality before high-cardinality** for better compression (more repeated values per granule).
3. **Equality predicates before range predicates**: range stops the index from being useful for subsequent columns (same as SQL composite index leftmost prefix rule).
4. **Monotonically increasing time** as the last key column often gives good compression and natural append order.

```sql
-- Typical time-series event table
ORDER BY (tenant_id, event_type, toDate(event_time), user_id)
-- tenant_id: equality filter on every query (low cardinality in many systems)
-- event_type: next equality filter
-- toDate(event_time): date-level granularity for range queries
-- user_id: fine-grained filter or sort
```

## Granule Size and index_granularity

Default `index_granularity = 8192` rows per granule. This controls the tradeoff between index size and read granularity:

- **Smaller granularity** (e.g., 1024): more index entries, smaller reads per granule, but larger index memory footprint and slower merges.
- **Larger granularity** (e.g., 65536): fewer index entries, coarser reads, suitable for very wide tables or analytical queries that always scan large ranges.

```sql
-- Custom granularity via SETTINGS
ENGINE = MergeTree()
ORDER BY (user_id, event_date)
SETTINGS index_granularity = 1024;
```

`index_granularity_bytes` (default: 10MB) also caps granule size by data volume — whichever limit is hit first determines the granule boundary.

## Selectivity Tradeoffs

The primary index is efficient when:
- The query predicates match the **leading columns** of the primary key.
- The filtered range maps to a **small number of granules** relative to total parts.

The primary index provides **no benefit** when:
- Querying on columns not in the primary key prefix.
- The table has only a few rows per granule that match (skipping indexes may help here instead).

**Example — Good selectivity:**
```sql
-- ORDER BY (user_id, event_date)
-- Query targets specific user + date range → few granules read
SELECT * FROM events WHERE user_id = 12345 AND event_date >= '2024-01-01';
```

**Example — Poor selectivity:**
```sql
-- Same ORDER BY
-- Querying only on event_date with no user_id filter → must scan many granules
SELECT * FROM events WHERE event_date = '2024-01-15';
-- Consider adding a skipping index on event_date, or restructuring ORDER BY
```

## Checking Index Usage

Use `EXPLAIN` to see how many granules are scanned:

```sql
EXPLAIN indexes = 1
SELECT count() FROM events WHERE user_id = 12345;
-- Look for: "Granules: X/Y" — X granules read out of Y total
```

A low X/Y ratio indicates the primary index is filtering effectively.
