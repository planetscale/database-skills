---
title: Neki ID Generation
description: Globally unique identifiers and sequences across shards
tags: neki, ids, sequences, uuid, identity, sharding
---

# ID Generation in Neki

Docs: https://planetscale.com/docs/neki/data-topology (Sequence placement)

On a single Postgres instance, `BIGINT GENERATED ALWAYS AS IDENTITY` or a `SEQUENCE` gives unique IDs. Across shards this needs care: a value must be unique everywhere it can appear.

## How sequences work in Neki

Each Postgres **sequence resolves to a shard group that contains exactly one shard**. A sequence owned only by unsharded tables follows them when they resolve to the same single-shard group (including the ownership link created by `SERIAL` and identity columns). Otherwise it falls back to the database's **authoritative shard group**, or to an explicit `shard_group` set on the sequence in the data topology.

Routers reserve a batch of values from the sequence shard's primary and hand them out locally; a larger `CACHE` on the sequence reduces trips back to that shard. Adding or removing routers doesn't move the sequence. Changing a sequence's placement does **not** copy its current value — coordinate placement changes with PlanetScale Support.

Because the counter lives on one shard, a plain sequence can become a bottleneck for high-rate inserts spread across many shards.

## Options for globally unique IDs

| Approach | Avoids a single hot shard? | Notes |
| --- | --- | --- |
| **UUIDv7** (`uuidv7()`) | Yes | Time-ordered, index-friendly; preferred for globally unique surrogate keys |
| **UUIDv4** (`gen_random_uuid()`) | Yes | Random; fragments indexes, 16 bytes; avoid as a hot primary key |
| **Application-generated (Snowflake-style)** | Yes | 64-bit time + node + counter; compact and sortable; needs a node-ID scheme |
| **Composite key (shard key + identity column)** | No — the identity column still draws from the sequence's shard | Keeps a tenant's rows together and lookups shard-local; keep the shard key leading |
| **Plain sequence or identity column** | No | Fine for unsharded tables and moderate insert rates |

## Recommended patterns

```sql
-- globally unique, index-friendly primary key
CREATE TABLE events (
  id UUID DEFAULT uuidv7() PRIMARY KEY,
  tenant_id BIGINT NOT NULL
);

-- child table with low to moderate write traffic: shard key leads, identity column secondary
CREATE TABLE order_items (
  tenant_id BIGINT NOT NULL,
  id BIGINT GENERATED ALWAYS AS IDENTITY,
  PRIMARY KEY (tenant_id, id)
);

-- child table with high write traffic: shard key leads, UUID column secondary, avoids a hot sequence
CREATE TABLE order_items (
  tenant_id BIGINT NOT NULL,
  id UUID DEFAULT uuidv7(),
  PRIMARY KEY (tenant_id, id)
);
```

## Guidelines

- Prefer `uuidv7()` (or application-generated IDs) for globally unique surrogate keys on sharded tables.
- Keep the shard key leading in composite primary keys. Raise the sequence's `CACHE` if many routers allocate from it at a high rate.
- When `COPY FROM` omits an identity column or a sequence-backed default, supply those values explicitly.
- Data-migration cutovers synchronize owned sequences to the maximum value on the target before switching writes (see [resharding-migration.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/resharding-migration.md)).
