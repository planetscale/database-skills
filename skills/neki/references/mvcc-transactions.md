---
title: Neki MVCC Transactions and Concurrency (per shard)
description: Postgres isolation levels, XID wraparound, and serialization on each shard
tags: neki, mvcc, isolation, xid-wraparound, concurrency, serialization
---

# MVCC Transactions and Concurrency in Neki

Docs: https://planetscale.com/docs/neki/query-planning (Transactions across shards)

This covers **per-shard** Postgres transaction behavior. Cross-shard routing, snapshots, and atomicity are in [transactions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/transactions.md) — Neki does **not** provide a shared snapshot or atomic commit across shards, so the guarantees below apply within a single shard.

## Isolation levels (within a shard)

- **READ COMMITTED** (default): a new snapshot per statement.
- **REPEATABLE READ**: snapshot at the first query; write conflicts raise serialization errors.
- **SERIALIZABLE**: strongest; requires application retry logic.

Readers don't block writers and writers don't block readers (only writer–writer conflicts on the same row). Isolation is local to each shard; there is no global serializable snapshot across shards. If the router rejects a transaction command, isolation level, or transaction option, it returns `NK013` with code 23, 24, or 25 (for example `PREPARE TRANSACTION`, code 23); `SET TRANSACTION SNAPSHOT` is code 29 — see [error-codes.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/error-codes.md).

## XID wraparound (per shard)

32-bit XIDs wrap at about 2 billion. `VACUUM FREEZE` replaces old XIDs so old rows stay visible; without it, wraparound makes rows appear to be "in the future" and become invisible. Each shard is independent — monitor and protect **every** shard:

```sql
SELECT datname, age(datfrozenxid),
       ROUND(100.0 * age(datfrozenxid) / 2147483648, 2) AS pct
FROM pg_database ORDER BY age(datfrozenxid) DESC;
```

Never disable autovacuum (see [mvcc-vacuum.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/mvcc-vacuum.md)).

## Long transactions

A single long-running transaction blocks dead-tuple cleanup on its shard, causing bloat and slower queries. Sessions left idle in a transaction are the most common culprit. Routers enforce `idle-in-transaction-session-timeout` (default 5 minutes); still keep transactions short and single-shard.

## Serialization errors

Applications must handle "could not serialize access" with retry logic (more common under REPEATABLE READ and SERIALIZABLE). Smaller, faster, single-shard transactions reduce conflicts.
