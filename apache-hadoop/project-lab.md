# 🐘 Apache Hadoop Project Mastery

> **Hey Fresher — Read This First!**
>
> Apache Hadoop is the original "big data on cheap hardware" idea turned into software: instead of buying one giant expensive server, you buy a rack of ordinary machines, spread your data across all of them (HDFS), and run your computation right where the data already sits (MapReduce). No single machine ever has to hold the whole dataset or do all the work alone — the cluster does it together. In this project you'll join **RouteWise Logistics**, a Bengaluru-based freight and last-mile delivery aggregator whose parcel-scan logs have outgrown every relational database the team has thrown at them. As the newest data engineer on the platform team, your first week's job is to get those logs onto a real distributed file system and process them with hand-written MapReduce jobs — no shortcuts, no managed services, just HDFS and the MapReduce programming model the way thousands of production clusters still run it today.

#### What You Will Learn and Build in This Project

You will build a small but real Hadoop pipeline for RouteWise: landing raw parcel-scan logs into HDFS, understanding how the NameNode and DataNodes actually store and replicate your files, writing and running MapReduce jobs using Hadoop Streaming with Python, speeding those jobs up with combiners and custom partitioners, tuning YARN memory and reducer counts so jobs stop failing on the shared cluster, and finally solving the classic "small files problem" that kills every beginner's HDFS cluster within a month. By the end you will be able to reason about a Hadoop job the way a production data engineer does — in terms of blocks, splits, shuffles, and containers, not just "run the script and hope."

HDFS architecture, block replication, MapReduce programming model, Hadoop Streaming, combiners, partitioners, custom Writables, YARN resource management, job tuning, small files problem, Hadoop Archives, SequenceFiles

> **📦 Phase 1 — Landing Raw Data in HDFS**
>
> Get RouteWise's daily parcel-scan logs off a single crashing MySQL box and into a distributed file system built for this scale.

> **📦 Phase 2 — HDFS Internals: Blocks, Replication, NameNode/DataNode**
>
> Understand exactly how HDFS splits, stores, and protects your files across the cluster — because tuning anything later requires knowing this cold.

> **📦 Phase 3 — Your First MapReduce Job**
>
> Write a mapper and reducer in Python using Hadoop Streaming to count parcel scans per city and per delivery hub.

> **📦 Phase 4 — Combiners, Partitioners & Custom Writables**
>
> Cut network shuffle traffic with a combiner, control which reducer gets which keys with a partitioner, and model composite keys properly.

> **📦 Phase 5 — YARN Tuning & Job Performance**
>
> Stop your jobs from being killed for using too much memory, and make them finish in minutes instead of hours.

> **📦 Phase 6 — Handling the Small Files Problem**
>
> Fix the thing that quietly destroys every new Hadoop cluster: millions of tiny scan-event files choking the NameNode.

**Scene 1 — RouteWise Logistics, Bengaluru | "MySQL Just Fell Over Again"**

> **Tanvi** _Junior Data Engineer_
>
> The ops dashboard has been down since 6 AM. I checked the logs — the nightly job that aggregates parcel scans by hub timed out again. We're at 40 GB of scan events a day now, and MySQL just can't keep up with the joins.

> **Roshan** _Senior Data Engineer_
>
> That's because you're trying to do distributed-scale work on a single box. We've got twelve old app servers sitting idle in the Bengaluru data center — enough disk and cores between them to hold six months of scan data with room to spare. Today we turn them into an HDFS cluster.

> **Priya** _Engineering Lead_
>
> Before you touch a single line of MapReduce code, I want you to understand what HDFS actually does differently from a normal filesystem. If you don't understand blocks and replication, every tuning decision you make later will be a guess. Roshan, walk her through it, then get her writing the first job by end of day.

> **Tanvi** _Junior Data Engineer_
>
> Understood. So step one — get the scan logs off MySQL's disk and into this cluster before tonight's batch even starts.

### 1. Phase 1 — Landing Raw Data in HDFS

**Business Problem:** RouteWise ingests roughly 40 GB/day of parcel-scan events — every barcode scan at every hub, van, and delivery point, arriving as newline-delimited JSON from hundreds of scanner devices. The existing pipeline dumps everything into a single MySQL table, and both the ingestion job and the nightly reporting query have started timing out. Before any processing can happen, this raw data needs a home that scales horizontally: HDFS.

#### 1.1 Setting Up the Landing Directory Structure

```bash
# Create a partitioned landing zone in HDFS, organized by ingestion date
hdfs dfs -mkdir -p /data/routewise/raw/scans
hdfs dfs -mkdir -p /data/routewise/raw/scans/dt=2026-08-12
hdfs dfs -mkdir -p /data/routewise/raw/scans/dt=2026-08-13

# Set ownership so only the ingestion service account can write here
hdfs dfs -chown -R ingest:routewise /data/routewise/raw

# Restrict write access, allow the analytics group to read
hdfs dfs -chmod -R 750 /data/routewise/raw
```

> **📖 What this does**
>
> `hdfs dfs -mkdir -p` creates the directory path in HDFS's namespace (much like a POSIX filesystem, but this hierarchy only exists as metadata on the NameNode). The `dt=YYYY-MM-DD` naming convention is a Hive-style partition layout — even though we're not using Hive yet, adopting this pattern from day one means every downstream tool (Hive, Spark, Presto) can later treat `dt` as a partition column for free. `-chown` and `-chmod` matter because HDFS enforces POSIX-like permissions, and an ingestion pipeline running as the wrong user is the single most common cause of "permission denied" tickets on a new cluster.

#### 1.2 Loading the Daily Scan Files

```bash
# Copy yesterday's scan export from the edge server's local disk into HDFS
hdfs dfs -put /var/spool/routewise/scans_2026-08-12.jsonl \
  /data/routewise/raw/scans/dt=2026-08-12/part-00000.jsonl

# Verify the file landed and check its size
hdfs dfs -ls -h /data/routewise/raw/scans/dt=2026-08-12/

# Confirm the data isn't corrupted using HDFS's built-in checksum
hdfs dfs -checksum /data/routewise/raw/scans/dt=2026-08-12/part-00000.jsonl
```

**Sample output:**

```
Found 1 items
-rw-r-----   3 ingest routewise    38.6 G  2026-08-13 01:12 /data/routewise/raw/scans/dt=2026-08-12/part-00000.jsonl
```

> **📖 What this does**
>
> `-put` streams the local file into HDFS, splitting it into blocks as it writes (we'll cover exactly how in Phase 2). The `3` in the `-ls` output is the replication factor — HDFS already made three copies of this file across different DataNodes without us asking. `-checksum` lets us prove to the ops team that the bytes that left MySQL's export job are the same bytes that landed in HDFS, which matters a lot when you're about to delete the source file.

**Comparing Ingestion Approaches**

- **Batch `hdfs dfs -put` (what we're doing here)** — simple, scriptable, perfect for RouteWise's once-a-day scan export where a few minutes of latency is fine.
- **Flume or a streaming ingestion agent** — appropriate if RouteWise later wants scan events to land in HDFS within seconds of being generated, at the cost of running an always-on ingestion service.

> **Key takeaways**
>
> - HDFS directories are metadata-only structures on the NameNode; the actual bytes live on DataNodes as blocks.
> - Adopting a `dt=YYYY-MM-DD` partition-style layout from day one costs nothing now and saves a painful re-layout later.
> - `hdfs dfs -put` automatically applies the cluster's default replication factor to every file it writes.

### 2. Phase 2 — HDFS Internals: Blocks, Replication, NameNode/DataNode

**Business Problem:** Roshan won't let Tanvi write a single MapReduce job until she can explain, without looking it up, what happens to that 38.6 GB file the moment `-put` finishes. If she doesn't understand blocks and replication, she won't understand why jobs run where they run, or why losing a DataNode at 2 AM doesn't page anyone.

**Scene — "Why Does a 38 GB File Need Three Copies?"**

> **Roshan** _Senior Data Engineer_
>
> That scan file you just uploaded — HDFS didn't store it as one 38.6 GB blob. It chopped it into 128 MB blocks, roughly 302 of them, and scattered three copies of each block across different DataNodes. Tell me why.

> **Tanvi** _Junior Data Engineer_
>
> Fault tolerance? If we lose a disk, we don't lose data.

> **Roshan** _Senior Data Engineer_
>
> Exactly, plus something you haven't hit yet: parallelism. When a MapReduce job reads that file, it doesn't need one machine to read 38.6 GB sequentially — 302 different map tasks can each read their own 128 MB block, on the machine that already has it, at the same time.

#### 2.1 Inspecting Block Layout and Replication

```bash
# See how a file is split into blocks and where each block's replicas live
hdfs fsck /data/routewise/raw/scans/dt=2026-08-12/part-00000.jsonl -files -blocks -locations

# Check the cluster-wide replication and block health summary
hdfs fsck /data/routewise/raw -files -blocks | tail -20

# Check overall HDFS capacity and usage
hdfs dfsadmin -report
```

> **📖 What this does**
>
> `hdfs fsck ... -files -blocks -locations` prints every block ID that makes up the file and the DataNodes each replica lives on — you'll see something like `blk_1073745823_5012 len=134217728 repl=3 [DataNode-07, DataNode-11, DataNode-04]`. `hdfs dfsadmin -report` shows total cluster capacity, used space, and the number of live vs. dead DataNodes, which is the first thing you check when someone says "the cluster feels slow."

#### 2.2 Changing Replication for Cold vs. Hot Data

```bash
# Scan logs older than 90 days are rarely queried — drop replication to save space
hdfs dfs -setrep -w 2 /data/routewise/raw/scans/dt=2026-05-01

# Today's actively-queried partition stays at the cluster default of 3
hdfs dfs -setrep -w 3 /data/routewise/raw/scans/dt=2026-08-12
```

> **📖 What this does**
>
> `-setrep -w` changes the replication factor for a path and waits (`-w`) until the NameNode has finished replicating or de-replicating blocks to match. Dropping cold, rarely-accessed partitions from 3 copies to 2 is a real cost lever on large clusters — it trades a small amount of fault tolerance for meaningfully less disk usage, which is a reasonable trade for data that's cheap to re-export from an upstream source if truly lost.

**Replication Factor 3 vs. Erasure Coding**

- **3x replication (default)** — simple, fast to read from (any of 3 nodes), best for hot data accessed by many jobs concurrently; costs 3x raw storage.
- **Erasure coding (HDFS 3.x)** — stores parity blocks instead of full copies, cutting storage overhead to roughly 1.4x–1.5x; better for cold, rarely-read archival data like RouteWise's scan logs older than a year, at the cost of slower reads during recovery.

**Quiz: RouteWise's default HDFS block size is 128 MB and a scan log file is 300 MB. How many blocks does HDFS split it into, and does the last block use the full 128 MB on disk?**
- 3 blocks (128 MB, 128 MB, 44 MB), and the last block only occupies 44 MB of actual disk space
- 3 blocks, and all three occupy a full 128 MB of disk space regardless of actual data size
- 1 block, because HDFS files are stored contiguously like a normal filesystem

> **Answer/explanation:** The first option is correct. HDFS splits files into fixed-size blocks (128 MB by default), so 300 MB becomes two full 128 MB blocks plus one 44 MB block. Critically, HDFS blocks are logical units, not physically pre-allocated disk space — the last, smaller block only consumes the 44 MB it actually needs on the DataNode's local disk (which itself uses a normal filesystem like ext4 or XFS underneath). The second option is a common misconception that leads people to wrongly fear "wasted space." The third option is wrong because HDFS is explicitly designed to split large files into blocks distributed across many machines — that's the entire point of the architecture.

> **Key takeaways**
>
> - HDFS blocks (default 128 MB) are the unit of storage, distribution, and parallel processing — not the file itself.
> - Every block is replicated (default factor 3) to different DataNodes for fault tolerance and read parallelism.
> - Replication factor is a per-path tunable trade-off between storage cost and durability/read-parallelism, not a fixed cluster-wide constant.

### 3. Phase 3 — Your First MapReduce Job

**Business Problem:** RouteWise's ops team wants one simple number refreshed every morning: how many parcels were scanned at each hub yesterday. This is the "hello world" of MapReduce, but it's also a real, useful report — and it's the foundation every later optimization in this project builds on.

#### 3.1 Writing the Mapper

```python
#!/usr/bin/env python3
# mapper.py — emits (hub_id, 1) for every scan event line
import sys
import json

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue
    try:
        event = json.loads(line)
    except json.JSONDecodeError:
        continue  # skip malformed scan records rather than crash the whole task

    hub_id = event.get("hub_id")
    if hub_id:
        # Hadoop Streaming expects "key\tvalue" on stdout
        print(f"{hub_id}\t1")
```

> **📖 What this does**
>
> In Hadoop Streaming, your mapper is just a program that reads lines from `stdin` and writes `key\tvalue` pairs to `stdout` — Hadoop handles feeding it the right block of input and collecting its output. Here, each JSON scan event is parsed, and if it has a `hub_id`, we emit `hub_id\t1`, meaning "one scan happened at this hub." We deliberately swallow `JSONDecodeError` rather than let a single malformed log line crash the entire map task — a real production lesson: one bad record should never take down a job processing millions of good ones.

#### 3.2 Writing the Reducer

```python
#!/usr/bin/env python3
# reducer.py — sums scan counts per hub_id
import sys

current_hub = None
current_count = 0

for line in sys.stdin:
    hub_id, count = line.strip().split("\t", 1)
    count = int(count)

    if current_hub == hub_id:
        current_count += count
    else:
        if current_hub is not None:
            print(f"{current_hub}\t{current_count}")
        current_hub = hub_id
        current_count = count

# emit the final hub's total after the loop ends
if current_hub is not None:
    print(f"{current_hub}\t{current_count}")
```

> **📖 What this does**
>
> Hadoop guarantees that all values for the same key arrive at the reducer **sorted and grouped together** — this is the "shuffle and sort" phase. That's why the reducer can use a simple running total instead of a dictionary: as soon as `hub_id` changes, we know we've seen every record for the previous hub and can safely emit its final count. This streaming-style aggregation pattern is memory-efficient even if one hub has millions of scans, because we never hold more than one hub's running total in memory at a time.

#### 3.3 Submitting the Job

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -D mapreduce.job.name="routewise_hub_scan_counts_2026-08-12" \
  -files mapper.py,reducer.py \
  -mapper mapper.py \
  -reducer reducer.py \
  -input /data/routewise/raw/scans/dt=2026-08-12 \
  -output /data/routewise/output/hub_counts/dt=2026-08-12

# Once it finishes, look at the results
hdfs dfs -cat /data/routewise/output/hub_counts/dt=2026-08-12/part-00000 | sort -k2 -n -r | head
```

> **📖 What this does**
>
> `hadoop-streaming-*.jar` is a bridge that lets you write MapReduce jobs in any language that can read stdin/write stdout, instead of Java. `-files` ships your Python scripts to every node that will run a task. `-input` points at the HDFS directory from Phase 1 — Hadoop automatically discovers every file (and every block) under it and creates one map task per input split. `-output` must **not** already exist, or the job fails immediately — this is intentional, protecting you from silently overwriting a previous run's results.

**Quiz: If `-input` points to a directory with 302 HDFS blocks, roughly how many map tasks does this job launch, and why?**
- Around 302 — one map task per input split, and by default one split corresponds to one HDFS block
- Exactly 1 — MapReduce always processes an entire input directory as a single map task
- It depends only on `-numReduceTasks`, which also controls the number of map tasks

> **Answer/explanation:** The first option is correct. The default `InputFormat` (`TextInputFormat` for streaming jobs) creates one input split per HDFS block, and Hadoop schedules one map task per split — so a 302-block input roughly produces 302 map tasks (it can vary slightly if a record spans a block boundary, which the InputFormat handles by reading a little past the boundary). The second option is wrong — that would defeat the entire purpose of parallel processing. The third is wrong because `-numReduceTasks` controls only the number of *reduce* tasks; map task count is driven by input splits, independent of the reducer count.

### 4. Phase 4 — Combiners, Partitioners & Custom Writables

**Business Problem:** The hub-count job from Phase 3 works, but Priya notices it's shuffling far more data across the network than necessary — every single `(hub_id, 1)` pair travels from mapper to reducer individually. With 40 GB of daily scans, that's millions of tiny key-value pairs crossing the network before any aggregation happens.

#### 4.1 Adding a Combiner

```python
#!/usr/bin/env python3
# combiner.py — pre-aggregates counts locally on each map node before shuffle
import sys

current_hub = None
current_count = 0

for line in sys.stdin:
    hub_id, count = line.strip().split("\t", 1)
    count = int(count)
    if current_hub == hub_id:
        current_count += count
    else:
        if current_hub is not None:
            print(f"{current_hub}\t{current_count}")
        current_hub, current_count = hub_id, count

if current_hub is not None:
    print(f"{current_hub}\t{current_count}")
```

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -files mapper.py,combiner.py,reducer.py \
  -mapper mapper.py \
  -combiner combiner.py \
  -reducer reducer.py \
  -input /data/routewise/raw/scans/dt=2026-08-12 \
  -output /data/routewise/output/hub_counts_v2/dt=2026-08-12
```

> **📖 What this does**
>
> A combiner runs on the **same machine as the mapper**, right after it, and does a local mini-reduce before anything crosses the network. Our combiner code is identical to the reducer here because summing counts is associative and commutative — RouteWise's mapper on one node might emit `(HUB-BLR-07, 1)` two thousand times; the combiner turns that into a single `(HUB-BLR-07, 2000)` before it's ever shuffled. This is purely an optimization — Hadoop makes no guarantee the combiner runs at all (it might skip it under memory pressure), so your logic must produce the same final result with or without it.

#### 4.2 Custom Partitioning for Balanced Reducers

```bash
# Route records so all scans for a given hub_id always land on the same reducer,
# and use a secondary sort key (delivery zone) within that
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -D stream.map.output.field.separator=\t \
  -D mapreduce.job.reduces=6 \
  -D mapreduce.partition.keypartitioner.options=-k1,1 \
  -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
  -files mapper.py,combiner.py,reducer.py \
  -mapper mapper.py \
  -combiner combiner.py \
  -reducer reducer.py \
  -input /data/routewise/raw/scans/dt=2026-08-12 \
  -output /data/routewise/output/hub_counts_v3/dt=2026-08-12
```

> **📖 What this does**
>
> `KeyFieldBasedPartitioner` with `-k1,1` tells Hadoop to partition strictly on the first field of the key (the `hub_id`), guaranteeing every record for `HUB-BLR-07` lands on the same one of the 6 reducers we requested — this is essential when the reducer logic depends on seeing *all* values for a key together, which ours does. Without a deliberate partitioner, Hadoop's default hash-based partitioner would still group by full key correctly for our simple case, but this pattern becomes critical once keys are composite (e.g., `hub_id + delivery_zone`) and you need partitioning on only part of the key.

**Hadoop Default HashPartitioner vs. KeyFieldBasedPartitioner**

- **Default HashPartitioner** — hashes the entire key to pick a reducer; fine for simple single-field keys like our `hub_id`.
- **KeyFieldBasedPartitioner** — partitions on a specified sub-field of a composite key, needed once you combine multiple dimensions (e.g., `hub_id\tzone`) into one key but still want all records for a given `hub_id` on the same reducer regardless of zone.

> **Key takeaways**
>
> - Combiners reduce shuffle volume by pre-aggregating on the mapper's node, but must be safe to skip — never rely on them for correctness.
> - Partitioners decide which reducer a key goes to; the default hash partitioner is fine until you have composite keys.
> - `mapreduce.job.reduces` directly controls output file count and reducer parallelism — too few creates a bottleneck, too many creates overhead.

### 5. Phase 5 — YARN Tuning & Job Performance

**Business Problem:** As RouteWise's scan volume grows, jobs start failing with `Container killed by YARN for exceeding memory limits`. Priya explains that this isn't a bug — it's YARN doing exactly what it's configured to do, and the fix is understanding resource requests, not just raising a number blindly.

**Scene — "The Job That Kept Getting Killed"**

> **Priya** _Engineering Lead_
>
> Look at this error: container killed, 2.1 GB of 2 GB physical memory used. That's not a crash — that's YARN protecting the cluster from one job hogging shared memory. Your mapper is buffering too much before writing out.

> **Tanvi** _Junior Data Engineer_
>
> So I just bump the memory setting and rerun it?

> **Priya** _Engineering Lead_
>
> You bump it *and* you understand why. If you just throw memory at every failing job, we'll run out of cluster capacity by Diwali. Let's look at what's actually configurable.

#### 5.1 Requesting the Right Container Memory

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -D mapreduce.map.memory.mb=3072 \
  -D mapreduce.map.java.opts=-Xmx2560m \
  -D mapreduce.reduce.memory.mb=4096 \
  -D mapreduce.reduce.java.opts=-Xmx3584m \
  -D mapreduce.job.reduces=6 \
  -files mapper.py,combiner.py,reducer.py \
  -mapper mapper.py -combiner combiner.py -reducer reducer.py \
  -input /data/routewise/raw/scans/dt=2026-08-12 \
  -output /data/routewise/output/hub_counts_v4/dt=2026-08-12
```

> **📖 What this does**
>
> `mapreduce.map.memory.mb` is the total memory YARN allocates to the container running each map task; `mapreduce.map.java.opts` sets the JVM heap size *inside* that container. The heap should always be smaller than the container memory (roughly 80–90%) to leave room for the JVM's own overhead and any off-heap usage — set them equal and you'll get killed again the moment there's the slightest overhead spike. We gave reducers more memory than mappers here because our reducer, while simple, can still be handed a large run of values for a busy hub like a central Bengaluru sorting facility.

#### 5.2 Monitoring a Running Job

```bash
# List currently running YARN applications
yarn application -list

# Get detailed status, including which containers were killed and why
mapred job -status job_1723526400000_0042

# Pull the actual container logs for a failed task attempt
yarn logs -applicationId application_1723526400000_0042 | grep -A5 "OutOfMemory\|Killed"
```

> **📖 What this does**
>
> `yarn application -list` shows every job currently competing for cluster resources — the first thing to check when RouteWise's morning report job seems slow is whether someone else's job is hogging the queue. `yarn logs` pulls the aggregated stdout/stderr from every container that ran, which is where you actually find the root cause (a Python traceback, an OOM kill, a corrupt input record) rather than guessing from the job's final status alone.

**Quiz: A map task keeps failing with "Container killed by YARN for exceeding physical memory limits, 3.2 GB of 3 GB physical memory used." What is the most correct first fix?**
- Increase `mapreduce.map.memory.mb` so the container's memory ceiling is raised, and keep `mapreduce.map.java.opts` heap size safely below it
- Increase `mapreduce.job.reduces` to spread the load across more reducers
- Ignore it — YARN memory limits are just soft warnings and don't actually kill anything

> **Answer/explanation:** The first option is correct — the container's YARN-allocated memory ceiling is too low for what the map task actually needs, so raising `mapreduce.map.memory.mb` (with the JVM heap in `mapreduce.map.java.opts` set comfortably below that ceiling) directly addresses the failure. The second option is wrong because `mapreduce.job.reduces` only affects reduce-side parallelism — it does nothing for a map task's memory. The third option is dangerously wrong: YARN's `NodeManager` actively monitors container memory and will `SIGKILL` any container exceeding its allocation, precisely to protect other jobs sharing the same physical node.

> **Key takeaways**
>
> - YARN container memory (`mapreduce.map/reduce.memory.mb`) is a hard ceiling enforced by the NodeManager, not a suggestion.
> - JVM heap size (`java.opts -Xmx`) should always be set below the container memory ceiling, never equal to it.
> - `yarn application -list` and `yarn logs` are your first two stops when diagnosing a slow or failed job on a shared cluster.

### 6. Phase 6 — Handling the Small Files Problem

**Business Problem:** Six months in, RouteWise's scanner devices have started writing one tiny file per scan batch instead of one file per day — someone changed the edge upload script without telling the platform team. The NameNode, which holds all file and block metadata in memory, is now straining under millions of files averaging a few KB each.

#### 6.1 Diagnosing the Damage

```bash
# Count how many files and blocks live under the raw scans directory
hdfs dfs -count /data/routewise/raw/scans

# Check average file size — anything well under 128 MB signals a small-files problem
hdfs dfs -du -s -h /data/routewise/raw/scans/dt=2026-08-12
```

**Sample output:**

```
       1   410382  6.1 G  /data/routewise/raw/scans
```

> **📖 What this does**
>
> `hdfs dfs -count` returns directories, files, and total size — 410,382 files averaging roughly 15 KB each for one day's data is exactly the small-files pattern that overwhelms the NameNode, because it has to hold metadata (filename, permissions, block locations) for every single one of those files in memory, regardless of how small they are.

#### 6.2 Consolidating with a Hadoop Archive

```bash
# Archive a day's worth of tiny files into a single .har container
hadoop archive -archiveName scans_2026-08-12.har \
  -p /data/routewise/raw/scans/dt=2026-08-12 \
  /data/routewise/archive/

# Files inside a .har are still individually addressable via the har:// scheme
hdfs dfs -ls har:///data/routewise/archive/scans_2026-08-12.har
```

> **📖 What this does**
>
> `hadoop archive` packs many small files into one large `.har` file (itself just a specially structured HDFS file plus an index), collapsing hundreds of thousands of NameNode metadata entries into a small, fixed number regardless of how many original files went in. It's a metadata-relief tool, not a compute-speedup tool — jobs reading through `har://` still pay a small overhead to look up the index, so it's best applied to cold data you rarely reprocess.

#### 6.3 The Real Fix: Consolidate at Ingestion Time

```python
#!/usr/bin/env python3
# consolidate_scans.py — run hourly to merge small per-batch files into one file per hub per day
import sys
import glob

def consolidate(input_glob, output_path):
    with open(output_path, "a") as out:
        for filepath in sorted(glob.glob(input_glob)):
            with open(filepath) as f:
                out.writelines(f.readlines())

if __name__ == "__main__":
    consolidate(sys.argv[1], sys.argv[2])
```

```bash
# Merge locally before the file ever reaches HDFS, then a single -put per hub per hour
python3 consolidate_scans.py "/var/spool/routewise/scans/*.jsonl" \
  /var/spool/routewise/merged/scans_2026-08-12_hour14.jsonl

hdfs dfs -put /var/spool/routewise/merged/scans_2026-08-12_hour14.jsonl \
  /data/routewise/raw/scans/dt=2026-08-12/
```

**Hadoop Archive (HAR) vs. Fixing Ingestion**

- **Hadoop Archive** — a fast, after-the-fact relief valve for small files that already exist in HDFS; doesn't prevent the problem from recurring tomorrow.
- **Consolidating at the ingestion source** (what we did in 6.3) — the actual fix: never let millions of tiny files reach HDFS in the first place. This is what Roshan insists on rolling out to the edge scanner upload script by the end of the week.

> **Key takeaways**
>
> - The NameNode stores all file/block metadata in RAM — millions of small files, regardless of their tiny total size, can exhaust it and slow the whole cluster.
> - `hadoop archive` (.har) is a stopgap for existing small files, not a substitute for fixing the ingestion pattern that created them.
> - The durable fix is almost always upstream: batch small records into fewer, larger files before they ever land in HDFS.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Write a second MapReduce job (mapper + reducer) that computes the average time-in-transit per parcel by joining scan-in and scan-out events for the same tracking ID, using a composite key of `tracking_id\tevent_type`.
2. Add a custom `Partitioner` (in Java, using the classic `org.apache.hadoop.mapreduce.Partitioner` API) that routes all of RouteWise's Mumbai-region hubs to one dedicated reducer, separate from all other regions, and explain in a comment why this could create a "hot reducer" problem.
3. Use `hdfs dfsadmin -report` before and after running `hdfs dfs -setrep -w 2` on three months of cold archive data, and calculate the actual disk space saved in GB.
4. Build a small monitoring script that runs `hdfs dfs -count` daily on the raw ingestion directory and alerts (prints a warning) if the average file size drops below 64 MB, catching the small-files regression before it becomes a NameNode incident.
5. Rerun the Phase 3 hub-count job with `-D mapreduce.job.reduces=1` and again with `-D mapreduce.job.reduces=12`, compare the wall-clock time and the number of output part-files, and write down which setting made sense for RouteWise's ~50 distinct hub IDs and why more reducers isn't always better.

### Apache Hadoop Project Complete 🎉

Over these six phases, RouteWise Logistics went from a single MySQL box buckling under 40 GB of daily scan logs to a working HDFS + MapReduce pipeline: raw data lands in a partitioned HDFS structure, a hand-written Streaming job aggregates hub-level scan counts, a combiner and custom partitioner cut shuffle overhead, YARN memory settings are tuned instead of guessed, and the small-files problem that would have eventually taken down the NameNode has both a stopgap and a permanent fix. Tanvi can now read a `fsck` report, explain a container-killed error to a teammate, and reason about why a job is slow instead of just rerunning it and hoping.

> **Roshan** _Senior Data Engineer_
>
> Three weeks ago you didn't know what a block was. Now you just diagnosed a NameNode memory pressure issue from a `dfs -count` output before I even looked at it.

> **Priya** _Engineering Lead_
>
> The hub-count job now finishes in under four minutes instead of the six-hour MySQL query it replaced. Ops has their morning numbers before their coffee's even cold.

> **Tanvi** _Junior Data Engineer_
>
> Next I want to see what this looks like when we stop writing mappers and reducers by hand for every single question we have about this data.

> **Next: Apache Spark**
>
> - See how DataFrames and a query optimizer replace hand-written mappers and reducers for the same kind of aggregation.
> - Learn why in-memory computation makes iterative and interactive analytics dramatically faster than disk-bound MapReduce.
> - Carry forward the same HDFS-partitioned data you just built and query it with PySpark instead of Hadoop Streaming.
