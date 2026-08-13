# Big Data — Hands-On Project Lab

> Design and run a batch pipeline over a large CSV dataset.

## Objective

Practice partitioning, cleaning, and aggregating data at a scale where naive approaches start to break.

## Prerequisites

- Python with pandas or PySpark
- A large sample dataset (1M+ rows)

## Steps

1. Profile the dataset: row count, null rates, and obvious data quality issues.
2. Partition the data by a logical key (e.g. date or region) into separate files.
3. Write a cleaning step that handles nulls and type mismatches explicitly (documented, not silently dropped).
4. Compute aggregates (sum/avg/count) per partition and write results to a summary file.
5. Measure and record processing time; then re-run with the partitioned approach and compare.
6. Document every data quality issue you found and how you handled it.

## Deliverable

A data-quality report plus a before/after runtime comparison between the naive and partitioned approach.

## Stretch goals

- Move the same pipeline to PySpark and compare runtime again at scale.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
