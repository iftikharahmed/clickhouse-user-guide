# 📖 Queries — SELECT, JOINs, Aggregations & More

> **Day 7 of the ClickHouse User Guide**

---

## Table of Contents

- [SELECT Basics](#select-basics)
- [Filtering with WHERE](#filtering-with-where)
- [GROUP BY and Aggregations](#group-by-and-aggregations)
- [ORDER BY and LIMIT](#order-by-and-limit)
- [JOINs](#joins)
- [Subqueries](#subqueries)
- [CTEs (WITH clause)](#ctes-with-clause)
- [Window Functions](#window-functions)
- [Array Functions](#array-functions)
- [String Functions](#string-functions)
- [Conditional Expressions](#conditional-expressions)
- [Query Clauses Order](#query-clauses-order)

---

## SELECT Basics

```sql
-- All columns
SELECT * FROM events LIMIT 10;

-- Specific columns with aliases
SELECT
    user_id,
    event_type                        AS type,
    toDate(event_time)                AS date,
    formatDateTime(event_time, '%H')  AS hour
FROM events
LIMIT 10;

-- Expressions in SELECT
SELECT
    price,
    quantity,
    price * quantity                           AS subtotal,
    price * quantity * 1.17                    AS subtotal_with_tax,
    round(price * quantity * 1.17, 2)          AS subtotal_rounded
FROM orders;
```

---

## Filtering with WHERE

```sql
-- Basic comparison
SELECT * FROM orders WHERE price > 100;

-- Multiple conditions
SELECT * FROM orders
WHERE category = 'Electronics'
  AND price BETWEEN 100 AND 500
  AND order_date >= '2024-01-01';

-- IN list
SELECT * FROM orders
WHERE country IN ('PK', 'US', 'DE');

-- NOT IN
SELECT * FROM orders
WHERE status NOT IN ('cancelled', 'refunded');

-- LIKE pattern matching
SELECT * FROM products WHERE name LIKE '%laptop%';
SELECT * FROM products WHERE name ILIKE '%LAPTOP%';  -- case-insensitive

-- NULL checks
SELECT * FROM contacts WHERE email IS NOT NULL;
SELECT * FROM contacts WHERE phone IS NULL;

-- Date range (very common pattern)
SELECT * FROM events
WHERE event_date >= today() - 7    -- last 7 days
  AND event_date < today();

-- Using toDate for datetime columns
SELECT * FROM logs
WHERE toDate(created_at) = today();
```

---

## GROUP BY and Aggregations

### Basic aggregations

```sql
SELECT
    category,
    count()                           AS total_orders,
    sum(price * quantity)             AS revenue,
    avg(price)                        AS avg_price,
    min(price)                        AS min_price,
    max(price)                        AS max_price,
    uniq(customer_id)                 AS unique_customers
FROM orders
GROUP BY category
ORDER BY revenue DESC;
```

### ClickHouse-specific aggregate functions

```sql
-- uniq: fast approximate unique count (HyperLogLog)
SELECT uniq(user_id) FROM events;

-- uniqExact: exact unique count (slower, more memory)
SELECT uniqExact(user_id) FROM events;

-- countIf: count with a condition
SELECT
    countIf(status = 'completed')  AS completed,
    countIf(status = 'cancelled')  AS cancelled,
    countIf(status = 'pending')    AS pending
FROM orders;

-- sumIf, avgIf: conditional aggregations
SELECT
    sumIf(revenue, country = 'PK')   AS pk_revenue,
    avgIf(price, category = 'Electronics') AS avg_electronics_price
FROM orders;

-- quantile: percentiles
SELECT
    quantile(0.50)(response_ms) AS p50,
    quantile(0.95)(response_ms) AS p95,
    quantile(0.99)(response_ms) AS p99
FROM api_logs;

-- groupArray: collect values into an array
SELECT user_id, groupArray(product_name) AS bought_products
FROM orders
GROUP BY user_id;

-- argMax: value of column A when column B is maximum
SELECT
    user_id,
    argMax(product_name, order_date) AS most_recent_purchase
FROM orders
GROUP BY user_id;
```

### HAVING (filter after aggregation)

```sql
SELECT
    customer_id,
    count()          AS total_orders,
    sum(price)       AS total_spent
FROM orders
GROUP BY customer_id
HAVING total_orders >= 5       -- only customers with 5+ orders
   AND total_spent > 1000
ORDER BY total_spent DESC;
```

---

## ORDER BY and LIMIT

```sql
-- Basic sort
SELECT * FROM orders ORDER BY price DESC LIMIT 10;

-- Multi-column sort
SELECT * FROM events
ORDER BY event_date DESC, user_id ASC
LIMIT 100;

-- LIMIT OFFSET (pagination — use with caution on large tables)
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;   -- page 3, 20 items per page

-- LIMIT BY: top N per group
SELECT *
FROM orders
ORDER BY price DESC
LIMIT 3 BY customer_id;   -- top 3 orders per customer
```

---

## JOINs

ClickHouse supports standard JOIN types. The **right table** is loaded into memory, so it should be the smaller one.

```sql
-- INNER JOIN
SELECT
    o.order_id,
    o.product_name,
    c.name AS customer_name,
    c.country
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;

-- LEFT JOIN
SELECT
    c.customer_id,
    c.name,
    count(o.order_id) AS total_orders
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;

-- USING (shorthand when column names match)
SELECT * FROM orders
JOIN customers USING (customer_id);
```

### JOIN performance tips

```sql
-- Put the LARGER table on the LEFT
-- Put the SMALLER table on the RIGHT (loaded into memory)

-- Use IN instead of JOIN for simple lookups
SELECT * FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM customers WHERE country = 'PK'
);

-- Semi-join using EXISTS
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM vip_customers v WHERE v.customer_id = o.customer_id
);
```

> ⚠️ ClickHouse JOINs are **not** as optimized as PostgreSQL's. If the right table is large (>10M rows), consider using a Dictionary or Distributed JOIN.

---

## Subqueries

```sql
-- Scalar subquery
SELECT
    order_id,
    price,
    price - (SELECT avg(price) FROM orders) AS diff_from_avg
FROM orders;

-- IN subquery
SELECT * FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM customers
    WHERE signup_date >= '2024-01-01'
);

-- Subquery in FROM
SELECT
    country,
    avg(order_count) AS avg_orders_per_customer
FROM (
    SELECT customer_id, country, count() AS order_count
    FROM orders
    JOIN customers USING (customer_id)
    GROUP BY customer_id, country
) AS customer_stats
GROUP BY country;
```

---

## CTEs (WITH clause)

CTEs make complex queries readable by naming intermediate steps:

```sql
-- Simple CTE
WITH high_value_customers AS (
    SELECT customer_id
    FROM orders
    GROUP BY customer_id
    HAVING sum(price) > 5000
)
SELECT c.name, c.email
FROM customers c
WHERE c.customer_id IN (SELECT customer_id FROM high_value_customers);

-- Multiple CTEs
WITH
monthly_revenue AS (
    SELECT
        toStartOfMonth(order_date) AS month,
        sum(price * quantity)      AS revenue
    FROM orders
    GROUP BY month
),
prev_month AS (
    SELECT
        month,
        revenue,
        lagInFrame(revenue) OVER (ORDER BY month) AS prev_revenue
    FROM monthly_revenue
)
SELECT
    month,
    revenue,
    prev_revenue,
    round(100 * (revenue - prev_revenue) / prev_revenue, 2) AS growth_pct
FROM prev_month
ORDER BY month;
```

---

## Window Functions

Window functions perform calculations across rows related to the current row — without collapsing them like GROUP BY does.

```sql
-- Running total
SELECT
    order_date,
    revenue,
    sum(revenue) OVER (ORDER BY order_date) AS running_total
FROM daily_revenue;

-- Rank within group
SELECT
    category,
    product_name,
    revenue,
    rank() OVER (PARTITION BY category ORDER BY revenue DESC) AS rank_in_category
FROM product_revenue;

-- Moving average (last 7 days)
SELECT
    date,
    daily_visits,
    avg(daily_visits) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_stats;

-- Row number
SELECT
    row_number() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_seq,
    customer_id,
    order_date,
    price
FROM orders;

-- LAG / LEAD: compare to previous/next row
SELECT
    date,
    revenue,
    lagInFrame(revenue) OVER (ORDER BY date)  AS prev_day_revenue,
    leadInFrame(revenue) OVER (ORDER BY date) AS next_day_revenue
FROM daily_revenue;

-- First and last value in window
SELECT
    user_id,
    event_time,
    firstValue(event_type) OVER (PARTITION BY user_id ORDER BY event_time) AS first_event,
    lastValue(event_type)  OVER (PARTITION BY user_id ORDER BY event_time) AS latest_event
FROM events;
```

---

## Array Functions

```sql
-- has: check if array contains a value
SELECT * FROM products WHERE has(tags, 'sale');

-- length: array size
SELECT user_id, length(purchased_items) AS items_count FROM carts;

-- arrayJoin: flatten array into rows
SELECT user_id, arrayJoin(tags) AS tag FROM products;

-- arrayFilter: filter elements
SELECT arrayFilter(x -> x > 100, [50, 150, 200, 80]) AS filtered;
-- Result: [150, 200]

-- arrayMap: transform elements
SELECT arrayMap(x -> x * 2, [1, 2, 3, 4]) AS doubled;
-- Result: [2, 4, 6, 8]

-- arraySum, arrayAvg
SELECT arraySum([10, 20, 30]) AS total;    -- 60
SELECT arrayAvg([10, 20, 30]) AS average;  -- 20

-- indexOf: find position of element
SELECT indexOf(['a', 'b', 'c'], 'b');  -- 2
```

---

## String Functions

```sql
SELECT
    lower('Hello World'),          -- hello world
    upper('hello'),                -- HELLO
    length('clickhouse'),          -- 10
    trim('  hello  '),             -- hello
    trimLeft('  hello'),           -- hello
    trimRight('hello  '),          -- hello
    substring('clickhouse', 1, 5), -- click  (1-indexed)
    concat('Hello', ', ', 'World'), -- Hello, World
    replace('foo bar', 'bar', 'baz'), -- foo baz
    splitByChar(',', 'a,b,c'),     -- ['a','b','c']
    toString(12345),               -- '12345'
    toInt32OrNull('abc')           -- NULL (safe conversion)
FROM system.one;

-- String matching
SELECT * FROM logs WHERE match(url, '^/api/v[0-9]+/');  -- regex match
SELECT * FROM products WHERE multiSearchAny(name, ['laptop', 'notebook', 'computer']);
```

---

## Conditional Expressions

```sql
-- IF
SELECT if(price > 100, 'expensive', 'cheap') AS tier FROM products;

-- CASE WHEN
SELECT
    order_id,
    CASE
        WHEN status = 'delivered' THEN 'Complete'
        WHEN status IN ('shipped', 'processing') THEN 'In Progress'
        WHEN status = 'cancelled' THEN 'Cancelled'
        ELSE 'Unknown'
    END AS status_label
FROM orders;

-- ifNull: return default if NULL
SELECT ifNull(email, 'unknown@email.com') FROM users;

-- nullIf: return NULL if condition is true
SELECT nullIf(quantity, 0) AS qty FROM inventory;  -- 0 becomes NULL

-- coalesce: first non-NULL value
SELECT coalesce(mobile, home_phone, work_phone, 'no-phone') AS contact
FROM contacts;
```

---

## Query Clauses Order

ClickHouse (like SQL) has a strict clause order:

```sql
SELECT   [DISTINCT] columns
FROM     table
[JOIN    other_table ON condition]
WHERE    row_filter
GROUP BY columns
HAVING   group_filter
ORDER BY columns [ASC|DESC]
LIMIT    [offset,] count
```

Execution order (different from written order):

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

> 💡 This is why you **cannot** use a SELECT alias in a WHERE clause — WHERE runs before SELECT. But you can use aliases in ORDER BY and HAVING.

---

## Summary

- ClickHouse supports full SQL with powerful extensions like `uniq`, `countIf`, `quantile`, `argMax`
- Always put the **larger table on the left** in JOINs
- Use **CTEs** to break complex queries into readable steps
- **Window functions** allow row-level analysis without GROUP BY collapsing
- `LIMIT N BY column` is a ClickHouse superpower for top-N-per-group queries

---

> 📅 **Tomorrow — Day 9:** Performance & Indexing — how to make your queries 10x faster with the right index strategy.

[← Table Engines](../05-table-engines/README.md) | [Back to Main Guide](../README.md) | [Next: Performance →](../07-performance/README.md)
