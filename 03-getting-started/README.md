# 📖 Getting Started

> **Day 4 of the ClickHouse User Guide**

---

## Table of Contents

- [Connect to ClickHouse](#connect-to-clickhouse)
- [Databases](#databases)
- [Creating Your First Table](#creating-your-first-table)
- [Inserting Data](#inserting-data)
- [Querying Data](#querying-data)
- [Updating and Deleting Data](#updating-and-deleting-data)
- [Importing CSV Data](#importing-csv-data)
- [Useful System Tables](#useful-system-tables)
- [A Complete Mini Project](#a-complete-mini-project)

---

## Connect to ClickHouse

```bash
# Docker users
docker exec -it clickhouse clickhouse-client

# Linux install users
clickhouse-client
```

---

## Databases

```sql
-- List all databases
SHOW DATABASES;

-- Create a database
CREATE DATABASE IF NOT EXISTS shop;

-- Switch to it
USE shop;

-- Drop a database (careful!)
DROP DATABASE IF EXISTS shop;
```

> 💡 ClickHouse comes with built-in `system` and `default` databases. The `system` database contains metadata about your server — never drop it.

---

## Creating Your First Table

ClickHouse requires you to pick a **table engine**. For most use cases, use `MergeTree`. (Deep dive in [Section 05](../05-table-engines/README.md))

```sql
CREATE TABLE shop.orders
(
    order_id     UInt64,
    customer_id  UInt64,
    product_name String,
    category     LowCardinality(String),
    quantity     UInt32,
    price        Decimal(10, 2),
    order_date   Date,
    created_at   DateTime DEFAULT now()
)
ENGINE = MergeTree()
ORDER BY (order_date, customer_id);
```

### Breaking it down

| Part | Meaning |
|------|---------|
| `ENGINE = MergeTree()` | Use the MergeTree storage engine |
| `ORDER BY (order_date, customer_id)` | Sort key — queries filtering on these columns will be fast |
| `LowCardinality(String)` | Optimization for columns with few unique values |
| `DEFAULT now()` | Auto-fill with current timestamp if not provided |

### Show table structure

```sql
DESCRIBE TABLE shop.orders;

-- Or
SHOW CREATE TABLE shop.orders;
```

---

## Inserting Data

### Basic INSERT

```sql
INSERT INTO shop.orders
    (order_id, customer_id, product_name, category, quantity, price, order_date)
VALUES
    (1, 101, 'Laptop',     'Electronics', 1, 999.99,  '2024-01-15'),
    (2, 102, 'Headphones', 'Electronics', 2, 149.50,  '2024-01-15'),
    (3, 101, 'Desk Chair', 'Furniture',   1, 299.00,  '2024-01-16'),
    (4, 103, 'Monitor',    'Electronics', 1, 450.00,  '2024-01-16'),
    (5, 102, 'Keyboard',   'Electronics', 1, 89.99,   '2024-01-17');
```

### INSERT from SELECT

```sql
-- Copy data from another table
INSERT INTO shop.orders_archive
SELECT * FROM shop.orders
WHERE order_date < '2024-01-01';
```

### Batch inserts (best practice)

> ⚠️ **Important:** ClickHouse is optimized for **large batch inserts**, not single-row inserts. Insert at least 1,000–10,000 rows per statement in production. Tiny inserts create too many small parts on disk.

```sql
-- Good: 10,000 rows in one INSERT
INSERT INTO shop.events SELECT * FROM generateRandom(...) LIMIT 10000;

-- Bad: 10,000 separate single-row INSERTs
```

---

## Querying Data

### Basic SELECT

```sql
SELECT * FROM shop.orders LIMIT 10;

SELECT
    order_id,
    product_name,
    price
FROM shop.orders
WHERE category = 'Electronics'
ORDER BY price DESC
LIMIT 5;
```

### Aggregations

```sql
-- Total revenue
SELECT sum(price * quantity) AS total_revenue
FROM shop.orders;

-- Revenue by category
SELECT
    category,
    count()              AS total_orders,
    sum(price * quantity) AS revenue,
    avg(price)           AS avg_price
FROM shop.orders
GROUP BY category
ORDER BY revenue DESC;

-- Orders per day
SELECT
    order_date,
    count() AS orders
FROM shop.orders
GROUP BY order_date
ORDER BY order_date;
```

### Filtering with WHERE

```sql
-- Date range filter
SELECT * FROM shop.orders
WHERE order_date BETWEEN '2024-01-15' AND '2024-01-16';

-- Multiple conditions
SELECT * FROM shop.orders
WHERE category = 'Electronics'
  AND price > 200
  AND quantity >= 1;

-- IN operator
SELECT * FROM shop.orders
WHERE category IN ('Electronics', 'Furniture');

-- LIKE pattern match
SELECT * FROM shop.orders
WHERE product_name LIKE '%Laptop%';
```

### Useful functions

```sql
-- Date functions
SELECT
    toYear(order_date)     AS year,
    toMonth(order_date)    AS month,
    toDayOfWeek(order_date) AS day_of_week
FROM shop.orders;

-- String functions
SELECT
    lower(product_name),
    upper(category),
    length(product_name)
FROM shop.orders;

-- Conditional
SELECT
    order_id,
    price,
    if(price > 300, 'expensive', 'affordable') AS price_tier
FROM shop.orders;
```

---

## Updating and Deleting Data

> ⚠️ Unlike PostgreSQL, updates and deletes in ClickHouse are **mutations** — asynchronous, heavy operations. Use them sparingly.

### UPDATE (mutation)

```sql
-- Update price for a specific order
ALTER TABLE shop.orders
UPDATE price = 899.99
WHERE order_id = 1;
```

### DELETE (mutation)

```sql
-- Delete old records
ALTER TABLE shop.orders
DELETE WHERE order_date < '2023-01-01';
```

### Check mutation status

```sql
SELECT *
FROM system.mutations
WHERE table = 'orders'
ORDER BY create_time DESC
LIMIT 5;
```

> 💡 **Best practice:** Design your schema so that you rarely need to update data. ClickHouse works best as an append-only store.

---

## Importing CSV Data

### Via clickhouse-client

```bash
# Import a CSV file
clickhouse-client \
  --query "INSERT INTO shop.orders FORMAT CSVWithNames" \
  < orders.csv
```

### Via HTTP API

```bash
curl -X POST "http://localhost:8123/?query=INSERT+INTO+shop.orders+FORMAT+CSVWithNames" \
  --data-binary @orders.csv
```

### From a URL

```sql
-- Read directly from a CSV on the web
INSERT INTO shop.orders
SELECT * FROM url(
    'https://example.com/data/orders.csv',
    CSVWithNames,
    'order_id UInt64, customer_id UInt64, product_name String, ...'
);
```

---

## Useful System Tables

```sql
-- All tables in current database
SELECT name, engine, total_rows, total_bytes
FROM system.tables
WHERE database = 'shop';

-- Running queries
SELECT query_id, user, elapsed, query
FROM system.processes;

-- Query history
SELECT query, query_duration_ms, read_rows
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 10;

-- Disk usage
SELECT
    database,
    table,
    formatReadableSize(sum(bytes)) AS size
FROM system.parts
GROUP BY database, table
ORDER BY sum(bytes) DESC;
```

---

## A Complete Mini Project

Let's build a small e-commerce analytics system from scratch:

```sql
-- 1. Create database
CREATE DATABASE IF NOT EXISTS ecommerce;
USE ecommerce;

-- 2. Create events table
CREATE TABLE user_events
(
    event_id    UUID DEFAULT generateUUIDv4(),
    user_id     UInt64,
    event_type  LowCardinality(String),  -- 'view', 'click', 'purchase'
    product_id  UInt64,
    session_id  String,
    event_time  DateTime,
    event_date  Date DEFAULT toDate(event_time)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id, event_time);

-- 3. Insert sample data
INSERT INTO user_events (user_id, event_type, product_id, session_id, event_time)
VALUES
    (1, 'view',     101, 'sess_a', '2024-01-15 09:00:00'),
    (1, 'click',    101, 'sess_a', '2024-01-15 09:01:00'),
    (1, 'purchase', 101, 'sess_a', '2024-01-15 09:05:00'),
    (2, 'view',     102, 'sess_b', '2024-01-15 10:00:00'),
    (2, 'view',     103, 'sess_b', '2024-01-15 10:02:00'),
    (3, 'view',     101, 'sess_c', '2024-01-15 11:00:00'),
    (3, 'purchase', 101, 'sess_c', '2024-01-15 11:03:00');

-- 4. Analyze: conversion rate per product
SELECT
    product_id,
    countIf(event_type = 'view')     AS views,
    countIf(event_type = 'purchase') AS purchases,
    round(100 * purchases / views, 2) AS conversion_pct
FROM user_events
GROUP BY product_id
ORDER BY conversion_pct DESC;
```

Output:
```
┌─product_id─┬─views─┬─purchases─┬─conversion_pct─┐
│        101 │     3 │         2 │          66.67 │
│        102 │     1 │         0 │              0 │
│        103 │     1 │         0 │              0 │
└────────────┴───────┴───────────┴────────────────┘
```

---

## Summary

- Use `CREATE DATABASE` and `CREATE TABLE ... ENGINE = MergeTree()` to set up your schema
- Always specify an `ORDER BY` clause — it's your primary index
- Insert data in **large batches** for best performance
- Updates and deletes are mutations — use them sparingly
- The `system` database is your best friend for monitoring and debugging

---

> 📅 **Tomorrow — Day 5:** Data Types — A complete cheatsheet of every ClickHouse data type with examples.

[← Installation](../02-installation/README.md) | [Back to Main Guide](../README.md) | [Next: Data Types →](../04-data-types/README.md)
