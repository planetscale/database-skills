---
title: SQLite Migrations and Operations
description: Safe migration patterns, backups, and integrity verification
tags: sqlite, migrations, ddl, backup, integrity-check
---

# SQLite Migrations and Operations

## Pre-migration checklist

- Capture a backup (`.backup` in sqlite3 CLI or file-level snapshot when safe).
- Record SQLite version (`SELECT sqlite_version();`) to validate feature support.
- Confirm foreign keys are enabled during migration sessions.

## Additive-first migration pattern

1. Add new nullable columns/tables/indexes.
2. Backfill in chunks.
3. Switch application reads/writes.
4. Enforce stricter constraints after backfill validation.

## Table rewrite pattern (create-copy-swap)

When direct `ALTER TABLE` support is limited:

```sql
BEGIN IMMEDIATE;
CREATE TABLE new_table (...);
INSERT INTO new_table (...) SELECT ... FROM old_table;
DROP TABLE old_table;
ALTER TABLE new_table RENAME TO old_table;
COMMIT;
```

Recreate indexes, triggers, and foreign keys explicitly as part of the migration.

## Post-migration validation

```sql
PRAGMA foreign_key_check;
PRAGMA integrity_check;
```

Also validate row counts and key business invariants before marking migration complete.
