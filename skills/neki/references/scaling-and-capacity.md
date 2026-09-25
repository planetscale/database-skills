---
title: Neki Scaling and Capacity
description: When to shard, shard counts, hot shards, workload isolation, and resizing
tags: neki, scaling, capacity, hot-shards, shard-groups, cluster-sizing
---

# Scaling and Capacity in Neki

Docs: https://planetscale.com/docs/neki/when-to-shard · https://planetscale.com/docs/neki/best-practices · https://planetscale.com/docs/neki/cluster-configuration/cluster-sizing

Neki scales in two ways: **scale up** a shard (larger cluster size, more replicas) and **shard horizontally** (spread a logical table across many shards, each with its own primary and replicas). You can start unsharded and shard later.

## When to shard

Shard when a single primary is the bottleneck **after** tuning configuration, queries, indexes, and cluster size (and considering Metal with local NVMe for IOPS). Signals:

- The primary is persistently limited by write throughput or disk IOPS.
- The working set no longer fits in one instance's RAM.
- You want to reduce the blast radius of one primary failure.
- A stable routing key keeps related data and common queries on one shard.

Evaluate single-shard improvements first with Query Insights and schema recommendations.

## Shard layout

- The **data topology** assigns rows to shards via shard groups, shard indexes, and key ranges (see [sharding-model.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/sharding-model.md)). Design so most queries reach **one** shard.
- Creating a shard adds capacity only; rows move there only after a topology change plus a data-migration workflow.
- Shards don't need to be evenly sized or identically configured — assign resources, replicas, parameters, and extensions per the data they hold, using separate configuration profiles.
- Use **shard groups** to place different tables or workloads (hot, cold, isolated) on the hardware they need.
- A two-way split of existing data uses **three** shards: the original stays as the authoritative single-shard group, and two new shards hold the split data.

## Hot shards and data skew

A poor shard key concentrates rows or traffic on one shard. Watch per-shard CPU, IOPS, and storage on the Metrics Shards tab and shard-call counts in Query Insights. Remedies: choose a higher-cardinality, more even shard key; split the hot shard with a Reshard workflow; or move a workload to its own shard group. Changing a shard key is a **resharding** operation, not an `ALTER` (see [resharding-migration.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/resharding-migration.md)).

## Resizing and workload isolation

- **Configuration profiles**: change a profile's cluster size (CPU, memory, storage) and replica count; changes roll through the profile's shards asynchronously (follow the Changes tab).
- **Router groups**: add router groups for independent router sizing and to isolate connection workloads (see [connections.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/connections.md)).
- **Admin**: sized separately; PlanetScale raises its memory automatically when needed.
- Profile, router, and admin changes are asynchronous — wait for one to finish before submitting a dependent change.

## Scale ceiling

Neki has been demonstrated at about 100 million queries per second and more than a petabyte across many shards. It also suits small unsharded databases, which still get online DDL, zero-downtime operations, connection pooling, and online version upgrades.
