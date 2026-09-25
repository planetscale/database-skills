---
title: Neki Partitioning vs Sharding
description: Per-shard Postgres partitioning and how it relates to sharding
tags: neki, partitioning, sharding, range, data-retention
---

# Partitioning in Neki

Docs: https://planetscale.com/docs/neki/when-to-shard (Partitioning vs sharding)

**Sharding and partitioning are different and complementary:**

- **Sharding** distributes a logical table's rows across independent Postgres shards (each with its own primary, storage, and replication). It removes the single-primary write and size bottleneck.
- **Partitioning** splits one table into physical partitions **within a single Postgres instance** (one shard). It helps maintenance (vacuum, index builds) and data pruning and retention, but doesn't remove a single-primary bottleneck or let data exceed one instance.

A common pattern: shard by `tenant_id`, and within each shard, range-partition a large time-series table by `created_at` for retention.

## When to partition (within a shard)

| Table type | Size threshold | Row threshold |
| --- | --- | --- |
| General tables | >100 GB (or larger than RAM) | >20M rows |
| Time-series and logs | >50 GB | >10M rows |

## Range partitioning

The partition key must be part of the primary key; on sharded tables the primary key already leads with the shard key:

```sql
CREATE TABLE events (
  tenant_id BIGINT NOT NULL,
  id BIGINT GENERATED ALWAYS AS IDENTITY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  payload JSONB,
  PRIMARY KEY (tenant_id, id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_01 PARTITION OF events
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

DDL sent through a router runs on every managed shard, so each shard gets the same partitions. Use managed workflows where you need coordination (see [schema-changes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/schema-changes.md)).

## Management

- `pg_partman` isn't in Neki's extension catalog; create future partitions with scheduled DDL instead (see [extensions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/extensions.md)).
- Drop old partitions for retention instead of `DELETE` (avoids bloat).
- Use `DETACH PARTITION ... CONCURRENTLY` to avoid `ACCESS EXCLUSIVE` locks on the parent.
- Create future partitions ahead of time to avoid insert failures.
- **Always confirm with a human before detaching or dropping partitions** — both are destructive.

## Guidelines and limitations

- Partition key columns must be in the `PRIMARY KEY` and any `UNIQUE` constraints.
- Postgres partitioning doesn't support global unique constraints on non-partition columns.
- Filter by the partition key to enable partition pruning, and keep the shard key present so the query still routes to one shard.
- Neki's shard indexes aren't Postgres range partitions — a key range maps part of a shard group's keyspace to a shard.
