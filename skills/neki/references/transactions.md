---
title: Neki Transactions and Consistency
description: Single-shard vs cross-shard transactions, snapshots, and routing modes
tags: neki, transactions, cross-shard, snapshot, tx-mode, advisory-locks
---

# Neki Transactions

Docs: https://planetscale.com/docs/neki/query-planning (Transactions across shards) · https://planetscale.com/docs/neki/platform-preview-limitations

Routing one statement to several shards does **not** make those shards one transactional database. The key distinction:

- **Single-shard transaction** — all statements touch one shard (same shard-key value). This is an ordinary Postgres transaction: full ACID, Postgres isolation levels, low latency.
- **Cross-shard transaction** — statements span shards. Neki commits each shard separately; there is **no** shared snapshot and **no** atomic commit.

## What cross-shard work does not provide

- **Reads don't share a snapshot** — a multi-shard read doesn't establish one Postgres snapshot across shards, so concurrent writes can make different shards reflect different points in time within one result.
- **Writes aren't atomic** — Neki commits each shard independently (no two-phase commit). If one shard fails after another commits, the statement can leave different outcomes per shard.

**Distributed atomic cross-shard transactions are not supported during Platform Preview.** `SET __neki.tx_mode = 'atomic'` is rejected (error code 26, "Atomic transaction mode is unavailable").

## Enforce a one-shard transaction

Default transaction routing is `multi` (a transaction may reach more than one shard, without atomic commit). Set `single` to make Neki reject an operation that would pull a second shard into the transaction:

```sql
SET __neki.tx_mode = 'single';
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE tenant_id = $1 AND id = $2;
  INSERT INTO ledger (tenant_id, account_id, delta) VALUES ($1, $2, -100);
COMMIT;
```

`__neki.tx_mode` (like `__neki.target` and `__neki.fanout`) cannot change inside a transaction — set it beforehand. Check the spelling: a wrong name such as `__neki.transaction_mode` is accepted silently as a custom parameter and has no effect. Confirm with `SHOW __neki.tx_mode;` — it should return `single`. To stop individual statements from fanning out, use `__neki.fanout` (see [query-serving.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/query-serving.md)).

## Advisory locks

Session advisory locks can't be held on multiple shards from one session (error code 60), and mixing session and transaction advisory locks is unavailable (error code 61). Keep advisory locking scoped to one shard key.

## Design guidance

- Model each business transaction to a single shard-key value. Include the shard key predicate in every query it runs against a sharded table (see [sharding-readiness.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/sharding-readiness.md)).
- Verify the plan with `EXPLAIN (NEKI_PLAN)` before relying on multi-statement behavior; use `__neki.tx_mode = 'single'` and `__neki.fanout = 'single'` in development and CI.
- For unavoidable cross-shard workflows, use application-level patterns (idempotent operations, sagas, outbox) rather than expecting distributed atomicity.
- Isolation is local to each shard; don't expect a global serializable snapshot.
- Replica reads don't guarantee read-your-writes or monotonic reads — send read-after-write paths to the primary (see [replication.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/replication.md)).
