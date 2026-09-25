---
title: Neki Architecture Overview
description: Neki components and how they fit together
tags: neki, architecture, router, sidecar, admin, replicator, topology
---

# Neki Architecture

Docs: https://planetscale.com/docs/neki/overview

Neki is distributed Postgres by PlanetScale. Every shard is real Postgres; Neki adds a router, per-instance sidecars, and a control plane to route, coordinate, and operate Postgres across many shards behind one Postgres connection string. Neki is built by the maintainers of Vitess but is a from-scratch system for Postgres, not a Vitess port.

> **Platform Preview:** Neki is in Platform Preview (a Beta Feature with no SLA). Single-node, non-HA configurations are not supported during the preview.

```
client ──▶ Neki router ──▶ sidecar ──▶ Postgres (primary/replica)   ← one shard
                 │                         (PostgresManager manages the instance)
        (parses, plans, routes,
         combines, buffers)
control plane: topology service · admin · Replicator
```

## Routers

A **router** accepts Postgres connections (port `5432`) and decides which shard(s) run each statement. It contains a full Postgres parser and planner, uses the data topology and table definitions to plan and route queries, and combines results when a query touches multiple shards. Routers are **stateless** (topology and schema are cached and kept in sync), so there is no leader — any router serves any connection, and they scale horizontally. Every Neki cluster ships with a minimum of 3 routers across three availability zones. Routers support **query buffering** to hold queries briefly during switchovers and failovers, so clients see elevated latency rather than errors.

## What runs alongside Postgres

A **shard** is one Postgres primary plus its replicas, and is its own failure domain (independent switchover and failover). Each Postgres instance runs two Neki components:

- **Sidecar** — the per-instance endpoint for router queries and admin operations; handles connection pooling and streams serving status to routers. All router-to-Postgres traffic goes through the sidecar.
- **PostgresManager** — manages startup, teardown, and the data directory of its Postgres instance.

## Control plane

- **Topology service** — stores the data topology and service-discovery records for routers, admins, replicators, and sidecars. Routers and sidecars pick up new topology without restarting.
- **Admin** — health-checks each Postgres instance through its sidecar and coordinates repairs. Handles planned switchovers and unplanned failovers (promotes the replica with the most caught-up WAL, then re-points the other replicas). Multiple admins can run; one holds recovery leadership.
- **Replicator** — runs data-movement workflows (MoveTables, Reshard, and the copy-and-stream behind Online DDL). Connects to Postgres directly, not through routers.
- **Orchestration layer (Neki operator)** — reconciles requested configuration into running routers, admins, shards, and Postgres instances; coordinates rolling changes, resizing, backups, and restores. It is outside the SQL query path.

## High availability

Each shard's primary replicates to its replicas via Postgres physical replication; on primary failure the admin promotes an eligible replica and updates topology. Because routers are stateless and redundant across availability zones, a router failure just drops its connections (clients reconnect to another router). A new shard's durability policy defaults to `sync` (the primary waits for one replica before confirming a commit). If a shard reports low disk, the router treats it as read-only until it is resized. See [replication.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/replication.md).

## Query lifecycle

1. Client sends a Postgres query to a router.
2. The router parses it and consults the data topology and schema.
3. It builds a plan: single-shard when a shard-key predicate is present, otherwise multi-shard or scatter (see [query-serving.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/query-serving.md)).
4. Work is sent to sidecars in parallel; each shard's Postgres executes locally.
5. The router combines results (aggregate, sort, limit) and returns one Postgres result.

See [sharding-model.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/sharding-model.md) for how data maps to shards.
