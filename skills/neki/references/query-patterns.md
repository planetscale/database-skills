---
title: Neki SQL Query Patterns
description: SQL anti-patterns and optimized alternatives (sharding-aware)
tags: neki, sql, query-optimization, n-plus-one, pagination, scatter-gather
---

# SQL Query Patterns in Neki

Docs: https://planetscale.com/docs/neki/query-planning · https://planetscale.com/docs/neki/best-practices

Standard Postgres query hygiene applies per shard, plus the key sharded rule: **include the shard key so a query routes to one shard** instead of scattering across the group (see [query-serving.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/query-serving.md)).

## Route to a single shard

```sql
-- single-shard (shard key present)
SELECT id, status FROM orders WHERE tenant_id = $1 AND status = 'pending';
-- scatter (no shard key) — every shard in the group
SELECT id, status FROM orders WHERE status = 'pending';
```

Verify with `EXPLAIN (NEKI_PLAN)`; use `SET __neki.fanout = 'single'` in development and CI to fail on unintended scatter. For lookups by a non-shard-key column, use a GSI rather than scattering (see [indexing.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/indexing.md)).

## Query structure

**Select specific columns** (less data to transfer and combine):
```sql
SELECT id, name, email FROM users WHERE tenant_id = $1;   -- not SELECT *
```

**Correlated subqueries → JOINs** (correlated subqueries re-execute per row and some shapes are rejected cross-shard):
```sql
SELECT u.id, COUNT(o.id)
FROM users u LEFT JOIN orders o ON o.tenant_id = u.tenant_id AND o.user_id = u.id
WHERE u.tenant_id = $1 GROUP BY u.id;
```

**Always LIMIT unbounded queries.** Unbounded scatter is especially costly.

**Keep functions off indexed columns:**
```sql
-- bad
WHERE date_trunc('day', created_at) = '2026-01-01'
-- good
WHERE created_at >= '2026-01-01' AND created_at < '2026-01-02'
```

## N+1 → batch (and stay shard-local)

```python
# bad: one query per id
for uid in ids: cur.execute("SELECT name FROM users WHERE tenant_id=%s AND id=%s", (tid, uid))
# good: batch, with the shard key fixed so it stays single-shard
cur.execute("SELECT id, name FROM users WHERE tenant_id=%s AND id = ANY(%s)", (tid, ids))
```

## Rewrites

- **UNION → UNION ALL** when duplicates are impossible or acceptable. `UNION` is supported; `INTERSECT` and `EXCEPT` are rejected for planned application queries during Platform Preview.
- **IN subquery → EXISTS** to short-circuit.
- **OFFSET → keyset (cursor) pagination** — `OFFSET` scans and discards rows and is worse when merged across shards:

```sql
SELECT id, title FROM articles
WHERE tenant_id = $1 AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

## Ordering and aggregation

- Without `ORDER BY` there is **no** cross-shard order guarantee — add explicit ordering when it matters.
- `FETCH FIRST ... WITH TIES` is unavailable across shards.
- Scope aggregates to the shard key where possible; a global `COUNT(*)` or `SUM()` is a scatter — for global stats, maintain rollup tables.

## Platform Preview query-shape limits

The router rejects `SELECT ... INTO`, `INTERSECT`/`EXCEPT` for planned application queries, `COMMIT/ROLLBACK AND CHAIN`, cross-database references, reading an inheritance parent without `ONLY`, and a `search_path` containing `pg_temp`. Shapes Neki hasn't implemented yet return SQLSTATE `NK013` with a numbered catalog code — see [error-codes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/error-codes.md).
