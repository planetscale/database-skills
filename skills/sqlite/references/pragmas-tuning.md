---
title: SQLite PRAGMAs and Tuning
description: Safe performance tuning using SQLite runtime settings
tags: sqlite, pragma, tuning, cache, memory
---

# SQLite PRAGMAs and Tuning

## Measure first

PRAGMAs change behavior globally for a connection or database file. Benchmark before and after under representative load.

## High-impact settings

```sql
PRAGMA cache_size = -32768;    -- about 32 MB cache (negative = KB)
PRAGMA temp_store = MEMORY;    -- keep temp objects in memory when feasible
PRAGMA mmap_size = 268435456;  -- 256 MB mapped I/O if platform supports it
```

- Larger cache helps read-heavy workloads, but increases memory use.
- `temp_store=MEMORY` can improve sorts/joins but may increase RSS.
- `mmap_size` can reduce syscall overhead on capable filesystems.

## Safety settings

```sql
PRAGMA foreign_keys = ON;
PRAGMA trusted_schema = OFF;
```

- Always enable foreign keys unless there is a specific documented reason not to.
- `trusted_schema=OFF` reduces risk from malicious schema-level SQL in untrusted files.

## Planner maintenance settings

```sql
PRAGMA analysis_limit = 2000;
PRAGMA optimize;
```

- `PRAGMA optimize` may run targeted `ANALYZE` work when statistics are stale or missing.
- Use explicit `ANALYZE` after large data shifts; use `PRAGMA optimize` as periodic maintenance.

## Avoid blanket advice

- `synchronous=OFF` is risky for most production apps.
- Revisit tuning after schema/index changes; bottlenecks move.
