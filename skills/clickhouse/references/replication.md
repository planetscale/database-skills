---
title: Replication and Clustering
description: ClickHouse ReplicatedMergeTree with ZooKeeper/ClickHouse Keeper, Distributed table engine, and sharding key selection
tags: clickhouse, replication, distributed, sharding, keeper, zookeeper, cluster
---

# Replication and Clustering

ClickHouse supports both replication (HA for each shard) and sharding (horizontal scale-out). These are configured independently.

## ReplicatedMergeTree

Add `Replicated` prefix to any MergeTree engine to enable replication:

```sql
CREATE TABLE events ON CLUSTER my_cluster (
    event_time  DateTime,
    user_id     UInt64,
    event_type  LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);
```

- First argument: **ZooKeeper/Keeper path** — uniquely identifies this table's replication group.
- Second argument: **replica name** — unique per node within the replication group.
- `{shard}` and `{replica}` are macros configured per-node in `config.xml`.

## ZooKeeper vs ClickHouse Keeper

Both serve as the coordination backend for `ReplicatedMergeTree`:

| | ZooKeeper | ClickHouse Keeper |
|---|---|---|
| Deployment | External Java service | Embedded in ClickHouse |
| Protocol | ZooKeeper protocol | ZooKeeper-compatible |
| Recommended for | Legacy/existing setups | New installations |
| Operational overhead | Higher | Lower |

ClickHouse Keeper is the recommended choice for new deployments — simpler operations, no JVM dependency, compatible drop-in replacement.

```xml
<!-- clickhouse-keeper config -->
<keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>
    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>clickhouse-01</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>2</id>
            <hostname>clickhouse-02</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>3</id>
            <hostname>clickhouse-03</hostname>
            <port>9234</port>
        </server>
    </raft_configuration>
</keeper_server>
```

Deploy Keeper on an odd number of nodes (3 or 5) for quorum fault tolerance.

## Distributed Table Engine

The `Distributed` engine acts as a virtual proxy table that routes queries to underlying shards:

```sql
-- Underlying replicated table (exists on each shard)
ENGINE = ReplicatedMergeTree(...)

-- Distributed table (created once, queries all shards)
CREATE TABLE events_dist ON CLUSTER my_cluster
AS events
ENGINE = Distributed(
    'my_cluster',   -- cluster name from config
    'default',      -- target database
    'events',       -- target table (local MergeTree table)
    user_id         -- sharding key expression
);
```

Inserts into `events_dist` route rows to shards based on `murmur_hash(sharding_key) % num_shards`.

Queries on `events_dist` fan out to all shards, collect results, and merge.

## Sharding Key Selection

The sharding key determines which shard stores each row. Poor selection causes **data skew** (hot shards with much more data than others):

**Good sharding keys:**
- `rand()` — perfectly uniform distribution, but colocated rows are spread across shards (JOINs become cross-shard).
- `user_id` — good if users are uniformly distributed; colocates all of a user's data on one shard.
- `intHash64(user_id)` — hashed user_id for better distribution if user_ids are not uniform.
- `cityHash64(tenant_id)` — for multi-tenant systems where tenant data should be colocated.

**Bad sharding keys:**
- A column with very low cardinality (e.g., `status` with 3 values → max 3 evenly loaded shards).
- A column with extreme skew (e.g., 90% of rows have the same value).
- Constant value — puts all data on one shard.

**Colocated JOINs:** If you JOIN two tables on `user_id`, shard both tables by `user_id` (same expression). Then JOIN on distributed tables uses local shard data and avoids cross-shard data transfer.

## Writing Directly vs Through Distributed Table

```sql
-- Write through Distributed table (simpler, automatic routing)
INSERT INTO events_dist VALUES (...);
-- ClickHouse routes to correct shard. Slightly higher latency due to routing.

-- Write directly to local MergeTree on each shard (faster, requires client-side routing)
INSERT INTO events VALUES (...);  -- run on each shard with pre-sharded data
```

For high-throughput ingestion (Kafka, batch loaders), writing directly to local tables and sharding at the source is more efficient.

## Cluster Configuration (config.xml)

```xml
<remote_servers>
    <my_cluster>
        <shard>
            <weight>1</weight>
            <internal_replication>true</internal_replication>
            <replica><host>clickhouse-01</host><port>9000</port></replica>
            <replica><host>clickhouse-02</host><port>9000</port></replica>
        </shard>
        <shard>
            <weight>1</weight>
            <internal_replication>true</internal_replication>
            <replica><host>clickhouse-03</host><port>9000</port></replica>
            <replica><host>clickhouse-04</host><port>9000</port></replica>
        </shard>
    </my_cluster>
</remote_servers>
```

`internal_replication = true`: when inserting through Distributed, ClickHouse writes to one replica per shard and lets ReplicatedMergeTree handle replication to the other replica. Recommended.

## Monitoring Replication

```sql
-- Replication queue — pending operations
SELECT table, type, create_time, required_quorum, source_replica, new_part_name
FROM system.replication_queue
ORDER BY create_time;

-- Replication lag per table/replica
SELECT database, table, replica_name, absolute_delay
FROM system.replicas
WHERE absolute_delay > 0
ORDER BY absolute_delay DESC;

-- Check Keeper/ZooKeeper connectivity
SELECT * FROM system.zookeeper WHERE path = '/';
```

High `absolute_delay` means a replica is behind — it will serve stale reads. Investigate with `system.replication_queue` for stuck tasks.
