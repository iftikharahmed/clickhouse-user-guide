# 📖 Introduction to ClickHouse

> **Day 2 of the ClickHouse User Guide**

---

## Table of Contents

- [What is ClickHouse?](#what-is-clickhouse)
- [Row-Oriented vs Column-Oriented](#row-oriented-vs-column-oriented)
- [How ClickHouse Achieves Speed](#how-clickhouse-achieves-speed)
- [When to Use ClickHouse](#when-to-use-clickhouse)
- [When NOT to Use ClickHouse](#when-not-to-use-clickhouse)
- [Real-World Use Cases](#real-world-use-cases)
- [ClickHouse vs Other Databases](#clickhouse-vs-other-databases)

---

## What is ClickHouse?

ClickHouse is an **open-source, column-oriented database management system (DBMS)** built for **online analytical processing (OLAP)**. It was created by Yandex (Russia's largest search engine) in 2008 to power their web analytics platform, Yandex.Metrica — one of the world's largest web analytics services.

In 2016, ClickHouse was released as open source. Today it is used by thousands of companies including **Cloudflare, Uber, eBay, Spotify, and ByteDance**.

### Key facts at a glance

| Property | Value |
|----------|-------|
| First released | 2016 (open source) |
| Written in | C++ |
| Query language | SQL (with extensions) |
| License | Apache 2.0 |
| Storage model | Column-oriented |
| Primary use case | Analytical queries (OLAP) |
| Minimum hardware | 4 GB RAM, any modern CPU |

---

## Row-Oriented vs Column-Oriented

This is the most important concept to understand about ClickHouse.

### Row-Oriented (PostgreSQL, MySQL)

Data is stored row by row on disk:

```
┌────┬──────────┬─────┬────────────┬────────┐
│ id │ name     │ age │ country    │ salary │
├────┼──────────┼─────┼────────────┼────────┤
│  1 │ Alice    │  30 │ Pakistan   │  50000 │  ← stored together
│  2 │ Bob      │  25 │ USA        │  75000 │  ← stored together
│  3 │ Carol    │  35 │ Germany    │  60000 │  ← stored together
└────┴──────────┴─────┴────────────┴────────┘
```

When you run `SELECT avg(salary) FROM employees`, the database must **read every column of every row** even though you only need `salary`.

### Column-Oriented (ClickHouse)

Data is stored column by column on disk:

```
id:      [1, 2, 3, ...]
name:    [Alice, Bob, Carol, ...]
age:     [30, 25, 35, ...]
country: [Pakistan, USA, Germany, ...]
salary:  [50000, 75000, 60000, ...]  ← only this is read
```

When you run `SELECT avg(salary) FROM employees`, ClickHouse **reads only the salary column**. Everything else stays on disk.

### Why this matters

| Scenario | Row DB reads | ClickHouse reads |
|----------|-------------|-----------------|
| Table has 10 columns, you query 1 | 100% of data | 10% of data |
| Table has 100 columns, you query 2 | 100% of data | 2% of data |
| Table has 1 billion rows | Slow | Fast |

> 💡 **Key insight:** Analytical queries typically touch a few columns across millions/billions of rows. Column storage is a perfect match.

---

## How ClickHouse Achieves Speed

ClickHouse isn't just about column storage. It combines multiple techniques:

### 1. Columnar Storage
Only reads the columns needed for a query (explained above).

### 2. Data Compression
Each column contains values of the same type, which compresses extremely well.

```
# A column of country names compresses from:
[Pakistan, Pakistan, Pakistan, USA, USA, Germany, Pakistan]

# To something like:
[Pakistan×3, USA×2, Germany×1, Pakistan×1]
```

Typical compression ratios: **5x to 10x** smaller than raw data.

### 3. Vectorized Execution
Instead of processing one row at a time, ClickHouse processes **batches of 8192 rows** using SIMD CPU instructions. This saturates modern CPU pipelines and gives massive throughput.

### 4. Sparse Indexing
ClickHouse does not index every row. Instead it stores the **primary key value of every 8192nd row** (called a "granule"). This makes indexes tiny and fast to load into memory.

### 5. Parallel Processing
A single query automatically uses **all CPU cores** on the machine. On a 32-core server, your query effectively runs 32 times in parallel.

### 6. MergeTree Storage Engine
The default storage engine that handles sorting, indexing, and background merges automatically. (Covered in detail in [Section 05](../05-table-engines/README.md))

---

## When to Use ClickHouse

ClickHouse excels at:

✅ **Time-series analytics** — logs, events, metrics over time  
✅ **Business intelligence dashboards** — aggregations over large datasets  
✅ **Ad-tech and clickstream analysis** — billions of events per day  
✅ **E-commerce analytics** — sales trends, user behavior  
✅ **Security and monitoring** — log aggregation, anomaly detection  
✅ **Financial reporting** — fast aggregations over transaction history  
✅ **Real-time reporting** — sub-second query response on fresh data  

### Example queries ClickHouse handles brilliantly

```sql
-- Count unique users per country in the last 30 days
SELECT country, uniq(user_id) AS unique_users
FROM events
WHERE event_date >= today() - 30
GROUP BY country
ORDER BY unique_users DESC;

-- Revenue by product category per month
SELECT
    toStartOfMonth(order_date) AS month,
    category,
    sum(revenue)               AS total_revenue
FROM orders
GROUP BY month, category
ORDER BY month, total_revenue DESC;
```

---

## When NOT to Use ClickHouse

ClickHouse is **not** designed for:

❌ **Frequent UPDATE/DELETE operations** — ClickHouse is append-optimized; mutations are expensive  
❌ **OLTP workloads** — point lookups, short transactions (use PostgreSQL/MySQL instead)  
❌ **Key-value lookups** — fetching a single row by primary key (use Redis or DynamoDB)  
❌ **Highly normalized relational data** — complex multi-table JOINs with small datasets  
❌ **Small datasets** — overhead not worth it below ~1 million rows  

> ⚠️ **Rule of thumb:** If your app does many small reads/writes (like a typical web app backend), use PostgreSQL. If your app needs to analyze large volumes of data quickly, use ClickHouse.

---

## Real-World Use Cases

### Cloudflare
Processes **over 10 million HTTP requests per second** and stores the logs in ClickHouse for real-time analytics and DDoS detection.

### Uber
Uses ClickHouse to power their internal analytics platform, querying **trillions of rows** for driver and rider behavior analysis.

### Spotify
Analyzes user listening patterns and generates content recommendations using ClickHouse for aggregation queries.

### Yandex.Metrica
The original use case — handles **200 billion events per day** with real-time reporting at sub-second latency.

---

## ClickHouse vs Other Databases

| Feature | ClickHouse | PostgreSQL | MySQL | BigQuery | Redshift |
|---------|-----------|-----------|-------|---------|---------|
| Type | OLAP | OLTP/OLAP | OLTP | OLAP | OLAP |
| Storage | Columnar | Row | Row | Columnar | Columnar |
| Analytics speed | ⚡ Fastest | Slow | Slow | Fast | Fast |
| UPDATE/DELETE | Limited | ✅ Full | ✅ Full | Limited | Limited |
| Self-hosted | ✅ | ✅ | ✅ | ❌ | ❌ |
| Cost | Free (OSS) | Free (OSS) | Free (OSS) | Pay per query | Expensive |
| SQL support | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## Summary

- ClickHouse is a **column-oriented database** built for **fast analytical queries**
- It achieves speed through **columnar storage, compression, vectorized execution, and parallelism**
- Best for **read-heavy analytical workloads** with large datasets
- Not suitable for **frequent updates or transactional workloads**
- Used by some of the world's largest companies to handle **trillions of rows**

---

> 📅 **Tomorrow — Day 3:** Installation & Setup — Docker, Linux packages, and connecting via CLI.

[← Back to Main Guide](../README.md) | [Next: Installation →](../02-installation/README.md)
