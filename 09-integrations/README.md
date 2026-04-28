# 📖 Integrations

> **Day 11 of the ClickHouse User Guide**

---

## Table of Contents

- [Python](#python)
- [Kafka](#kafka)
- [Amazon S3](#amazon-s3)
- [PostgreSQL](#postgresql)
- [MySQL](#mysql)
- [dbt](#dbt)
- [Grafana](#grafana)
- [Node.js](#nodejs)
- [JDBC / BI Tools](#jdbc--bi-tools)

---

## Python

### clickhouse-connect (official, recommended)

```bash
pip install clickhouse-connect
```

```python
import clickhouse_connect

# Connect
client = clickhouse_connect.get_client(
    host='localhost',
    port=8123,
    username='default',
    password='',
    database='default'
)

# Simple query
result = client.query('SELECT version()')
print(result.result_rows)

# Query to DataFrame (pandas)
import pandas as pd
df = client.query_df('SELECT * FROM events LIMIT 1000')
print(df.head())

# Insert a DataFrame
import pandas as pd
df = pd.DataFrame({
    'user_id': [1, 2, 3],
    'event_type': ['view', 'click', 'purchase'],
    'event_date': ['2024-01-15', '2024-01-15', '2024-01-15']
})
client.insert_df('events', df)

# Insert raw rows
data = [[1, 'Alice', 'alice@example.com'], [2, 'Bob', 'bob@example.com']]
client.insert('users', data, column_names=['id', 'name', 'email'])

# Parameterized query (safe from SQL injection)
result = client.query(
    'SELECT * FROM events WHERE user_id = {uid:UInt64}',
    parameters={'uid': 42}
)
```

### clickhouse-driver (native TCP protocol)

```bash
pip install clickhouse-driver
```

```python
from clickhouse_driver import Client

client = Client(host='localhost', port=9000)

# Execute query
rows = client.execute('SELECT count() FROM events')
print(rows)  # [(1234567,)]

# Insert many rows efficiently
rows = [(i, f'event_{i}', '2024-01-15') for i in range(100000)]
client.execute(
    'INSERT INTO events (event_id, event_type, event_date) VALUES',
    rows
)

# Stream large results (memory efficient)
for row in client.execute_iter('SELECT * FROM events', settings={'max_block_size': 10000}):
    process(row)
```

---

## Kafka

ClickHouse can **consume directly from Kafka topics** using the Kafka table engine.

### Setup

```sql
-- 1. Create a Kafka engine table (reads from topic)
CREATE TABLE kafka_events_queue
(
    user_id    UInt64,
    event_type String,
    event_time DateTime,
    page_url   String
)
ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list  = 'user-events',
    kafka_group_name  = 'clickhouse-consumer',
    kafka_format      = 'JSONEachRow';

-- 2. Create the target storage table
CREATE TABLE events
(
    user_id    UInt64,
    event_type LowCardinality(String),
    event_time DateTime,
    event_date Date DEFAULT toDate(event_time),
    page_url   String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);

-- 3. Create a Materialized View to bridge them
CREATE MATERIALIZED VIEW kafka_events_mv TO events AS
SELECT
    user_id,
    event_type,
    event_time,
    page_url
FROM kafka_events_queue;
```

### How it works

```
Kafka Topic → Kafka Engine Table → Materialized View → MergeTree Table
              (reads messages)      (transforms)        (stores data)
```

Data flows automatically — ClickHouse polls Kafka and inserts into your table.

### Supported formats

```sql
kafka_format = 'JSONEachRow'    -- {"field": "value"} per line
kafka_format = 'Avro'           -- Avro binary
kafka_format = 'CSV'            -- CSV
kafka_format = 'Protobuf'       -- Protocol Buffers
```

### Monitor Kafka consumption

```sql
SELECT * FROM system.kafka_consumers;
```

---

## Amazon S3

ClickHouse can **read and write directly from/to S3** without loading data locally.

### Read from S3

```sql
-- Read a single file
SELECT * FROM s3(
    'https://my-bucket.s3.amazonaws.com/data/events.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
);

-- Read with wildcard (multiple files)
SELECT count() FROM s3(
    'https://my-bucket.s3.amazonaws.com/events/2024/01/*.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
);

-- With schema inference
DESCRIBE TABLE s3('https://my-bucket.s3.amazonaws.com/data.csv', 'CSV');
```

### Write to S3

```sql
-- Export query result to S3
INSERT INTO FUNCTION s3(
    'https://my-bucket.s3.amazonaws.com/exports/events.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
)
SELECT * FROM events WHERE event_date = '2024-01-15';
```

### S3 as a table engine (data lake)

```sql
CREATE TABLE s3_events
(
    user_id    UInt64,
    event_type String,
    event_date Date
)
ENGINE = S3(
    'https://my-bucket.s3.amazonaws.com/events/*.parquet',
    'ACCESS_KEY_ID',
    'SECRET_ACCESS_KEY',
    'Parquet'
);

-- Now query S3 like a normal table
SELECT count(), uniq(user_id)
FROM s3_events
WHERE event_date >= '2024-01-01';
```

### Store credentials in config (recommended)

```xml
<!-- /etc/clickhouse-server/config.xml -->
<s3>
    <my_bucket>
        <endpoint>https://my-bucket.s3.amazonaws.com</endpoint>
        <access_key_id>ACCESS_KEY_ID</access_key_id>
        <secret_access_key>SECRET_ACCESS_KEY</secret_access_key>
    </my_bucket>
</s3>
```

---

## PostgreSQL

### Read from PostgreSQL

```sql
-- One-time query
SELECT * FROM postgresql(
    'postgres-host:5432',
    'mydb',
    'users',
    'pg_user',
    'pg_password'
);

-- Create a permanent table backed by PostgreSQL
CREATE TABLE pg_users
(
    id    UInt64,
    name  String,
    email String
)
ENGINE = PostgreSQL(
    'postgres-host:5432',
    'mydb',
    'users',
    'pg_user',
    'pg_password'
);

-- JOIN ClickHouse data with PostgreSQL data
SELECT
    e.user_id,
    u.name,
    count() AS event_count
FROM events e
JOIN pg_users u ON e.user_id = u.id
GROUP BY e.user_id, u.name
ORDER BY event_count DESC;
```

### Replicate from PostgreSQL (CDC)

Use the `MaterializedPostgreSQL` engine for real-time replication via logical replication:

```sql
CREATE DATABASE pg_replica
ENGINE = MaterializedPostgreSQL(
    'postgres-host:5432',
    'source_db',
    'pg_user',
    'pg_password'
)
SETTINGS materialized_postgresql_tables_list = 'users,orders,products';
```

---

## MySQL

```sql
-- Query MySQL directly
SELECT * FROM mysql(
    'mysql-host:3306',
    'mydb',
    'orders',
    'mysql_user',
    'mysql_password'
);

-- Permanent MySQL-backed table
CREATE TABLE mysql_orders
ENGINE = MySQL(
    'mysql-host:3306',
    'mydb',
    'orders',
    'mysql_user',
    'mysql_password'
);
```

---

## dbt

[dbt-clickhouse](https://github.com/ClickHouse/dbt-clickhouse) is the official adapter.

```bash
pip install dbt-clickhouse
```

### profiles.yml

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: clickhouse
      host: localhost
      port: 8123
      user: default
      password: ""
      database: analytics
      schema: analytics
```

### dbt model example

```sql
-- models/marts/daily_revenue.sql
{{ config(
    materialized='incremental',
    engine='MergeTree()',
    order_by='(order_date)',
    partition_by='toYYYYMM(order_date)',
    unique_key='order_date'
) }}

SELECT
    order_date,
    sum(price * quantity) AS revenue,
    count()               AS total_orders,
    uniq(customer_id)     AS unique_customers
FROM {{ ref('stg_orders') }}
{% if is_incremental() %}
WHERE order_date >= (SELECT max(order_date) FROM {{ this }})
{% endif %}
GROUP BY order_date
```

```bash
dbt run --select daily_revenue
dbt test
```

---

## Grafana

1. Install the **ClickHouse plugin** in Grafana:
   ```
   grafana-cli plugins install grafana-clickhouse-datasource
   ```
2. Add data source: ClickHouse → `http://localhost:8123`
3. Use SQL queries in dashboards:

```sql
-- Time-series panel: events per minute
SELECT
    toStartOfMinute(event_time) AS time,
    count()                     AS events
FROM events
WHERE $__timeFilter(event_time)
GROUP BY time
ORDER BY time;
```

> 💡 Use `$__timeFilter(column)` — Grafana replaces this with the dashboard's time range automatically.

---

## Node.js

```bash
npm install @clickhouse/client
```

```javascript
import { createClient } from '@clickhouse/client';

const client = createClient({
    host: 'http://localhost:8123',
    username: 'default',
    password: '',
    database: 'default',
});

// Query
const result = await client.query({
    query: 'SELECT * FROM events WHERE event_date = today() LIMIT 100',
    format: 'JSONEachRow',
});
const rows = await result.json();
console.log(rows);

// Insert
await client.insert({
    table: 'events',
    values: [
        { user_id: 1, event_type: 'view', event_date: '2024-01-15' },
        { user_id: 2, event_type: 'click', event_date: '2024-01-15' },
    ],
    format: 'JSONEachRow',
});

await client.close();
```

---

## JDBC / BI Tools

| Tool | How to connect |
|------|---------------|
| **DBeaver** | New Connection → ClickHouse → host/port/db |
| **Tableau** | Use JDBC driver from [clickhouse.com/docs](https://clickhouse.com/docs) |
| **Power BI** | ODBC driver → ClickHouse DSN |
| **Metabase** | Add database → ClickHouse |
| **Apache Superset** | `pip install clickhouse-connect` → ClickHouse dialect |
| **Redash** | ClickHouse data source built-in |

---

## Summary

- **Python**: use `clickhouse-connect` (HTTP) or `clickhouse-driver` (TCP)
- **Kafka**: Kafka engine + Materialized View = real-time streaming pipeline
- **S3**: query data lake files without ETL using the `s3()` function
- **PostgreSQL/MySQL**: query external databases directly or replicate via CDC
- **dbt**: use `dbt-clickhouse` for SQL-based transformation pipelines
- **Grafana**: official plugin for building real-time dashboards

---

> 📅 **Tomorrow — Day 12:** Administration — users, roles, backups, monitoring, and keeping ClickHouse healthy in production.

[← Replication](../08-replication-clustering/README.md) | [Back to Main Guide](../README.md) | [Next: Administration →](../10-administration/README.md)
