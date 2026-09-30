---
title: Neki WAL and Checkpoint Operations (per shard)
description: Postgres WAL, checkpoints, and archiving on each Neki shard
tags: neki, wal, checkpoints, archiving, durability, crash-recovery
---

# WAL and Checkpoints in Neki

Docs: https://planetscale.com/docs/neki/cluster-configuration/parameters (Write-ahead log) · https://planetscale.com/docs/neki/monitoring/metrics (Storage)

Each shard is real Postgres, so WAL and checkpoint behavior is standard Postgres — **per shard**. Each shard's WAL also feeds its physical replicas (see [replication.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/replication.md)), the Replicator's change streaming (see [resharding-migration.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/resharding-migration.md)), and per-shard CDC. Backups plus archived WAL enable point-in-time recovery (see [backup-recovery.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/backup-recovery.md)).

## Fundamentals

Postgres writes changes to WAL before modifying data files; on `COMMIT` it flushes WAL and returns success (data files are updated lazily). A checkpoint flushes dirty pages and writes a checkpoint record; crash recovery replays WAL from the last checkpoint.

## Configuration-profile parameters

Tune on the profile Postgres tab (applies to every shard on the profile): `max_wal_size` (checkpoint trigger; raise it if logs report checkpoints occurring too frequently), `checkpoint_timeout`, `min_wal_size`, `wal_buffers` (restart), `wal_compression` (compresses full-page images; helps replication bandwidth), `wal_level` (restart), and `archive_timeout`. Replication-related: `max_wal_senders`, `max_replication_slots`, `max_slot_wal_keep_size`.

## Archiving and disk (Metrics Storage tab)

WAL archiving underpins PITR; archive failures block WAL recycling and can fill the disk. The Metrics **Storage** tab shows WAL storage, WAL archive success and failure rates, WAL archive age, and unarchived WAL bytes. Investigate rising archive age, failed archives, and unarchived WAL together with disk usage and IOPS. `max_slot_wal_keep_size` caps WAL retained for replication slots so a lagging consumer can't exhaust the disk.

## Logical replication and CDC

Each shard has its own WAL and logical replication path, so CDC is consumed as per-shard streams and replicating a whole sharded database means one stream per shard. Replication from a replica is unavailable (error code 2). A shard added later receives existing publications with the rest of the schema. Subscription DDL applies to the shards that exist when it runs and is not restored when a shard is added or rebuilt. Contact PlanetScale Support before combining logical replication with shard lifecycle changes.

## Guidance

- Aim for mostly time-based checkpoints; raise `max_wal_size` if checkpoints are size-triggered too often.
- Enable `wal_compression` when replication bandwidth or WAL volume is a concern.
- Watch WAL growth per shard; a shard reporting low disk is treated as read-only by the router until it is resized.
