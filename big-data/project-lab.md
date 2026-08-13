# 📊 Big Data Project Mastery

> **Hey Fresher — Read This First!**
>
> "Big data" isn't a specific tool — it's a set of design habits you adopt once your data stops fitting comfortably on one machine or finishing in one script's runtime. It's the difference between "run this Python script and wait" and "design a pipeline that partitions, retries, checks its own quality, and scales without a rewrite every time volume doubles." In this project you'll join **StreamGaon**, a Hyderabad-based regional video streaming platform serving 40+ Indian cities, as a data engineering intern turned full-time hire on the analytics platform team. Your job: take the pipeline that computes daily viewership and engagement metrics — currently a single pandas script that a founding engineer wrote two years ago and now silently fails once a week — and redesign it as a real batch pipeline that can survive StreamGaon's growth from 2 million to 20 million monthly active viewers.

#### What You Will Learn and Build in This Project

You will design and build a batch data pipeline the way a production data engineering team actually does it: profiling a large, messy dataset before touching it, choosing a partitioning key that matches how the data will actually be queried, writing idempotent transformation steps that can safely be rerun without duplicating data, building explicit data quality checks instead of silently dropping bad rows, orchestrating the whole thing with dependency-aware scheduling, and finally scaling the pipeline from a single-machine pandas job to a distributed Spark job once volume outgrows one box. By the end you'll be able to explain, with real numbers, exactly *why* a naive pipeline breaks at scale and what specific design change fixes each failure mode.

Data profiling, partitioning strategy, idempotent writes, data quality validation, pipeline orchestration, backfills, incremental processing, schema evolution, scaling from pandas to Spark, batch pipeline design patterns

> **📦 Phase 1 — Profiling Before You Pipeline**
>
> Understand exactly what's in StreamGaon's raw viewership logs before writing a single transformation.

> **📦 Phase 2 — Partitioning Strategy**
>
> Choose a partition key that matches how the pipeline and downstream analysts will actually query the data.

> **📦 Phase 3 — Idempotent, Rerunnable Transforms**
>
> Rebuild the cleaning and aggregation logic so a failed or rerun job never produces duplicate or corrupted output.

> **📦 Phase 4 — Data Quality as Code**
>
> Replace silent `dropna()` calls with explicit, logged, alertable data quality checks.

> **📦 Phase 5 — Orchestration & Backfills**
>
> Move from a cron job that "just runs the script" to a dependency-aware scheduler that can backfill history correctly.

> **📦 Phase 6 — Scaling Past One Machine**
>
> Rewrite the pandas pipeline in PySpark once StreamGaon's daily volume outgrows a single box's memory.

**Scene 1 — StreamGaon, Hyderabad | "The Script That Fails Every Friday"**

> **Karthik** _Junior Data Engineer_
>
> Priya, the daily viewership job failed again last night. It's the third Friday in a row. I checked — it's not even an infrastructure issue, the script just runs out of memory around 45 minutes in.

> **Priya** _Senior Data Engineer_
>
> Friday is our highest-traffic day — new show releases go out Thursday night, so Friday's log volume is almost double a normal day. The script was written when we had 2 million monthly viewers. We're at 11 million now and climbing toward a Diwali push that's supposed to bring in another 9 million.

> **Roshan** _Data Platform Lead_
>
> This isn't a "add more RAM" problem, it's a design problem. A pipeline that only works at last year's volume isn't a pipeline, it's a liability with a deadline. Karthik, I want this redesigned properly — profiled, partitioned, idempotent, quality-checked, and ready to run on Spark before Diwali, not patched with a bigger EC2 instance the night before launch.

> **Karthik** _Junior Data Engineer_
>
> Understood. Let's start by actually looking at what's in this data, because I don't think anyone has in a while.

### 1. Phase 1 — Profiling Before You Pipeline

**Business Problem:** StreamGaon's raw viewership logs land as daily gzipped CSVs — one row per playback event (user_id, content_id, device_type, watch_seconds, city, timestamp). Nobody has checked null rates, duplicate rates, or schema drift in over a year, and the pipeline just assumes the data is clean. Before redesigning anything, Karthik needs real numbers.

#### 1.1 Profiling Row Counts and Null Rates

```python
import pandas as pd

# Load a representative day's raw log — this alone takes 90 seconds and 6 GB of RAM
df = pd.read_csv("s3://streamgaon-raw/viewership/dt=2026-08-07/events.csv.gz")

print(f"Row count: {len(df):,}")
print(f"Columns: {list(df.columns)}")

# Null rate per column — the first thing that tells you where the landmines are
null_report = df.isna().mean().sort_values(ascending=False) * 100
print(null_report.round(2))

# Duplicate playback events — same user, content, and timestamp logged twice
dupes = df.duplicated(subset=["user_id", "content_id", "event_timestamp"]).sum()
print(f"Duplicate events: {dupes:,} ({dupes / len(df) * 100:.2f}%)")
```

**Sample output:**

```
Row count: 14,206,933
city               8.41
device_type        0.03
watch_seconds       0.00
Duplicate events: 61,204 (0.43%)
```

> **📖 What this does**
>
> `read_csv` on a single day's gzipped log already pulls 14.2 million rows into memory — that number alone tells you a naive `for` loop or row-by-row logic would never survive this pipeline. The null report shows `city` missing in 8.4% of events, which turns out to be devices that failed reverse-geocoding — not random noise, a specific upstream bug worth flagging to the mobile team rather than just filling with "Unknown." The duplicate check catches retried playback pings from StreamGaon's Android app under poor network conditions, which is exactly the kind of thing that silently inflates a naive `SUM(watch_seconds)` metric if nobody checks for it.

#### 1.2 Checking for Schema Drift Across Days

```python
import pandas as pd
import json

def get_schema(date):
    df = pd.read_csv(f"s3://streamgaon-raw/viewership/dt={date}/events.csv.gz", nrows=1000)
    return set(df.columns)

schema_last_month = get_schema("2026-07-08")
schema_today = get_schema("2026-08-07")

added = schema_today - schema_last_month
removed = schema_last_month - schema_today

print(f"Columns added since last month: {added}")
print(f"Columns removed since last month: {removed}")
```

**Sample output:**

```
Columns added since last month: {'subtitle_language', 'is_offline_download'}
Columns removed since last month: set()
```

> **📖 What this does**
>
> Reading just the header and first 1,000 rows (`nrows=1000`) is enough to compare column sets cheaply without loading full days. Finding `subtitle_language` and `is_offline_download` as new columns means the mobile team shipped a feature last month without telling the data platform team — a pipeline that hardcodes a fixed list of expected columns would have either silently ignored these or, worse, crashed on a strict schema validation with no useful error message.

**Quiz: The profiling step found 8.4% of `city` values are null, concentrated entirely in Android app events from a specific app version. What is the correct next action?**
- Investigate and flag the upstream cause (a geocoding bug in that app version) rather than silently dropping or filling those rows
- Immediately drop all rows with a null `city` value, since 8.4% is small enough not to matter
- Fill all null `city` values with `"Hyderabad"` since that's StreamGaon's headquarters city, to keep row counts consistent

> **Answer/explanation:** The first option is correct — a null rate concentrated in one app version is a signal of a real upstream bug, not random missingness, and dropping or blindly imputing it would silently under-report viewership from whatever cities those users are actually in, potentially skewing regional content decisions. The second option throws away real watch-time data that's still valid for every metric except geography. The third option is actively wrong and dangerous — inventing a city value fabricates data that will directly mislead regional programming decisions.

> **Key takeaways**
>
> - Always profile row counts, null rates, duplicate rates, and schema before writing a single transformation — assumptions about "clean" data are usually wrong at scale.
> - Concentrated null patterns (by app version, device, or region) are usually bugs, not noise, and deserve investigation rather than silent handling.
> - Reading a small `nrows` sample is enough to detect schema drift cheaply, without loading full datasets.

### 2. Phase 2 — Partitioning Strategy

**Business Problem:** The old pipeline reads one giant CSV per day into memory in a single pass. Karthik needs to choose a partitioning scheme that matches how the data is actually queried — by StreamGaon's content team asking "how did this show perform this week" and by the finance team asking "what's our total watch time this month by city."

#### 2.1 Choosing and Applying a Partition Key

```python
import pandas as pd

df = pd.read_csv("s3://streamgaon-raw/viewership/dt=2026-08-07/events.csv.gz")
df["event_timestamp"] = pd.to_datetime(df["event_timestamp"])

# Partition by date (already the ingestion grain) and by city, since regional
# breakdowns are the single most common downstream query pattern at StreamGaon
for city, city_df in df.groupby("city"):
    if pd.isna(city):
        city = "unknown"
    out_path = f"s3://streamgaon-clean/viewership/dt=2026-08-07/city={city}/events.parquet"
    city_df.to_parquet(out_path, index=False)
```

> **📖 What this does**
>
> Partitioning by `dt` (already the natural ingestion boundary) and then by `city` means a downstream query like "watch time in Mumbai for the last 7 days" only reads the 7 relevant date partitions' Mumbai files — not all 40+ cities' worth of data for those days. This is called **partition pruning**, and it's the single biggest lever for making batch queries fast without touching a line of query logic, because the query engine skips reading files it knows can't match the filter.

**Choosing a Partition Key**

- **Partition by date only** — simplest, works well when almost every query filters by a date range (which is true here), but every query still scans all cities for those dates.
- **Partition by date + city (chosen here)** — matches StreamGaon's actual query pattern of "this city, this date range," at the cost of many small partition directories if some cities have very low traffic.
- **Partition by date + content_id** — would help a "how did this specific show perform" query, but content_id has far higher cardinality than city, risking a small-files problem (the same one you'd hit in an HDFS-based system) if applied at the daily grain.

#### 2.2 Verifying Partition Sizes Are Reasonable

```bash
# Check that no single partition is too small (small-files problem) or too large (skew)
aws s3 ls --recursive s3://streamgaon-clean/viewership/dt=2026-08-07/ --human-readable --summarize \
  | awk '{print $3, $4, $5}' | sort -k1 -h
```

> **📖 What this does**
>
> This lists every partition file with its human-readable size, sorted smallest to largest, so Karthik can eyeball whether any city partition is suspiciously tiny (a few KB — a small-files problem waiting to compound over a year of daily partitions) or suspiciously huge relative to the others (data skew — usually a single mega-city like Mumbai or Delhi dominating volume, which may need a further sub-partition by hour during peak days).

> **Key takeaways**
>
> - A good partition key mirrors the most common filter in downstream queries — for StreamGaon that's "this date range, this city."
> - Partition pruning lets a query engine skip reading irrelevant files entirely, which is a far bigger performance win than almost any code-level optimization.
> - Always check partition size distribution — too many tiny partitions and too few giant ones are both real problems, just opposite ones.

### 3. Phase 3 — Idempotent, Rerunnable Transforms

**Business Problem:** When Friday's job failed halfway through, it had already written half the day's partitions. Rerunning it from scratch appended duplicate rows into the partitions that had already succeeded — StreamGaon's Friday watch-time numbers were reported almost 40% too high before anyone caught it.

**Scene — "The Job That Doubled Friday's Numbers"**

> **Priya** _Senior Data Engineer_
>
> Look at this: Friday shows 6.2 million hours watched. Every other day this month is between 3.8 and 4.5 million. That's not a traffic spike, that's a rerun that appended instead of replacing.

> **Karthik** _Junior Data Engineer_
>
> Right — the script uses `to_parquet` in append mode by default in some of our older jobs. If it dies at partition 30 of 40 and we just rerun it, the first 30 get written twice.

> **Priya** _Senior Data Engineer_
>
> Exactly. A batch job that can't be safely rerun isn't production-ready, full stop. Every write in this pipeline needs to be idempotent — running it once or running it five times must produce the exact same result.

#### 3.1 Writing an Idempotent Overwrite Pattern

```python
import pandas as pd
import boto3

def write_partition_idempotent(df, dt, city, bucket="streamgaon-clean"):
    prefix = f"viewership/dt={dt}/city={city}/"
    s3 = boto3.client("s3")

    # Delete any existing objects under this exact partition before writing new ones —
    # this makes the write idempotent: rerun as many times as needed, same result
    existing = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
    for obj in existing.get("Contents", []):
        s3.delete_object(Bucket=bucket, Key=obj["Key"])

    out_path = f"s3://{bucket}/{prefix}events.parquet"
    df.to_parquet(out_path, index=False)

for city, city_df in df.groupby("city"):
    city = "unknown" if pd.isna(city) else city
    write_partition_idempotent(city_df, "2026-08-07", city)
```

> **📖 What this does**
>
> Instead of appending, every partition write first deletes whatever already exists at that exact `dt=/city=` path, then writes the fresh result. This "delete-then-write" pattern (the batch equivalent of `INSERT OVERWRITE`) guarantees that running this job once, or crashing halfway and rerunning it five times, always leaves each partition in exactly the state the last successful run produced — never duplicated, never partially double-counted.

#### 3.2 Making the Whole Job Atomic at the Day Level

```python
def run_daily_pipeline(dt):
    staging_prefix = f"viewership_staging/dt={dt}/"
    final_prefix = f"viewership/dt={dt}/"

    # 1. Write everything to a staging location first
    df = load_and_clean(dt)
    for city, city_df in df.groupby("city"):
        city = "unknown" if pd.isna(city) else city
        city_df.to_parquet(f"s3://streamgaon-clean/{staging_prefix}city={city}/events.parquet", index=False)

    # 2. Only after ALL partitions succeed, atomically "promote" staging to final
    #    by copying staging objects over the final prefix, then clearing staging
    promote_staging_to_final(staging_prefix, final_prefix)
```

> **📖 What this does**
>
> Writing to a staging prefix first, then promoting only once every partition for the day has succeeded, prevents the exact Friday failure: a partial, half-written day never becomes visible to downstream readers under the real `viewership/dt=2026-08-07/` path. If the job dies at partition 30 of 40, the staging area is simply discarded and the job reruns cleanly — the final path either has a complete, correct day or the previous day's data, never a partial one.

**Quiz: A batch job crashes after writing 30 of 40 partitions for the day, using the delete-then-write idempotent pattern from 3.1, but without the staging/promote pattern from 3.2. What happens if downstream dashboards query the data before the job is rerun?**
- They see a partial day — 30 complete, correct partitions and 10 missing ones, likely under-reporting the day's totals
- They see duplicated data in all 40 partitions
- Nothing — the delete-then-write pattern alone also guarantees atomicity at the whole-day level

> **Answer/explanation:** The first option is correct. Delete-then-write makes each *individual partition* idempotent and correct once written, but without staging and promotion, partitions become visible to readers the moment they're written — so a crash mid-run leaves 30 correct partitions and 10 simply absent, which downstream dashboards will read as "the day's totals" without any error, silently under-reporting. The second option describes the append-only bug from before 3.1 was applied, not this scenario. The third option is wrong because idempotency (safe to rerun) and atomicity (all-or-nothing visibility) are different guarantees — 3.1 solves the first, 3.2 solves the second, and you need both.

> **Key takeaways**
>
> - A rerunnable ("idempotent") batch job must produce the same result whether it runs once or five times — delete-then-write or `INSERT OVERWRITE` semantics achieve this per partition.
> - Idempotency alone doesn't guarantee atomicity — a job can still leave a *partial* day visible to readers if it crashes mid-run.
> - Staging-then-promote patterns make an entire day's output atomic: downstream readers see either a complete, correct day or the previous complete day, never a partial one.

### 4. Phase 4 — Data Quality as Code

**Business Problem:** The old pipeline had a single line, `df = df.dropna()`, that silently deleted any row with a null in any column — including rows where only the low-priority `subtitle_language` field was missing, throwing away perfectly good `watch_seconds` data along with it. Roshan wants explicit, auditable rules instead of one blunt line.

#### 4.1 Explicit, Rule-Based Validation

```python
import pandas as pd

def validate_viewership(df: pd.DataFrame) -> tuple[pd.DataFrame, dict]:
    issues = {}

    # Rule 1: watch_seconds must be non-negative and under 24 hours (sanity bound)
    bad_duration = df[(df["watch_seconds"] < 0) | (df["watch_seconds"] > 86400)]
    issues["invalid_watch_seconds"] = len(bad_duration)
    df = df[~df.index.isin(bad_duration.index)]

    # Rule 2: user_id and content_id are required — these rows are unusable, not fixable
    missing_ids = df[df["user_id"].isna() | df["content_id"].isna()]
    issues["missing_required_ids"] = len(missing_ids)
    df = df.dropna(subset=["user_id", "content_id"])

    # Rule 3: city is nice-to-have — keep the row, just tag it explicitly instead of guessing
    df["city"] = df["city"].fillna("UNKNOWN_GEO")
    issues["city_backfilled"] = int((df["city"] == "UNKNOWN_GEO").sum())

    return df, issues

clean_df, quality_report = validate_viewership(df)
print(quality_report)
```

**Sample output:**

```
{'invalid_watch_seconds': 412, 'missing_required_ids': 89, 'city_backfilled': 1194338}
```

> **📖 What this does**
>
> Each rule is a named, independently reasoned decision instead of one silent `dropna()`: rows with impossible watch durations (negative, or over 24 hours — almost certainly a logging bug) are removed and counted; rows missing the identifiers needed to attribute a view are removed because they're genuinely unusable; rows missing only `city` are **kept** and explicitly tagged `UNKNOWN_GEO` rather than dropped, because a missing city shouldn't cost StreamGaon real watch-time data. The `quality_report` dictionary becomes a metric you can log and alert on — a sudden jump in `invalid_watch_seconds` tomorrow tells you something broke upstream before finance ever sees a wrong number.

#### 4.2 Alerting on Quality Thresholds

```python
def check_quality_gates(quality_report: dict, total_rows: int):
    invalid_rate = quality_report["invalid_watch_seconds"] / total_rows
    missing_id_rate = quality_report["missing_required_ids"] / total_rows

    if invalid_rate > 0.01:
        raise ValueError(
            f"Data quality gate failed: {invalid_rate:.2%} invalid watch_seconds "
            f"(threshold 1%). Pipeline halted before writing output."
        )
    if missing_id_rate > 0.005:
        raise ValueError(
            f"Data quality gate failed: {missing_id_rate:.2%} rows missing required IDs "
            f"(threshold 0.5%). Pipeline halted before writing output."
        )

check_quality_gates(quality_report, len(df))
```

> **📖 What this does**
>
> This turns data quality from a passive report into an active gate: if more than 1% of a day's rows have invalid watch durations, the pipeline **stops and raises an error before writing any output** — because a spike that large usually means an upstream logging bug, and it's far better for StreamGaon's dashboards to show yesterday's data with a visible staleness warning than today's data silently wrong.

> **Key takeaways**
>
> - Replace blanket `dropna()` calls with named, per-column rules that make an explicit decision (drop, fix, or tag) for each kind of data problem.
> - Every quality rule should produce a countable metric, not just silently clean the data — you can't alert on what you don't measure.
> - Quality gates that halt the pipeline on threshold breaches are safer than always producing output, even wrong output.

### 5. Phase 5 — Orchestration & Backfills

**Business Problem:** The old pipeline is a single cron entry running a Python script. There's no dependency tracking (it doesn't wait for the raw log upload to finish), no automatic retry, and reprocessing three weeks of history after the Phase 3 and 4 fixes means someone manually editing a date variable and rerunning the script 21 times by hand.

#### 5.1 A Dependency-Aware DAG

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.sensors.s3_key_sensor import S3KeySensor
from datetime import datetime, timedelta

default_args = {
    "owner": "data-platform",
    "retries": 3,
    "retry_delay": timedelta(minutes=10),
}

with DAG(
    dag_id="streamgaon_viewership_daily",
    schedule_interval="0 3 * * *",  # 3 AM IST, after overnight log rotation
    start_date=datetime(2026, 7, 1),
    catchup=False,
    default_args=default_args,
) as dag:

    wait_for_raw_logs = S3KeySensor(
        task_id="wait_for_raw_logs",
        bucket_name="streamgaon-raw",
        bucket_key="viewership/dt={{ ds }}/events.csv.gz",
        timeout=60 * 60,
        poke_interval=300,
    )

    def _run_pipeline(ds, **_):
        run_daily_pipeline(dt=ds)

    run_pipeline = PythonOperator(
        task_id="run_daily_pipeline",
        python_callable=_run_pipeline,
    )

    wait_for_raw_logs >> run_pipeline
```

> **📖 What this does**
>
> `S3KeySensor` makes the pipeline actually **wait** for the raw log file to exist before running, instead of assuming it landed by 3 AM and failing confusingly when it's late — a real problem StreamGaon hits during high-traffic release nights when log rotation runs late. `retries=3` with a 10-minute delay handles transient failures (a flaky S3 read, a brief network blip) without paging anyone at 3 AM. `{{ ds }}` is Airflow's templated "execution date," which is what makes the next trick — backfilling — possible without editing code.

#### 5.2 Backfilling Three Weeks of History

```bash
# Airflow re-runs the DAG once per missing date in the range, each with its own {{ ds }}
airflow dags backfill streamgaon_viewership_daily \
  --start-date 2026-07-18 \
  --end-date 2026-08-07 \
  --reset-dagruns
```

> **📖 What this does**
>
> Because the pipeline was written to be idempotent (Phase 3) and the DAG parameterizes everything by `{{ ds }}` instead of a hardcoded date, `airflow dags backfill` can safely regenerate three weeks of history in one command — each day reruns independently, with its own retries, and each day's output atomically replaces whatever was there before. This is the entire payoff of Phases 1–4: without idempotent, atomic, quality-gated transforms, a backfill this size would risk corrupting three weeks of reporting instead of fixing it.

**Cron vs. a Real Orchestrator**

- **Plain cron (the old setup)** — simple, but has no concept of upstream dependencies, no automatic retries, and no built-in way to backfill a date range without manual scripting.
- **Airflow (or similar DAG orchestrator)** — models the pipeline as a graph of dependent tasks, waits for real preconditions (a sensor), retries transient failures automatically, and treats "run for date X" as a first-class, repeatable operation — which is exactly what backfills need.

> **Key takeaways**
>
> - A sensor that waits for real upstream data (not just a fixed clock time) prevents an entire class of "ran too early" failures.
> - Parameterizing a pipeline by execution date (rather than hardcoding "today") is what makes backfills a one-command operation instead of a manual chore.
> - Backfills are only safe once the underlying transforms are idempotent and atomic — orchestration doesn't fix a pipeline that corrupts data on rerun, it just runs that corruption on a schedule.

### 6. Phase 6 — Scaling Past One Machine

**Business Problem:** With Diwali traffic projected to double StreamGaon's daily event volume to roughly 28 million rows/day, the pandas-based pipeline — even with all the Phase 1–5 fixes — will simply run out of memory on a single machine. Roshan wants the same logic ported to PySpark before the volume hits.

#### 6.1 Porting the Cleaning and Aggregation Logic to PySpark

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder.appName("streamgaon_viewership_daily").getOrCreate()

raw = spark.read.csv(
    "s3://streamgaon-raw/viewership/dt=2026-08-07/events.csv.gz",
    header=True,
    inferSchema=True,
)

# Same three quality rules from Phase 4, expressed as Spark DataFrame filters
clean = (
    raw
    .filter((F.col("watch_seconds") >= 0) & (F.col("watch_seconds") <= 86400))
    .dropna(subset=["user_id", "content_id"])
    .withColumn("city", F.coalesce(F.col("city"), F.lit("UNKNOWN_GEO")))
)

daily_summary = (
    clean.groupBy("city", "content_id")
    .agg(
        F.sum("watch_seconds").alias("total_watch_seconds"),
        F.countDistinct("user_id").alias("unique_viewers"),
    )
)

# Same idempotent, partitioned write pattern as Phase 2/3, expressed natively in Spark
(
    daily_summary.write
    .mode("overwrite")
    .partitionBy("city")
    .parquet("s3://streamgaon-clean/viewership_summary/dt=2026-08-07/")
)

spark.stop()
```

> **📖 What this does**
>
> The logic is identical to the pandas version — same three quality rules, same partition-by-city, same overwrite semantics — but now Spark distributes the 28 million rows across a cluster of executors instead of loading them into one process's memory. `.mode("overwrite")` with `.partitionBy("city")` gives the same idempotent-per-partition guarantee as our manual delete-then-write pandas code, but Spark handles it natively. Notice how little actually changed conceptually — every hard-won lesson from Phases 1 through 5 (partition key choice, idempotency, explicit quality rules) carried over directly; only the execution engine changed.

**Pandas vs. PySpark for This Pipeline**

- **Pandas (Phases 1–5)** — fast to iterate on, fine up to roughly the size of one machine's RAM (StreamGaon's ~14M rows/day fit, barely, on a large single box); simplest to debug locally.
- **PySpark (Phase 6)** — necessary once data no longer fits in one machine's memory or one machine's processing time budget, distributing both storage and computation across a cluster; the trade-off is more operational complexity (a cluster to manage, a Spark session to tune) in exchange for headroom that scales to hundreds of millions of rows.

**Quiz: Why did Roshan insist on fixing idempotency, atomicity, and data quality (Phases 3–4) in the pandas version *before* porting to Spark, instead of just rewriting everything in Spark directly?**
- Because the design flaws (unsafe reruns, silent data loss, no quality gates) are engine-independent — porting broken logic to a faster engine just produces the same wrong answers faster
- Because PySpark doesn't support `dropna()` or overwrite writes, so those had to be solved in pandas first
- Because pandas code always runs faster than PySpark code for small daily batches, so there was no real reason to rewrite it at all

> **Answer/explanation:** The first option is correct — none of the bugs fixed in Phases 3 and 4 (unsafe reruns duplicating data, silent quality issues, non-atomic partial writes) are specific to pandas; they're pipeline *design* flaws that would reproduce identically, just faster and at higher volume, in a naive PySpark rewrite. Fixing the design first in the simpler, easier-to-debug environment made the eventual Spark port in 6.1 almost mechanical. The second option is false — PySpark has both `dropna()` and `mode("overwrite")`, as shown directly in section 6.1. The third option is also false in general — pandas is often faster for genuinely small data, but the entire premise of this phase is that StreamGaon's Diwali volume is about to exceed what a single machine can hold in memory at all, regardless of speed.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Add a fourth data quality rule to Phase 4 that flags `watch_seconds` greater than a given title's actual runtime (join against a `content_catalog` table) as a likely tracking bug, and report what percentage of rows it catches.
2. Extend the Phase 5 Airflow DAG with a second task, downstream of `run_daily_pipeline`, that only runs on Fridays and Saturdays and computes a rolling 7-day unique-viewer count — using `BranchPythonOperator` or a day-of-week check.
3. Simulate a partial-day failure by manually deleting 5 of 40 partitions after a successful run, then rerun the Phase 3 staging/promote pipeline and confirm the final output is complete and correct, not a mix of old and new data.
4. In the PySpark version from Phase 6, replace `inferSchema=True` with an explicit `StructType` schema, and explain in a comment why explicit schemas are safer for a scheduled production job than schema inference.
5. Using the partition size check from section 2.2, identify which of StreamGaon's 40+ cities has the smallest daily partition, and propose (in writing) whether it should be merged into a broader "Tier-3 Cities" bucket to avoid a long-term small-files problem.

### Big Data Project Complete 🎉

StreamGaon's viewership pipeline went from a single unmonitored pandas script that failed nearly every high-traffic Friday to a properly designed batch pipeline: profiled and understood before being touched, partitioned by date and city to match real query patterns, idempotent and atomic so reruns and backfills are safe, gated by explicit, measurable data quality rules instead of a blind `dropna()`, orchestrated by a dependency-aware DAG that waits for real preconditions and retries automatically, and finally ported to PySpark so it has headroom for the Diwali traffic surge and well beyond.

> **Karthik** _Junior Data Engineer_
>
> I backfilled three weeks of corrected history in one command this morning. Eight months ago that would have been a full day of manually editing a date variable.

> **Priya** _Senior Data Engineer_
>
> And Friday's numbers finally look like Friday's numbers instead of a rerun artifact. Finance stopped double-checking our watch-time totals against their own spreadsheet two weeks ago — that's the real sign this pipeline is trusted now.

> **Roshan** _Data Platform Lead_
>
> Diwali volume hit 31 million events on the peak day — 10% over our projection — and the Spark job finished on schedule without anyone touching it. That's what "designed for scale" actually looks like.

> **Next: Apache Spark**
>
> - Go deeper into the DataFrame API and query optimizer that made the Phase 6 rewrite possible.
> - Learn to read a Spark physical plan and understand exactly where shuffles and stage boundaries come from.
> - Practice caching, broadcast joins, and partition tuning on datasets far larger than StreamGaon's daily volume.
