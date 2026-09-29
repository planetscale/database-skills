---
title: Neki Data Topology and Sharding Model
description: How Neki maps tables and shard keys to shards
tags: neki, data-topology, shard-group, shard-index, key-range, sharding
---

# Neki Data Topology and Sharding Model

Docs: https://planetscale.com/docs/neki/data-topology · https://planetscale.com/docs/neki/terminology

The **data topology** is a JSON document that defines how a Neki cluster's data is distributed. It declares four object kinds:

- **Databases** — logical databases, schemas, tables, and sequences (same shape as a Postgres catalog).
- **Shards** — units of physical storage (a Postgres primary plus replicas). Shards are created on the cluster configuration page, not in the topology.
- **Shard groups** — named key-range layouts. Each is a list of key ranges, each pointing at a shard. Tables attach to a shard group.
- **Shard indexes** — named routing functions (type plus columns). Shard groups and tables reference them by name.

## Shard key, shard index, key range

A table's **shard key** columns decide placement. A **shard index** turns those values into a routing value; the router compares that value against a shard group's **key ranges** to pick the destination shard.

Neki supports exactly three shard-index types (any other is rejected):

| Type | Use | Notes |
| --- | --- | --- |
| `xxhash` | Distribute values evenly across a hex keyspace | `XXH3-64`; accepts `text`, `varchar`, `bpchar`, `bytea`, `int2/4/8`, `float4/8`, `numeric`, `date`, `timestamp`, `timestamptz`, `uuid`; not `json`, `jsonb`, arrays, or records |
| `modulo` | Map an integer key into a fixed number of buckets | requires `modulus` in `index_params`; `int2/4/8` |
| `range` | Route an integer key by its value | `int2/4/8` |

Prefer `xxhash` unless you have a specific reason not to: it distributes rows evenly. After defaults and overrides resolve, `columns` holds exactly one entry — a column name or a deterministic expression over a single unqualified column (for example `"lower(email)"` or `"tenant_id * 1000"`). An expression that names more than one column, such as `"tenant_id * 1000 + user_id"`, is stored, but `INSERT` returns `NK013` code `10` (`multi column shard index inserts are not supported`). `SELECT` and `DELETE` scatter unless `=` names every base column of that expression, which returns code `115`. A comparison other than `=` still scatters. `UPDATE` of a column that is not part of the expression follows the same rule. `UPDATE` of a base column of the expression returns code `145`, including when there is no `WHERE`. A `columns` list with more than one entry is rejected when the topology is applied, also code `10` (`multi-column partitioning indexes are not supported`).

```json
{
  "databases": {
    "analytics": { "schemas": { "public": { "tables": {
      "events": { "shard_group": "tenant_data" }
    } } } }
  },
  "shard_groups": [
    { "uid": "tenant_data", "default_shard_index": "xxhash_tenant_id",
      "key_ranges": [
        { "shard_uid": "shard-a", "end": "40" },
        { "shard_uid": "shard-b", "start": "40", "end": "80" },
        { "shard_uid": "shard-c", "start": "80", "end": "c0" },
        { "shard_uid": "shard-d", "start": "c0" }
      ] },
    { "uid": "authoritative", "key_ranges": [ { "shard_uid": "shard-z" } ] }
  ],
  "shard_indexes": { "xxhash_tenant_id": { "type": "xxhash", "columns": ["tenant_id"] } },
  "authoritative_shard_group": "authoritative"
}
```

Changing a group's shard-index type without changing its key ranges can concentrate rows on one shard or leave values with no destination.

## Shard-group resolution and defaults

A table's group resolves in order: the table's `shard_group` → the schema's `default_shard_group` → the database's `default_shard_group` → the topology's `default_shard_group`. Unlisted tables inherit the effective default; if nothing resolves, the table is not routable.

## Authoritative shard group

Every topology must name one **authoritative shard group** containing exactly one unbounded key range (one shard). It provides canonical Postgres OIDs and schema events; the router rewrites result-set type identifiers so clients always see the authoritative OIDs. It is also the default home for sequences that don't derive placement from an owned unsharded table. A new cluster starts as a single shard that is both default and authoritative; after sharding, leave that shard as the standalone authoritative group and size it for catalog work, sequence reservations, and unsharded tables. Authority currently cannot move to a different physical shard.

## Co-location

To keep joins shard-local, bind related tables to the **same shard group** and route them through the **same shard index** (for example both `events` and `event_metadata` on `tenant_data` via `xxhash_tenant_id`). Rows with the same key then land on the same shard.

## Reference tables and GSIs

For access that the shard key can't satisfy, a **reference table** keeps a full copy of shared data on every shard in a group (local joins), and a **global secondary index (GSI)** maps another key to the owner row's shard key via a lookup table. Both add write cost. Both are declared inside a schema, next to `tables`:

```json
"databases": {
  "postgres": {
    "global_secondary_indexes": {
      "by_email": { "schema": "public", "table": "users_by_email", "owner_table": "users",
                    "columns": ["email"], "owner_sk_columns": ["tenant_id"],
                    "unique": true, "ignore_null": false, "enabled": false }
    },
    "schemas": {
      "public": {
        "tables": {
          "users": { "shard_group": "tenant_data",
                     "secondary_indexes": [ { "name": "by_email", "columns": ["email"] } ] },
          "users_by_email": { "shard_group": "authoritative" }
        },
        "reference_tables": {
          "countries": { "shard_groups": ["tenant_data"] }
        }
      }
    }
  }
}
```

`reference_tables` maps a table name to the list of shard groups that hold a copy. A GSI needs both the database-level `global_secondary_indexes` entry and the owner table's `secondary_indexes` entry, and its lookup table needs its own table binding. Details and safe activation in [indexing.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/indexing.md).

## Editing the topology

An update is a full replacement, not a patch. View or replace it in the dashboard (Clusters → Data topology), the CLI (`pscale branch data-topology get|ls|update`), or SQL:

```sql
SELECT * FROM __neki.set_data_topology($topology$ { ... } $topology$, true,
  '{"comment":"...", "expected_revision":42}');
SELECT __neki.wait_for_data_topology(<revision>);   -- wait for all routers to see it
```

Use `overwrite => false` only to create the first topology; use `expected_revision` for safe concurrent edits; use `force` only with PlanetScale Support.
