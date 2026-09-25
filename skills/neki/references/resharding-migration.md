---
title: Neki Data Migration and Resharding
description: MoveTables, Reshard, verification, cutover, and imports
tags: neki, resharding, movetables, migration, differ, workflows, import
---

# Data Migration and Resharding in Neki

Docs: https://planetscale.com/docs/neki/data-migration · https://planetscale.com/docs/neki/replication · https://planetscale.com/docs/neki/imports/postgres

Neki moves data online with the **Replicator**: it copies existing rows, then streams source changes to the target until you switch traffic. The source keeps serving until cutover.

## Two workflows

| Workflow | What changes |
| --- | --- |
| **MoveTables** | Move selected tables to another database, or to a shard group whose shards are not the source shards |
| **Reshard** | Redistribute declared tables from one source shard group into a target shard group in the same database (this is how you shard an imported unsharded database) |

During Platform Preview these are driven by **SQL metafunctions on a router** (no dashboard or CLI workflow). Mutations require the `neki_operator` role; status and report functions require `neki_viewer`. A workflow keeps running after the SQL session that created it disconnects.

## Before a Reshard

- Add the destination shards first and wait for each to have a ready primary; don't reuse the source shard as a target.
- Declare every physical base table that resolves to the source group in the data topology (including tables that inherit it from a default) and keep foreign-key-related tables in the same group. Reshard does not repoint dependent views.
- Splitting the initial authoritative group leaves authority on the original shard, so a two-way split of existing data uses three shards.

## Lifecycle

```sql
-- create (Reshard shown; move_tables_create for MoveTables). Starts unless create_stopped.
SELECT __neki.reshard_create('reshard_events','postgres','imported',
  '{"default_shard_index":"xxhash_tenant_id","key_ranges":[
     {"shard_uid":"<SHARD_2>","end":"80"},{"shard_uid":"<SHARD_3>","start":"80"}]}',
  'events_by_tenant', '{"create_stopped": true}');
SELECT __neki.workflow_start('reshard_events');
SELECT * FROM __neki.workflow_status('reshard_events');   -- wait for phase 'streaming'

-- verify with a differ BEFORE switching
SELECT __neki.differ_create('reshard_events','pre_cutover');
SELECT * FROM __neki.differ_report('reshard_events','pre_cutover');

-- switch (staged or together), then complete
SELECT * FROM __neki.workflow_switch_reads('reshard_events');
SELECT * FROM __neki.workflow_switch_writes('reshard_events');
-- or: SELECT * FROM __neki.workflow_switch_traffic('reshard_events');
SELECT __neki.workflow_complete('reshard_events');
```

Phases: `initializing` → `copying` → `streaming` (initial copy done, changes still applied). Reaching `streaming` does **not** switch traffic. `workflow_stop` and `workflow_start` preserve progress; `workflow_cancel` removes the workflow (drops target data unless `{"keep_data": true}`) and can't be used after traffic moves. `SELECT name, arguments, purpose FROM __neki.list_metafuncs();` lists what your router exposes.

Useful create options: `create_stopped`, `stop_after_copy`, `copy_batch_size`, `copy_phase_duration`, `read_from_standby` (Postgres 16+ source), `on_ddl` (`ON_DDL_ACTION_STOP` needs Postgres 14+ and a superuser source; default `ON_DDL_ACTION_IGNORE`), `skip_defer_secondary_keys`. Only `copy_batch_size` and `copy_phase_duration` change on a running workflow (`__neki.workflow_set_options`). Migration streams do not apply source DDL to the target — keep schemas compatible separately.

## Verify before cutover

A **differ** compares source and target rows. Approve cutover only when the report has `complete: true`, `mismatch: false`, no shard errors, every expected table present with `completed: true`, and no missing, extra, or mismatched rows. A clean differ is necessary but not sufficient — also confirm every table and stream was included and run an application-level check. After the differ runs, confirm all streams are `running` in `streaming` again (cutover refuses streams that aren't).

## Cutover details

A staged switch moves non-primary reads (`replica`/`rdonly`) first, then writes. During a write switch Neki buffers affected queries, stops new source writes to the moving tables, waits for changes to reach the target, then updates topology. It also synchronizes owned sequences to the maximum value on the target before switching. **Reversing a write switch is not a documented Platform Preview rollback path** — don't plan cutover around it.

## Completing and source cleanup

`workflow_complete` retires migration resources; source data is retained by default. Pass `{"drop_source_data": true}` to truncate old source rows. Reshard keeps table identifiers (source tables are truncated on cleanup, not renamed); MoveTables can instead rename old source tables with an `_old` suffix. An old source shard can be removed later if it holds no other data and isn't the authoritative group.

## Importing external Postgres

External sources are **not** supported for Neki migration workflows during Platform Preview. Import offline: stop source writes, `pg_dump --format=custom --no-owner --no-privileges`, `pg_restore --no-owner --no-privileges --exit-on-error` into a fresh **unsharded** Neki database, `ANALYZE`, validate (row counts, checksums, representative queries, identity inserts, roles, TLS), then cut over. Afterwards use **Reshard** to shard the tables. Check source extensions against the profile's Extensions tab first. For help: migrations@planetscale.com.
