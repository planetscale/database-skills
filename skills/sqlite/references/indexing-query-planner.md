---
title: SQLite Indexing and Query Planner
description: Index design and planner behavior for fast SQLite queries
tags: sqlite, indexes, query-planner, explain, performance
---

# SQLite Indexing and Query Planner

## Composite index ordering

Put equality predicates first, then range/sort predicates.

```sql
-- Query pattern:
-- WHERE tenant_id = ? AND status = ? AND created_at >= ? ORDER BY created_at DESC
CREATE INDEX idx_orders_tenant_status_created
  ON orders(tenant_id, status, created_at DESC);
```

## Covering indexes

If a query only needs indexed columns, SQLite can avoid table lookups.

```sql
-- SELECT id, created_at FROM orders WHERE tenant_id=? ORDER BY created_at DESC LIMIT 50
CREATE INDEX idx_orders_tenant_created_id
  ON orders(tenant_id, created_at DESC, id);
```

## Expression and partial indexes

Use them only when the predicate is stable and common.

```sql
CREATE INDEX idx_user_lower_email ON user(lower(email));
CREATE INDEX idx_task_open ON task(updated_at) WHERE status = 'open';
```

## Common planner pitfalls

- Functions on indexed columns usually prevent normal index use unless expression index matches exactly.
- `OR` conditions can split plans; rewrite to `UNION ALL` when selective branches exist.
- Deep `OFFSET` pagination gets slower; prefer keyset pagination.

## Keep stats fresh

Run after major data changes so the planner chooses better paths:

```sql
ANALYZE;
PRAGMA optimize;
```
