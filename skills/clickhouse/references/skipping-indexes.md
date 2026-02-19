---
title: Skipping Indexes
description: ClickHouse data-skipping indexes (minmax, set, bloom_filter, ngrambf_v1) — when they help and their overhead costs
tags: clickhouse, skipping-indexes, bloom-filter, minmax, secondary-index
---

# Skipping Indexes

Data-skipping indexes (also called secondary indexes) help ClickHouse skip **granules** that cannot possibly match a query predicate. They are stored alongside the data and checked before reading granule data.

## How Skipping Indexes Work

A skipping index stores a small summary per block of granules (controlled by `GRANULARITY`). At query time, ClickHouse checks each block's summary:
- If the summary **rules out** matching rows → skip the block entirely.
- If the summary is **inconclusive** → read the block normally.

They complement the primary index but have overhead: each INSERT and background merge must update index summaries.

## Creating a Skipping Index

```sql
ALTER TABLE events
    ADD INDEX idx_user_id user_id TYPE bloom_filter(0.01) GRANULARITY 4;

-- Or inline in CREATE TABLE:
CREATE TABLE events (
    ...
    INDEX idx_user_id user_id TYPE bloom_filter(0.01) GRANULARITY 4,
    INDEX idx_type event_type TYPE set(100) GRANULARITY 2
)
ENGINE = MergeTree()
ORDER BY (date, session_id);
```

`GRANULARITY N` means the index covers N primary-index granules per entry. The default when omitted is **1**. Higher GRANULARITY = fewer index entries = less overhead, but coarser skipping.

## Index Types

### minmax

Stores min and max values for the column within each block:

```sql
INDEX idx_price price TYPE minmax GRANULARITY 4
```

- Effective for range predicates on numeric or date columns.
- Cheapest to build and store.
- Example use: `WHERE price BETWEEN 10 AND 50`, `WHERE created_at > '2024-01-01'`.
- **Not useful** for equality predicates on high-cardinality columns (min ≠ max in most blocks).

### set

Stores the set of distinct values within each block:

```sql
INDEX idx_status status TYPE set(100) GRANULARITY 4
-- 100 = max values stored per block; 0 = unlimited (use cautiously)
```

- Effective for equality predicates on low-to-medium cardinality columns.
- Example use: `WHERE status = 'active'`, `WHERE country IN ('US', 'CA')`.
- If a block has >100 distinct values, the index becomes inconclusive and provides no benefit for that block.
- Overhead grows with cardinality of values per block.

### bloom_filter

Probabilistic filter — answers "definitely not in block" or "maybe in block":

```sql
INDEX idx_user_id user_id TYPE bloom_filter(0.01) GRANULARITY 4
-- 0.01 = 1% false positive rate (lower = larger index, more storage)
```

- Effective for equality (`=`, `IN`) predicates on high-cardinality columns.
- No false negatives — if the filter says "not present", the block is safely skipped.
- Has false positives — some blocks are read unnecessarily (controlled by the false positive rate).
- Supports `String`, `FixedString`, numeric types, and arrays.
- Example use: `WHERE user_id = 12345`, `WHERE session_id IN (...)`.

### ngrambf_v1

Bloom filter over n-grams of a string — enables substring/LIKE search acceleration:

```sql
INDEX idx_url url TYPE ngrambf_v1(4, 1024, 2, 0) GRANULARITY 4
-- Parameters: ngram_size, bloom_filter_size (bytes), hash_functions, seed
```

- Effective for `LIKE '%substring%'` and `hasToken()` predicates.
- Only helps with predicates that can be expressed as n-gram containment.
- **Does not help** with prefix-only `LIKE 'prefix%'` (use `tokenbf_v1` or primary key for that).
- Larger `bloom_filter_size` = lower false positive rate but more storage.

### tokenbf_v1

Like `ngrambf_v1` but tokenizes on non-alphanumeric boundaries instead of fixed n-grams:

```sql
INDEX idx_log_message message TYPE tokenbf_v1(1024, 2, 0) GRANULARITY 4
```

- Better for natural-language text search with token-based predicates (`hasToken`, `IN`).

### text (Beta as of v25.11)

Builds an inverted index over tokenized string data, enabling deterministic full-text search:

```sql
INDEX idx_message message TYPE text GRANULARITY 1
```

- More accurate than `ngrambf_v1`/`tokenbf_v1` for full-text search (no false positives).
- Moved from experimental to **beta in v25.11**; on older versions enable with `allow_experimental_full_text_index = 1`.
- Supports `hasToken()`, `multiSearchAny()`, and similar predicates.
- Supports `Array` columns as of v26.1.

### vector_similarity (Experimental)

Enables approximate nearest-neighbor (ANN) search on `Array(Float32)` embedding columns:

```sql
INDEX idx_embedding embedding TYPE vector_similarity('hnsw', 'cosineDistance') GRANULARITY 1
```

- Currently experimental — enable with `allow_experimental_vector_similarity_index = 1`.
- Use for semantic search / embedding similarity workloads.

## When Skipping Indexes Help vs Hurt

**Skipping indexes help when:**
- The query predicate targets a column that is NOT in the leading `ORDER BY` key.
- Data has natural clustering in the column (e.g., events from the same user tend to be written together).
- The false-skip rate is high (e.g., a user's events span few granules).

**Skipping indexes hurt or provide no benefit when:**
- Data is uniformly random in the indexed column (every block contains all values — nothing to skip).
- The column has very high cardinality with no clustering.
- Write throughput is critical — each insert and merge updates the index.

**OR condition support:** Prior to recent versions, skip indexes only applied to AND predicates. As of 2025, `WHERE a = 1 OR b = 2`-style queries can use skip indexes when `use_skip_indexes_for_disjunctions = 1` is set.

**Tip:** After creating a skipping index, run `EXPLAIN indexes = 1` to verify it is being considered and check `system.query_log` for read row counts before/after.

## Projections

Projections are embedded sub-tables stored inside the same MergeTree table with their own `ORDER BY`. They give primary-index-level query performance for a second access pattern without maintaining a separate table.

```sql
-- Add a projection for queries that filter by user_id instead of the table's leading ORDER BY
ALTER TABLE events
    ADD PROJECTION proj_by_user (
        SELECT * ORDER BY user_id, event_date
    );

-- Materialize on existing data (runs in background)
ALTER TABLE events MATERIALIZE PROJECTION proj_by_user;
```

ClickHouse automatically chooses the projection at query time if it produces a smaller scan than the primary index. No query changes needed.

**When to use projections vs skipping indexes:**

| Scenario | Use |
|---|---|
| Different sort order needed (e.g., table is `ORDER BY date`, but you query by `user_id`) | Projection |
| Same sort order, just need to skip non-matching granules | Skipping index |
| Pre-aggregated rollup (different GROUP BY key) | Projection with aggregate SELECT |
| Column is clustered — data for each value is already local | Skipping index |

**Projection cost:** Every INSERT writes data to both the main table and each projection — roughly 2× write I/O per projection. Storage roughly doubles. Only add projections that are queried frequently enough to justify this overhead.

**Partial projection (subset of columns):**
```sql
-- Store only the columns needed by the alternative query pattern
ALTER TABLE events
    ADD PROJECTION proj_user_summary (
        SELECT user_id, event_date, event_type ORDER BY user_id, event_date
    );
```

Reduces the storage overhead vs a full `SELECT *` projection.
