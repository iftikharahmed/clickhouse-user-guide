# 📖 Replication & Clustering

> **Day 10 of the ClickHouse User Guide**

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [ClickHouse Keeper (Coordination)](#clickhouse-keeper)
- [Replicated Tables](#replicated-tables)
- [Distributed Tables (Sharding)](#distributed-tables-sharding)
- [Cluster Configuration](#cluster-configuration)
- [Practical Setup: 3-Node Cluster](#practical-setup-3-node-cluster)
- [Monitoring Your Cluster](#monitoring-your-cluster)

---

## Core Concepts

| Concept | What it means |
|---------|--------------|
| **Replication** | Same data on multiple nodes — for **high availability** |
| **Sharding** | Different data on different nodes — for **horizontal scaling** |
| **Replica** | A node that holds a copy of a shard |
| **Shard** | A partition of the full dataset |
| **Keeper** | Coordination service (like ZooKeeper) that tracks replica state |

A production cluster typically combines both:

```
                  ┌─────────────────────────────┐
                  │    Distributed Table Layer   │
                  └──────────┬──────────────┬───┘
                             │              │
                    ┌────────▼──────┐  ┌────▼────────┐
                    │   Shard 1     │  │   Shard 2   │
                    │  (50% data)   │  │  (50% data) │
                    └──┬────────┬──┘  └──┬────────┬──┘
                       │        │        │        │
                  Replica 1  Replica 2  Replica 1  Replica 2
                  (primary)  (backup)   (primary)  (backup)
```

---

## ClickHouse Keeper

ClickHouse Keeper is the coordination service that manages replication state, leader election, and data consistency. It replaces the older Apache ZooKeeper dependency.

### Setup (in `config.xml` or separate `keeper_config.xml`)

```xml
<keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>    <!-- unique per node: 1, 2, 3 -->

    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>ch-node-1</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>2</id>
            <hostname>ch-node-2</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>3</id>
            <hostname>ch-node-3</hostname>
            <port>9234</port>
        </server>
    </raft_configuration>
</keeper_server>
```

> 💡 Run Keeper on 3 nodes for quorum — it can tolerate 1 node failure. Run on 5 nodes to tolerate 2 failures.

### Check Keeper health

```sql
SELECT * FROM system.zookeeper WHERE path = '/';
SELECT * FROM system.keeper_map_data_losses;
```

---

## Replicated Tables

Replace `MergeTree` with `ReplicatedMergeTree` to enable replication:

```sql
-- On every node in the replica set
CREATE TABLE events
(
    event_id   UInt64,
    user_id    UInt64,
    event_date Date,
    event_time DateTime
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',  -- Keeper path (unique per shard)
    '{replica}'                           -- replica name (unique per node)
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);
```

`{shard}` and `{replica}` are macros defined in `config.xml` per node:

```xml
<!-- On node ch-node-1 (shard 1, replica 1) -->
<macros>
    <shard>01</shard>
    <replica>ch-node-1</replica>
</macros>

<!-- On node ch-node-2 (shard 1, replica 2) -->
<macros>
    <shard>01</shard>
    <replica>ch-node-2</replica>
</macros>
```

### How replication works

1. You insert data into any replica
2. That replica writes it locally and **logs the insert** in Keeper
3. Other replicas **pull** the data in the background
4. If a replica is down, it **catches up** when it comes back online

```sql
-- Check replication lag
SELECT
    database, table, replica_name,
    absolute_delay AS lag_seconds,
    queue_size
FROM system.replicas
ORDER BY lag_seconds DESC;
```

---

## Distributed Tables (Sharding)

A Distributed table is a **virtual table** that routes queries across all shards:

```sql
-- Create the local ReplicatedMergeTree table (on every node)
CREATE TABLE events_local
ON CLUSTER 'my_cluster'      -- creates on all nodes automatically
(
    event_id   UInt64,
    user_id    UInt64,
    event_date Date,
    event_time DateTime
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);

-- Create the Distributed table (on every node)
CREATE TABLE events
ON CLUSTER 'my_cluster'
AS events_local
ENGINE = Distributed(
    'my_cluster',    -- cluster name
    'default',       -- database
    'events_local',  -- local table name
    rand()           -- sharding key: rand() = random round-robin
);
```

### Sharding keys

```sql
-- Round robin (equal distribution, random)
ENGINE = Distributed('cluster', 'db', 'table', rand())

-- By user (same user always goes to same shard)
ENGINE = Distributed('cluster', 'db', 'table', user_id)

-- By hash for more even distribution
ENGINE = Distributed('cluster', 'db', 'table', cityHash64(user_id))
```

> 💡 **Rule:** If you often query by `user_id`, shard by `user_id`. This ensures user data lives on one shard and avoids cross-shard JOINs.

### Querying distributed tables

```sql
-- This query fans out to all shards automatically
SELECT count(), uniq(user_id)
FROM events                  -- the Distributed table
WHERE event_date = today();
```

---

## Cluster Configuration

Define your cluster in `/etc/clickhouse-server/config.xml`:

```xml
<remote_servers>
    <my_cluster>
        <shard>
            <!-- Shard 1: 2 replicas -->
            <replica>
                <host>ch-node-1</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>ch-node-2</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <!-- Shard 2: 2 replicas -->
            <replica>
                <host>ch-node-3</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>ch-node-4</host>
                <port>9000</port>
            </replica>
        </shard>
    </my_cluster>
</remote_servers>
```

### Verify cluster config

```sql
SELECT cluster, shard_num, replica_num, host_name
FROM system.clusters
WHERE cluster = 'my_cluster';
```

---

## Practical Setup: 3-Node Cluster

Minimal production-ready setup (3 nodes, 1 shard with 2 replicas + 1 Keeper quorum):

```
Node 1: ch-node-1  →  Keeper (id=1) + Shard 1 Replica 1
Node 2: ch-node-2  →  Keeper (id=2) + Shard 1 Replica 2
Node 3: ch-node-3  →  Keeper (id=3) only (coordination)
```

### Docker Compose for local 3-node test cluster

```yaml
version: "3.8"
services:
  ch1:
    image: clickhouse/clickhouse-server:latest
    hostname: ch1
    ports: ["8123:8123", "9000:9000"]
    volumes:
      - ./config/ch1:/etc/clickhouse-server/config.d
      - ch1-data:/var/lib/clickhouse

  ch2:
    image: clickhouse/clickhouse-server:latest
    hostname: ch2
    ports: ["8124:8123", "9001:9000"]
    volumes:
      - ./config/ch2:/etc/clickhouse-server/config.d
      - ch2-data:/var/lib/clickhouse

  ch3:
    image: clickhouse/clickhouse-server:latest
    hostname: ch3
    ports: ["8125:8123", "9002:9000"]
    volumes:
      - ./config/ch3:/etc/clickhouse-server/config.d
      - ch3-data:/var/lib/clickhouse

volumes:
  ch1-data:
  ch2-data:
  ch3-data:
```

---

## Monitoring Your Cluster

```sql
-- All replicas and their status
SELECT
    database,
    table,
    replica_name,
    is_leader,
    is_readonly,
    absolute_delay  AS replication_lag_sec,
    queue_size      AS pending_operations
FROM system.replicas;

-- Replication queue (what's waiting to be replicated)
SELECT *
FROM system.replication_queue
ORDER BY create_time DESC
LIMIT 20;

-- Cluster health overview
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    errors_count,
    estimated_recovery_time
FROM system.clusters;

-- Parts distribution across shards
SELECT
    hostName()             AS node,
    table,
    count()                AS parts,
    sum(rows)              AS total_rows,
    formatReadableSize(sum(bytes)) AS size
FROM clusterAllReplicas('my_cluster', system.parts)
WHERE active = 1
GROUP BY node, table
ORDER BY node, table;
```

---

## Summary

- **Replication** = same data on multiple nodes (high availability)
- **Sharding** = different data on different nodes (horizontal scale)
- Use `ReplicatedMergeTree` for replication; wrap with `Distributed` for sharding
- **ClickHouse Keeper** is the coordination layer — run it on 3 or 5 nodes
- `ON CLUSTER` clause creates tables on all nodes in one command
- Monitor replication lag via `system.replicas`

---

> 📅 **Tomorrow — Day 11:** Integrations — connect ClickHouse to Kafka, S3, Python, dbt, and more.

[← Performance](../07-performance/README.md) | [Back to Main Guide](../README.md) | [Next: Integrations →](../09-integrations/README.md)
