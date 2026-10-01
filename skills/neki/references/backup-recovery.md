---
title: Neki Backup and Recovery
description: Scheduled and manual backups, restore to a new branch, and PITR
tags: neki, backups, restore, pitr, branches, recovery
---

# Backup and Recovery in Neki

Docs: https://planetscale.com/docs/neki/backups

A Neki backup covers the schema and data on **every managed shard** in a branch. Shards back up in parallel, and the overall backup succeeds only when every shard succeeds. Backups capture each shard's **primary** (replicas aren't backed up separately). Restoring **creates a new branch** — it never overwrites the source.

## Automatic and manual backups

- **Automatic (required)**: every 12 hours, retained for 2 days; can't be modified or removed; covered by the included allowance.
- **Manual**: one at a time per branch, with a retention you choose (hours to years). An **Emergency backup** option exists for critical situations (it may affect performance).
- **Custom schedules**: hourly, daily, weekly, or monthly per branch type, with retention; required schedules can't be changed.

## Restore to a new branch

From Backups, pick a successful backup → **Restore to new branch**. Choose a cluster size and replica count per configuration profile (same CPU architecture as the source; all development sizes produce a development branch, all production sizes a production branch — no mixing) and a router size and replicas per availability zone for each router group. The new branch is billed independently.

## Point-in-time recovery (PITR)

PITR restores a new branch to a chosen time by starting from an eligible backup taken at or before that time and replaying each shard's WAL to that point. The window runs from the oldest eligible backup to about 5 minutes ago.

```bash
pscale branch create <DB> <NEW_BRANCH> --from <SOURCE_BRANCH> \
  --restore-point 2026-09-09T18:00:00Z
```

Adding a shard can create a gap in the PITR window: a restore point after the shard was added is unavailable until a later successful backup includes every shard in the new set.

## Retention and protection

Backups expire according to their retention. Enable **Prevent backup deletion** to block both automatic and manual deletion; required backups and backups used by an active restore can't be deleted. Included backup storage equals twice the branch's allocated disk; usage above that, including longer retention, is billed per GB-month.

## Operational guidance

- Periodically **restore into a disposable branch** — a successful backup proves data was captured; a restore test proves recovery works.
- Backups also provision and repair replicas (a new or unrecoverable replica is restored from the last backup and then rejoined; see [replication.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/replication.md)).
