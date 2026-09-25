---
title: Neki Process and Connection Architecture
description: Router, sidecar, and Postgres processes and connection handling
tags: neki, processes, connections, router, sidecar, pooling
---

# Process and Connection Architecture in Neki

Docs: https://planetscale.com/docs/neki/overview · https://planetscale.com/docs/neki/cluster-configuration/parameters

Neki changes how connections reach Postgres. Applications connect to a **router**, not to Postgres directly. The query path is **client → router → sidecar → Postgres**.

## Components in the path

- **Router** — stateless; full Postgres parser and planner; plans and routes each statement; combines multi-shard results; buffers during switchovers and failovers. Scale routers vertically and horizontally, and use multiple **router groups** to isolate workloads.
- **Sidecar** — one per Postgres instance; the endpoint for router-to-Postgres traffic and the **connection pool** to that instance.
- **PostgresManager** — manages the Postgres instance's lifecycle and data directory.
- **Postgres** — still the multi-process engine on each shard (one backend process per server connection).

## Why this scales connections

Direct Postgres connections are expensive (one backend process each), and PgBouncer helps but has its own limits. Neki routers pool more effectively and connect through sidecars beside each Postgres instance, so many application connections map to far fewer backend Postgres connections. Prefer scaling application-to-router connections and router capacity over raising Postgres `max_connections`.

## Relevant parameters

- **Sidecar pools** (profile Sidecars tab): `pool-capacity`, `pool-min-conns`, `pool-max-lifetime`, `pool-max-wait-time`, `pool-idle-timeout`, `tx-idle-timeout`, `external-conn-reservation`. Applied without restarting Postgres. `pool-capacity + external-conn-reservation` must fit the profile's connection budget.
- **Router** (router group Parameters tab): `idle-session-timeout` (default 30m), `idle-in-transaction-session-timeout` (default 5m), `stream-pool-size`, `sidecar-conns`, and the replica-lag thresholds.
- **Postgres** (profile Postgres tab): `max_connections` (restart), parallelism (`max_parallel_workers`, `max_parallel_workers_per_gather`), and per-operation `work_mem` (see [memory-management-ops.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/memory-management-ops.md)).

## Background Postgres processes (per shard)

Each Postgres instance still runs the WAL writer, background writer, checkpointer, autovacuum launcher and workers, and archiver — tuned through the configuration profile.

## Monitoring

Router metrics (queries per second, latency, errors, CPU, memory, restarts) and per-shard Postgres connection and lock metrics are on the Metrics Routers and Shards tabs. Investigate connection exhaustion or out-of-memory events through pooling settings and cluster size rather than reactively raising limits. See [monitoring.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/monitoring.md).
