---
title: Neki Sharding Readiness and Best Practices
description: Schema and query design that keeps a Postgres database shard-ready on Neki
tags: neki, sharding, schema-design, shard-key, best-practices, readiness
---

# Sharding Readiness and Best Practices

Docs: https://planetscale.com/docs/neki/best-practices · https://planetscale.com/docs/neki/when-to-shard

Neki runs unsharded or sharded, and you can start unsharded and shard later. Designing for sharding up front makes that transition cheap. This guide covers choosing a shard key and shaping schema and queries so most work stays on one shard. For when to shard and how to size the cluster, see [scaling-and-capacity.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/scaling-and-capacity.md). For topology mechanics (shard groups, shard indexes, key ranges), see [sharding-model.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/sharding-model.md).

## Choosing a shard key

Pick a shard key from the queries and transactions you rely on. Prefer a key that:

- Distributes data and writes evenly (avoids hot shards).
- Appears in the predicates of latency-sensitive queries (so they route to one shard).
- Keeps rows that are joined or updated together in the same shard group.
- Stays stable for a row's lifetime (changing it relocates the row — a resharding operation).

A tenant key (`tenant_id`, `org_id`, `account_id`, `customer_id`) commonly keeps a tenant's rows together. The `xxhash` shard index accepts `text`, `varchar`, `bytea`, integer, float, `numeric`, date/time, and `uuid` columns (not `json`, `jsonb`, or arrays).

## Primary keys and IDs

- A single-column primary key is fine when it is the shard key. For child tables, lead a composite primary key with the shard key so lookups stay shard-local.
- For globally unique surrogate keys on sharded tables use `uuidv7()` or application-generated IDs. See [id-generation.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/id-generation.md).

## Co-location, uniqueness, foreign keys

- **Co-locate** frequently joined tables by binding them to the same shard group and routing them through the same shard index; include the shard key in join predicates.
- **Uniqueness**: scope unique constraints to include the shard key so they hold globally; a plain `UNIQUE(email)` only holds within a shard.
- **Foreign keys**: keep FK-related tables in the same shard group so references stay shard-local; cross-shard references need application-level enforcement.
- Use the same shard-key column type across co-located tables.

## Access paths the shard key can't serve

- **Reference tables** duplicate small shared data on every shard in a group so joins stay local.
- **GSIs** map another key to the owner row's shard key via a lookup table (best for selective lookups).

Both add write cost and must be populated and verified before use. See [indexing.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/indexing.md).

## Transactions and scatter

- Keep transactions on one shard-key value; cross-shard transactions have no shared snapshot or atomic commit (see [transactions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/transactions.md)).
- Watch for scatter queries with `EXPLAIN (NEKI_PLAN)` and Query Insights; add a routing predicate or revisit the topology.

## Readiness checklist

1. Shard key chosen from real query and transaction predicates; present on every tenant-scoped table.
2. Composite primary keys lead with the shard key; globally unique IDs use `uuidv7()` or application-generated IDs.
3. Shard key in hot query `WHERE` clauses, indexes (leading), and join conditions.
4. Unique constraints scoped to include the shard key.
5. FK-related tables co-located in the same shard group; cross-shard references planned for application-level enforcement.
6. Reference tables and GSIs identified for non-shard-key access paths.
7. Transactions scoped to one shard-key value.
8. Global aggregations identified; rollup or async plan in place.
9. Authoritative shard group left as a standalone shard, sized for catalog work, sequences, and unsharded tables.
