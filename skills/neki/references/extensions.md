---
title: Neki Postgres Extensions
description: Built-in and community extensions and how to enable them
tags: neki, extensions, pgvector, configuration-profile, create-extension
---

# Postgres Extensions on Neki

Docs: https://planetscale.com/docs/neki/extensions

Neki runs real Postgres on each shard, so extensions install with `CREATE EXTENSION` — but enabling a profile-level setting and installing database objects are **separate** steps, and the dashboard catalog for the **configuration profile** is the authoritative list of what's available. Don't assume an extension you'd find on single-instance Postgres is available — check the catalog, or run `SELECT name FROM pg_available_extensions;` on the branch. If a user says an unlisted extension is installed, verify before giving commands for it.

Common extensions that are **not** in the catalog, and what to use instead:

| Not available | Use instead |
| --- | --- |
| `pg_repack` | Tune autovacuum per table and keep transactions short; `REINDEX INDEX CONCURRENTLY` for bloated indexes; `VACUUM FULL` only in a quiet window (it rewrites the table on every shard, even when `__neki.shard` is set, and blocks access to it) |
| `pg_partman` | Create and drop partitions with scheduled DDL |
| `pgstattuple` | Compare table and index sizes (`pg_relation_size`) and dead-tuple counts (`pg_stat_user_tables`) per shard |
| `pg_stat_statements` | Query Insights (backed by the always-enabled `pginsights`) |

## Two-step model

1. **Profile-level enablement** (when required): some extensions need server configuration (such as a preload library). Enable them on the profile's **Extensions** tab — a configuration-profile change that applies to every shard on the profile and may restart Postgres. Some extensions are **always enabled**; others need no toggle.
2. **Database install**: after any required profile change completes, run `CREATE EXTENSION IF NOT EXISTS <name>;` in each logical database that needs it.

The catalog is also available with `pscale branch config-profile extensions`, and `... enable` / `... disable` toggle the ones that can be toggled.

## Always enabled

`neki_xxhash` (hash functions used for distribution), `pg_pscale_utils` (privileged actions without superuser), `pgextwlist` (extension allow-list), `pginsights` (per-query statistics for Query Insights), `plpgsql`.

## Built-in Postgres extensions

No profile toggle needed; install per database: `bloom`, `btree_gin`, `btree_gist`, `citext`, `cube`, `fuzzystrmatch`, `hstore`, `insert_username`, `intarray`, `ltree`, `moddatetime`, `pg_trgm`, `pgcrypto`, `tcn`, `tsm_system_rows`, `tsm_system_time`, `unaccent`, `uuid-ossp`.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SELECT name, installed_version FROM pg_available_extensions
WHERE installed_version IS NOT NULL ORDER BY name;
```

## Community extensions

- **`vector` (pgvector)** — enable on the profile, then `CREATE EXTENSION vector`. Works on unsharded and sharded Neki, with query-shape limits.
- **`vectorscale`** — requires `vector`; enable on the profile to expose DiskANN query parameters (`diskann.query_search_list_size`, `diskann.query_rescore`).

### pgvector on sharded tables

These shapes work on a sharded table:

- `ORDER BY` the distance expression, or project it once and order by the alias. The same is true for cosine (`<=>`), inner product (`<#>`), and L1 (`<+>`) on `vector`, `halfvec`, and `sparsevec`, and for Hamming (`<~>`) and Jaccard (`<%>`) on `bit`:
  ```sql
  SELECT tenant_id, item_id, embedding <-> '[1,0,0]'::vector AS distance
  FROM public.vector_items ORDER BY distance LIMIT 20;
  ```
- Vector literals (`'[1,0,0]'::vector`) and `ARRAY[1,0,0]::vector`. Cast the literal when the distance is in the select list: an uncast `embedding <-> '[1,0,0]'` there fails with `operator does not exist: vector <-> text` for every distance operator, while the uncast form resolves in `ORDER BY` and `WHERE`.
- `avg(embedding)` and `sum(embedding)` across shards. The router scatters and combines the aggregate.
- `INSERT ... SELECT` of vector values, including a statement that reads more than one shard.
- A prepared parameter declared as `vector`.

`l2_norm(embedding)` is ambiguous (`42725`) even when the column type is `vector`. Raise `ivfflat.probes` or `hnsw.ef_search` for better recall. Neki may not implement every function, cast, or cross-shard shape Postgres accepts. If a shape is rejected, the error carries an `NK013` code.

## Preview and cleanup notes

- During Platform Preview, don't use `ALTER EXTENSION ... ADD` to attach your own objects to an extension (a rebuilt shard may not recreate them) — contact Support first.
- Disabling a profile extension does **not** run `DROP EXTENSION` or remove objects; remove dependent objects first. `DROP EXTENSION` can remove dependent objects — review them before running it.
- When importing, inventory source extensions (`SELECT extname, extversion FROM pg_extension`) and compare them with the target profile's catalog before `pg_restore`.
