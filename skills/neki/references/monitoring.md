---
title: Neki Monitoring
description: Metrics, logs, Query Insights, anomalies, and schema recommendations
tags: neki, monitoring, metrics, query-insights, anomalies, logs
---

# Monitoring a Neki Database

Docs: https://planetscale.com/docs/neki/monitoring · https://planetscale.com/docs/neki/monitoring/metrics

Monitor the routers, the Postgres instances in each shard, and query patterns across the database. PlanetScale provides five complementary tools:

| Tool | Use it to |
| --- | --- |
| **Metrics** | Router traffic and latency, per-shard Postgres utilization, storage, WAL activity, replication lag, instance health |
| **Logs** | Search individual events, filtered by shard, server, severity, and time |
| **Query Insights** | Find expensive or frequent query patterns and their shard-level cost |
| **Anomalies** | Find periods when queries run slower than their baseline |
| **Schema recommendations** | Review automatic DDL and index suggestions from production telemetry |

## Metrics (Shards, Storage, Routers tabs)

- **Routers**: queries per second (by logical database), p50/p95/p99 latency, query errors per second, router CPU and memory, container and out-of-memory restarts, pod status.
- **Shards**: primary and replica CPU, memory, IOPS, connections by state, locks, transaction rate, replication lag, container out-of-memory restarts, pod status.
- **Storage**: primary and replica disk usage and bytes, WAL storage, WAL archive success and failure rates, WAL archive age, unarchived WAL.

Live mode refreshes about every 30 seconds; the default range is 12 hours (custom ranges up to 7 days). Filter Shards and Storage by configuration profile and Routers by router group.

## Query Insights

Groups executions into query patterns with latency, execution count, rows read and written, errors, tags, **shard calls**, and parallel-worker activity. A high shard-call count means fanout across shards or repeated dispatch to one shard — review the query plan (`EXPLAIN (NEKI_PLAN)`) and the data topology. *Qualified table* shows the resolved `database.schema.table`.

## Investigation flow

1. Query Insights → identify the affected pattern and time range.
2. Metrics → check router latency, shard resource use, storage and WAL, and replication lag in that window.
3. Logs → find errors and events on the affected shards and instances.
4. Compare across shards to see whether the issue is isolated or branch-wide.

Interpret metrics against each branch's **baseline** — there's no single healthy value. If one shard is hotter (CPU, IOPS, storage) than the others, review its routed workload and topology; if router latency rises without shard saturation, inspect routing, fanout, and result combination.

## Database-wide activity view

The `__neki.stat_get_activity()` SQL metafunction lists client sessions across all routers, enriched with what each backend is doing.

## Per-shard Postgres views

Standard Postgres views (`pg_stat_activity`, `pg_stat_user_tables`, `pg_stat_user_indexes`) work on a given shard (`SET __neki.shard = '<shard_uid>'`, or the dashboard web console). Dead tuples, index usage, and WAL are ordinary Postgres — use the [postgres skill](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/postgres/SKILL.md). Per-query statistics for Query Insights come from the always-enabled `pginsights` extension.
