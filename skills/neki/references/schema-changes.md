---
title: Neki Schema Changes
description: Native DDL and managed Online/Direct DDL workflows across shards
tags: neki, schema-changes, online-ddl, direct-ddl, workflows, ddl
---

# Schema Changes in Neki

Docs: https://planetscale.com/docs/neki/schema-changes

A Neki database has one logical table definition across its managed shards, so every shard must end with the same definition. Neki offers **native DDL** and **managed DDL**.

## Native DDL

Issue a supported statement (for example `ALTER TABLE`) through a router; Neki sends it to every managed shard, where Postgres executes it immediately. Native DDL has no workflow record, progress tracking, readiness gate, or completion step, and is **not atomic** across the deployment — a failure can leave some shards changed and others not.

After native DDL commits, other routers may not have refreshed their schema view. The accepting router emits a notice with the exact call to run before sending dependent SQL through other routers:

```sql
SELECT __neki.wait_for_ddl(<schema_version>, <cluster_version>);
```

## Managed DDL (workflows)

Managed DDL records the requested DDL, tracks per-shard progress, waits until every shard is ready, and gives explicit control over completion and cleanup. Two execution paths share the same workflow:

- **Online DDL** — for disruptive changes (row rewrites, indexing a large table). Neki builds a **shadow table**, applies the DDL to it, copies existing rows in batches, and streams ongoing changes until caught up. **Cutover** is short: routers buffer queries for the table, Neki locks it, applies final changes, and swaps original and shadow in one transaction. If it can't get the lock within the cutover timeout it backs off and retries. Do **not** use `CONCURRENTLY` when creating an index via Online DDL.
- **Direct DDL** — sends the change to Postgres in a transaction; the only managed path for statements a shadow-table copy can't carry (`CREATE`/`DROP TABLE`, types, sequences, views). Gets the same tracking, readiness, and completion as Online DDL.

Neki picks the path automatically; force direct by passing `'{"applyDirect": true}'` as the fifth argument to `online_ddl_create`. Each workflow must resolve to **one** table and one execution category. Changes that touch multiple tables or mix "online" and "direct" execution paths must be split into multiple workflows.

### Workflow lifecycle (SQL metafunctions)

```sql
-- create (workflow name, database, DDL, migration id; '' = auto-generate)
SELECT * FROM __neki.online_ddl_create(
  'add-refund-state', 'orders_db',
  'ALTER TABLE public.orders ADD COLUMN refund_state text NOT NULL DEFAULT ''none''', '');

-- track (wait for every shard: ddl_status "running", current_readiness true)
SELECT workflow, status, jsonb_pretty(status_by_shard) FROM __neki.online_ddl_status('add-refund-state');

-- complete (explicit; online shards cut over, direct shards apply)
SELECT __neki.workflow_complete('add-refund-state');

-- clean up (drop the retained original table and artifacts after success)
SELECT __neki.online_ddl_cleanup('add-refund-state');

-- or cancel before completion
SELECT __neki.workflow_cancel('add-refund-state');
```

Completion is coordinated but **not** one atomic transaction across the database (shards usually finish within seconds of each other). Repeated completion requests are safe. The workflow reports `completed` when every shard is done. If a shard fails, recover **forward**: `online_ddl_cleanup` the failed attempt, reissue `online_ddl_create` with the stored configuration, then complete so every shard converges. Cancellation is rejected while a shard is mid-cutover; if some shards completed and others cancelled, Neki keeps the workflow — reissue create and complete rather than cleaning up blindly.

## Choosing a path

- **Native** when the change is safe to run immediately and needs no tracking.
- **Online** when direct execution would block traffic, rewrite a large live table, or build an index aggressively.
- **Direct (managed)** when no copy is needed or the statement can't use a shadow table, but you want coordination.

Common single-table `ALTER TABLE` (add or drop column, change type, add or drop constraint, add index), `CREATE INDEX`, and `DROP INDEX` use the online path. `CREATE`/`DROP TABLE` and type, sequence, and view changes use direct.

## Shard-key changes are not schema changes

Changing a table's shard key or primary routing index moves data between shards — use a Reshard workflow, not DDL. See [resharding-migration.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/resharding-migration.md).
