---
title: Neki Error Codes
description: How to read NK013 "not implemented" errors and what the catalog codes mean
tags: neki, errors, nk013, not-implemented, limitations
---

# Neki Error Codes

Docs: https://planetscale.com/docs/neki/error-codes · https://planetscale.com/docs/neki/platform-preview-limitations

When the router can't plan or execute a statement because an implementation is still missing, it returns SQLSTATE **`NK013`** with a message like:

```
not implemented: [108] This subquery shape is unavailable in ORDER BY.
```

The bracketed number is a **stable catalog code**: it identifies one specific gap and is never reused, even after the gap closes. Quote it when reporting a rejected query, and match rejections in your application against it.

- `NK013` means Neki intends to close the gap.
- A limit that is a design decision is reported as **not supported** instead (see the Platform Preview limitations page).
- One statement can hit several gaps; you see the first one the planner reaches, so fixing it may reveal another.
- Some rejections depend on the data topology, not the SQL text — the same statement can plan under a different topology.

## Catalog by area (representative codes)

| Area | Examples |
| --- | --- |
| Statements, transactions, sessions | `1` `SELECT ... INTO` (use `CREATE TABLE AS`); `21` inheritance parent without `ONLY`; `22` `AND CHAIN`; `23`–`25` unsupported transaction command, isolation level, or option (for example `PREPARE TRANSACTION`); `26` atomic transaction mode; `27`–`29` unsupported `SET` variants, `SET FROM CURRENT`, `SET TRANSACTION SNAPSHOT`; `30`–`32` unsupported `EXPLAIN (NEKI_PLAN)` shapes and options |
| Topology and evaluation context | `2` replication from a replica; `10` a `columns` list with more than one entry, rejected when the topology is applied, and an expression over more than one column, which is stored; `INSERT ... VALUES` into that table returns this code, and `INSERT ... SELECT` returns code `117`; `115` equality on every base column of an expression that names more than one column; client `COPY TO STDOUT` works for sharded tables, reference tables, and materialized views, including binary; `41` and `43` file `COPY`; `44`–`55` other `COPY` limits (for example `COPY TO PROGRAM`, `COPY` from a query, `COPY FROM` into a table with an active GSI, `COPY` in the extended protocol or a multi-statement query) |
| Advisory locks | `60` session advisory locks on multiple shards; `61` mixing session and transaction advisory locks; `62`, `139` unavailable evaluation context or plan position |
| PL/pgSQL (router-side execution) | `80`–`95` unsupported targets, statements, and `RAISE`/`RETURN` forms |
| Query shapes and routing | `63` join types; `100`–`153` subquery, CTE, window-function, `INSERT ... SELECT`, `ON CONFLICT`, `MERGE`, view, set-operation, and GSI routing shapes; `138` `FETCH FIRST ... WITH TIES` across shards; `145` updating an index column |
| Evaluation engine | `200`–`216` built-in functions, type input functions, casts, and user-defined functions, operators, and aggregates the router can't evaluate |
| DDL | `300`–`338` operator classes and families, custom text search, event triggers, rules, large objects, custom languages, tablespaces, access methods, collations, foreign data wrappers and tables, `LOAD`, `ALTER DOMAIN`, column storage and compression; `338` is temporary tables, temporary views, and temporary sequences only — ordinary views and sequences are allowed |

## What to do

1. Read the bracketed code and look it up on the error-codes page; each entry shows a triggering example and what to change.
2. Rewrite to a supported shape — for example `CREATE TABLE AS` instead of `SELECT ... INTO`, keyset pagination instead of `WITH TIES`, the pgvector alias pattern for cross-shard nearest-neighbor search (see [extensions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/extensions.md)).
3. If the statement only touches one shard's data, add the shard key so it routes to a single shard — many gaps apply only to shapes the router must combine across shards.
4. For a gap you can't work around, report it to PlanetScale Support with the code and statement.

Related: [query-patterns.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/query-patterns.md) (Platform Preview query-shape limits) and [schema-design.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/schema-design.md) (unsupported objects).
