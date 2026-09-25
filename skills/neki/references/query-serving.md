---
title: Neki Query Planning and Routing
description: Cross-shard routing, plans, fanout, and read targeting
tags: neki, query-planning, routing, scatter-gather, fanout, explain
---

# Neki Query Planning

Docs: https://planetscale.com/docs/neki/query-planning

A router parses each statement and builds a **plan** from the SQL and current data topology. The plan contains routes that send work to shards and, when needed, router operations that combine shard results. With the simple query protocol, Neki parameterizes eligible literals (`tenant_id = 12` and `tenant_id = 13` both become `tenant_id = $1`) so plans are reusable; the value bound at execution decides the route. Preparing a statement does not pin its plan — it can be rebuilt after a schema or topology change.

## Reading a plan

Use `EXPLAIN (NEKI_PLAN)` (add `FORMAT TEXT` for a readable tree, `COSTS OFF` to drop estimates):

```sql
EXPLAIN (NEKI_PLAN, COSTS OFF, FORMAT TEXT)
SELECT event_id FROM public.events WHERE tenant_id = 4821;
```
```text
Route [EqualUnique]
  Query: SELECT event_id FROM public.events WHERE tenant_id = $1
  ShardGroup: tenant_data
  Values: $1
```

| Plan | Meaning | Cost |
| --- | --- | --- |
| `Route [EqualUnique]` | Single shard via shard-key equality | Best |
| `Route [IN]` (under `Collapse`) | Bounded set of shards from an `IN` list | Good |
| `Route [Scatter]` (under `Collapse`/`Aggregate`) | Every shard in the group | Expensive |

Add `NEKI_PG_PLAN` to include each shard's Postgres plan inside the routes. A plain Postgres `EXPLAIN` works only when the plan reduces to a single route; plans with router-side operators are rejected with a pointer to `NEKI_PLAN`. `EXPLAIN (NEKI_PLAN, ANALYZE)` executes the statement.

**If you expect single-shard but see `Route [Scatter]`:** add a shard-key predicate, or use a GSI lookup (see [indexing.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/indexing.md)).

## Controlling fanout

`__neki.fanout` rejects `SELECT` and DML whose routing shape exceeds the level (default `scatter`):

```sql
SET __neki.fanout = 'single';   -- or 'multi' or 'scatter'
```

`single` = single-shard with at most one shard-touching operation; `multi` = bounded set of shards; `scatter` = anything, including every shard. Use a stricter setting in development or CI to catch overly broad queries. It does not apply to DDL, and fanout needed to maintain reference tables and GSIs is exempt.

## Choosing where reads run

Reads default to the shard **primary**. Change with `__neki.target`:

```sql
SET __neki.target = 'replica';   -- 'primary' (the only target that accepts DML), 'replica', or 'rdonly'
```

Replica selection is tuned by `__neki.replica_recency` (`prefer`/`require`/`off`; prefers ≤30s lag), `__neki.replica_locality` (`prefer`/`require`/`off`; the router's availability zone), and `__neki.replica_affinity` (`session`/`none`). Replicas above the maximum lag (default 15 minutes) or with unknown lag are excluded; the query fails if no candidate remains.

## Targeting one shard directly

`SET __neki.shard = 'shard-a';` forwards data statements (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE`) to that shard UID without topology routing. Schema-changing DDL, `COPY`, `DO`, and data queries that also call router-managed functions (`nextval`, `current_setting`, `set_config`) are rejected while it is set. `RESET __neki.shard;` restores routing. Use it for debugging one shard, not for application writes.

All `__neki.*` routing settings (`target`, `shard`, `fanout`, `tx_mode`, and the replica policies) cannot change inside a transaction — set them before `BEGIN`. A misspelled name (for example `__neki.transaction_mode`) is accepted silently as an ordinary custom parameter and has no effect, so confirm a setting took hold with `SHOW __neki.tx_mode;`.

## Cross-shard execution and combining

Multi-shard plans dispatch in parallel and combine results (partial aggregates, cross-shard ordering and limits, joins that can't run on one shard). A query without `ORDER BY` has **no** cross-shard order guarantee — add explicit ordering when it matters. As a shard group grows, scatter queries touch more shards; avoid scatter in hot paths.

`COPY FROM STDIN` into a sharded table works when the input includes the shard-key column; it is rejected for a table with an enabled GSI (COPY doesn't maintain lookup rows). Other `COPY` limits: file-based `COPY` works for unsharded tables only; `COPY TO PROGRAM`, `COPY FROM PROGRAM`, `COPY (SELECT ...) TO`, and `COPY FROM ... WHERE` are rejected; `COPY` must use the simple query protocol and be the only statement in the query. If a `COPY` shape is rejected, the error carries an `NK013` catalog code (see [error-codes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/error-codes.md)).

## DDL and transactions (brief)

DDL sent through a router runs synchronously on every managed shard and is **not atomic** across the deployment — use managed workflows for coordination (see [schema-changes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/schema-changes.md)). Multi-shard reads don't share a snapshot and multi-shard writes aren't atomic — keep transactions single-shard (see [transactions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/transactions.md)).
