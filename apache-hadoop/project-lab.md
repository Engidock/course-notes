# Apache Hadoop — Hands-On Project Lab

> Run a MapReduce word-count job on a local Hadoop cluster.

## Objective

Understand the map/shuffle/reduce model by running the classic job end to end yourself.

## Prerequisites

- A local pseudo-distributed Hadoop setup (or Docker image)
- Java basics

## Steps

1. Start HDFS and YARN in pseudo-distributed mode.
2. Upload a text dataset (e.g. a book or log file) into HDFS.
3. Write (or use the bundled) word-count Mapper and Reducer classes.
4. Package and submit the job with `hadoop jar`.
5. Inspect the job in the YARN ResourceManager UI while it runs.
6. Pull the output back out of HDFS and verify the top 10 most frequent words.

## Deliverable

The job's output file plus a screenshot of the completed job in the YARN UI.

## Stretch goals

- Re-run the same logic as a Spark job and compare runtime.

---
*Part of the [EngiDock](https://www.engidock.com) course library.*
