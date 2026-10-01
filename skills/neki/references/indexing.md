---
title: Neki Indexing, Reference Tables, and GSIs
description: Per-shard indexing plus reference tables and global secondary indexes
tags: neki, indexes, gsi, reference-tables, composite, sharding
---

# Indexing in Neki

Docs: https://planetscale.com/docs/neki/reference-tables-and-gsis

Each shard is Postgres, so ordinary Postgres indexing applies **per shard**. Two distributed additions matter: lead a sharded table index with the shard key if the index needs to be globally unique, and use **reference tables** or **global secondary indexes (GSIs)** for access paths the shard key can't serve.

## Per-shard index rules

1. If a composite index on a sharded table must be unique between shards, **lead the index with the shard key**, then equality, range, and sort columns.
2. Always index foreign key columns (Postgres does not create these automatically).
3. Index columns used in `WHERE`, `JOIN`, and `ORDER BY`.
4. Don't over-index — each extra index is written on every shard. Audit usage per shard (`SET __neki.shard`); an index unused on one shard may be used on another. Schema recommendations surface candidates. For the audit queries, use the [postgres skill](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/postgres/SKILL.md).
5. Build indexes through Online DDL, and do **not** add `CONCURRENTLY` (see [schema-changes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/schema-changes.md)).

```sql
-- WHERE tenant_id = $1 AND status = 'active' AND created_at > $2
CREATE INDEX orders_tenant_status_created_idx ON orders (tenant_id, status, created_at);
```

Index types work per shard as in Postgres: B-tree (default), GIN (JSONB, arrays, full text), GiST (ranges, geometry, full text), BRIN (large append-only, time-ordered). Partial and covering indexes also work.

## Reference tables

A **reference table** keeps a full copy of shared data on every shard in a group, so joins to sharded data stay local. Good for small, rarely changing, broadly joined lookups (for example `countries`). Declare it under the schema's `reference_tables`, listing the shard groups that hold a copy:

```json
"schemas": { "public": {
  "tables": { "customers": { "shard_group": "tenant_data" } },
  "reference_tables": { "countries": { "shard_groups": ["tenant_data"] } }
} }
```

- Neki assumes the copies already exist and match — it does **not** verify them, so populate every copy before binding or start with an empty table and fill it after binding.
- A write to a reference table runs on **every** shard holding a copy (one write → N shard executions), and those per-shard commits are independent.
- Client `COPY TO STDOUT` works for a reference table, including text, CSV, and binary. File `COPY` does not (`NK013` code `43`).

## Global secondary indexes (GSIs)

A **GSI** maps another key to the owner row's shard key via a **lookup table**, so a query without the shard key can still route to one shard. Example: `users` sharded by `tenant_id`, looked up by `email`.

- A GSI has two linked declarations: a database-level `global_secondary_indexes` entry (lookup table, owner table, `columns`, `owner_sk_columns`, `unique`, `ignore_null`, `enabled`) and a `secondary_indexes` entry on the owner table. The lookup table also needs an ordinary table binding:

```json
"databases": { "postgres": {
  "global_secondary_indexes": {
    "by_email": { "schema": "public", "table": "users_by_email", "owner_table": "users",
                  "columns": ["email"], "owner_sk_columns": ["tenant_id"],
                  "unique": true, "ignore_null": false, "enabled": false }
  },
  "schemas": { "public": { "tables": {
    "users": { "shard_group": "tenant_data",
               "secondary_indexes": [ { "name": "by_email", "columns": ["email"] } ] },
    "users_by_email": { "shard_group": "authoritative" }
  } } }
} }
```
- Applications can't query or write the lookup table directly through a router.
- A **unique** GSI can serve a covering read when the lookup row has every needed column; a **non-unique** GSI may return several shard keys and narrow to those shards (it must also set `owner_pk_columns`).
- Use GSIs for **highly selective** lookups — if the value looked up in the GSI appears on most shards, the query approaches a scatter and the lookup isn't saving much work. The GSI also adds overhead when it's kept in sync with the owner table, so savings from GSI lookups must be worth the overhead.
- `UPDATE`/`DELETE` routed through a GSI are rejected; updating a GSI column is supported only for a single-row update of one active, globally unique, single-column GSI.

### Global uniqueness via GSI

Postgres enforces uniqueness within each lookup-table shard. For a global unique constraint, equal indexed values must always reach the same shard: use an unsharded lookup table, or shard the lookup table by the GSI's indexed columns. Neki rejects a lookup table sharded by a non-indexed column.

### Activate a GSI safely

An enabled but incomplete GSI can return incomplete results with no error. Create the lookup table, declare both GSI entries with `enabled: false`, backfill while following owner changes, verify every owner row has a lookup row, then set `enabled: true` at cutover. `COPY FROM` into a sharded owner with an enabled GSI is rejected — keep the GSI disabled while copying, then backfill, verify, and enable.

## Choosing what to duplicate

Use a **reference table** when a small shared dataset is needed for reads or joins on many shards. Use a **GSI** when a large sharded table needs selective access through another key. Both add write cost and neither replaces a good primary shard-key design. Check routing with `EXPLAIN (NEKI_PLAN)` (see [query-serving.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/query-serving.md)).
