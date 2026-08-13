# SQL — Hands-On Project Lab

> Answer real business questions against a sample schema using joins and window functions.

## Objective

Go beyond basic SELECTs into the SQL patterns that actually show up in analytics work.

## Prerequisites

- Any SQL engine (Postgres/SQLite/MySQL)
- A sample multi-table schema (orders, customers, products)

## Steps

1. Write a query joining 3+ tables to list each order with customer name and product name.
2. Write a query using `GROUP BY` + `HAVING` to find customers with more than 5 orders.
3. Use a window function (`ROW_NUMBER()` or `RANK()`) to find each customer's most recent order.
4. Write a query computing a running total of revenue by month using `SUM() OVER (ORDER BY ...)`.
5. Write a CTE that finds products that have never been ordered (anti-join).
6. Use `EXPLAIN` on your slowest query and add an index that measurably improves it.

## Deliverable

All 6 queries with their results, plus the `EXPLAIN` before/after showing the index improvement.

## Stretch goals

- Turn the running-total query into a reusable view.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
