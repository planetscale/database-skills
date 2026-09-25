---
title: Neki Index Optimization
description: Per-shard index audit queries and write-amplification control
tags: neki, indexes, unused-indexes, duplicate-indexes, bloat, hot
---

# Index Optimization in Neki

Docs: https://planetscale.com/docs/neki/monitoring/schema-recommendations

Each shard is Postgres, so index audits run **per shard**. Under sharding, index bloat and write amplification are multiplied across shards, so keep index counts lean. PlanetScale **schema recommendations** and Query Insights surface index opportunities from production telemetry — use them alongside the queries below.

## Unused indexes (run per shard)

```sql
SELECT s.schemaname, s.relname AS table_name, s.indexrelname AS index_name,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size
FROM pg_catalog.pg_stat_user_indexes s
JOIN pg_catalog.pg_index i ON s.indexrelid = i.indexrelid
WHERE s.idx_scan = 0
  AND 0 <> ALL (i.indkey)
  AND NOT i.indisunique
  AND NOT EXISTS (SELECT 1 FROM pg_catalog.pg_constraint c WHERE c.conindid = s.indexrelid)
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

Check statistics age first. An index unused on one shard may be used on another, so audit **all** shards before concluding. Target a shard with `SET __neki.shard = '<shard_uid>'` (or use the dashboard web console).

## Duplicate and invalid indexes

```sql
-- duplicates
SELECT schemaname || '.' || tablename AS table, array_agg(indexname) AS dupes
FROM pg_indexes WHERE schemaname NOT IN ('pg_catalog','information_schema')
GROUP BY schemaname, tablename, regexp_replace(indexdef, 'INDEX \S+ ON ', 'INDEX ON ')
HAVING count(*) > 1;

-- invalid (failed index builds)
SELECT indexrelname FROM pg_stat_user_indexes s
JOIN pg_index i ON s.indexrelid = i.indexrelid WHERE NOT i.indisvalid;
```

> **Warning:** Confirm with a human before dropping any index; "duplicates" may differ in operator class, collation, predicate, or sort order.

## Per-table index count

| Count | Recommendation |
| --- | --- |
| <5 | Normal |
| 5–10 | Review for unused and duplicate indexes |
| >10 | Audit — write overhead is multiplied per shard |

## Bloat and HOT updates

- VACUUM doesn't compact empty index pages — rebuild a bloated index with `REINDEX INDEX CONCURRENTLY` (maintenance statements sent through a router run on every shard). `pg_repack` and `pgstattuple` are not in Neki's extension catalog, so compare index size to table size and row counts per shard to spot bloat.
- Target a HOT-update ratio above 90% on write-heavy tables: set `fillfactor = 70–80`, and don't index frequently updated columns (`status`, `updated_at`) unless queries need them.

```sql
SELECT relname, round(100.0 * n_tup_hot_upd / nullif(n_tup_upd,0), 1) AS hot_pct
FROM pg_stat_user_tables WHERE n_tup_upd > 0 ORDER BY n_tup_upd DESC;
```

## Write amplification

Every index adds write cost on each shard, and reference tables and GSIs add cross-shard write work. Keep only indexes that earn their keep, lead them with the shard key, and prefer selective GSIs over broad ones (see [indexing.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/indexing.md)).
