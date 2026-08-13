# Apache Spark — Hands-On Project Lab

> Write a PySpark job that cleans, aggregates, and writes data to Parquet.

## Objective

Get comfortable with the DataFrame API and understand what actually triggers computation.

## Prerequisites

- PySpark installed (or Databricks Community Edition)
- Basic Python

## Steps

1. Load a CSV dataset into a Spark DataFrame.
2. Clean the data: drop/fill nulls, fix data types, deduplicate rows.
3. Perform a groupBy + aggregation (e.g. total sales per region per month).
4. Cache the cleaned DataFrame and explain in the README why you chose that point to cache.
5. Write the result out partitioned by one column, in Parquet format.
6. Use `.explain()` on your final query and identify one stage in the physical plan.

## Deliverable

The Parquet output directory plus your README explanation of the `.explain()` output.

## Stretch goals

- Rewrite the aggregation using Spark SQL instead of the DataFrame API and compare readability.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
