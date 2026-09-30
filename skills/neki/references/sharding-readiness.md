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

A tenant key (`tenant_id`, `org_id`, `account_id`, `customer_id`) commonly keeps a tenant's rows together. Which types a shard index accepts is in [sharding-model.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/sharding-model.md).

Keys, uniqueness, and foreign keys are in [schema-design.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/schema-design.md). Globally unique IDs are in [id-generation.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/id-generation.md). Reference tables and GSIs are in [indexing.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/indexing.md). Single-shard transactions are in [transactions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/transactions.md).

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
