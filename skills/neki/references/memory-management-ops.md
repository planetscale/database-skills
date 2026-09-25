---
title: Neki Memory Management (per shard)
description: Postgres memory tuning per shard and out-of-memory prevention
tags: neki, memory, shared_buffers, work_mem, oom, cluster-size
---

# Memory Management in Neki

Docs: https://planetscale.com/docs/neki/cluster-configuration/parameters (Resource usage) · https://planetscale.com/docs/neki/monitoring/metrics

Each shard is real Postgres, so Postgres memory rules apply **per shard**. A shard's total memory comes from its **cluster size** (set on its configuration profile); memory parameters are set on the profile Postgres tab and apply to every shard on that profile.

## Memory areas (per Postgres instance)

- **Shared**: `shared_buffers` (main data cache; restart to change).
- **Per backend**: `work_mem` (per sort, hash, or join operation), `maintenance_work_mem` (VACUUM, `CREATE INDEX`), temp buffers.
- **Planner hint**: `effective_cache_size` (not allocated; set to about 50–75% of the instance's RAM).

## Memory multiplication risk

`work_mem` is per operation, not per query: the worst case is roughly `work_mem × operations_per_query × (parallel_workers + 1) × concurrent_connections` on a shard. High concurrency plus large sorts and hashes is the usual cause of out-of-memory events. Sidecar pooling keeps backend counts lower than direct Postgres connections would, but still size `work_mem` conservatively and set `max_parallel_workers_per_gather` deliberately.

## Tuning levers (configuration profile)

`shared_buffers` (restart), `work_mem`, `maintenance_work_mem`, `effective_cache_size`, `max_parallel_workers`, `max_parallel_workers_per_gather`, `max_worker_processes` (restart), and `huge_pages` (restart; must be a supported combination with `shared_buffers`), plus `statement_timeout` and `lock_timeout` to bound runaway work. Put a memory-intensive shard on its own profile so its settings don't apply to the whole fleet. Defaults depend on the cluster size, and change when the size changes unless you set a value manually.

## Out-of-memory prevention and monitoring

- The Metrics Shards tab shows primary and replica memory use and **container out-of-memory restarts** (with a warning banner when one occurred); the Routers tab shows router memory and restarts.
- Correlate an out-of-memory event with memory, connections, and query activity before resizing; if instances repeatedly approach their limits, increase the profile's cluster size.
- Keep `work_mem` modest globally and raise it per session only for heavy queries; set `statement_timeout` for runaway queries; keep parallelism modest under high concurrency.
