---
title: Schema Design
description: ClickHouse column types, ordering, compression, and schema best practices
tags: clickhouse, schema, data-types, compression, nullable, lowcardinality
---

# Schema Design

ClickHouse is a columnar database — schema decisions around column types, order, and codecs directly affect compression ratios, query speed, and write throughput.

## Column Ordering for Compression

ClickHouse compresses each column independently. Within each column, data is sorted by the table's `ORDER BY` key. To maximize compression:

- Put **low-cardinality columns first** in `ORDER BY` (e.g., `country`, `event_type`), then higher-cardinality columns (e.g., `user_id`, `timestamp`).
- Low-cardinality prefix columns create long runs of identical values, which compress extremely well (often 10–100x).

```sql
-- Good: low-cardinality columns lead, high-cardinality trail
ORDER BY (country, event_type, user_id, event_timestamp)

-- Bad: high-cardinality leads, destroys compression for following columns
ORDER BY (user_id, event_timestamp, country, event_type)
```

## LowCardinality Type

`LowCardinality(T)` uses dictionary encoding — stores an integer index + a dictionary. Dramatically reduces memory and storage for string columns with <10,000 distinct values:

```sql
-- Good for: status, country, event_type, device_type, etc.
event_type  LowCardinality(String),
country     LowCardinality(String),

-- Avoid for: UUIDs, URLs, freeform text -- cardinality too high
url         String,     -- not LowCardinality
```

- Works with `String`, `FixedString`, numeric types, `Date`, `DateTime`.
- Queries on `LowCardinality` columns can be 2–10x faster than plain `String` due to dictionary operations.

## Nullable Columns — Use Sparingly

`Nullable(T)` stores an extra bitmask column alongside the data column:

- Roughly doubles the storage for that column in the worst case.
- Many ClickHouse functions do not propagate `NULL` the same way as SQL — can produce surprising results.
- Prevents certain query optimizations.

**Guideline:** Use a sentinel value instead when practical (`0`, `''`, `'unknown'`). Only use `Nullable` when `NULL` is semantically meaningful and callers will check for it.

```sql
-- Prefer sentinel
user_agent String DEFAULT '',

-- Use Nullable only when absence is meaningful
deleted_at Nullable(DateTime),
```

## Numeric Types — Right-Size Your Columns

| Type | Bytes | Range |
|---|---|---|
| `UInt8` | 1 | 0–255 |
| `UInt16` | 2 | 0–65 535 |
| `UInt32` | 4 | 0–4.3B |
| `UInt64` | 8 | 0–18.4Q |
| `Int32` | 4 | ±2.1B |
| `Int64` | 8 | ±9.2Q |
| `Float32` | 4 | ~7 sig digits |
| `Float64` | 8 | ~15 sig digits |

Use the smallest type that covers your range. Smaller types = better compression and faster SIMD processing.

## Date and DateTime Types

- `Date`: 2 bytes, day resolution.
- `Date32`: 4 bytes, broader range (for dates before 1970 or after 2149).
- `DateTime`: 4 bytes, second resolution, timezone-aware.
- `DateTime64(precision, timezone)`: 8 bytes, sub-second precision.

```sql
-- For event timestamps with millisecond precision
event_time DateTime64(3, 'UTC'),

-- For partition key (day/month granularity only)
event_date Date,
```

## Array and Nested Types

`Array(T)` stores variable-length arrays per row:

```sql
tags Array(LowCardinality(String)),
scores Array(Float32),
```

- Arrays are stored as two columns internally: offsets + values.
- Can be queried with array functions: `has(tags, 'clickhouse')`, `arrayJoin`, `length`.

`Nested` is syntactic sugar for parallel arrays — each field becomes `Array(T)` internally:

```sql
-- Nested definition
metrics Nested (
    name  LowCardinality(String),
    value Float64
)
-- Accessed as: metrics.name, metrics.value (both Array types)
```

**Prefer explicit `Array` columns** over `Nested` for clarity.

## Codec Selection

ClickHouse supports per-column compression codecs:

```sql
timestamp   DateTime  CODEC(Delta, ZSTD(1)),
counter     UInt64    CODEC(T64, LZ4),
payload     String    CODEC(ZSTD(3)),
```

| Codec | Best For |
|---|---|
| `Delta` | Monotonically increasing integers or timestamps |
| `DoubleDelta` | Slowly changing numeric sequences |
| `T64` | Integers with narrow range (uses min/max to strip high bits) |
| `Gorilla` | Floating point time series |
| `LZ4` | Fast compression/decompression, moderate ratio |
| `ZSTD(level)` | Higher ratio at cost of CPU; level 1–22 |

**Default codec** is `LZ4` unless overridden. For timestamp columns, `Delta + ZSTD` is usually a strong choice.

## Fixed vs Variable Length Strings

- `FixedString(N)`: fixed N bytes, padded with null bytes. Only use when all values are exactly N bytes (e.g., hash digests, fixed-width codes).
- `String`: variable length, no overhead for short strings, better for most text data.
