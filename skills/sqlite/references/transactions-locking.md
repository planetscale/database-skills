---
title: SQLite Transactions and Locking
description: Contention patterns and reliable transaction usage in SQLite
tags: sqlite, transactions, locking, concurrency, busy-timeout
---

# SQLite Transactions and Locking

## Transaction modes

```sql
BEGIN;            -- deferred (default)
BEGIN IMMEDIATE;  -- reserve write lock up front
BEGIN EXCLUSIVE;  -- strongest lock mode
```

- `BEGIN` (deferred) delays locking until first write.
- `BEGIN IMMEDIATE` is usually best for write paths; it surfaces contention early.
- Keep write transactions short to reduce `database is locked` errors.

## Busy handling

Set timeout per connection and add bounded app retries.

```sql
PRAGMA busy_timeout = 5000;
```

- Do not retry forever; use jitter and a max attempt count.
- Log lock wait time to identify saturation.
- Some lock-upgrade conflicts fail fast with `SQLITE_BUSY` to avoid deadlock; design retry logic accordingly.

## WAL-aware behavior

- In WAL mode, readers do not block writers, but one writer still serializes writes.
- Long-lived readers can delay checkpoints and grow WAL files.

## Savepoints for partial rollback

```sql
BEGIN IMMEDIATE;
SAVEPOINT step1;
-- changes
ROLLBACK TO step1; -- optional partial rollback
RELEASE step1;
COMMIT;
```

Use savepoints to isolate chunks inside larger business transactions.
