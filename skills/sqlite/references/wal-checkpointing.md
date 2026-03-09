---
title: SQLite WAL and Checkpointing
description: WAL mode setup, checkpoint strategy, and durability trade-offs
tags: sqlite, wal, checkpoint, durability, operations
---

# SQLite WAL and Checkpointing

## Recommended baseline

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
```

- `WAL` improves read/write concurrency for most app workloads.
- `NORMAL` is a common balance; use `FULL` when durability requirements are strict.

## Checkpoint behavior

Checkpoint moves WAL pages back into the main database file.

```sql
PRAGMA wal_checkpoint(PASSIVE);
PRAGMA wal_checkpoint(TRUNCATE);
```

- `PASSIVE` avoids blocking writers.
- `TRUNCATE` aggressively shrinks the WAL file when safe.

## Autocheckpoint tuning

```sql
PRAGMA wal_autocheckpoint = 1000;
```

- Default threshold is 1000 pages; tune using real write volume and latency goals.
- Too frequent checkpoints increase write amplification; too infrequent can bloat WAL.

## Checkpoint mode selection

- `PASSIVE`: best default for background maintenance; avoids blocking writers.
- `FULL`: waits for readers and writers as needed to complete more checkpoint work.
- `RESTART`/`TRUNCATE`: use for maintenance windows when you want WAL reset/shrink behavior.

## Operational signals

- WAL file grows without shrinking -> likely long readers or checkpoint starvation.
- Frequent sync stalls -> revisit transaction size, checkpoint cadence, and storage latency.
