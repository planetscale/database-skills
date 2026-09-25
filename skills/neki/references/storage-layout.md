---
title: Neki Storage Layout (per shard)
description: Postgres storage internals, TOAST, and fillfactor on each shard
tags: neki, storage, toast, fillfactor, disk, per-shard
---

# Storage Layout in Neki

Docs: https://planetscale.com/docs/neki/monitoring/metrics (Storage) · https://planetscale.com/docs/neki/cluster-configuration/cluster-sizing

Each Neki shard is real Postgres with its own storage. Per-shard storage internals are standard Postgres; a shard's disk capacity comes from its **cluster size** (set on its configuration profile), and you monitor it on the Metrics **Storage** tab.

## Per-shard Postgres storage

- **Data directory**: `base/` (per-database files), `global/` (shared catalogs), `pg_wal/` (WAL), `pg_xact/` (commit status). Each table and index is split into 1 GB segment files, with a free space map and a visibility map alongside.
- **TOAST**: rows over about 2 kB have large values compressed and/or moved out of line to a `pg_toast` table. `SELECT *` fetches every TOASTed column — select only what you need, and move large, rarely read columns to separate tables. TOAST tables also need VACUUM.
- **Fillfactor**: a lower value (70–80) leaves room on each page for HOT updates on update-heavy tables, reducing index churn and bloat; keep 100 for insert-only or read-mostly tables.

Tablespaces are unavailable in Neki during Platform Preview, and so are column storage and compression overrides (`ALTER TABLE ... SET STORAGE` / `SET COMPRESSION`).

## Storage sizing and monitoring

- Cluster size sets each instance's CPU, memory, and **storage**. Metal (local NVMe) offers higher IOPS and lower latency than network-attached storage (see [scaling-and-capacity.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/scaling-and-capacity.md)).
- The Metrics Storage tab shows primary and replica disk usage and bytes, WAL storage, and WAL archiving health.
- A shard reporting low disk is treated as read-only by the router until it is resized — watch disk per shard.
- `pg_total_relation_size('t')`, `pg_relation_size('t')`, and `pg_database_size('db')` report one shard's sizes; target a specific shard with `SET __neki.shard` or the dashboard web console.

## Sharding note

Sharding is the main lever for total data size: spreading a logical table across shards keeps any one instance's storage bounded (see [sharding-model.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/sharding-model.md)). Within a shard, partitioning and TOAST/fillfactor tuning manage large tables (see [partitioning.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/partitioning.md)).
