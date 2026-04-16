# 🚀 ClickHouse User Guide

> A concise, practical, and beginner-friendly guide to mastering ClickHouse — the fastest open-source OLAP database in the world.

[![ClickHouse](https://img.shields.io/badge/ClickHouse-FFCC01?style=for-the-badge&logo=clickhouse&logoColor=black)](https://clickhouse.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=for-the-badge)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=for-the-badge)](CONTRIBUTING.md)
[![Stars](https://img.shields.io/github/stars/iftikharahmed/clickhouse-user-guide?style=for-the-badge&color=yellow)](https://github.com/iftikharahmed/clickhouse-user-guide/stargazers)

## 🎯 Who Is This For?

| You are... | You'll learn... |
|-----------|----------------|
| A SQL developer | How to write blazing-fast analytical queries |
| A backend engineer | How to integrate ClickHouse into your stack |
| A data engineer | How to ingest, store, and query billions of rows |
| A complete beginner | Everything from installation to production |

---

## 📚 Table of Contents

| # | Section | Description |
|---|---------|-------------|
| 01 | [Introduction to ClickHouse](01-introduction/README.md) | What it is, how it works, when to use it |
| 02 | [Installation & Setup](02-installation/README.md) | Docker, Linux packages, ClickHouse Cloud |
| 03 | [Getting Started](03-getting-started/README.md) | First DB, tables, INSERT, SELECT |
| 04 | [Data Types](04-data-types/README.md) | Complete data types cheatsheet |
| 05 | [Table Engines](05-table-engines/README.md) | MergeTree, ReplacingMergeTree, and more |
| 06 | [Queries](06-queries/README.md) | SELECT, JOINs, Window Functions, CTEs |
| 07 | [Performance & Indexing](07-performance/README.md) | Indexes, partitioning, materialized views |
| 08 | [Replication & Clustering](08-replication-clustering/README.md) | HA setup, sharding, distributed tables |
| 09 | [Integrations](09-integrations/README.md) | Kafka, S3, Python, dbt, Grafana |
| 10 | [Administration](10-administration/README.md) | Users, backups, monitoring, troubleshooting |

---

## ⚡ What is ClickHouse?

ClickHouse is an **open-source column-oriented database** designed for **real-time analytics** at massive scale.

```sql
-- Query 1 billion rows in under a second
SELECT
    toStartOfHour(event_time) AS hour,
    count()                   AS events,
    uniq(user_id)             AS unique_users
FROM events
WHERE event_date = today()
GROUP BY hour
ORDER BY hour;
```

> ✅ Result: **1,000,000,000 rows scanned in 0.8 seconds**

---

## 🏆 Why ClickHouse?

- **100x faster** than traditional row-oriented databases for analytics
- **Column-oriented storage** — reads only the columns you need
- **Vectorized query execution** — processes data in CPU-friendly batches
- **Massive compression** — stores data 5–10x more efficiently
- **Linear scalability** — add nodes, get more speed
- **SQL interface** — no new query language to learn

---

## 🗺️ Quick Architecture Overview

```
┌─────────────────────────────────────────────┐
│              ClickHouse Server               │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Column  │  │  Column  │  │  Column  │  │
│  │  Store   │  │  Store   │  │  Store   │  │
│  │  (id)    │  │  (name)  │  │  (date)  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │     MergeTree Storage Engine        │    │
│  │  (Sorting + Indexing + Compression) │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

---

## 🚀 Quick Start (30 seconds)

```bash
# Run ClickHouse with Docker
docker run -d \
  --name clickhouse \
  -p 8123:8123 \
  -p 9000:9000 \
  clickhouse/clickhouse-server

# Connect via CLI
docker exec -it clickhouse clickhouse-client

# Your first query
SELECT 'Hello, ClickHouse!' AS message;
```

---

## 🤝 Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.  
Found a mistake? Open an issue. Have an improvement? Send a PR!

---

## 📄 License

MIT © [iftikharahmed](https://github.com/iftikharahmed)

---

<div align="center">

**📅 New content added daily — [⭐ Star this repo](https://github.com/iftikharahmed/clickhouse-user-guide) to follow along!**

</div>
