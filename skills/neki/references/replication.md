---
title: Neki Replication and High Availability
description: Per-shard physical replication, replicas, failover, and read routing
tags: neki, replication, replicas, high-availability, failover, lag
---

# Replication and HA in Neki

Docs: https://planetscale.com/docs/neki/replicas · https://planetscale.com/docs/neki/overview

Each Neki shard is a highly available Postgres cluster: one **primary** and, on a production branch, **at least two replicas**. Replicas copy changes via Postgres **physical replication** (WAL), serve read-only queries, and can be promoted on failover. This per-shard replication is separate from the **Replicator** workflows used for migrations and Online DDL (see [resharding-migration.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/resharding-migration.md)).

> **Platform Preview:** single-instance, non-HA configurations are not supported.

## Failover and switchover

- **Failover** (unplanned): the **admin** health-checks each instance through its sidecar. On primary loss it promotes an eligible replica — preferring one that has replayed all available WAL, otherwise the one that received the most WAL — then re-points the other replicas (repairing any it can't update on a later health check).
- **Switchover** (planned, e.g. before maintenance or during a rolling upgrade): Neki makes the old primary read-only, waits for the chosen replica to catch up, then promotes it, preserving all commits. If it fails before promotion, the old primary becomes writable again.

Routers buffer queries during both, so clients usually see a short latency spike; still **retry transient errors**. The durability policy defaults to `sync` (the primary waits for one replica to receive a commit), so there is normally a fully caught-up promotion candidate.

If a replica needs a WAL file no source can supply, Neki fences it, restores it from the shard's last backup, and rejoins it to the primary — without changing the primary.

## Read routing and lag

Reads default to the **primary**. Send reads to replicas with `__neki.target = 'replica'` (see [connections.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/connections.md)). Per shard, the router starts from serving replicas with known lag and orders them by:

| Policy | Default |
| --- | --- |
| Recency (`__neki.replica_recency`) | `prefer` replicas with ≤30s lag |
| Locality (`__neki.replica_locality`) | `prefer` the router's availability zone |
| Affinity (`__neki.replica_affinity`) | `none` — choose again per statement |

Replicas above the maximum lag (default 15 minutes) or with unknown lag are excluded; if none qualify, a replica-targeted query **fails** rather than falling back to the primary. The thresholds are the router group parameters `replication-lag-tolerable-min` (30s) and `replication-lag-tolerable-max` (15m).

## Consistency caveats

WAL can be durable on a replica before the replica has **replayed** it, so a successful write doesn't guarantee an immediate replica read sees it. Replica reads don't guarantee read-your-writes or monotonic reads (a later read can move to a different replica). Send read-after-write paths to the primary. For multi-shard reads, Neki picks a replica per shard, so results can reflect slightly different points in time (see [transactions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/transactions.md)).

## Managing replicas

Add replicas per configuration profile on the Clusters page (applies to every shard on that profile). Each new replica is restored from the shard's last backup, then joined to the primary. Monitor lag on the dashboard and the Metrics Shards tab (`0s` = caught up); investigate high or unknown lag via replica CPU, IOPS, and connections alongside WAL activity. See [monitoring.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/monitoring.md).
