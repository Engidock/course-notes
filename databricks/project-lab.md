# Databricks — Hands-On Project Lab

> Build an ingest-transform-visualize notebook pipeline in Databricks.

## Objective

Practice the full notebook-driven workflow Databricks is built around, including scheduling.

## Prerequisites

- Databricks Community Edition (free)
- Basic Python/SQL

## Steps

1. Ingest a CSV into a Databricks table (or Delta table) via the UI or `spark.read`.
2. Write a transform notebook cell that cleans and reshapes the data.
3. Save the cleaned result as a managed Delta table.
4. Build a simple visualization (bar/line chart) directly in the notebook from a query against the Delta table.
5. Create a Databricks Job that runs the notebook on a schedule.
6. Check the Delta table's history with `DESCRIBE HISTORY` and explain what each version represents.

## Deliverable

A scheduled Job run history plus the `DESCRIBE HISTORY` output explained in your README.

## Stretch goals

- Add a second notebook that reads the same Delta table and demonstrates time travel to an earlier version.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
