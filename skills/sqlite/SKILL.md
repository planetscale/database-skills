---
name: sqlite
description: Plan and review SQLite schema, indexing, query plans, transactions, and operational settings. Use when building embedded/local databases, tuning mobile or edge workloads, planning migrations, or debugging lock/contention behavior.
---

# SQLite

Use this skill to make safe, measurable SQLite changes for production apps and local tooling.

## Workflow
1. Define environment and workload (SQLite version, app runtime, storage medium, read/write pattern, concurrency model).
2. Read only the relevant reference files linked in each section below.
3. Propose the smallest change that solves the issue, including trade-offs.
4. Validate with evidence (`EXPLAIN QUERY PLAN`, `ANALYZE`, timing, lock behavior, and rollback steps).
5. For production changes, include migration safety, checkpoint strategy, and post-deploy verification.

## Data Modeling
- Keep hot tables narrow; use `INTEGER PRIMARY KEY` when possible for rowid-backed lookups.
- Use `STRICT` tables (3.37+) for type safety in app-facing schemas.
- Enable and enforce foreign keys (`PRAGMA foreign_keys = ON`) on every connection.
- Store timestamps in one consistent format (usually UTC ISO-8601 text) and document it.

References:
- [data-modeling](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/sqlite/references/data-modeling.md)

## Indexing and Query Planning
- Build composite indexes in predicate order (equality columns before range/sort columns).
- Prefer covering indexes for high-frequency reads.
- Avoid functions on indexed columns in `WHERE` unless expression indexes are used.
- Re-run `ANALYZE` after major data distribution changes; use `PRAGMA optimize` periodically for low-touch maintenance.

References:
- [indexing-query-planner](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/sqlite/references/indexing-query-planner.md)
- [diagnostics](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/sqlite/references/diagnostics.md)

## Transactions and Concurrency
- Use explicit transactions for write batches; keep them short.
- Prefer `BEGIN IMMEDIATE` for write-heavy code paths to fail fast on lock contention.
- Configure a `busy_timeout` and implement bounded retries with jitter.
- Keep long-running reads from blocking checkpoints and writers.

References:
- [transactions-locking](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/sqlite/references/transactions-locking.md)
- [wal-checkpointing](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/sqlite/references/wal-checkpointing.md)

## Pragmas and Performance Tuning
- Prefer `journal_mode=WAL` for mixed read/write workloads.
- Tune `synchronous` to durability needs (`FULL` for max safety, `NORMAL` for many app workloads).
- Size cache and temp storage intentionally (`cache_size`, `temp_store`, `mmap_size`).
- Avoid blanket PRAGMA changes without benchmarking representative traffic.

References:
- [pragmas-tuning](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/sqlite/references/pragmas-tuning.md)

## Migrations and Operations
- Use additive migrations first; backfill before enforcing new constraints.
- For table rewrites, use create-copy-swap patterns with integrity checks.
- Always run `PRAGMA foreign_key_check` and `PRAGMA integrity_check` after risky schema changes.
- Add backup and restore steps before destructive migrations.

References:
- [migrations-operations](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/sqlite/references/migrations-operations.md)

## Guardrails
- Prefer evidence over assumptions; include measured before/after results.
- Call out version-specific behavior (for example, `STRICT` tables, `RETURNING`, `ALTER TABLE` support).
- Ask for explicit human approval before destructive data operations (drops/deletes/truncates).
