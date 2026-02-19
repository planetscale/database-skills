---
title: TTL (Time to Live)
description: ClickHouse row-level TTL, column TTL, TTL MOVE for tiered storage, and TTL DELETE patterns
tags: clickhouse, ttl, data-retention, tiered-storage, expiry
---

# TTL (Time to Live)

ClickHouse TTL allows automatic expiry or movement of data based on time expressions. TTL is evaluated during merges — data is not removed instantly when it expires but on the next merge of the affected part.

## Row-Level TTL (DELETE)

Deletes entire rows after a specified time:

```sql
CREATE TABLE events (
    event_time  DateTime,
    user_id     UInt64,
    payload     String
)
ENGINE = MergeTree()
ORDER BY (user_id, event_time)
TTL event_time + INTERVAL 90 DAY;
```

- Rows where `event_time + 90 days < now()` are removed during the next merge of that part.
- TTL expression must reference a `Date` or `DateTime` column.
- Use `ALTER TABLE` to add or modify TTL on an existing table:

```sql
ALTER TABLE events MODIFY TTL event_time + INTERVAL 90 DAY;
```

## Column-Level TTL

Resets specific columns to their default value after a time period, keeping the row:

```sql
CREATE TABLE user_profiles (
    user_id         UInt64,
    last_login      DateTime,
    raw_user_agent  String TTL last_login + INTERVAL 30 DAY,
    ip_address      String TTL last_login + INTERVAL 30 DAY
)
ENGINE = MergeTree()
ORDER BY user_id;
```

- After `last_login + 30 days`, `raw_user_agent` and `ip_address` are set to `''` (empty string default).
- Useful for GDPR/privacy compliance: anonymize PII after a retention window without deleting the row.
- Column TTL fires independently of row TTL.

## TTL MOVE — Tiered Storage

Move data to a different storage volume as it ages:

```sql
CREATE TABLE events (
    event_time DateTime,
    ...
)
ENGINE = MergeTree()
ORDER BY (event_time)
TTL
    event_time + INTERVAL 7 DAY TO VOLUME 'warm',
    event_time + INTERVAL 90 DAY TO VOLUME 'cold';
```

- Requires multiple storage volumes configured in `storage_policy` in `config.xml` or via `SETTINGS`.
- Parts are physically moved between disk volumes (e.g., SSD → HDD → object storage).
- `TO DISK 'disk_name'` moves to a specific disk; `TO VOLUME 'volume_name'` moves to any disk in a volume.

Storage policy — local disks:
```xml
<storage_configuration>
    <disks>
        <hot_ssd><path>/var/lib/clickhouse/</path></hot_ssd>
        <warm_hdd><path>/mnt/hdd/clickhouse/</path></warm_hdd>
    </disks>
    <policies>
        <tiered>
            <volumes>
                <hot><disk>hot_ssd</disk><max_data_part_size_bytes>1073741824</max_data_part_size_bytes></hot>
                <warm><disk>warm_hdd</disk></warm>
            </volumes>
        </tiered>
    </policies>
</storage_configuration>
```

Storage policy — S3 cold tier (common in cloud deployments):
```xml
<storage_configuration>
    <disks>
        <hot_local>
            <type>local</type>
            <path>/var/lib/clickhouse/</path>
        </hot_local>
        <s3_cold>
            <type>s3</type>
            <endpoint>https://s3.us-east-1.amazonaws.com/my-bucket/clickhouse/</endpoint>
            <access_key_id>YOUR_KEY</access_key_id>
            <secret_access_key>YOUR_SECRET</secret_access_key>
        </s3_cold>
    </disks>
    <policies>
        <hot_to_s3>
            <volumes>
                <hot><disk>hot_local</disk></hot>
                <cold><disk>s3_cold</disk></cold>
            </volumes>
        </hot_to_s3>
    </policies>
</storage_configuration>
```

Then apply the policy on the table:
```sql
CREATE TABLE events (...)
ENGINE = MergeTree()
ORDER BY (event_time)
SETTINGS storage_policy = 'hot_to_s3'
TTL event_time + INTERVAL 30 DAY TO VOLUME 'cold';
```

S3-compatible stores (MinIO, Cloudflare R2, GCS) use the same `type=s3` disk type with the appropriate endpoint URL.

## TTL Evaluation and Triggers

TTL is applied **lazily during merges** — not on a schedule. To force immediate TTL application:

```sql
-- Force TTL evaluation on all parts now
ALTER TABLE events MATERIALIZE TTL;

-- Or optimize to trigger merges (not recommended in production regularly)
OPTIMIZE TABLE events FINAL;
```

**Do not rely on TTL for exact-time deletion** — data may persist beyond the TTL period until a merge occurs. For strict retention (e.g., compliance), combine TTL with scheduled `DROP PARTITION` for coarse-grained time ranges.

## Multiple TTL Expressions

A table can have multiple TTL rules with different actions:

```sql
TTL
    event_time + INTERVAL 7 DAY TO VOLUME 'warm',
    event_time + INTERVAL 30 DAY TO VOLUME 'cold',
    event_time + INTERVAL 365 DAY DELETE;
```

Rules are evaluated in order:
1. After 7 days: move to `warm` volume.
2. After 30 days: move to `cold` volume.
3. After 365 days: delete rows.

## Monitoring TTL

```sql
-- Check TTL settings on a table
SHOW CREATE TABLE events;

-- Find parts with expired TTL not yet processed
SELECT name, min_time, max_time, rows, active
FROM system.parts
WHERE table = 'events'
  AND max_time < now() - INTERVAL 90 DAY
  AND active = 1;

-- Monitor TTL merge activity
SELECT * FROM system.merges WHERE is_ttl_delete_merge = 1;
```

## Common Pitfalls

- **TTL doesn't fire on empty tables or tables with no merges**: trigger `OPTIMIZE` or wait for enough inserts to cause merges.
- **Row TTL uses the part's min time, not per-row granularity for part selection**: a part with even one non-expired row won't have rows removed until a merge.
- **Column TTL and row TTL use separate clocks**: adding both to the same table is valid but apply independently.
- **Altering TTL doesn't retroactively apply**: use `MATERIALIZE TTL` after changing TTL expressions on existing data.
