# SQL Project Mastery

> **👋 Hey Fresher — Read This First!**

> - SQL (Structured Query Language) is the **universal language for all relational databases** — MySQL, PostgreSQL, SQL Server, Oracle, SQLite, Azure SQL, Amazon RDS. Learn SQL once, work with any of them.
> - SQL is not just for "database admins." Every backend developer, data analyst, data engineer, DevOps engineer, and cloud engineer uses SQL **every single working day.**
> - Every block in this document covers exactly one concept. Every explanation is bullet points. Query results are shown in table previews so you see what each query returns.
> - **Company in this project:** DataPulse Analytics — a Hyderabad analytics startup building a SaaS product for Indian e-commerce companies. They track orders, products, customers, and sales performance. You just joined as a Junior Data/Backend Engineer. Your lead is Ananya. You will write real SQL queries that power DataPulse's dashboards and backend APIs.
> - **Database used:** PostgreSQL 16 — the most popular open-source SQL database and the default for most modern projects. Every query also works in MySQL/MariaDB with minor syntax differences noted where they differ.

📦 Where SQL Lives in a Real Project Tech Stack

React Frontend → Node.js / Django / Spring API → PostgreSQL / MySQL (SQL) → AWS RDS / Azure SQL

Kafka / RabbitMQ → Python ETL → PostgreSQL Warehouse (SQL) → Grafana / Metabase

Django ORM / SQLAlchemy → SQL Queries (under the hood) → PostgreSQL → Redis Cache

SQL sits at the heart of nearly every production system. Even when you use an ORM (Django/SQLAlchemy/Hibernate), it generates SQL. Understanding raw SQL means you can debug slow queries, optimise ORM performance, and write reports that ORMs can't generate efficiently.

#### What You Will Learn in This Project

> **🏗️ Phase 1 — Tables & Basic Queries**

> Create tables, insert data, SELECT with WHERE filters, ORDER BY, LIMIT. The foundation of every SQL query you'll ever write.

> Combine data from multiple tables using INNER, LEFT, RIGHT, and FULL joins. Understand foreign keys and referential integrity.

> COUNT, SUM, AVG, MIN, MAX. GROUP BY for category summaries. HAVING to filter groups. Power the DataPulse sales dashboard.

> Queries inside queries. WITH clauses for readable complex logic. ROW_NUMBER, RANK, LAG, LEAD for analytics — used in every data engineering role.

> Make queries fast with indexes. Ensure data safety with transactions. Reuse complex logic with stored procedures. Production-grade database skills.

CREATE TABLE, SELECT / WHERE, JOIN, GROUP BY, Subqueries, Window Functions, Indexes, Transactions

**Scene 1 — DataPulse Analytics, Hyderabad | First Sprint Task**

> **Ananya** _Senior Data Engineer — DataPulse Analytics_
> 
> Rishi, our client FlipMart just onboarded. They need a sales dashboard showing daily revenue, top-selling products, and customer retention. All of their transactional data sits in PostgreSQL — orders, customers, products, order_items. Your job is to write the SQL queries that power every metric on that dashboard. The frontend engineer is React. The backend is Django. But the real data logic — joins, aggregations, window functions — that is 100% SQL. Every query you write today will be called from the Django REST API hundreds of times a day.

> **Rishi (You)** _Junior Data Engineer — Day 1 at DataPulse_
> 
> I know basic SELECT statements but I've never written JOINs across four tables or used window functions. Will I need to understand the Django ORM too?

> **Ananya** _Senior Data Engineer_
> 
> The ORM is just a wrapper over SQL. When Django's ORM generates a slow query, you need to understand the SQL it produces to fix it. When a client asks "why is the daily revenue report wrong?", you will write raw SQL to debug it — not Python. SQL is the only language that talks directly to the truth. Everything else — ORMs, dashboards, APIs — just translates between your code and SQL. Master SQL and every layer above it becomes easier to debug.

> **Vikram** _Backend Lead — DataPulse Analytics_
> 
> And one more thing — SQL performance matters at scale. A query that takes 50ms for 1,000 orders takes 3 minutes for 10 million orders if there's no index. DataPulse clients have 2 to 50 million records. You'll learn indexes and query optimisation this week — not as theory but as fixes for real slow queries that you'll encounter in the FlipMart dataset.

```
DataPulse — FlipMart PostgreSQL Schema
=========================================

  customers          products              orders              order_items
  ──────────────     ──────────────────    ───────────────     ────────────────────
  id (PK)            id (PK)               id (PK)             id (PK)
  name               name                  customer_id (FK)    order_id (FK)
  email              category              status              product_id (FK)
  city               price                 total_amount        quantity
  created_at         stock_qty             created_at          unit_price
                     is_active             delivered_at

  Stack:  React → Django REST API → PostgreSQL → AWS RDS (prod)
  Tools:  psql CLI, pgAdmin, DBeaver, Django ORM, SQLAlchemy
  Used by: Dashboard queries (Metabase), API queries (Django), ETL (Python + pandas)
```

### 1. Phase 1 — Creating Tables and Basic Queries

🔧 Where This Phase Is Used in the Stack

Django Migrations → CREATE TABLE (SQL DDL) → PostgreSQL

Django ORM .objects.filter() → SELECT + WHERE (SQL DQL) → Returns Python objects

DDL (Data Definition Language) = CREATE/ALTER/DROP tables. DML = INSERT/UPDATE/DELETE data. DQL = SELECT to read data. Every Django migration generates DDL SQL. Every ORM query generates DQL SQL.

#### 1.1 CREATE TABLE — Define the Schema

```
CREATE TABLE customers (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(150) UNIQUE NOT NULL,
  city VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW()
);
```

**📖 Column Types & Constraints**

- **SERIAL** — auto-incrementing integer. PostgreSQL-specific. In MySQL use: `INT AUTO_INCREMENT`. Standard SQL: `INT GENERATED ALWAYS AS IDENTITY`.
- **PRIMARY KEY** — uniquely identifies each row. Automatically creates an index. Every table should have one.
- **VARCHAR(100)** — variable-length text up to 100 chars. Use TEXT if length is unknown. Use CHAR(n) only for fixed-length codes like country codes.
- **NOT NULL** — the column must have a value. The database rejects any insert that omits this column.
- **UNIQUE** — no two rows can have the same value. Automatically creates an index. Perfect for emails and usernames.
- **DEFAULT NOW()** — if no value is given on insert, PostgreSQL fills in the current timestamp automatically.

```
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  customer_id INT NOT NULL,
  status VARCHAR(20)
CHECK (status IN
                 ('pending','shipped','delivered')),
  total_amount NUMERIC(10,2) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (customer_id)
    REFERENCES customers(id)
    ON DELETE CASCADE
);
```

**📖 Foreign Keys & Constraints**

- **FOREIGN KEY** — links orders.customer_id to customers.id. The database refuses to insert an order for a non-existent customer. This is referential integrity.
- **ON DELETE CASCADE** — if a customer is deleted, all their orders are automatically deleted too. Alternative: ON DELETE SET NULL (sets customer_id to null) or ON DELETE RESTRICT (blocks deletion if orders exist).
- **CHECK constraint** — restricts the allowed values for a column. The database rejects any status value not in the list. Catches bad data at the database level, not just in application code.
- **NUMERIC(10,2)** — stores a decimal number with up to 10 total digits and exactly 2 decimal places. Use NUMERIC for money — FLOAT is imprecise for financial calculations.

#### 1.2 INSERT — Add Data

```
-- Single row insert
INSERT INTO customers (name, email, city)
VALUES ('Priya Sharma', 'priya@email.com', 'Mumbai');

-- Multi-row insert (much faster than one-by-one)
INSERT INTO customers (name, email, city)
VALUES
  ('Arjun Mehta',  'arjun@email.com', 'Delhi'),
  ('Divya Rao',    'divya@email.com', 'Bangalore'),
  ('Karan Singh',  'karan@email.com', 'Pune')
RETURNING id, name;
```

**📖 INSERT Essentials**

- Always specify column names in INSERT — if the table schema changes, hardcoded positional inserts break. Explicit column names are self-documenting and safe.
- **Multi-row INSERT** is far faster than inserting one row at a time in a loop. For bulk loads of thousands of rows use `COPY` command or `pg_copy` — 10–100x faster than INSERT.
- **RETURNING** — PostgreSQL extension. Returns data from the inserted row immediately without a second SELECT. In MySQL: use `LAST_INSERT_ID()` after the insert.
- The `id` column is not listed — SERIAL fills it automatically from the sequence.

#### 1.3 SELECT — Read Data with Filters

```dockerfile
-- Get all delivered orders above ₹5000
SELECT
id,
  customer_id,
  total_amount,
  created_at
FROM orders
WHERE status = 'delivered'
AND total_amount > 5000
ORDER BY total_amount DESC
LIMIT 10;
```

**📖 SELECT Clause Order**

- **SELECT** — which columns to return. Use `SELECT *` only for quick exploration, never in application code (returns unnecessary data, breaks if schema changes).
- **FROM** — which table to read from.
- **WHERE** — filter rows. Only rows where the condition is true are returned. Multiple conditions with AND/OR.
- **ORDER BY ... DESC** — sort results. DESC = highest first. ASC (default) = lowest first.
- **LIMIT 10** — return only first 10 rows. Always use LIMIT in application queries to prevent returning millions of rows accidentally. In SQL Server: `TOP 10`. In Oracle: `FETCH FIRST 10 ROWS ONLY`.

#### 1.4 Useful WHERE Conditions

```
-- LIKE: pattern matching (slow on large tables)
SELECT * FROM customers
WHERE name LIKE '%Sharma%';

-- IN: match any value in list
SELECT * FROM customers
WHERE city IN ('Mumbai', 'Delhi', 'Bangalore');

-- BETWEEN: range (inclusive on both ends)
SELECT * FROM orders
WHERE created_at
BETWEEN '2026-01-01' AND '2026-03-31';

-- IS NULL: check for missing values
SELECT * FROM orders
WHERE delivered_at IS NULL;
```

**📖 WHERE Operators Explained**

- **LIKE '%text%'** — % matches zero or more characters. `'%Sharma%'` finds "Sharma", "Priya Sharma", "Sharma Enterprises". Case-sensitive by default in PostgreSQL. Use `ILIKE` for case-insensitive.
- **IN (list)** — cleaner than writing `city='Mumbai' OR city='Delhi'`. Works with subqueries: `WHERE id IN (SELECT customer_id FROM orders)`.
- **BETWEEN a AND b** — includes both endpoints. Equivalent to `created_at >= '2026-01-01' AND created_at <= '2026-03-31'`.
- **IS NULL / IS NOT NULL** — never use `= NULL` (it always returns nothing in SQL). Always use `IS NULL`. NULL represents an unknown or missing value, not a zero or empty string.

### 2. Phase 2 — JOINs: Combining Tables

**Business Problem:** The FlipMart dashboard needs to show "customer name + city" alongside their orders. Orders only stores customer_id. To get the name, you JOIN orders to customers. This is the most important SQL skill — joins power 90% of real-world queries.

🔧 Where JOINs Are Used in the Stack

Dashboard Query (Metabase/Grafana) → JOIN 3–5 tables in SQL → Returns combined result set

Django: Order.objects.select_related('customer') → Generates INNER JOIN SQL → PostgreSQL executes it

Understanding JOINs lets you debug N+1 query problems in Django (where ORM generates 100 queries instead of 1 JOIN), optimize Hibernate queries in Java, and write efficient API endpoints.

#### 2.1 INNER JOIN — Matching Rows Only

```dockerfile
-- Show each order with customer name
SELECT
o.id AS order_id,
  c.name AS customer_name,
  c.city,
  o.total_amount,
  o.status
FROM orders o
INNER JOIN customers c
ON o.customer_id = c.id
WHERE o.status = 'delivered'
LIMIT 5;
```

**📖 INNER JOIN**

- **INNER JOIN** — returns only rows where a match exists in BOTH tables. An order with no matching customer (orphan data) is excluded.
- **Aliases** — `orders o` and `customers c` are shortcuts. Instead of writing `orders.total_amount` you write `o.total_amount`. Always alias tables in multi-table queries.
- **ON clause** — the join condition. Links the foreign key in one table to the primary key in the other. The database uses this to match rows.
- **AS alias_name** — rename a column in the output. `o.id AS order_id` makes the result column readable. Especially important when multiple tables have columns with the same name.

order_id

customer_name

city

total_amount

status

1042

Priya Sharma

Mumbai

8750.00

delivered

1055

Arjun Mehta

Delhi

3200.00

delivered

1061

Divya Rao

Bangalore

12400.00

delivered

#### 2.2 LEFT JOIN — Keep All Left Table Rows

```dockerfile
-- Show all customers, even those with no orders
SELECT
c.name,
  c.city,
  COUNT(o.id) AS total_orders
FROM customers c
LEFT JOIN orders o
ON c.id = o.customer_id
GROUP BY c.id, c.name, c.city
ORDER BY total_orders DESC;
```

**📖 LEFT JOIN**

- **LEFT JOIN** — returns ALL rows from the left table (customers), and matching rows from the right table (orders). If a customer has no orders, their orders columns appear as NULL.
- This is how you find **customers who have never ordered**: LEFT JOIN + `WHERE o.id IS NULL`.
- **COUNT(o.id)** — counts non-NULL values of o.id. For customers with no orders, o.id is NULL so COUNT returns 0. `COUNT(*)` would count the NULL row and return 1 — wrong. Always COUNT a specific column for LEFT JOINs.
- When in doubt, use LEFT JOIN — INNER JOIN silently drops rows that don't match, which can cause missing data bugs that are hard to find.

#### 2.3 JOIN Three Tables Together

```dockerfile
-- Order details: customer + product name + quantity
SELECT
o.id AS order_id,
  c.name AS customer,
  p.name AS product,
  oi.quantity,
  oi.unit_price
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products p ON p.id = oi.product_id
WHERE o.id = 1042;
```

**📖 Multi-Table JOINs**

- Chain JOIN clauses one after another. Each JOIN adds another table to the query's working set.
- The **order matters for readability**, not performance. PostgreSQL's query planner reorders JOINs for efficiency regardless of how you write them.
- **Tip for understanding multi-joins**: start from the central table (orders), then ask "what does each JOIN add?" — JOIN customers adds customer name, JOIN order_items adds line items, JOIN products adds product names.
- When column names collide across tables (e.g. both orders and customers have a `name` column), always use `table_alias.column_name` notation to be explicit.

### 3. Phase 3 — Aggregations and GROUP BY

**Business Problem:** FlipMart's dashboard needs: total revenue per day, total orders per city, best-selling product categories, and average order value per customer. All of these require aggregation functions and GROUP BY.

🔧 Where Aggregations Are Used in the Stack

Metabase / Grafana Dashboard → GROUP BY + SUM/COUNT SQL → Draws bar/line charts

Django API: sales_by_category endpoint → GROUP BY + SUM query → JSON response to React

Aggregation queries generate the JSON data that powers every chart in every analytics dashboard. Learning GROUP BY is learning how every business metric is calculated.

#### 3.1 Aggregate Functions

```dockerfile
-- Daily revenue — powers the revenue line chart
SELECT
DATE(created_at)    AS order_date,
  COUNT(*)            AS total_orders,
  SUM(total_amount)  AS daily_revenue,
  AVG(total_amount)  AS avg_order_value,
  MAX(total_amount)  AS largest_order
FROM orders
WHERE status = 'delivered'
GROUP BY DATE(created_at)
ORDER BY order_date DESC
LIMIT 7;
```

**📖 Aggregate Functions**

- **COUNT(*)** — counts all rows in each group. Use `COUNT(column)` to count non-NULL values only.
- **SUM(column)** — total of all values in the group. NULL values are ignored.
- **AVG(column)** — arithmetic mean. Returns NUMERIC. NULLs are excluded from the calculation.
- **MAX / MIN** — largest/smallest value. Works on numbers, text (alphabetical), and dates.
- **GROUP BY** — all non-aggregate columns in SELECT must appear in GROUP BY. The query calculates one row per unique value of the GROUP BY expression.
- **DATE(created_at)** — extracts the date portion only. Groups all orders on the same calendar day together. In SQL Server: `CAST(created_at AS DATE)`.

order_date

total_orders

daily_revenue

avg_order_value

largest_order

2026-03-29

142

₹4,82,350.00

₹3,397.53

₹48,200.00

2026-03-28

128

₹3,91,800.00

₹3,061.00

₹35,600.00

2026-03-27

156

₹5,23,100.00

₹3,353.00

₹52,000.00

#### 3.2 HAVING — Filter After Grouping

```dockerfile
-- Cities with total revenue above ₹10 lakh
SELECT
c.city,
  COUNT(o.id)          AS order_count,
  SUM(o.total_amount) AS city_revenue
FROM customers c
INNER JOIN orders o
ON c.id = o.customer_id
GROUP BY c.city
HAVING SUM(o.total_amount) > 1000000
ORDER BY city_revenue DESC;
```

**📖 HAVING vs WHERE**

- **WHERE** filters individual rows *before* grouping. You cannot use aggregate functions (SUM, COUNT) in WHERE.
- **HAVING** filters groups *after* GROUP BY runs. Used when you want to filter based on an aggregated value.
- Rule: `WHERE city = 'Mumbai'` filters rows before counting. `HAVING SUM(...) > 1000000` filters groups after summing.
- Query execution order: FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT. Understanding this order explains why you can't use alias names in WHERE (alias is created in SELECT, which runs after WHERE).

### 4. Phase 4 — Subqueries, CTEs and Window Functions

**Business Problem:** FlipMart needs: customers who spent more than the average customer, the rank of each product by revenue within its category, and month-over-month revenue change. These require subqueries, CTEs, and window functions — the tools that separate junior SQL writers from senior data engineers.

🔧 Where These Are Used in the Stack

Data Engineering (Python + PostgreSQL) → CTEs + Window Functions → Pandas DataFrame or Parquet file

Analytics SQL (BigQuery, Snowflake, Redshift) → Window Functions (ROW_NUMBER, RANK, LAG) → BI Tool Charts

Window functions are the most-tested SQL skill in data engineering and analytics interviews. RANK, ROW_NUMBER, LAG, and LEAD appear in nearly every data analyst job's technical round.

#### 4.1 Subquery — Query Inside a Query

```dockerfile
-- Customers who spent MORE than the average
SELECT name, city
FROM customers
WHERE id IN (
  SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING SUM(total_amount)
         > (SELECT AVG(total_amount)
              FROM orders)
);
```

**📖 Subqueries**

- A subquery is a SELECT inside another SELECT. The inner query runs first, its result is used by the outer query.
- **Scalar subquery** — returns a single value. The `(SELECT AVG(total_amount) FROM orders)` inside HAVING returns one number.
- **IN subquery** — the inner query returns a list of values. Outer query filters by that list.
- When the same subquery appears multiple times, or when a subquery is complex, use a **CTE** instead — far more readable.
- Subqueries in WHERE can be slow on large tables. Consider rewriting as a JOIN or CTE for better performance.

#### 4.2 CTE (Common Table Expression) — Named Temporary Query

```dockerfile
-- Monthly revenue with CTE for clarity
WITH monthly_revenue AS (
  SELECT
TO_CHAR(created_at, 'YYYY-MM') AS month,
    SUM(total_amount) AS revenue
FROM orders
WHERE status = 'delivered'
GROUP BY TO_CHAR(created_at, 'YYYY-MM')
)
SELECT *
FROM monthly_revenue
ORDER BY month;
```

**📖 CTEs — WITH Clause**

- **WITH name AS (...)** — defines a named temporary result set. The CTE name can then be used in the main query like any table.
- **Readability** — break complex queries into named steps. Instead of deeply nested subqueries, you give each step a meaningful name.
- **Multiple CTEs** — chain them: `WITH cte1 AS (...), cte2 AS (SELECT ... FROM cte1) SELECT ... FROM cte2`.
- **Recursive CTEs** — a CTE that references itself. Used for hierarchical data (org charts, category trees). `WITH RECURSIVE` syntax.
- In most databases, CTEs are not materialised by default — they're inlined into the query plan. Use `WITH ... AS MATERIALIZED` in PostgreSQL to force caching when a CTE is reused multiple times.

#### 4.3 Window Functions — Calculations Across Related Rows

> **Window Functions — The Core Concept**

> - A window function performs a calculation across a set of related rows (a "window") without collapsing them into one row — unlike GROUP BY which aggregates rows into fewer rows.
> - Syntax: `FUNCTION() OVER (PARTITION BY ... ORDER BY ...)`
> - **PARTITION BY** — divides rows into groups (like GROUP BY but keeps all rows). **ORDER BY** — defines the order within each partition for running totals and rankings.
> - Available in: PostgreSQL, MySQL 8+, SQL Server, Oracle, BigQuery, Snowflake, Redshift. This is standard SQL — works everywhere.

```dockerfile
-- Rank products by revenue within each category
SELECT
p.name AS product,
  p.category,
  SUM(oi.quantity * oi.unit_price) AS revenue,
  RANK() OVER (
    PARTITION BY p.category
ORDER BY SUM(oi.quantity * oi.unit_price) DESC
  ) AS rank_in_category
FROM products p
INNER JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.name, p.category;
```

**📖 RANK() Window Function**

- **RANK()** — assigns rank 1 to highest revenue within each category. If two products tie, both get rank 2 and the next gets rank 4 (gap).
- **PARTITION BY p.category** — restart ranking for each category. Electronics has rank 1, 2, 3. Clothing has its own rank 1, 2, 3.
- **ROW_NUMBER()** — sequential numbers, no ties. Even if two products have the same revenue, they get different numbers.
- **DENSE_RANK()** — like RANK but no gaps. Ties both get 2, next gets 3 (not 4).
- Use RANK for top-N queries: wrap this in a CTE and filter `WHERE rank_in_category <= 3` to get top 3 per category.

```dockerfile
-- Month-over-month revenue change with LAG
WITH rev AS (
  SELECT
DATE_TRUNC('month', created_at) AS month,
    SUM(total_amount)               AS revenue
FROM orders
GROUP BY 1
)
SELECT
month,
  revenue,
  LAG(revenue) OVER (ORDER BY month)   AS prev_revenue,
  revenue - LAG(revenue) OVER (ORDER BY month)
                                            AS change
FROM rev
ORDER BY month;
```

**📖 LAG and LEAD**

- **LAG(column)** — returns the value from the *previous* row in the ordered window. First row gets NULL (no previous row).
- **LEAD(column)** — returns the value from the *next* row. Last row gets NULL.
- Used to compare current period vs previous: month-over-month change, day-over-day growth, customer churn (last purchase vs current).
- **DATE_TRUNC('month', date)** — truncates a timestamp to the start of the month. `2026-03-15` becomes `2026-03-01`. All March dates now group together. Available in PostgreSQL; in MySQL use `DATE_FORMAT(date, '%Y-%m-01')`.
- **GROUP BY 1** — GROUP BY the first column in the SELECT list. Shorthand for GROUP BY DATE_TRUNC(...). Useful for long expressions.

### 5. Phase 5 — Indexes, Transactions and Stored Procedures

**Business Problem:** FlipMart's daily revenue query takes 8 seconds on 10 million orders. The API is slow. Inserting an order with payment must be atomic — if payment fails, the order must not be created. Frequently run reports need reusable code. Indexes, transactions, and stored procedures solve these.

🔧 Where These Are Used in the Stack

Django makemigrations → CREATE INDEX in migration SQL → Query planner uses index

Payment Gateway Webhook → BEGIN / COMMIT TRANSACTION → Atomic order + payment update

Scheduled ETL / cron job → CALL stored_procedure() → Complex multi-step data transform

Indexes are what make queries fast at scale. Transactions ensure data consistency when multiple related changes must succeed or fail together. Every backend developer must understand both.

#### 5.1 Indexes — Make Queries Fast

```
-- Check current query speed (EXPLAIN ANALYZE)
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE customer_id = 4521;

-- Result: Seq Scan (full table scan) → 3,200ms
-- Create an index on customer_id
CREATE INDEX idx_orders_customer
  ON orders(customer_id);

-- Re-run EXPLAIN ANALYZE
-- Result: Index Scan → 2ms
```

**📖 Indexes**

- **EXPLAIN ANALYZE** — shows how PostgreSQL executes a query and how long each step takes. "Seq Scan" = reading every row (slow for large tables). "Index Scan" = using the index (fast).
- An **index** is a sorted lookup structure — like a book's index. PostgreSQL can jump directly to rows matching customer_id=4521 instead of reading all 10 million rows.
- Indexes are **automatic** for PRIMARY KEY and UNIQUE columns. You must manually create indexes for foreign key columns and frequently filtered columns.
- **Index trade-off:** reads become faster, writes become slightly slower (index must be updated on every INSERT/UPDATE/DELETE). Don't create indexes on every column — only on columns used frequently in WHERE, JOIN ON, and ORDER BY.

```dockerfile
-- Composite index: queries filtering both columns
CREATE INDEX idx_orders_status_date
  ON orders(status, created_at);

-- This query now uses the index
SELECT * FROM orders
WHERE status = 'delivered'
AND created_at > '2026-01-01';

-- List all indexes on a table
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders';
```

**📖 Composite Indexes**

- **Composite index** — index on multiple columns. Useful when queries always filter on both columns together.
- Column order matters: `(status, created_at)` works for `WHERE status='delivered'` but NOT efficiently for `WHERE created_at > '2026-01-01'` alone. Put the most selective column first.
- **Partial index** — index only a subset of rows: `CREATE INDEX ON orders(created_at) WHERE status='delivered'`. Smaller, faster for that specific query.
- **pg_indexes** — PostgreSQL system view listing all indexes. Use it to audit indexes on a table and find missing or redundant ones.

#### 5.2 Transactions — Atomic All-or-Nothing Changes

```
-- Place order + deduct stock atomically
BEGIN;

INSERT INTO orders (customer_id, total_amount, status)
VALUES (4521, 8750.00, 'pending')
RETURNING id INTO v_order_id;

INSERT INTO order_items
  (order_id, product_id, quantity, unit_price)
VALUES (v_order_id, 12, 2, 4375.00);

UPDATE products
SET stock_qty = stock_qty - 2
WHERE id = 12;

COMMIT; -- or ROLLBACK if something failed
```

**📖 ACID Transactions**

- **BEGIN** — starts a transaction. All statements after this are part of the same atomic unit.
- **COMMIT** — saves all changes permanently. Other connections can see the changes only after COMMIT.
- **ROLLBACK** — cancels all changes since BEGIN. Database goes back to the state before BEGIN.
- **ACID properties:** Atomicity (all or nothing), Consistency (data stays valid), Isolation (transactions don't interfere), Durability (committed data is permanent even after a crash).
- In Django/SQLAlchemy: `with transaction.atomic():` / `with db.session.begin():` — these generate BEGIN/COMMIT automatically. Understanding transactions helps you debug integrity errors.

#### 5.3 Stored Procedure — Reusable Database Logic

```dockerfile
-- Stored procedure: monthly sales report
CREATE OR REPLACE PROCEDURE
generate_monthly_report(report_month TEXT)
LANGUAGE plpgsql AS
$$
BEGIN
INSERT INTO monthly_reports
SELECT
report_month,
    SUM(total_amount)  AS revenue,
    COUNT(*)            AS orders,
    COUNT(DISTINCT customer_id) AS customers
FROM orders
WHERE TO_CHAR(created_at, 'YYYY-MM')
        = report_month;
END;
$$;

-- Call it
CALL generate_monthly_report('2026-03');
```

**📖 Stored Procedures**

- **Stored procedure** — pre-compiled SQL logic stored in the database. Called by name from application code, SQL clients, or scheduled jobs.
- **Advantages:** logic lives in the database, not duplicated across multiple applications; re-use from Python, Node.js, and Java apps equally; database runs it efficiently.
- **plpgsql** — PostgreSQL's procedural language. Supports IF/ELSE, loops, variables, exceptions. MySQL uses its own stored procedure syntax. SQL Server uses T-SQL.
- **COUNT(DISTINCT customer_id)** — counts unique customers, not total orders. If one customer places 5 orders, COUNT(*) = 5, COUNT(DISTINCT customer_id) = 1.
- Use stored procedures for multi-step data transforms, batch jobs, and operations that must run inside the database. For simpler read queries, plain SQL or ORM is fine.

### 6. SQL in Real Project Tech Stacks

- **🌐 Full-Stack Web App Stack** — 

- **📊 Data Engineering Stack** — 

- **📈 Analytics/BI Stack** — 

- **☁️ DevOps/Cloud Stack** — 

### 7. SQL Quick Reference

SQL Command / Concept

What It Does

Real-World Use

SELECT col FROM table

Read specific columns from a table

Every API endpoint reads data this way

WHERE + AND/OR/NOT

Filter rows by conditions

Search, filter, date ranges in every query

LIKE / ILIKE

Pattern matching with wildcards

Name search, email lookup, partial matching

INNER JOIN

Combine rows from two tables where match exists

Orders + customers, products + categories

LEFT JOIN

All rows from left, matched rows from right (NULL if no match)

Customers with no orders, products not sold

GROUP BY

Collapse rows into groups, aggregate each group

Revenue by day, orders by city, sales by category

COUNT / SUM / AVG

Aggregate functions on groups

Every dashboard metric

HAVING

Filter groups after GROUP BY

Cities with revenue > threshold

WITH ... AS (CTE)

Named temporary query for readability

Complex analytics, multi-step transformations

ROW_NUMBER() OVER

Sequential row number per partition

Pagination, deduplication (keep latest per customer)

RANK() / DENSE_RANK()

Rank rows within groups

Top products per category, leaderboards

LAG() / LEAD()

Access previous/next row's value

MoM growth, day-over-day change, retention

SUM() OVER

Running total within a window

Cumulative revenue chart

CREATE INDEX

Build a lookup structure for faster queries

Speed up slow WHERE and JOIN queries

EXPLAIN ANALYZE

Show query execution plan and timing

Debug slow queries, verify index is being used

BEGIN / COMMIT / ROLLBACK

Atomic multi-step transaction

Order placement + stock update + payment

CREATE OR REPLACE PROCEDURE

Reusable stored logic in database

Monthly reports, batch updates, ETL

DISTINCT

Remove duplicate rows

Unique customer count, unique product list

COALESCE(val, default)

Return first non-NULL value

Replace NULLs with 0 or 'N/A' in reports

CASE WHEN ... THEN ... END

Conditional logic in a query

Categorize orders by size, status labels

DATE_TRUNC / DATE_FORMAT

Truncate date to day/week/month/year

Group by week, monthly aggregations

FOREIGN KEY ... REFERENCES

Link two tables, enforce referential integrity

Every relational data model

CREATE VIEW

Saved SELECT query — acts like a table

Simplify complex joins for dashboards and APIs

### 8. Interview Questions — SQL

##### Interview Q&A — Fresher Level (0–1 Year SQL Experience)

**Q: Q1. What is the difference between WHERE and HAVING?**

A: **WHERE** — filters individual rows before any grouping happens. Runs before GROUP BY. Cannot contain aggregate functions (COUNT, SUM, AVG) because aggregation hasn't happened yet.
**HAVING** — filters groups after GROUP BY has run. Can contain aggregate functions. Used when you want to keep only groups that meet a condition on their aggregate value.
Example: `WHERE status='delivered'` keeps only delivered orders (row filter). `HAVING SUM(total_amount) > 100000` keeps only groups (cities) with revenue over ₹1 lakh (group filter).
You can use both in the same query: WHERE filters rows first, GROUP BY aggregates, HAVING filters the results.

**Q: Q2. What is the difference between INNER JOIN and LEFT JOIN?**

A: **INNER JOIN** — returns only rows that have a match in BOTH tables. Customers with no orders are excluded. Orders with no customer (orphan rows) are excluded.
**LEFT JOIN** — returns ALL rows from the left table, and matching rows from the right. If no match, the right table columns are NULL. Customers with no orders appear with NULL order columns.
**Use case for LEFT JOIN:** find customers who have NEVER ordered — `LEFT JOIN orders ON ... WHERE orders.id IS NULL`. This pattern is called anti-join.
**RIGHT JOIN** — same as LEFT JOIN but keeps all rows from the RIGHT table. Rarely used — just swap the table order and use LEFT JOIN instead.
**FULL OUTER JOIN** — keeps all rows from both tables, NULL-filling where no match. Used for reconciliation (find records that exist in one table but not the other).

**Q: Q3. What is the difference between DELETE, TRUNCATE, and DROP?**

A: **DELETE** — removes rows matching a WHERE condition (or all rows if no WHERE). Logged row by row. Can be rolled back inside a transaction. Triggers fire. Slow for large deletions.
**TRUNCATE** — removes ALL rows from a table instantly. Minimally logged. Much faster than DELETE for clearing a table. Cannot be used with WHERE. In PostgreSQL, can be rolled back. Triggers may or may not fire depending on the database.
**DROP** — deletes the entire table (and all its data, indexes, and constraints) permanently. Cannot be rolled back in most databases.
When accidentally asked "delete all data from this table", use TRUNCATE for speed. When asked to "delete the table completely", use DROP.

**Q: Q4. What is a window function and how is it different from GROUP BY?**

A: **GROUP BY** — collapses multiple rows into one row per group. If you GROUP BY city, you get one row per city. Individual order rows are gone — you only see the aggregate.
**Window function** — performs a calculation across related rows but keeps every row in the result. `RANK() OVER (PARTITION BY city ORDER BY total_amount DESC)` adds a rank column to every order row without removing any rows.
Use GROUP BY when you want a summary (one row per group). Use window functions when you want to add information to each row based on its group context.
Common interview question: "Get the top 3 products by sales in each category." — This requires RANK() or DENSE_RANK() with PARTITION BY category, then filter WHERE rank <= 3 in a CTE. Cannot be done with GROUP BY alone.

**Q: Q5. Why are indexes important and when should you NOT create one?**

A: Indexes speed up SELECT queries by allowing the database to find rows directly (like a book index) instead of reading every row (sequential scan). A query on customer_id that takes 3 seconds on 10M rows takes 2ms with an index.
**Always index:** foreign key columns (customer_id, product_id), columns frequently used in WHERE, columns used in JOIN ON conditions, columns used in ORDER BY on large result sets.
**Don't create indexes on:** columns with very few distinct values (like status with 3 values — the database still reads 30% of the table for each value), small tables (full scan is faster than index lookup overhead), columns rarely used in queries.
**Index maintenance cost:** every INSERT, UPDATE, or DELETE on a table must also update its indexes. A table with 10 indexes is 10x slower for writes than the same table with no indexes. Over-indexing hurts write performance.
Use EXPLAIN ANALYZE to verify an index is actually being used — the query planner may choose a sequential scan if it estimates the index isn't faster for that particular query.

**Q: Q6. What is a transaction and why is it needed?**

A: A transaction groups multiple SQL statements into a single atomic unit — either ALL of them succeed, or NONE of them do. There is no partial success.
**Classic example:** Transfer ₹10,000 between bank accounts: `UPDATE accounts SET balance = balance - 10000 WHERE id = 1` and `UPDATE accounts SET balance = balance + 10000 WHERE id = 2`. Without a transaction, if the server crashes between these two statements, ₹10,000 disappears from account 1 but never appears in account 2.
With a transaction: if the second UPDATE fails, the ROLLBACK reverses the first UPDATE. Both accounts stay unchanged. No money is lost.
**ACID:** Atomicity (all or nothing), Consistency (data stays valid), Isolation (concurrent transactions don't interfere with each other), Durability (committed changes survive crashes).
ORMs handle transactions with context managers: Django's `with transaction.atomic():`, SQLAlchemy's `with session.begin():`. Both generate BEGIN/COMMIT/ROLLBACK SQL.

**Quiz: Quiz 1 — You run: SELECT city, COUNT(*) FROM customers LEFT JOIN orders ON customers.id = orders.customer_id GROUP BY city ORDER BY COUNT(*) DESC. A customer in Pune has no orders. What happens to them?**

- A) They are excluded because LEFT JOIN only keeps matching rows
- B) They appear in the result with COUNT(*) = 1 (the NULL row from orders is still counted by COUNT(*))
- C) They appear with COUNT(*) = 0 if you use COUNT(orders.id) instead of COUNT(*)
- D) Both B and C — COUNT(*) counts 1, COUNT(orders.id) counts 0

> **Answer/explanation:** ✅ Answer: **D — COUNT(*) counts 1, COUNT(orders.id) counts 0**
LEFT JOIN keeps the Pune customer even with no orders. Their orders columns are NULL.
**COUNT(*)** — counts rows, including rows where orders columns are NULL. The Pune customer appears as one row → COUNT(*) = 1. This is wrong if you want "number of orders."
**COUNT(orders.id)** — counts non-NULL values of orders.id. Since orders.id is NULL for the Pune customer → COUNT(orders.id) = 0. This is correct.
Rule: when using LEFT JOIN + COUNT, always COUNT a column from the right table (COUNT(orders.id)), not COUNT(*). This is a classic interview trick question.

**Quiz: Quiz 2 — A query on orders WHERE status='delivered' AND created_at > '2026-01-01' is slow. You add CREATE INDEX idx1 ON orders(created_at). Will this fix the performance?**

- A) Yes — any index on the table makes all queries faster
- B) Partially — it helps with created_at filtering but a composite index on (status, created_at) would be better since both columns are in WHERE
- C) No — indexes only help with PRIMARY KEY lookups
- D) No — you should index status instead because it has fewer distinct values

> **Answer/explanation:** ✅ Answer: **B — Composite index on (status, created_at) is optimal**
An index on created_at alone helps partially — PostgreSQL can use it to find rows after Jan 1, then scan those for status='delivered'.
A **composite index on (status, created_at)** is better — it matches both WHERE conditions. PostgreSQL can find all rows where status='delivered' AND created_at > '2026-01-01' directly from the index without touching the table.
Option D is wrong — indexing a column with only 3 distinct values (pending/shipped/delivered) is often less effective. The planner may skip the index and do a sequential scan because each value matches ~33% of rows.
Use EXPLAIN ANALYZE before and after to measure the actual improvement. Never assume an index helps — verify with the query plan.

**Quiz: Quiz 3 — What does RANK() return if two products have the same revenue in their category?**

- A) Both get different ranks — RANK() never ties
- B) Both get the same rank, and the next rank is skipped (e.g. both get rank 2, next product gets rank 4)
- C) Both get the same rank, and the next rank continues without a gap (e.g. both get rank 2, next gets rank 3)
- D) The query throws an error — ties are not allowed in RANK()

> **Answer/explanation:** ✅ Answer: **B — Same rank with a gap after**
**RANK()** — tied rows get the same rank. The next rank skips. If rows A and B both get rank 2, row C gets rank 4 (not 3).
**DENSE_RANK()** — tied rows get the same rank, but no gap. Rows A and B get rank 2, row C gets rank 3.
**ROW_NUMBER()** — no ties ever. Even if A and B have identical revenue, one gets 2 and the other gets 3 (order between ties is arbitrary unless you add a tiebreaker to ORDER BY).
For "top 3 products" queries: use DENSE_RANK so you always get at least 3 distinct ranks even with ties. RANK might skip rank 3 entirely if there's a 2-way tie at rank 2.

> **SQL Project — Core Takeaways for Freshers**

> - **SQL is the universal data language** — MySQL, PostgreSQL, SQL Server, BigQuery, Redshift, Snowflake all speak SQL. Learn it once, work anywhere. Every backend, data, and cloud job requires it daily.
> - **Never use SELECT *** in application code — always list the specific columns you need. SELECT * returns unnecessary data, breaks when schema changes, and prevents the query planner from using covering indexes.
> - **Use LEFT JOIN + COUNT(right_table.id)** — never COUNT(*) with a LEFT JOIN. COUNT(*) counts NULL rows; COUNT(column) counts only non-NULL values. This distinction causes wrong dashboard numbers constantly.
> - **HAVING filters groups, WHERE filters rows** — they run at different stages of the query. If your WHERE contains an aggregate function and the query fails, the fix is HAVING.
> - **Use CTEs for complex queries** — a query with 5 nested subqueries is nearly unreadable. The same logic as 3 named CTEs is clear and maintainable. Reviewers judge you on readability as much as correctness.
> - **Window functions (RANK, LAG, ROW_NUMBER)** are the most-tested SQL skill in data engineering interviews. They appear in every analytics role technical assessment. Master these and you are ahead of 80% of candidates.
> - **Add indexes before performance problems** — always index foreign key columns and columns you know will be used in WHERE. Use EXPLAIN ANALYZE to verify the index is being used. Don't over-index — every index slows writes.
> - **Transactions protect data integrity** — any operation involving multiple related changes (order + payment + stock update) must be wrapped in a transaction. Without it, partial failures corrupt your data in ways that are nearly impossible to detect and fix.

##### SQL Code Standards — DataPulse Engineering Rules

- Write SQL keywords in UPPERCASE and table/column names in lowercase — standard convention across all companies makes code instantly readable
- Always alias table names in multi-table queries: `FROM orders o INNER JOIN customers c` — without aliases, queries with 4 tables become impossible to read and debug
- Use NUMERIC, not FLOAT, for monetary amounts — floating-point arithmetic produces results like ₹99.99999999 instead of ₹100.00 due to binary representation of decimals
- Format complex queries vertically — each clause (SELECT, FROM, WHERE, GROUP BY) on its own line. This is how senior engineers write SQL and how code reviewers expect to read it
- Never put `LIMIT` in your development queries and forget to add it in production — always limit result sets in application queries to prevent accidentally loading millions of rows into memory
- Version control your schema changes with a migration tool (Alembic for SQLAlchemy, Django migrations, Flyway for Java, Liquibase) — never run raw ALTER TABLE statements directly on a production database without a recorded, reversible migration file

##### 🏋️ Hands-On Exercises — Build DataPulse's FlipMart Dashboard Queries

1. **Customer Segmentation Query:** Classify FlipMart's customers into three tiers using CASE WHEN: "Platinum" (total spend > ₹50,000), "Gold" (₹20,000–₹50,000), "Silver" (below ₹20,000). Use a CTE to calculate total spend per customer, then a second CTE to assign the tier, then SELECT the final count per tier. This is the "customer cohort" metric on every e-commerce dashboard.
2. **Product Inventory Alert Query:** Write a query that finds all products where stock_qty is below the average units sold per day in the last 30 days. JOIN products with order_items, calculate average daily units sold as a subquery, then compare against stock_qty. Return product name, current stock, and average daily demand. This is what drives automated reorder alerts in inventory management systems.
3. **Running Total (Cumulative Revenue):** Write a query showing daily revenue AND the running cumulative total revenue for March 2026. Use SUM() as a window function with ORDER BY created_date. The cumulative total for day N is the sum of all daily revenues from day 1 to day N. This powers the "progress toward monthly target" chart on the FlipMart dashboard.
4. **Customer Retention Analysis:** Find customers who placed an order in January 2026 AND also placed an order in February 2026 (retained customers). Use two CTEs: jan_customers and feb_customers, each selecting DISTINCT customer_ids from that month. Then INNER JOIN them to find the intersection. Calculate the retention rate: retained / jan_customers × 100. This is the most important metric in any subscription or repeat-purchase e-commerce business.
5. **Slow Query Optimisation:** Create a table with 1 million rows (use `generate_series` in PostgreSQL: `INSERT INTO orders SELECT gs, (gs%1000)+1, 'delivered', random()*10000, NOW()-interval '1 day'*random()*365 FROM generate_series(1,1000000) gs`). Run EXPLAIN ANALYZE on `WHERE customer_id=500 AND status='delivered'`. Record the execution time. Create the appropriate index. Run EXPLAIN ANALYZE again. Record the improvement. Write down what changed in the query plan (Seq Scan → Index Scan) and the time reduction.

### SQL Project Complete 🎉

You have mastered the SQL skills that power DataPulse's FlipMart analytics — table design with proper constraints, SELECT with filters and sorting, multi-table JOINs, aggregations for dashboard metrics, CTEs for readable complex logic, window functions for rankings and trend analysis, indexes for performance, and transactions for data integrity. These are the queries that run in production every minute of every day.

> **Ananya**
> 
> "Rishi, FlipMart's CTO asked why the daily revenue report was accurate while three competitors' dashboards showed wrong numbers. The answer was your LEFT JOIN with COUNT(orders.id) instead of COUNT(*). That one line is the difference between correct and incorrect customer order counts when customers have zero orders. The CTO didn't care about Python or React. He cared that the numbers were right. That is SQL."

> **Vikram**
> 
> "The monthly revenue query was taking 8 seconds. You added one composite index. It now runs in 12 milliseconds. The Django API endpoint went from timing out to returning in under 100ms total. The React frontend stopped showing loading spinners. All of that from one CREATE INDEX statement. That is why every backend developer needs SQL — not just data engineers. The ORM generated the query. SQL expertise fixed it."

> **Next: Advanced SQL — Query Optimisation, Partitioning & Analytics Databases**

> - Query optimisation deep dive — reading EXPLAIN plans, index-only scans, covering indexes, vacuum and ANALYZE
> - Table partitioning — range partitioning by date (crucial for time-series data: orders, logs, events tables with 100M+ rows)
> - Views and Materialised Views — save complex SQL as reusable "virtual tables"; materialised views for pre-computed expensive aggregations
> - dbt (data build tool) — write SQL CTEs as modular, version-controlled, testable data transformations used by every modern data team
> - Analytics databases — BigQuery, Snowflake, AWS Redshift; columnar storage, how they differ from PostgreSQL for analytical queries
> - SQL for ETL — INSERT INTO ... SELECT, UPSERT (INSERT ON CONFLICT), MERGE for incremental data loads in data pipelines
