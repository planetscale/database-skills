---
title: Neki MVCC and VACUUM (per shard)
description: Postgres MVCC, VACUUM/autovacuum tuning, and bloat prevention on each shard
tags: neki, mvcc, vacuum, autovacuum, xid, bloat
---

# MVCC and VACUUM in Neki

Docs: https://planetscale.com/docs/neki/cluster-configuration/parameters (Autovacuum) · https://planetscale.com/docs/neki/monitoring

Each Neki shard is real Postgres, so MVCC and VACUUM behave exactly as on a single Postgres instance — **per shard**. Autovacuum runs independently on every shard's primary. Tune it through the shard's **configuration profile** (Postgres tab), which applies to every shard on that profile.

## MVCC

Every `UPDATE` creates a new tuple version and marks the old one dead; `DELETE` marks tuples dead. Dead tuples accumulate until VACUUM reclaims them. Each transaction gets a 32-bit XID; VACUUM must freeze old XIDs to prevent wraparound.

## VACUUM vs VACUUM FULL

`VACUUM` is non-blocking (`SHARE UPDATE EXCLUSIVE`) and marks dead space reusable. `VACUUM FULL` rewrites the table under `ACCESS EXCLUSIVE` — a last resort. `VACUUM`, `ANALYZE`, and `VACUUM FULL` sent through a router run on every shard, including when `__neki.shard` is set. A shard pin limits data statements; it does not limit these maintenance commands. `VACUUM FULL` takes its exclusive lock on every shard. `pg_repack` is not in Neki's extension catalog. Prevent bloat with autovacuum tuning and short transactions; if a table is badly bloated, schedule `VACUUM FULL` for a quiet window or contact PlanetScale Support.

## Autovacuum tuning (configuration-profile parameters)

The profile Postgres tab exposes `autovacuum`, `autovacuum_vacuum_scale_factor` (default 0.2; lower to 0.01–0.05 for large tables), `autovacuum_vacuum_cost_delay`, `autovacuum_vacuum_cost_limit`, `autovacuum_max_workers`, `autovacuum_naptime`, `autovacuum_analyze_scale_factor`, and the insert-vacuum thresholds. Changes apply to **every shard on the profile** — put a hot shard on its own profile if it needs different settings. Prefer per-table overrides for individual hot tables:

```sql
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.02);
```

## Monitoring (per shard)

```sql
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;
SELECT datname, age(datfrozenxid) AS xid_age FROM pg_database ORDER BY xid_age DESC;
SELECT pid, state, now() - xact_start AS tx_age FROM pg_stat_activity
 WHERE xact_start IS NOT NULL ORDER BY xact_start;
```

Run these on the shard you're investigating (`SET __neki.shard = '<shard_uid>'`, or the dashboard web console). Query Insights and Metrics also surface slow queries and per-shard resource use.

## Best practices

- Keep transactions short. Routers enforce `idle-in-transaction-session-timeout` (default 5 minutes), but fix long transactions at the source — one long transaction blocks dead-tuple cleanup on its shard.
- Alert when `age(datfrozenxid)` approaches wraparound; **never disable autovacuum**.
- Tune autovacuum per hot table first; only then change profile-level defaults.
