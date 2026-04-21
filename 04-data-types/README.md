# 📖 Data Types

> **Day 5 of the ClickHouse User Guide**

Choosing the right data type is one of the most impactful decisions in ClickHouse. The right type = better compression, faster queries, less disk space.

---

## Table of Contents

- [Integer Types](#integer-types)
- [Floating Point Types](#floating-point-types)
- [Decimal Type](#decimal-type)
- [String Types](#string-types)
- [Date and Time Types](#date-and-time-types)
- [Boolean](#boolean)
- [UUID](#uuid)
- [Arrays](#arrays)
- [Nullable](#nullable)
- [LowCardinality](#lowcardinality)
- [Enum](#enum)
- [Nested](#nested)
- [Type Cheatsheet](#type-cheatsheet)

---

## Integer Types

ClickHouse has both **signed** and **unsigned** integers at multiple sizes.

| Type | Range | Storage |
|------|-------|---------|
| `Int8` | -128 to 127 | 1 byte |
| `Int16` | -32,768 to 32,767 | 2 bytes |
| `Int32` | -2.1B to 2.1B | 4 bytes |
| `Int64` | -9.2 quintillion to 9.2 quintillion | 8 bytes |
| `Int128` | Very large | 16 bytes |
| `Int256` | Extremely large | 32 bytes |
| `UInt8` | 0 to 255 | 1 byte |
| `UInt16` | 0 to 65,535 | 2 bytes |
| `UInt32` | 0 to 4.3B | 4 bytes |
| `UInt64` | 0 to 18.4 quintillion | 8 bytes |

```sql
CREATE TABLE counters
(
    id         UInt64,   -- IDs are always positive: use UInt
    age        UInt8,    -- 0-255 is enough for age
    score      Int32,    -- can be negative
    population UInt32    -- city population, always positive
)
ENGINE = MergeTree() ORDER BY id;
```

> 💡 **Rule:** Use the **smallest type** that fits your data. `UInt8` for status flags, `UInt32` for most IDs, `UInt64` for high-volume auto-increment IDs.

---

## Floating Point Types

| Type | Precision | Storage |
|------|-----------|---------|
| `Float32` | ~7 decimal digits | 4 bytes |
| `Float64` | ~15 decimal digits | 8 bytes |

```sql
SELECT
    toFloat32(3.14159265358979) AS f32,  -- 3.1415927 (loses precision)
    toFloat64(3.14159265358979) AS f64;  -- 3.14159265358979
```

> ⚠️ **Warning:** Floats have rounding errors. **Never use Float for money.** Use `Decimal` instead.

---

## Decimal Type

For exact numeric calculations (money, measurements):

```
Decimal(P, S)
  P = total significant digits (precision)
  S = digits after decimal point (scale)
```

| Type | Max Precision | Storage |
|------|--------------|---------|
| `Decimal32(S)` | 1–9 | 4 bytes |
| `Decimal64(S)` | 1–18 | 8 bytes |
| `Decimal128(S)` | 1–38 | 16 bytes |

```sql
CREATE TABLE transactions
(
    tx_id    UInt64,
    amount   Decimal(18, 2),   -- up to 9,999,999,999,999,999.99
    tax_rate Decimal(5, 4)     -- e.g. 0.1750 (17.5%)
)
ENGINE = MergeTree() ORDER BY tx_id;

INSERT INTO transactions VALUES (1, 1999.99, 0.1750);

SELECT amount * (1 + tax_rate) AS total FROM transactions;
-- Result: 2349.9882 (exact, no float errors)
```

---

## String Types

### String

Stores arbitrary-length strings. No length limit.

```sql
CREATE TABLE articles
(
    id      UInt64,
    title   String,    -- short or long, doesn't matter
    content String     -- can be megabytes
)
ENGINE = MergeTree() ORDER BY id;
```

### FixedString(N)

Fixed-length string of exactly N bytes. Faster for known-length data.

```sql
country_code FixedString(2),   -- 'PK', 'US', 'DE'
ip_address   FixedString(16)   -- IPv6 address as bytes
```

> 💡 Use `FixedString` for things like ISO country codes, currency codes, and hash values where length is always the same.

---

## Date and Time Types

| Type | Range | Storage | Precision |
|------|-------|---------|-----------|
| `Date` | 1970-01-01 to 2149-06-06 | 2 bytes | Day |
| `Date32` | 1900-01-01 to 2299-12-31 | 4 bytes | Day |
| `DateTime` | 1970-01-01 to 2106-02-07 | 4 bytes | Second |
| `DateTime64(N)` | wider range | 8 bytes | Up to nanosecond |

```sql
CREATE TABLE events
(
    event_id   UInt64,
    event_date Date,               -- just the date: 2024-01-15
    event_time DateTime,           -- date + time: 2024-01-15 09:30:00
    precise_ts DateTime64(3),      -- millisecond: 2024-01-15 09:30:00.123
    nano_ts    DateTime64(9)       -- nanosecond precision
)
ENGINE = MergeTree() ORDER BY event_date;
```

### Useful date functions

```sql
SELECT
    today(),                             -- current date
    now(),                               -- current datetime
    toYear(today()),                     -- 2024
    toMonth(today()),                    -- 1
    toDayOfWeek(today()),                -- 1=Mon, 7=Sun
    toStartOfMonth(today()),             -- 2024-01-01
    toStartOfWeek(today()),              -- start of current week
    addDays(today(), 7),                 -- 7 days from now
    dateDiff('day', '2024-01-01', today()) AS days_since_newyear;
```

---

## Boolean

ClickHouse uses `Bool` (alias for `UInt8`):

```sql
CREATE TABLE users
(
    user_id    UInt64,
    is_active  Bool,
    is_premium Bool
)
ENGINE = MergeTree() ORDER BY user_id;

INSERT INTO users VALUES (1, true, false), (2, true, true);

SELECT * FROM users WHERE is_active = true AND is_premium = false;
```

---

## UUID

Universally Unique Identifier — stored as 16 bytes (compact, no dashes in storage):

```sql
CREATE TABLE sessions
(
    session_id UUID DEFAULT generateUUIDv4(),
    user_id    UInt64,
    started_at DateTime DEFAULT now()
)
ENGINE = MergeTree() ORDER BY started_at;

INSERT INTO sessions (user_id) VALUES (42);

SELECT session_id, user_id FROM sessions;
-- Output: 550e8400-e29b-41d4-a716-446655440000  42
```

---

## Arrays

Store a list of values in a single column:

```sql
CREATE TABLE product_tags
(
    product_id UInt64,
    tags       Array(String),
    scores     Array(Float32)
)
ENGINE = MergeTree() ORDER BY product_id;

INSERT INTO product_tags VALUES
    (1, ['electronics', 'laptop', 'gaming'], [4.5, 3.8, 4.9]),
    (2, ['furniture', 'chair', 'office'],    [4.0, 4.2, 3.5]);

-- Query: find products with 'gaming' tag
SELECT product_id, tags
FROM product_tags
WHERE has(tags, 'gaming');

-- Expand array rows
SELECT product_id, arrayJoin(tags) AS tag
FROM product_tags;
```

---

## Nullable

Allows a column to store `NULL` values. Use sparingly — adds overhead.

```sql
CREATE TABLE contacts
(
    contact_id UInt64,
    email      Nullable(String),    -- email might be unknown
    phone      Nullable(String)
)
ENGINE = MergeTree() ORDER BY contact_id;

-- isNull / isNotNull functions
SELECT * FROM contacts WHERE isNotNull(email);

-- ifNull: return default if NULL
SELECT contact_id, ifNull(email, 'no-email@unknown.com') FROM contacts;
```

> ⚠️ `Nullable` adds a separate 1-bit column to track NULLs. Avoid it on high-cardinality columns or in `ORDER BY` keys.

---

## LowCardinality

A wrapper that applies **dictionary encoding** to any type. Massive performance boost for columns with few unique values.

```sql
-- Good candidates for LowCardinality
country    LowCardinality(String),   -- ~200 unique values
status     LowCardinality(String),   -- 'active', 'inactive', 'pending'
category   LowCardinality(String),   -- product categories
currency   LowCardinality(String),   -- 'USD', 'EUR', 'PKR'
```

```sql
-- Comparison: same data, different types
CREATE TABLE test_lc
(
    v1 String,
    v2 LowCardinality(String)
) ENGINE = MergeTree() ORDER BY v1;

-- v2 will use ~10x less disk space and query faster
-- when the column has < 10,000 unique values
```

> 💡 **Rule of thumb:** Use `LowCardinality` for any `String` column with fewer than 10,000 unique values.

---

## Enum

Stores strings as integers internally — great for fixed sets of values:

```sql
CREATE TABLE orders
(
    order_id UInt64,
    status   Enum8('pending'=1, 'processing'=2, 'shipped'=3, 'delivered'=4, 'cancelled'=5)
)
ENGINE = MergeTree() ORDER BY order_id;

INSERT INTO orders VALUES (1, 'pending'), (2, 'shipped');

SELECT * FROM orders WHERE status = 'shipped';
```

| Type | Max values | Storage |
|------|-----------|---------|
| `Enum8` | 256 | 1 byte |
| `Enum16` | 65,536 | 2 bytes |

> 💡 `Enum` vs `LowCardinality`: Use `Enum` when the set of values is **fixed and known at schema design time**. Use `LowCardinality` when values may change or grow.

---

## Nested

A column that stores a structured sub-table (array of structs):

```sql
CREATE TABLE orders_with_items
(
    order_id    UInt64,
    order_date  Date,
    items Nested
    (
        product_id UInt64,
        name       String,
        quantity   UInt32,
        price      Decimal(10,2)
    )
)
ENGINE = MergeTree() ORDER BY order_id;
```

> 💡 Nested columns are stored as parallel arrays internally. They're powerful but less common — consider using a separate table with a foreign key for simpler designs.

---

## Type Cheatsheet

| Use case | Recommended type |
|----------|-----------------|
| Auto-increment ID | `UInt64` |
| Age, small count | `UInt8` or `UInt16` |
| Money / price | `Decimal(18, 2)` |
| Percentage / rate | `Decimal(5, 4)` |
| GPS latitude | `Float64` |
| Date only | `Date` |
| Date + time | `DateTime` |
| Millisecond timestamp | `DateTime64(3)` |
| Short text (name, title) | `String` |
| Status / category | `LowCardinality(String)` |
| Fixed set of values | `Enum8` or `Enum16` |
| Country / currency code | `FixedString(2)` or `LowCardinality(String)` |
| UUID / GUID | `UUID` |
| Tags / multi-value | `Array(String)` |
| Optional field | `Nullable(Type)` — use sparingly |
| Flag (true/false) | `Bool` |

---

## Summary

- Use the **smallest integer type** that fits your range
- **Never use Float for money** — use `Decimal`
- `LowCardinality(String)` is a free performance boost for low-unique columns
- Avoid `Nullable` unless truly needed — it has overhead
- `Date` and `DateTime` are the workhorses of time-series data

---

> 📅 **Tomorrow — Day 6:** Table Engines — MergeTree, ReplacingMergeTree, AggregatingMergeTree and more explained clearly.

[← Getting Started](../03-getting-started/README.md) | [Back to Main Guide](../README.md) | [Next: Table Engines →](../05-table-engines/README.md)
