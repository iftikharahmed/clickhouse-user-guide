# 📖 Administration

> **Day 12 of the ClickHouse User Guide**

---

## Table of Contents

- [Users and Roles](#users-and-roles)
- [Access Control](#access-control)
- [Backups and Restore](#backups-and-restore)
- [Storage Policies and Tiering](#storage-policies-and-tiering)
- [TTL — Automatic Data Expiry](#ttl-automatic-data-expiry)
- [Quotas and Resource Limits](#quotas-and-resource-limits)
- [Monitoring](#monitoring)
- [Common Admin Tasks](#common-admin-tasks)
- [Troubleshooting](#troubleshooting)

---

## Users and Roles

### Create users

```sql
-- Create a user with password
CREATE USER analyst IDENTIFIED WITH sha256_password BY 'SecurePass123!';

-- Create a user allowed only from specific IPs
CREATE USER app_user IDENTIFIED WITH sha256_password BY 'AppPass456!'
    HOST IP '10.0.0.0/8', IP '192.168.1.0/24';

-- User that can only connect from localhost
CREATE USER readonly_user IDENTIFIED WITH sha256_password BY 'ReadPass!'
    HOST LOCAL;

-- List all users
SHOW USERS;
SELECT * FROM system.users;
```

### Create roles

```sql
-- Create roles
CREATE ROLE analyst_role;
CREATE ROLE etl_role;
CREATE ROLE admin_role;

-- Grant privileges to roles
GRANT SELECT ON analytics.* TO analyst_role;
GRANT SELECT, INSERT ON default.* TO etl_role;
GRANT ALL ON *.* TO admin_role WITH GRANT OPTION;

-- Assign roles to users
GRANT analyst_role TO analyst;
GRANT etl_role TO app_user;
GRANT admin_role TO admin_user;

-- Users can have multiple roles
GRANT analyst_role, etl_role TO power_user;

-- List roles
SHOW ROLES;
SELECT * FROM system.roles;

-- See what a user has access to
SHOW GRANTS FOR analyst;
```

### Change and drop users

```sql
-- Change password
ALTER USER analyst IDENTIFIED WITH sha256_password BY 'NewPass789!';

-- Revoke a role
REVOKE analyst_role FROM analyst;

-- Drop user
DROP USER IF EXISTS analyst;

-- Drop role
DROP ROLE IF EXISTS analyst_role;
```

---

## Access Control

### Row-level security (row policies)

```sql
-- User 'analyst' can only see their country's data
CREATE ROW POLICY pk_only ON analytics.events
    FOR SELECT USING country = 'PK'
    TO analyst;

-- Multiple conditions
CREATE ROW POLICY recent_data ON analytics.events
    FOR SELECT USING event_date >= today() - 90
    TO readonly_user;
```

### Column-level access

```sql
-- Grant access to specific columns only
GRANT SELECT(user_id, event_type, event_date) ON analytics.events TO analyst;
-- analyst CANNOT see page_url, ip_address, or other sensitive columns
```

### Profiles (settings constraints)

```sql
-- Create a settings profile
CREATE SETTINGS PROFILE analyst_limits
    SETTINGS max_memory_usage = 10000000000,   -- 10 GB max
             max_execution_time = 30,           -- 30 second timeout
             max_rows_to_read = 1000000000;    -- 1B rows max

-- Apply profile to user
ALTER USER analyst SETTINGS PROFILE 'analyst_limits';
```

---

## Backups and Restore

### Backup to local disk or S3

```sql
-- Full backup to local disk
BACKUP DATABASE analytics TO Disk('backups', 'analytics_backup_20240115');

-- Backup specific table
BACKUP TABLE analytics.events TO Disk('backups', 'events_backup_20240115');

-- Backup to S3
BACKUP DATABASE analytics
TO S3('https://my-bucket.s3.amazonaws.com/backups/20240115/', 'KEY', 'SECRET');

-- Incremental backup (only changed parts since last backup)
BACKUP DATABASE analytics
TO S3('https://my-bucket.s3.amazonaws.com/backups/20240116/', 'KEY', 'SECRET')
SETTINGS base_backup = S3('https://my-bucket.s3.amazonaws.com/backups/20240115/', 'KEY', 'SECRET');
```

### Monitor backup progress

```sql
SELECT *
FROM system.backups
ORDER BY start_time DESC;
```

### Restore from backup

```sql
-- Restore full database
RESTORE DATABASE analytics
FROM Disk('backups', 'analytics_backup_20240115');

-- Restore to a different database name
RESTORE DATABASE analytics AS analytics_restored
FROM Disk('backups', 'analytics_backup_20240115');

-- Restore specific table
RESTORE TABLE analytics.events
FROM Disk('backups', 'events_backup_20240115');
```

### Configure backup disk in config.xml

```xml
<storage_configuration>
    <disks>
        <backups>
            <type>local</type>
            <path>/var/lib/clickhouse/backups/</path>
        </backups>
    </disks>
</storage_configuration>
<backups>
    <allowed_disk>backups</allowed_disk>
</backups>
```

---

## Storage Policies and Tiering

Move cold data automatically to cheaper storage (e.g., HDD or S3):

```xml
<!-- config.xml -->
<storage_configuration>
    <disks>
        <hot>
            <type>local</type>
            <path>/mnt/ssd/clickhouse/</path>
        </hot>
        <cold>
            <type>local</type>
            <path>/mnt/hdd/clickhouse/</path>
        </cold>
        <glacier>
            <type>s3</type>
            <endpoint>https://my-bucket.s3.amazonaws.com/cold/</endpoint>
            <access_key_id>KEY</access_key_id>
            <secret_access_key>SECRET</secret_access_key>
        </glacier>
    </disks>

    <policies>
        <hot_to_cold>
            <volumes>
                <hot_volume>
                    <disk>hot</disk>
                    <max_data_part_size_bytes>1073741824</max_data_part_size_bytes> <!-- 1GB -->
                </hot_volume>
                <cold_volume>
                    <disk>cold</disk>
                </cold_volume>
            </volumes>
            <move_factor>0.2</move_factor> <!-- move when hot disk is 80% full -->
        </hot_to_cold>
    </policies>
</storage_configuration>
```

```sql
-- Apply storage policy to a table
CREATE TABLE events (...)
ENGINE = MergeTree()
ORDER BY event_date
SETTINGS storage_policy = 'hot_to_cold';
```

---

## TTL — Automatic Data Expiry

Automatically delete or move old data without manual intervention:

```sql
-- Delete rows older than 1 year
CREATE TABLE events
(
    event_id   UInt64,
    event_date Date,
    event_time DateTime
)
ENGINE = MergeTree()
ORDER BY event_date
TTL event_date + INTERVAL 1 YEAR DELETE;

-- Move to cold storage after 90 days, delete after 1 year
CREATE TABLE events
(
    event_id   UInt64,
    event_date Date
)
ENGINE = MergeTree()
ORDER BY event_date
TTL
    event_date + INTERVAL 90 DAY TO DISK 'cold',
    event_date + INTERVAL 1 YEAR DELETE;

-- Add TTL to existing table
ALTER TABLE events MODIFY TTL event_date + INTERVAL 2 YEAR DELETE;

-- Check TTL settings
SHOW CREATE TABLE events;

-- Force TTL processing (normally runs in background)
ALTER TABLE events MATERIALIZE TTL;
```

---

## Quotas and Resource Limits

Prevent any single user or query from overwhelming the server:

```sql
-- Create a quota
CREATE QUOTA analyst_quota
    FOR INTERVAL 1 hour
        MAX queries = 100,
        MAX read_rows = 1000000000,   -- 1B rows per hour
        MAX result_rows = 1000000
    FOR INTERVAL 1 day
        MAX queries = 1000
    TO analyst_role;

-- View quotas
SHOW QUOTAS;
SELECT * FROM system.quotas;

-- Check current usage
SELECT * FROM system.quota_usage;
```

---

## Monitoring

### Key system tables

```sql
-- Live running queries
SELECT
    query_id,
    user,
    elapsed,
    formatReadableSize(memory_usage) AS memory,
    read_rows,
    query
FROM system.processes
ORDER BY elapsed DESC;

-- Kill a slow/stuck query
KILL QUERY WHERE query_id = 'abc123';

-- Query history with performance stats
SELECT
    query,
    toStartOfMinute(event_time) AS minute,
    query_duration_ms            AS duration_ms,
    formatReadableSize(read_bytes) AS data_read,
    formatReadableSize(memory_usage) AS memory,
    exception
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date = today()
ORDER BY duration_ms DESC
LIMIT 20;

-- Error log
SELECT event_time, level, message
FROM system.text_log
WHERE level IN ('Error', 'Fatal')
ORDER BY event_time DESC
LIMIT 50;

-- Disk usage per table
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS compression_ratio
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC;

-- Asynchronous metrics (memory, CPU, connections)
SELECT metric, value
FROM system.asynchronous_metrics
WHERE metric LIKE '%Memory%' OR metric LIKE '%CPU%'
ORDER BY metric;
```

### Recommended Prometheus metrics to monitor

| Metric | Alert threshold |
|--------|----------------|
| `ClickHouseAsyncMetrics_MemoryResident` | > 80% of RAM |
| `ClickHouseMetrics_Query` | Sudden spike |
| `ClickHouseMetrics_BackgroundMergesAndMutationsPoolTask` | > 20 |
| `ClickHouseProfileEvents_FailedQuery` | > 0 for 5 min |
| `ClickHouseProfileEvents_ReplicatedPartFetches` | Sustained high = replica lag |

---

## Common Admin Tasks

### Check and fix too many parts

```sql
-- Too many parts = too many small inserts
SELECT table, count() AS parts
FROM system.parts
WHERE active = 1
GROUP BY table
HAVING parts > 300
ORDER BY parts DESC;

-- Force merge to reduce part count
OPTIMIZE TABLE events FINAL;          -- merge all parts
OPTIMIZE TABLE events PARTITION '202401';  -- merge one partition only
```

### Reclaim disk space

```sql
-- After DROP PARTITION or ALTER DELETE, free disk space
-- (happens automatically, but you can force it)
SYSTEM DROP MARK CACHE;
SYSTEM DROP UNCOMPRESSED CACHE;
SYSTEM RELOAD DICTIONARIES;
```

### Reload config without restart

```sql
SYSTEM RELOAD CONFIG;
```

### Restart ClickHouse (Linux)

```bash
sudo systemctl restart clickhouse-server
sudo systemctl status clickhouse-server
```

---

## Troubleshooting

### "Memory limit exceeded"

```sql
-- Increase for one query
SELECT ... SETTINGS max_memory_usage = 20000000000;  -- 20 GB

-- Check who's using the most memory
SELECT user, sum(memory_usage) AS total_mem
FROM system.processes
GROUP BY user
ORDER BY total_mem DESC;
```

### "Too many parts"

```sql
-- Optimize the problematic table
OPTIMIZE TABLE events FINAL;

-- Long-term fix: increase batch insert size, or enable async_insert
SET async_insert = 1;
```

### "Replication is lagging"

```sql
-- Check lag
SELECT replica_name, absolute_delay FROM system.replicas;

-- Force sync
SYSTEM SYNC REPLICA events;
```

### Query is slow

```sql
-- Step 1: check query log for the slow query
SELECT * FROM system.query_log WHERE query_id = 'YOUR_ID';

-- Step 2: explain the query plan
EXPLAIN SELECT ...;

-- Step 3: check if index is used
SET send_logs_level = 'trace';
SELECT ...;
-- Look for "Selected N/M granules" — if N ≈ M, index isn't helping
```

---

## Administration Checklist

For a production ClickHouse deployment, make sure you have:

- [ ] Non-default users with strong passwords
- [ ] Role-based access control configured
- [ ] Row policies for sensitive data
- [ ] TTL set on all time-series tables
- [ ] Daily automated backups to S3
- [ ] Prometheus + Grafana monitoring set up
- [ ] Alerting on memory, query failures, replication lag
- [ ] Storage tiering policy for tables > 1 TB
- [ ] Query quotas per user/role
- [ ] `max_execution_time` set globally to prevent runaway queries

---

## Summary

- Manage access with **Users, Roles, and Row Policies** — never use the `default` user in production
- **BACKUP/RESTORE** commands handle full and incremental backups natively
- **TTL** automates data expiry and cold storage tiering
- **system.processes** and **system.query_log** are your primary debugging tools
- `OPTIMIZE TABLE FINAL` fixes the "too many parts" problem but is expensive — better to fix insert patterns

---

> 🎉 **You've completed the ClickHouse User Guide!**
> 
> You now know everything from basics to production-grade administration.
> 
> If this guide helped you, please [⭐ star the repo](../README.md) — it helps others find it too!
> 
> Found an error or want to contribute? See [CONTRIBUTING.md](../CONTRIBUTING.md).

[← Integrations](../09-integrations/README.md) | [Back to Main Guide](../README.md)
