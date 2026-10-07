---
title: Neki CLI, Metafunctions, and Insights
description: pscale commands, __neki router metafunctions, session settings, and query telemetry
tags: neki, cli, pscale, metafunctions, query-insights, mcp
---

# CLI, Metafunctions, and Insights

Docs: https://planetscale.com/docs/neki/connecting · https://planetscale.com/docs/neki/data-migration · https://planetscale.com/docs/neki/monitoring/query-insights

Neki is managed through the PlanetScale dashboard, the `pscale` CLI, the API and Terraform provider, and — for topology, schema-change, and data-movement work — SQL **metafunctions** in the `__neki` schema, run on a router connection. During Platform Preview, MoveTables and Reshard have no dashboard or CLI workflow and are driven by metafunctions.

## pscale CLI

```bash
# connect with temporary credentials; pick a router group or replica routing
pscale shell <DB> <BRANCH> --org <ORG> [--router <GROUP>] [--replica]

# data topology
pscale branch data-topology get <DB> <BRANCH> --org <ORG>
pscale branch data-topology ls  <DB> <BRANCH> --org <ORG>
pscale branch data-topology update <DB> <BRANCH> --org <ORG> --format json < topology.json

# shards, extensions, point-in-time restore
pscale branch shard create <DB> <BRANCH> --config-profile default --count 2
pscale branch config-profile extensions [enable|disable] ...
pscale branch create <DB> <NEW_BRANCH> --from <SOURCE_BRANCH> --restore-point <RFC3339>

# application roles
pscale role ...
```

## Session settings

Set before a transaction (see [query-serving.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/query-serving.md)):

| Setting | Values (default first) |
| --- | --- |
| `__neki.target` | `primary`, `replica`, `rdonly` |
| `__neki.fanout` | `scatter`, `multi`, `single` |
| `__neki.tx_mode` | `multi`, `single` (`atomic` is rejected) |
| `__neki.shard` | a shard UID (empty = normal routing) |
| `__neki.replica_recency` | `prefer`, `require`, `off` |
| `__neki.replica_locality` | `prefer`, `require`, `off` |
| `__neki.replica_affinity` | `none`, `session` |

## Router metafunctions (`__neki.*`)

- **Topology**: `set_data_topology(json, overwrite, options)`, `wait_for_data_topology(revision)`, `list_shards()`.
- **Client activity**: `stat_get_activity`.
- **Schema changes**: `online_ddl_create(workflow, database, ddl, migration_id[, options])`, `online_ddl_status`, `workflow_complete`, `workflow_cancel`, `online_ddl_cleanup`, `wait_for_ddl(schema_version, cluster_version)` (see [schema-changes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/schema-changes.md)).
- **Data movement**: `reshard_create`, `move_tables_create`, `workflow_start`, `workflow_stop`, `workflow_status`, `list_workflows`, `workflow_set_options`, `differ_create`, `differ_status`, `differ_report`, `differ_delete`, `workflow_switch_reads`, `workflow_switch_writes`, `workflow_switch_traffic`, `workflow_complete`, `workflow_cancel` (see [resharding-migration.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/resharding-migration.md)). Mutations require `neki_operator`; status and report functions require `neki_viewer`.
- **Discovery**: `SELECT name, arguments, purpose FROM __neki.list_metafuncs();` lists what your router exposes.
- **Plans**: `EXPLAIN (NEKI_PLAN[, NEKI_PG_PLAN][, ANALYZE], COSTS OFF, FORMAT TEXT) <query>`.

## Insights and automation

- **Query Insights** — expensive and frequent query patterns, latency, rows, errors, tags, **shard calls**, and parallel workers. The router parameter `insights-raw-queries` sends full query text for slow, large, or failed queries (the text may contain sensitive data).
- **Anomalies** — periods when queries run slower than their baseline.
- **Schema recommendations** — automatic DDL and index suggestions from production telemetry.
- **MCP** — PlanetScale's hosted MCP server exposes these to agents; schedule an agent to review Query Insights, query errors, and schema recommendations.

## Web console

Run Postgres queries and DDL from the dashboard web console — handy for inspecting one shard (`SET __neki.shard`) or a logical database without a local client.
