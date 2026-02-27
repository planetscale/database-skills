---
title: SQLite Data Modeling
description: Practical schema design patterns for maintainable SQLite databases
tags: sqlite, schema, data-types, foreign-keys, strict-tables
---

# SQLite Data Modeling

## Prefer predictable primary keys

```sql
CREATE TABLE account (
  id INTEGER PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL
) STRICT;
```

- `INTEGER PRIMARY KEY` aliases the rowid and is the most efficient lookup path.
- Avoid random-text primary keys for hot write paths; keep external IDs in a separate indexed column.

## Use STRICT tables when available

`STRICT` (SQLite 3.37+) enforces type constraints more like server databases.

```sql
CREATE TABLE event (
  id INTEGER PRIMARY KEY,
  kind TEXT NOT NULL,
  payload TEXT NOT NULL,
  happened_at TEXT NOT NULL
) STRICT;
```

## Foreign keys are disabled by default

Enable on every connection before statements that rely on referential integrity:

```sql
PRAGMA foreign_keys = ON;
```

Validate after migrations:

```sql
PRAGMA foreign_key_check;
```

## Represent booleans and timestamps intentionally

- Booleans: store as `INTEGER` (`0`/`1`) with `CHECK (flag IN (0,1))` when needed.
- Timestamps: store as UTC ISO-8601 `TEXT` consistently, or Unix epoch `INTEGER` consistently.

## Add constraints early

- Use `NOT NULL`, `CHECK`, and `UNIQUE` constraints to prevent bad data entering the file.
- Add app-level validation too; SQLite constraints are the final safety net, not the first.
