---
title: Neki Schema Design
description: Schema design for Neki (sharding-aware)
tags: neki, schema, primary-keys, data-types, foreign-keys, sharding
---

# Schema Design in Neki

Docs: https://planetscale.com/docs/neki/best-practices

Each Neki shard is real Postgres, so ordinary Postgres schema design applies — with the shard key shaping every sharded table. See [sharding-readiness.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/sharding-readiness.md) for the readiness checklist and [sharding-model.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/sharding-model.md) for topology.

## Primary keys

- A single-column primary key is fine when it is the shard key (for example `user_id` on `users`).
- For child tables, lead a composite primary key with the shard key so lookups stay shard-local.
- Avoid random UUIDv4 as a hot primary key (fragmentation, 16 bytes); use `uuidv7()` when you need a UUID. See [id-generation.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/id-generation.md).

```sql
CREATE TABLE users (user_id BIGINT PRIMARY KEY, email TEXT NOT NULL);

CREATE TABLE orders (
  tenant_id BIGINT NOT NULL,
  id BIGINT GENERATED ALWAYS AS IDENTITY,
  status TEXT NOT NULL CHECK (status IN ('pending','shipped','delivered')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (tenant_id, id)
);
```

## Data types

| Use | Avoid |
| --- | --- |
| `TEXT`, `VARCHAR` | `json`, `jsonb`, or arrays for any column that may become a shard key |
| `JSONB` | Custom `ENUM` (prefer `CHECK` — easier to change) |
| `TIMESTAMPTZ` | `TIMESTAMP` without time zone |
| `BIGINT`, `INTEGER`, `UUID` | Platform-specific types |

Use the **same shard-key type** across co-located tables — routing and joins depend on it.

## Foreign keys

- Always index FK columns (Postgres does not create these automatically).
- Keep FK-related tables in the **same shard group** so references stay shard-local; cross-shard references need application-level enforcement.

```sql
CREATE TABLE order_items (
  tenant_id BIGINT NOT NULL,
  id BIGINT GENERATED ALWAYS AS IDENTITY,
  order_id BIGINT NOT NULL,
  PRIMARY KEY (tenant_id, id),
  FOREIGN KEY (tenant_id, order_id) REFERENCES orders (tenant_id, id)
);
CREATE INDEX order_items_tenant_order_idx ON order_items (tenant_id, order_id);
```

## Uniqueness

Scope unique constraints to include the shard key so uniqueness holds globally; a plain `UNIQUE(email)` only holds within a shard. For global uniqueness on a non-shard-key column, use a unique GSI (see [indexing.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/indexing.md)).

```sql
ALTER TABLE orders ADD CONSTRAINT uq_order_number UNIQUE (tenant_id, order_number);
```

## Unsupported objects during Platform Preview

The router rejects objects tied to one host's storage or code: tablespaces, large objects, procedural languages with custom handlers, `LOAD`, user-defined text search configurations and dictionaries, and temporary tables, views, and sequences. See [error-codes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/error-codes.md).

## General guidelines

- Tables and columns: singular `snake_case`; indexes `{table}_{column}_idx`.
- Add `NOT NULL` widely; add `created_at TIMESTAMPTZ DEFAULT NOW()` to tables.
- Propagate the shard key onto every tenant-scoped table, even when it feels redundant — it keeps joins and transactions shard-local.
- Declare small shared lookup tables as reference tables rather than sharding them.
- Apply schema changes through Neki's managed workflows (see [schema-changes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/schema-changes.md)).
