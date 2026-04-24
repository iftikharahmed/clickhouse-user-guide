# 📖 Performance & Indexing

> **Day 9 of the ClickHouse User Guide**

---

## Table of Contents

- [How ClickHouse Reads Data](#how-clickhouse-reads-data)
- [Primary Key and ORDER BY](#primary-key-and-order-by)
- [Skipping Indexes (Secondary Indexes)](#skipping-indexes-secondary-indexes)
- [Partitioning for Performance](#partitioning-for-performance)
- [Projections](#projections)
- [Materialized Views](#materialized-views)
- [Query Profiling](#query-profiling)
- [Performance Tips Cheatsheet](#performance-tips-cheatsheet)

---

## How ClickHouse Reads Data

Understanding this is the key to writing fast queries.

```
Query: SELECT count() FROM events WHERE event_date = '2024-01-15' AND user_id = 42

Step 1: PARTITION PRUNING
  ├── Table is partitioned by toYYYYMM(event_date)
  └── Only partition '202401' is opened — all others skipped ✅

Step 2: PRIMARY INDEX LOOKUP
  ├── ORDER BY (event_date, user_id) — sparse index exists
  ├── Binary search finds which granules may contain our rows
  └── Only 2 granules out of 10,000 are read ✅

Step 3: COLUMN READS
  ├── Only event_date and user_id columns are read (column pruning)
  └── Other 8 columns are never touched ✅

Step 4: DECOMPRESSION + FILTERING
  └── Decompress granules, apply WHERE filter, return count
```

This layered skipping is why ClickHouse is so fast.

---

## Primary Key and ORDER BY

The `ORDER BY` clause defines both the **sort order** and the **sparse primary index**.

### Choosing the right ORDER BY

```sql
-- Good: query patterns drive the choice
-- If you mostly query by date + country:
ORDER BY (event_date, country, user_id)

-- If you mostly query by user:
ORDER BY (user_id, event_date)

-- Rule: most-filtered column FIRST, highest-cardinality LAST
```

### Cardinality matters

```sql
-- Bad: high-cardinality first means poor pruning
ORDER BY (user_id, event_date)
-- Filtering by event_date alone won't use the index effectively

-- Better: low-cardinality first
ORDER BY (event_date, country, user_id)
-- Filtering by event_date is very selective and efficient
```

### Check if your query uses the index

```sql
-- Enable trace logging for a query
SET send_logs_level = 'trace';
SELECT count() FROM events WHERE event_date = '2024-01-15';
-- Look for: "Selected 2/10000 parts by partition key"
-- And: "Selected 3/500 granules by primary key"
```

---

## Skipping Indexes (Secondary Indexes)

When you need to filter on a column that's **not** in `ORDER BY`, use a skipping index.

### minmax — range queries

```sql
ALTER TABLE events
ADD INDEX idx_price price TYPE minmax GRANULARITY 4;

-- Effective for: WHERE price > 100 AND price < 500
```

### set — equality/IN queries

```sql
ALTER TABLE events
ADD INDEX idx_status status TYPE set(100) GRANULARITY 4;
-- 100 = max unique values per granule to index

-- Effective for: WHERE status = 'active'
-- Effective for: WHERE status IN ('active', 'pending')
```

### bloom_filter — high-cardinality equality

```sql
ALTER TABLE events
ADD INDEX idx_email email TYPE bloom_filter(0.01) GRANULARITY 4;
-- 0.01 = 1% false positive rate

-- Effective for: WHERE email = 'user@example.com'
```

### ngrambf_v1 — substring search

```sql
ALTER TABLE articles
ADD INDEX idx_content content TYPE ngrambf_v1(3, 256, 2, 0) GRANULARITY 1;

-- Effective for: WHERE content LIKE '%clickhouse%'
```

### Build index on existing data

```sql
-- After adding an index, rebuild it on existing data
ALTER TABLE events MATERIALIZE INDEX idx_price;
```

---

## Partitioning for Performance

### Why partition?

- Queries filtered on the partition key skip entire partition directories
- DROP PARTITION is instant — no row-level deletion needed
- Data tiering: move old partitions to slower/cheaper storage

### Partition by month (most common)

```sql
PARTITION BY toYYYYMM(event_date)
```

### Partition by day (for very high volume)

```sql
PARTITION BY toYYYYMMDD(event_date)
-- Use when inserting millions of rows/day
```

### Check partition sizes

```sql
SELECT
    partition,
    count()                          AS parts,
    sum(rows)                        AS rows,
    formatReadableSize(sum(bytes))   AS size
FROM system.parts
WHERE table = 'events' AND active = 1
GROUP BY partition
ORDER BY partition DESC;
```

### Drop old data instantly

```sql
-- Drop all data from December 2023 — instant operation
ALTER TABLE events DROP PARTITION '202312';
```

> 💡 **Tip:** Never use mutations (`DELETE WHERE event_date < ...`) to delete old data. Always `DROP PARTITION` instead — it's 1000x faster.

---

## Projections

Projections are **pre-sorted copies** of your data stored inside the same table. They allow a single table to have multiple sort orders.

```sql
CREATE TABLE orders
(
    order_id   UInt64,
    customer_id UInt64,
    product_id UInt64,
    order_date Date,
    revenue    Decimal(18,2),

    -- Main sort: time-based queries
    PROJECTION by_customer (
        SELECT * ORDER BY customer_id, order_date
    ),
    PROJECTION revenue_by_product (
        SELECT product_id, sum(revenue) GROUP BY product_id
    )
)
ENGINE = MergeTree()
ORDER BY (order_date, order_id);
```

```sql
-- ClickHouse automatically uses the best projection
SELECT * FROM orders WHERE customer_id = 42;    -- uses by_customer projection
SELECT sum(revenue) FROM orders GROUP BY product_id;  -- uses revenue_by_product
```

---

## Materialized Views

Materialized Views automatically process and store query results as new data arrives. They're the most powerful tool for real-time pre-aggregation.

```sql
-- Source table
CREATE TABLE raw_events
(
    user_id    UInt64,
    country    LowCardinality(String),
    event_time DateTime,
    event_date Date DEFAULT toDate(event_time)
)
ENGINE = MergeTree()
ORDER BY event_date;

-- Aggregation target table
CREATE TABLE hourly_country_stats
(
    hour     DateTime,
    country  LowCardinality(String),
    events   UInt64,
    users    AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (hour, country);

-- Materialized view: runs on every INSERT into raw_events
CREATE MATERIALIZED VIEW mv_hourly_stats
TO hourly_country_stats
AS
SELECT
    toStartOfHour(event_time)     AS hour,
    country,
    count()                       AS events,
    uniqState(user_id)            AS users
FROM raw_events
GROUP BY hour, country;

-- Query the pre-aggregated data (fast!)
SELECT
    hour,
    country,
    sum(events)          AS total_events,
    uniqMerge(users)     AS unique_users
FROM hourly_country_stats
GROUP BY hour, country
ORDER BY hour DESC;
```

---

## Query Profiling

### See what a query does before running it

```sql
EXPLAIN SELECT count() FROM events WHERE event_date = today();
EXPLAIN PIPELINE SELECT count() FROM events WHERE event_date = today();
```

### Find slow queries

```sql
SELECT
    query,
    query_duration_ms,
    read_rows,
    formatReadableSize(read_bytes) AS read_size,
    result_rows
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_duration_ms > 1000   -- queries > 1 second
ORDER BY query_duration_ms DESC
LIMIT 20;
```

### Check parts and merges

```sql
-- Active parts per table (too many = too many small inserts)
SELECT table, count() AS parts
FROM system.parts
WHERE active = 1
GROUP BY table
ORDER BY parts DESC;

-- Ongoing merges
SELECT * FROM system.merges ORDER BY elapsed DESC;
```

---

## Performance Tips Cheatsheet

### Schema design

| Tip | Why |
|-----|-----|
| Use `LowCardinality(String)` for low-unique columns | 5–10x compression |
| Use `Date` not `DateTime` for partition key | Smaller index |
| Avoid `Nullable` unless truly needed | Extra storage per row |
| Put low-cardinality columns first in `ORDER BY` | Better index selectivity |
| Use `Decimal` not `Float` for money | Exact arithmetic |

### Query writing

| Tip | Why |
|-----|-----|
| Filter on partition key first | Skip entire partitions |
| Filter on ORDER BY prefix | Use sparse index |
| Use `uniq()` not `COUNT(DISTINCT)` | ~3x faster (approximate) |
| Prefer `IN` over `JOIN` for lookups | Less memory |
| Use `LIMIT` always on dev/exploration queries | Prevent accidental full scans |
| Use `countIf` instead of subqueries | Single pass |

### Data ingestion

| Tip | Why |
|-----|-----|
| Batch inserts ≥ 10,000 rows | Avoid too many small parts |
| Insert in background async | Don't block application |
| Use async_insert mode for many small clients | ClickHouse buffers internally |
| Use `Buffer` table engine for tiny event streams | Batch before writing |

```sql
-- Enable async insert (buffer small inserts automatically)
SET async_insert = 1;
SET wait_for_async_insert = 0;
```

---

## Summary

- ClickHouse performance comes from **partition pruning → index skipping → column pruning**
- Choose `ORDER BY` based on your **most common query filters**
- Use **skipping indexes** for non-ORDER-BY columns you filter on
- **Materialized Views** are the secret weapon for real-time dashboards
- Always **DROP PARTITION** to delete old data — never use mutations
- Batch your inserts — tiny inserts are the #1 performance killer

---

> 📅 **Tomorrow — Day 10:** Replication & Clustering — how to run ClickHouse across multiple servers for high availability and scale.

[← Queries](../06-queries/README.md) | [Back to Main Guide](../README.md) | [Next: Replication →](../08-replication-clustering/README.md)
