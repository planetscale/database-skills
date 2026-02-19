---
title: Data Ingestion
description: ClickHouse insert best practices — batch sizing, async inserts, avoiding small inserts, and deduplication
tags: clickhouse, ingestion, inserts, async-inserts, batch, parts, deduplication
---

# Data Ingestion

ClickHouse's write model is fundamentally different from row-oriented databases. Understanding how inserts create parts is critical to avoiding common performance and reliability problems.

## How Inserts Create Parts

Each `INSERT` statement creates at least one new **part** per partition touched:

```
INSERT 1000 rows (one partition) → 1 new part
INSERT 1 row (one partition)    → 1 new part
INSERT 1 row × 1000 times       → 1000 new parts
```

Parts are merged in the background by ClickHouse's merge scheduler. Too many small parts:
- Triggers `Too many parts` errors (default threshold: 300 parts → delay, 1000 → error).
- Slows down queries (must read many small files).
- Increases memory usage (each part has metadata overhead).

**Core rule: always batch inserts. Aim for ≥1000 rows per INSERT; 10,000–100,000 is better.**

## Batching Strategies

### Application-Level Batching

Accumulate rows in memory and flush as a batch:

```python
batch = []
for event in event_stream:
    batch.append(event)
    if len(batch) >= 10000:
        client.insert('events', batch)
        batch = []
# flush remaining
if batch:
    client.insert('events', batch)
```

- Simple and effective for moderate write rates.
- Requires handling failures (retry, at-least-once semantics).

### Buffer Table Engine

Use a `Buffer` table to absorb small writes and flush to the real table in batches:

```sql
CREATE TABLE events_buffer AS events
ENGINE = Buffer(
    'default',   -- target database
    'events',    -- target table
    4,           -- num_layers (parallel buffers)
    10,          -- min_time (seconds before flush)
    100,         -- max_time (seconds before forced flush)
    10000,       -- min_rows
    1000000,     -- max_rows
    10485760,    -- min_bytes (10MB)
    104857600    -- max_bytes (100MB)
);

-- Application inserts into the buffer table
INSERT INTO events_buffer VALUES (...);
```

- Flushes to `events` when any threshold (time, rows, bytes) is exceeded.
- Data in buffers is lost on server restart — not suitable for critical data without a queue upstream.

### Message Queue Integration (Kafka)

For high-throughput streaming ingestion, use the Kafka table engine:

```sql
-- Kafka source table (reads from topic)
CREATE TABLE events_kafka (
    event_time  DateTime,
    user_id     UInt64,
    event_type  String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'broker:9092',
    kafka_topic_list = 'events',
    kafka_group_name = 'clickhouse_consumer',
    kafka_format = 'JSONEachRow';

-- Target MergeTree table
CREATE TABLE events (...) ENGINE = MergeTree() ORDER BY (...);

-- MV to move data from Kafka to MergeTree
CREATE MATERIALIZED VIEW events_mv TO events AS
SELECT * FROM events_kafka;
```

- ClickHouse reads from Kafka in configurable batches.
- Provides at-least-once delivery; implement deduplication if needed.

## Async Inserts

Available in ClickHouse 21.11+. The server buffers small inserts and flushes them as a batch:

```sql
-- Enable for a session
SET async_insert = 1;
SET wait_for_async_insert = 1;  -- wait for flush (synchronous from client perspective)

-- Or per-query
INSERT INTO events SETTINGS async_insert = 1, wait_for_async_insert = 0 VALUES (...);
```

With async inserts:
- Small individual INSERTs are buffered server-side.
- The server flushes when `async_insert_max_data_size` or `async_insert_busy_timeout_ms` is reached.
- Reduces part count from many small inserts without client-side buffering.

**Tradeoffs:**
- `wait_for_async_insert = 0`: fire-and-forget — low latency but no confirmation of persistence.
- `wait_for_async_insert = 1`: waits for the insert to flush — safer but higher per-insert latency.
- Data is in memory until flushed — risk of loss on server crash (mitigated by async_insert_deduplicate).

## Insert Deduplication

ClickHouse MergeTree tables have built-in insert deduplication based on the content hash of each inserted block:

```sql
-- Deduplication is on by default for ReplicatedMergeTree
-- For non-replicated MergeTree, controlled by:
INSERT INTO events SETTINGS insert_deduplication_token = 'my-idempotency-key' VALUES (...);
```

- If the same block is inserted twice within the dedup window (controlled by `replicated_deduplication_window`), the second insert is silently ignored.
- Dedup is based on block content hash (or explicit token) — does not deduplicate individual rows.
- Use `insert_deduplication_token` for explicit idempotency keys (useful for retry logic).

## Format Selection for Bulk Loads

For loading large data files, prefer binary or compact formats:

| Format | Notes |
|---|---|
| `Native` | ClickHouse native binary — fastest, most compact |
| `Parquet` | Columnar, widely supported, good for S3/file loads |
| `Arrow` | Fast columnar format, good for in-process transfers |
| `JSONEachRow` | One JSON object per line — easy but 3–5x larger than binary |
| `CSV` | Universal but slow for large loads |

```sql
-- Load from file (clickhouse-client)
clickhouse-client --query="INSERT INTO events FORMAT Parquet" < data.parquet

-- Load from S3
INSERT INTO events SELECT * FROM s3('s3://bucket/data/*.parquet', 'Parquet');
```

## Monitoring Insert Health

```sql
-- Current parts per partition (watch for high counts)
SELECT partition, count() AS parts
FROM system.parts
WHERE table = 'events' AND active
GROUP BY partition
ORDER BY parts DESC;

-- Active merges
SELECT table, partition, elapsed, progress, rows_read, rows_written
FROM system.merges
ORDER BY elapsed DESC;

-- Recent insert errors
SELECT event_time, exception, query
FROM system.query_log
WHERE type = 'ExceptionWhileProcessing'
  AND query LIKE 'INSERT%'
ORDER BY event_time DESC
LIMIT 20;
```
