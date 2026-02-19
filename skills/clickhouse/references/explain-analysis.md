---
title: EXPLAIN Plan Analysis
description: Reading ClickHouse EXPLAIN, EXPLAIN PIPELINE, and EXPLAIN PLAN output to identify full scans and optimize queries
tags: clickhouse, explain, query-plan, performance, pipeline
---

# EXPLAIN Analysis

ClickHouse provides several `EXPLAIN` variants for inspecting how a query is planned and executed.

## EXPLAIN Variants

```sql
-- Default: syntax tree (not very useful for optimization)
EXPLAIN SELECT count() FROM events WHERE user_id = 1;

-- Query plan with index and partition information
EXPLAIN indexes = 1, actions = 1
SELECT count() FROM events WHERE user_id = 1;

-- Physical execution pipeline (processors and their connections)
EXPLAIN PIPELINE
SELECT count() FROM events WHERE user_id = 1;

-- Logical plan steps
EXPLAIN PLAN
SELECT count() FROM events WHERE user_id = 1;
```

## EXPLAIN with indexes = 1

The most useful form for diagnosing index usage:

```sql
EXPLAIN indexes = 1
SELECT count() FROM events WHERE user_id = 12345 AND event_date >= '2024-01-01';
```

Sample output:
```
Expression ((Projection + Before ORDER BY))
  Aggregating
    Expression (Before GROUP BY)
      Filter (WHERE)
        ReadFromMergeTree (default.events)
        Indexes:
          PrimaryKey
            Keys: user_id, event_date
            Condition: and((user_id in [12345, 12345]), (event_date in [2024-01-01, +Inf)))
            Parts: 3/12         ← 3 of 12 parts need to be read
            Granules: 5/980     ← 5 of 980 granules need to be read
```

Key fields:
- **Parts X/Y**: how many parts will be read out of total. Lower = better partition pruning.
- **Granules X/Y**: how many granules will be read out of total. Lower = better primary index usage.
- **Condition**: the predicate ClickHouse applies to the index.

A high Granules ratio (e.g., 900/980) means the primary index is barely helping — consider restructuring `ORDER BY` or adding a skipping index.

## EXPLAIN PIPELINE

Shows the physical execution pipeline of processors:

```sql
EXPLAIN PIPELINE
SELECT user_id, count() FROM events GROUP BY user_id;
```

Sample output:
```
(Expression)
ExpressionTransform
  (Aggregating)
  Resize 1 → 1
    AggregatingTransform × 8      ← parallel aggregation on 8 threads
      (Expression)
      ExpressionTransform × 8
        (ReadFromMergeTree)
        MergeTreeThread × 8       ← reading on 8 threads
```

What to look for:
- **MergeTreeThread × N**: N parallel reader threads. Low N means little parallelism (few parts).
- **Resize**: thread count changes (funnel or fanout).
- **RemoteSource**: data coming from remote shards in a distributed query.
- Unexpectedly sequential steps (×1 where you expect ×N) can indicate bottlenecks.

## EXPLAIN PLAN

Shows logical plan steps without physical pipeline details:

```sql
EXPLAIN PLAN actions = 1
SELECT user_id, count() FROM events WHERE status = 'active' GROUP BY user_id;
```

Useful for seeing:
- Filter pushdown (is `WHERE` applied before or after aggregation?)
- Expression evaluation steps
- JOIN implementation strategies

## Common Red Flags

| Symptom | Likely Cause | Action |
|---|---|---|
| `Granules: X/X` (all granules) | Primary index not used | Restructure `ORDER BY` or add skipping index |
| `Parts: X/X` (all parts) | No partition pruning | Add partition key filter to query |
| Very high `Parts:` count | Over-partitioning or too many small inserts | Reduce partition granularity, batch inserts |
| `Resize 32 → 1` | Funnel to single thread for aggregation | May be expected; check for slow aggregation |
| `RemoteSource` with high latency | Slow shard or network | Check shard health, consider replication |

## system.query_log for Post-Execution Analysis

`EXPLAIN` shows the plan before execution. For actual runtime stats, query `system.query_log`:

```sql
SELECT
    query,
    read_rows,
    read_bytes,
    result_rows,
    memory_usage,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date = today()
  AND query LIKE '%events%'
ORDER BY query_duration_ms DESC
LIMIT 10;
```

Key metrics:
- `read_rows`: total rows read from storage (before filtering). High value = poor index usage.
- `read_bytes`: total bytes read. High value = reading too many columns or too many granules.
- `memory_usage`: peak memory for the query. High value may indicate expensive aggregation or large joins.
- `query_duration_ms`: wall-clock time.

## system.query_thread_log

For per-thread breakdown of parallel query execution:

```sql
SELECT
    thread_name,
    read_rows,
    read_bytes,
    query_duration_ms
FROM system.query_thread_log
WHERE query_id = '<your-query-id>'
ORDER BY read_rows DESC;
```

Useful for identifying data skew in parallel reads.
