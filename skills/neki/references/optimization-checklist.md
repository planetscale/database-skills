---
title: Neki Optimization Checklist
description: Routing and per-shard tuning checklist for Neki
tags: neki, optimization, routing, indexes, sharding, maintenance
---

# Neki Optimization Checklist

Docs: https://planetscale.com/docs/neki/best-practices · https://planetscale.com/docs/neki/monitoring

Work through routing first (Neki-specific), then per-shard Postgres tuning.

## Routing (do this first)

- Confirm hot queries include the **shard key** and route to a single shard — check with `EXPLAIN (NEKI_PLAN)` and Query Insights (watch shard-call counts).
- Enforce `SET __neki.fanout = 'single'` in development and CI to catch unintended scatter.
- Verify frequently joined tables are co-located in the same shard group via the same shard index.
- Keep transactions on one shard-key value (`SET __neki.tx_mode = 'single'` to enforce).
- Identify global aggregations; move them to rollup tables or scope them to the shard key.
- Use reference tables and selective GSIs for non-shard-key access paths.
- Look for hot shards and data skew from an uneven shard key (see [scaling-and-capacity.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/scaling-and-capacity.md)).

## Per-shard Postgres

- Find unused, duplicate, and invalid indexes; keep per-table index counts lean (write cost is multiplied per shard). Audit **all** shards.
- Ensure indexes lead with the shard key; scope unique constraints to include it.
- Review large tables for in-shard partitioning (>100 GB general, >50 GB time-series).
- Check dead tuples and HOT-update ratios; tune autovacuum per hot table or per configuration profile.
- Confirm cluster size and `work_mem`/`shared_buffers` fit the workload; check Metrics for saturated shards.
- Verify required extensions are enabled on the profile (see [extensions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/extensions.md)).

## Use PlanetScale telemetry

- Query Insights for expensive or frequent patterns and shard calls; Anomalies for baseline regressions; schema recommendations for index and DDL suggestions.

## Safety

- **Always confirm with a human before removing indexes, dropping partitions, dropping data, or other destructive actions.**
