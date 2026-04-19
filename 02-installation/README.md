# 📖 Installation & Setup

> **Day 3 of the ClickHouse User Guide**

---

## Table of Contents

- [Installation Options](#installation-options)
- [Option 1: Docker (Recommended for Beginners)](#option-1-docker)
- [Option 2: Linux Package Install](#option-2-linux-package-install)
- [Option 3: ClickHouse Cloud](#option-3-clickhouse-cloud)
- [Connecting to ClickHouse](#connecting-to-clickhouse)
- [Verifying Your Installation](#verifying-your-installation)

---

## Installation Options

| Method | Best For | Effort |
|--------|----------|--------|
| Docker | Local dev, testing | ⭐ Easiest |
| Linux package | Production servers | Medium |
| ClickHouse Cloud | Managed, no ops | Zero setup |
| Kubernetes (Helm) | Scalable production | Advanced |

---

## Option 1: Docker

### Single command setup

```bash
docker run -d \
  --name clickhouse \
  --ulimit nofile=262144:262144 \
  -p 8123:8123 \
  -p 9000:9000 \
  clickhouse/clickhouse-server:latest
```

| Port | Protocol | Used by |
|------|----------|---------|
| `8123` | HTTP | REST API, web UI, drivers |
| `9000` | TCP native | clickhouse-client, high-perf drivers |

### With persistent data (recommended)

```bash
docker run -d \
  --name clickhouse \
  --ulimit nofile=262144:262144 \
  -p 8123:8123 \
  -p 9000:9000 \
  -v $(pwd)/clickhouse-data:/var/lib/clickhouse \
  -v $(pwd)/clickhouse-logs:/var/log/clickhouse-server \
  clickhouse/clickhouse-server:latest
```

> 💡 Without a volume, your data is lost when the container stops.

### Docker Compose setup

```yaml
version: "3.8"
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse-data:/var/lib/clickhouse
      - clickhouse-logs:/var/log/clickhouse-server
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    environment:
      CLICKHOUSE_DB: mydb
      CLICKHOUSE_USER: myuser
      CLICKHOUSE_PASSWORD: mypassword
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1

volumes:
  clickhouse-data:
  clickhouse-logs:
```

```bash
docker compose up -d    # start
docker compose down     # stop
```

---

## Option 2: Linux Package Install

### Ubuntu / Debian

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] \
  https://packages.clickhouse.com/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/clickhouse.list

sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
sudo systemctl enable --now clickhouse-server
```

### RHEL / CentOS / Fedora

```bash
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
sudo yum install -y clickhouse-server clickhouse-client
sudo systemctl enable --now clickhouse-server
```

### Check service status

```bash
sudo systemctl status clickhouse-server
# Expected: Active: active (running)
```

---

## Option 3: ClickHouse Cloud

1. Go to [clickhouse.cloud](https://clickhouse.cloud)
2. Sign up (free trial available)
3. Create a service — pick the region closest to you
4. Copy the credentials and use them in any connection method below

> 💡 Best for production — no server management, backups, or upgrades needed.

---

## Connecting to ClickHouse

### Method 1: clickhouse-client (CLI)

```bash
# Docker
docker exec -it clickhouse clickhouse-client

# With credentials
docker exec -it clickhouse clickhouse-client \
  --user myuser --password mypassword --database mydb
```

You'll see:
```
ClickHouse client version 24.x.x
clickhouse :)
```

### Method 2: HTTP API (curl)

```bash
# Simple GET
curl "http://localhost:8123/?query=SELECT+version()"

# POST for longer queries
curl -X POST "http://localhost:8123/" \
  -d "SELECT name, database FROM system.tables LIMIT 5"

# With auth
curl "http://localhost:8123/?user=myuser&password=mypassword" \
  -d "SELECT version()"
```

### Method 3: Play UI (built-in web editor)

Open in browser:
```
http://localhost:8123/play
```

### Method 4: DBeaver (GUI)

1. New Connection → ClickHouse
2. Host: `localhost`, Port: `8123`
3. User: `default`, Password: (empty by default)
4. Test → Finish

---

## Security Setup

```sql
-- Create a user with a password
CREATE USER myuser IDENTIFIED WITH sha256_password BY 'StrongPassword123';

-- Grant all privileges
GRANT ALL ON *.* TO myuser WITH GRANT OPTION;
```

> ⚠️ **Warning:** Never expose ClickHouse ports to the internet without authentication and a firewall.

---

## Verifying Your Installation

```sql
SELECT version();                    -- ClickHouse version
SELECT uptime();                     -- Server uptime in seconds
SHOW DATABASES;                      -- List all databases

-- Quick benchmark — should complete in < 1 second
SELECT count() FROM numbers(1000000000);
-- Expected: 1000000000 rows in ~0.2 sec
```

---

## Summary

- **Docker** is the fastest way to get started
- **Linux packages** are best for production server installs
- **ClickHouse Cloud** is fully managed — no ops required
- Always set a password and restrict network access in production
- Connect via CLI, HTTP API, the built-in Play UI, or DBeaver

---

> 📅 **Tomorrow — Day 4:** Getting Started — Create your first database, table, insert data, and run your first queries.

[← Introduction](../01-introduction/README.md) | [Back to Main Guide](../README.md) | [Next: Getting Started →](../03-getting-started/README.md)
