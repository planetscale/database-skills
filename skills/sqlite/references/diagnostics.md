---
title: SQLite Diagnostics
description: Query plan inspection and troubleshooting workflow
tags: sqlite, explain, diagnostics, analyze, performance
---

# SQLite Diagnostics

## Plan inspection

```sql
EXPLAIN QUERY PLAN
SELECT id, status FROM orders
WHERE tenant_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

Look for:

- `SCAN table` -> full scan (often a missing/unused index)
- `SEARCH table USING INDEX ...` -> index usage
- `USING COVERING INDEX ...` -> no table lookup needed
- `USE TEMP B-TREE` -> sort/group spill; check index support

## Repeatable tuning loop

1. Capture baseline latency and row counts.
2. Inspect `EXPLAIN QUERY PLAN` output.
3. Add or adjust indexes/query shape.
4. Run `ANALYZE` and compare metrics again.
5. Keep only changes with measurable gains.

## Helpful commands

```sql
SELECT sqlite_version();
ANALYZE;
PRAGMA optimize;
PRAGMA compile_options;
```

Version and compile options explain feature availability and behavior differences across environments.
