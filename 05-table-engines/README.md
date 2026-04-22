# 📖 Table Engines

> **Day 6 of the ClickHouse User Guide**

The **table engine** is the most important decision when creating a table. It determines how data is stored, indexed, merged, and queried.

---

## Table of Contents

- [Engine Families](#engine-families)
- [MergeTree — The Workhorse](#mergetree-the-workhorse)
- [ReplacingMergeTree — Deduplication](#replacingmergetree-deduplication)
- [SummingMergeTree — Pre-aggregation](#summingmergetree-pre-aggregation)
- [AggregatingMergeTree — Custom Aggregations](#aggregatingmergetree-custom-aggregations)
- [CollapsingMergeTree — Event Sourcing](#collapsingmergetree-event-sourcing)
- [Log Engines — Lightweight Storage](#log-engines-lightweight-storage)
- [Memory Engine — In-Memory Tables](#memory-engine-in-memory-tables)
- [Distributed Engine — Multi-Node Queries](#distributed-engine-multi-node-queries)
- [Engine Selection Guide](#engine-selection-guide)

---

## Engine Families

```
ClickHouse Table Engines
│
├── MergeTree Family (for production data)
│   ├── MergeTree            — general purpose
│   ├── ReplacingMergeTree   — deduplication
│   ├── SummingMergeTree     — pre-aggregate sums
│   ├── AggregatingMergeTree — custom aggregations
│   ├── CollapsingMergeTree  — update simulation
│   └── VersionedCollapsingMergeTree
│
├── Log Family (for small/temp data)
│   ├── Log
│   ├── TinyLog
│   └── StripeLog
│
├── Integration Engines
│   ├── Kafka
│   ├── S3
│   ├── MySQL
│   └── PostgreSQL
│
└── Special Engines
    ├── Memory
    ├── Distributed
    ├── View / MaterializedView
    └── Dictionary
```

---

## MergeTree — The Workhorse

The foundation of almost every ClickHouse table. Use this unless you have a specific reason to use a variant.

```sql
CREATE TABLE events
(
    event_id   UInt64,
    user_id    UInt64,
    event_type LowCardinality(String),
    event_date Date,
    event_time DateTime,
    page_url   String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)    -- split data into monthly partitions
ORDER BY (event_date, user_id)       -- primary sort key (also the sparse index)
PRIMARY KEY (event_date, user_id)    -- optional: defaults to ORDER BY
TTL event_date + INTERVAL 1 YEAR    -- optional: auto-delete old data
SETTINGS index_granularity = 8192;  -- rows per index granule (default is fine)
```

### How MergeTree works

1. **Data is written** as immutable "parts" (like mini-files)
2. **Background merges** combine small parts into larger ones
3. **Sparse index** maps every 8192nd row to a file offset
4. **Queries** use the index to skip irrelevant granules

```
Disk layout:
events/
  ├── 20240101_1_5_2/       ← a merged part
  │   ├── event_date.bin    ← column data (compressed)
  │   ├── user_id.bin
  │   ├── primary.idx       ← sparse index
  │   └── ...
  └── 20240102_6_10_1/
```

### PARTITION BY

```sql
PARTITION BY toYYYYMM(event_date)    -- monthly (most common)
PARTITION BY toYYYYMMDD(event_date)  -- daily (for high volume)
PARTITION BY toYear(event_date)      -- yearly (for cold data)
```

Benefits:
- Queries filtered on the partition key **skip entire partitions**
- You can `DROP PARTITION` to delete a month of data instantly (no mutation needed)
- Moves/copies of partitions are very fast

```sql
-- Drop a specific partition
ALTER TABLE events DROP PARTITION '202312';

-- List all partitions
SELECT partition, rows, bytes_on_disk
FROM system.parts
WHERE table = 'events' AND active = 1
ORDER BY partition;
```

### ORDER BY is your index

```sql
-- Fast: filters on ORDER BY prefix
SELECT count() FROM events WHERE event_date = '2024-01-15';
SELECT count() FROM events WHERE event_date = '2024-01-15' AND user_id = 42;

-- Slower: filter on non-ORDER BY column
SELECT count() FROM events WHERE page_url = '/home';
-- (ClickHouse still works but reads more data)
```

> 💡 **Key rule:** Put your most commonly filtered columns first in `ORDER BY`. High-cardinality columns should come after low-cardinality ones.

---

## ReplacingMergeTree — Deduplication

Automatically removes duplicate rows with the same `ORDER BY` key, keeping only the **latest version**.

```sql
CREATE TABLE users
(
    user_id    UInt64,
    name       String,
    email      String,
    updated_at DateTime
)
ENGINE = ReplacingMergeTree(updated_at)  -- keep row with highest updated_at
ORDER BY user_id;

-- Insert original record
INSERT INTO users VALUES (1, 'Alice', 'alice@old.com', '2024-01-01 00:00:00');

-- "Update" by inserting a new version
INSERT INTO users VALUES (1, 'Alice', 'alice@new.com', '2024-01-15 00:00:00');

-- Query with deduplication (FINAL forces merge)
SELECT * FROM users FINAL WHERE user_id = 1;
-- Returns: 1, Alice, alice@new.com, 2024-01-15
```

> ⚠️ Without `FINAL`, you may see duplicates because background merges haven't happened yet. `FINAL` is slower but always correct.

---

## SummingMergeTree — Pre-aggregation

Automatically sums numeric columns when rows with the same `ORDER BY` key are merged. Great for counters and metrics.

```sql
CREATE TABLE daily_stats
(
    date       Date,
    country    LowCardinality(String),
    page_views UInt64,
    clicks     UInt64,
    revenue    Decimal(18, 2)
)
ENGINE = SummingMergeTree()     -- sums all numeric columns
ORDER BY (date, country);

INSERT INTO daily_stats VALUES
    ('2024-01-15', 'PK', 1000, 50, 500.00),
    ('2024-01-15', 'PK',  500, 30, 200.00);  -- same key

-- After merge, these become:
-- 2024-01-15, PK, 1500, 80, 700.00

SELECT date, country, sum(page_views), sum(revenue)
FROM daily_stats
GROUP BY date, country;
```

> 💡 Always use `sum()` in queries even with SummingMergeTree — merges happen in the background and may not be complete.

---

## AggregatingMergeTree — Custom Aggregations

Stores **intermediate aggregation states** instead of raw data. Used with Materialized Views for real-time rollups.

```sql
-- Store aggregation states
CREATE TABLE hourly_pageviews
(
    hour       DateTime,
    country    LowCardinality(String),
    views      AggregateFunction(count),
    uniq_users AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (hour, country);

-- Query using special merge functions
SELECT
    hour,
    country,
    countMerge(views)      AS total_views,
    uniqMerge(uniq_users)  AS unique_users
FROM hourly_pageviews
GROUP BY hour, country;
```

---

## CollapsingMergeTree — Event Sourcing

Simulates updates by inserting "cancel" rows (`sign = -1`) and "state" rows (`sign = 1`):

```sql
CREATE TABLE user_balances
(
    user_id UInt64,
    balance Decimal(18, 2),
    sign    Int8    -- 1 = current state, -1 = cancel previous state
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY user_id;

-- Initial balance
INSERT INTO user_balances VALUES (1, 1000.00, 1);

-- To "update" balance: insert a cancel + new state
INSERT INTO user_balances VALUES
    (1, 1000.00, -1),   -- cancel old state
    (1, 1500.00,  1);   -- new state

-- Query: sum(balance * sign) gives current state
SELECT user_id, sum(balance * sign) AS current_balance
FROM user_balances
GROUP BY user_id
HAVING current_balance > 0;
```

---

## Log Engines — Lightweight Storage

For small tables, temporary data, or test datasets:

| Engine | Best for | Concurrent writes | Index |
|--------|----------|------------------|-------|
| `TinyLog` | Very small tables < 1M rows | No | No |
| `Log` | Small tables, temp data | No | No |
| `StripeLog` | Slightly larger temp tables | No | No |

```sql
CREATE TABLE temp_import
(
    id   UInt64,
    data String
)
ENGINE = Log;
```

> ⚠️ Log engines don't support concurrent writes or indexes. Only use for dev/test or staging tables.

---

## Memory Engine — In-Memory Tables

Stores data entirely in RAM. Extremely fast but data is lost on server restart.

```sql
CREATE TABLE cache_table
(
    key   String,
    value String,
    ttl   DateTime
)
ENGINE = Memory;
```

Use cases:
- Temporary lookup tables
- Small reference data that's rebuilt on startup
- Join targets (small dimension tables)

---

## Distributed Engine — Multi-Node Queries

Routes queries across a cluster of ClickHouse servers:

```sql
CREATE TABLE events_distributed
AS events  -- same schema as local table
ENGINE = Distributed(
    'my_cluster',   -- cluster name from config
    'default',      -- database
    'events',       -- local table name
    rand()          -- sharding key (rand() = round robin)
);
```

Queries on the Distributed table automatically fan out to all shards and merge results. (Covered in detail in [Section 08](../08-replication-clustering/README.md))

---

## Engine Selection Guide

```
What is your use case?
│
├── General analytics / time-series data
│   └── MergeTree ✅
│
├── You need to "update" records (user profiles, product catalog)
│   └── ReplacingMergeTree ✅
│
├── You need real-time counters (page views, clicks per day)
│   └── SummingMergeTree ✅
│
├── You need real-time dashboards with complex aggregations
│   └── AggregatingMergeTree + Materialized Views ✅
│
├── Small temp table or staging area
│   └── Log or TinyLog ✅
│
├── In-memory lookup table
│   └── Memory ✅
│
└── Multi-server cluster
    └── Distributed ✅
```

---

## Summary

- **MergeTree** is the default engine for almost all production tables
- **PARTITION BY** splits data into manageable chunks; use it for time-series
- **ORDER BY** is your primary index — choose it carefully
- **ReplacingMergeTree** handles deduplication / "update" patterns
- **SummingMergeTree** pre-aggregates counters automatically
- **Log engines** are for small, non-critical temporary tables only

---

> 📅 **Tomorrow — Day 7:** Queries — SELECT, JOINs, GROUP BY, Window Functions, and CTEs with real examples.

[← Data Types](../04-data-types/README.md) | [Back to Main Guide](../README.md) | [Next: Queries →](../06-queries/README.md)
