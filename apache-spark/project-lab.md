# ⚡ Apache Spark Project Mastery

> **Hey Fresher — Read This First!**
>
> Apache Spark took the MapReduce idea — split work across a cluster — and rebuilt it around in-memory computation and a smart query optimizer, so the same distributed jobs that used to take hours now take minutes, and you write far less code to get there. Instead of hand-writing mappers and reducers, you describe *what* you want (a DataFrame transformation) and Spark's Catalyst optimizer figures out the fastest *how*. In this project you'll join **KartMandi**, a fast-growing Indian e-commerce marketplace selling everything from electronics to groceries across 500+ pin codes, as a data engineer on the seller analytics team. Your task: replace a painfully slow Hive-on-MapReduce job that computes seller performance metrics with a PySpark pipeline that runs in a fraction of the time — and actually understand why it's faster, not just that it is.

#### What You Will Learn and Build in This Project

You will build a real PySpark pipeline against KartMandi's order data: loading and inspecting DataFrames, understanding lazy evaluation and what actually triggers a Spark job, cleaning and transforming order records, joining orders against a seller dimension table the efficient way, computing seller-level aggregations and rankings with window functions, deciding when and what to cache, and writing partitioned Parquet output that downstream BI tools can query fast. Along the way you'll learn to read a `.explain()` physical plan well enough to spot a shuffle before it costs you twenty minutes of wasted cluster time.

PySpark DataFrame API, lazy evaluation, transformations vs actions, joins and broadcast joins, groupBy aggregations, window functions, caching and persistence, partitioned Parquet writes, Spark SQL, reading physical query plans

> **📦 Phase 1 — Loading and Exploring Order Data**
>
> Get KartMandi's raw order exports into Spark DataFrames and understand what lazy evaluation actually means in practice.

> **📦 Phase 2 — Cleaning and Transforming Orders**
>
> Fix data types, handle cancellations and returns, and prepare a clean orders DataFrame for analysis.

> **📦 Phase 3 — Joining Orders with Seller Data**
>
> Bring in seller details efficiently, understanding when Spark broadcasts a join and when it shuffles.

> **📦 Phase 4 — Aggregations and Window Functions**
>
> Compute seller-level revenue metrics and rank sellers within each product category.

> **📦 Phase 5 — Caching and Persistence Strategy**
>
> Decide exactly where to cache the pipeline, and prove with `.explain()` why that choice matters.

> **📦 Phase 6 — Writing Partitioned Output for BI**
>
> Write the final seller scorecard to partitioned Parquet so KartMandi's dashboards stay fast as data grows.

**Scene 1 — KartMandi, Pune | "The Seller Dashboard That Takes Forty Minutes"**

> **Meera** _Junior Data Engineer_
>
> The seller performance dashboard refreshes at 5 AM, and it's now taking 40 minutes on our Hive-on-MapReduce job. Sellers are complaining they can't see yesterday's numbers until almost lunchtime some days when the cluster is busy.

> **Karthik** _Senior Data Engineer_
>
> That job hasn't been touched since we had a tenth of the order volume. MapReduce writes intermediate results to disk between every stage — for a pipeline with a join, three aggregations, and a rank calculation, that's a lot of disk I/O we don't need anymore.

> **Priya** _Engineering Lead_
>
> Rewrite it in PySpark. Keep it under five minutes end to end, and I want you to actually understand what Spark is doing differently — not just "it's faster," but why, so when it's slow again in six months you know where to look.

> **Meera** _Junior Data Engineer_
>
> Fair. Let's start with just getting the order data into a DataFrame and seeing what Spark does — and doesn't — do immediately.

### 1. Phase 1 — Loading and Exploring Order Data

**Business Problem:** KartMandi's orders land as daily partitioned Parquet files in cloud storage, roughly 6 million order line-items per day across electronics, groceries, fashion, and home goods categories. Before building anything, Meera needs to load this into Spark and understand exactly when computation actually happens.

#### 1.1 Reading Order Data into a DataFrame

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("kartmandi_seller_scorecard")
    .config("spark.sql.shuffle.partitions", "200")
    .getOrCreate()
)

orders = spark.read.parquet("s3://kartmandi-lake/orders/dt=2026-08-12/")

orders.printSchema()
print(f"Partitions: {orders.rdd.getNumPartitions()}")
```

**Sample output:**

```
root
 |-- order_id: string (nullable = true)
 |-- seller_id: string (nullable = true)
 |-- product_id: string (nullable = true)
 |-- category: string (nullable = true)
 |-- quantity: integer (nullable = true)
 |-- unit_price: double (nullable = true)
 |-- order_status: string (nullable = true)
 |-- order_ts: timestamp (nullable = true)

Partitions: 48
```

> **📖 What this does**
>
> `spark.read.parquet` doesn't actually load any data into memory yet — Spark just reads the Parquet footer metadata (schema, row counts, file layout) to build a query plan. `printSchema()` and `getNumPartitions()` only need that metadata, so they run instantly even against 6 million rows. `spark.sql.shuffle.partitions` sets how many partitions Spark uses whenever it shuffles data (during joins and groupBys) — the default of 200 is a reasonable starting point we'll revisit once we see real shuffle behavior.

#### 1.2 Transformations vs. Actions: What Actually Runs

```python
# This is a transformation — Spark only records it in a logical plan, nothing executes
electronics_orders = orders.filter(orders.category == "electronics")

# Still a transformation — still nothing has executed
high_value = electronics_orders.filter(orders.unit_price * orders.quantity > 5000)

# THIS is an action — it forces Spark to actually execute the plan built above
print(f"High-value electronics orders: {high_value.count()}")
```

> **📖 What this does**
>
> Spark DataFrames are **lazy**: `.filter()` doesn't touch a single row of data, it just adds a step to a logical plan. Nothing runs until an **action** — `.count()`, `.show()`, `.collect()`, or a `.write` — forces Spark to execute. This matters enormously for performance: Spark's Catalyst optimizer sees the *entire* chain of transformations before running anything, so it can reorder filters, push predicates down to the Parquet reader (only reading the `electronics` rows from disk in the first place), and combine steps — something a hand-written MapReduce job could never do automatically.

**Quiz: A colleague writes `df.filter(df.category == "electronics").filter(df.quantity > 1)` and says "this line is slow." Is that possible, and why?**
- No — by itself this line does nothing at all yet, because both `.filter()` calls are lazy transformations; it can't be "slow" until an action triggers execution
- Yes — each `.filter()` call scans the full DataFrame immediately, so two filters means two full passes over the data
- No — Spark automatically caches every DataFrame the moment it's created, so filters always run instantly

> **Answer/explanation:** The first option is correct. Both `.filter()` calls only build up a logical query plan; Spark performs zero actual computation until an action like `.count()`, `.show()`, or `.write()` is called. The line itself executes in microseconds regardless of data size, because it's just plan-building. The second option describes eager evaluation, which is exactly what Spark's laziness avoids — Catalyst can even merge both filters into a single pass once an action does trigger. The third option is false: Spark never caches automatically; caching is always an explicit `.cache()` or `.persist()` call, which we'll cover in Phase 5.

> **Key takeaways**
>
> - Reading a DataFrame and calling `.filter()` builds a logical plan — no data is read or processed yet.
> - Only actions (`.count()`, `.show()`, `.collect()`, `.write()`) trigger real execution.
> - Laziness lets Spark's Catalyst optimizer see and optimize the whole chain of transformations before running anything — including pushing filters down into the Parquet reader.

### 2. Phase 2 — Cleaning and Transforming Orders

**Business Problem:** Raw order data includes cancelled and returned orders that must be excluded from seller revenue calculations, and `unit_price` occasionally arrives as a string due to an upstream export bug. Meera needs a clean, typed orders DataFrame before any aggregation makes sense.

#### 2.1 Fixing Types and Filtering Invalid Orders

```python
from pyspark.sql import functions as F
from pyspark.sql.types import DoubleType

clean_orders = (
    orders
    .withColumn("unit_price", F.col("unit_price").cast(DoubleType()))
    .filter(F.col("order_status").isin("delivered", "shipped"))  # exclude cancelled/returned
    .filter(F.col("quantity") > 0)
    .withColumn("line_total", F.round(F.col("unit_price") * F.col("quantity"), 2))
)

# Quick sanity check: how many rows did we drop, and why
dropped = orders.count() - clean_orders.count()
print(f"Dropped {dropped:,} rows (cancelled, returned, or invalid quantity)")
```

> **📖 What this does**
>
> `.cast(DoubleType())` explicitly converts `unit_price` regardless of whether the upstream export wrote it as a string or a number — any value that can't be parsed becomes `null` rather than crashing the job, which we then need to check for. Filtering `order_status` to only `delivered` and `shipped` is a business rule, not a data quality fix: KartMandi doesn't want cancelled or returned orders inflating a seller's revenue metrics, even though those rows are perfectly valid data. `withColumn("line_total", ...)` adds a derived column computed from two others — this is still a lazy transformation, so no computation has happened yet.

#### 2.2 Handling Nulls After the Cast

```python
# Find rows where the price cast failed, so we can report them rather than silently drop
bad_price_rows = clean_orders.filter(F.col("unit_price").isNull())
print(f"Rows with unparseable unit_price: {bad_price_rows.count()}")

# For the pipeline output, drop them but log the count for the data quality dashboard
clean_orders = clean_orders.dropna(subset=["unit_price"])
```

> **📖 What this does**
>
> Rather than letting a bad cast silently disappear into a `null`-filled `line_total`, we isolate and count those rows first — this is the same "measure before you drop" discipline that matters at any data scale, whether you're in pandas or Spark. Only after logging the count do we call `dropna()`, so if this number ever spikes, it shows up on a monitoring dashboard instead of quietly under-reporting seller revenue.

> **Key takeaways**
>
> - `.cast()` converts types defensively — failures become `null` rather than crashing the job, but you must explicitly check for and handle those nulls.
> - Business-rule filtering (excluding cancelled orders) and data-quality filtering (excluding bad casts) are different concerns — keep them as separate, clearly named steps.
> - Every dropped row should be counted and logged before it disappears, not silently discarded.

### 3. Phase 3 — Joining Orders with Seller Data

**Business Problem:** The clean orders DataFrame has a `seller_id` but no seller name, city, or tier. Meera needs to join it against a much smaller seller dimension table — and understand why the naive join approach that worked fine on a 10,000-row test dataset caused a massive shuffle in production.

#### 3.1 A Naive Join and Its Cost

```python
sellers = spark.read.parquet("s3://kartmandi-lake/dim_sellers/")  # ~45,000 rows, small

joined = clean_orders.join(sellers, on="seller_id", how="left")

joined.explain()
```

**Relevant excerpt of the physical plan:**

```
== Physical Plan ==
*(5) Project [...]
+- *(5) SortMergeJoin [seller_id], [seller_id], LeftOuter
   :- *(2) Sort [seller_id ASC]
   :  +- Exchange hashpartitioning(seller_id, 200)
   :     +- *(1) Filter ...
   +- *(4) Sort [seller_id ASC]
      +- Exchange hashpartitioning(seller_id, 200)
         +- *(3) Scan parquet dim_sellers
```

> **📖 What this does**
>
> `SortMergeJoin` with two `Exchange hashpartitioning` steps means Spark is **shuffling both sides of the join across the network** — repartitioning 6 million order rows *and* 45,000 seller rows by `seller_id` before sorting and merging them. That's expensive, and it's completely unnecessary here: the seller table is tiny enough to fit comfortably in memory on every executor.

#### 3.2 Forcing a Broadcast Join

```python
from pyspark.sql.functions import broadcast

joined = clean_orders.join(broadcast(sellers), on="seller_id", how="left")

joined.explain()
```

**Relevant excerpt of the physical plan:**

```
== Physical Plan ==
*(2) Project [...]
+- *(2) BroadcastHashJoin [seller_id], [seller_id], LeftOuter, BuildRight
   :- *(2) Filter ...
   +- BroadcastExchange HashedRelationBroadcastMode
      +- *(1) Scan parquet dim_sellers
```

> **📖 What this does**
>
> `broadcast(sellers)` tells Spark to send the entire small seller table to every executor once, so each executor can do the join purely in local memory — no shuffle of the large orders DataFrame at all. `BroadcastHashJoin` replaces the two expensive `Exchange` steps with a single one-time `BroadcastExchange` of just the small table, which for KartMandi's 45,000-row seller dimension against 6 million order rows turns a multi-minute shuffle-heavy join into a near-instant one. Spark's optimizer actually does this automatically for tables under `spark.sql.autoBroadcastJoinThreshold` (default 10 MB) — the explicit `broadcast()` hint here is Meera making sure it happens even if the seller table grows past that threshold later.

**Sort-Merge Join vs. Broadcast Join**

- **Sort-Merge Join (default for large-large joins)** — shuffles and sorts both sides by the join key; necessary and correct when both tables are too large to fit in memory on a single executor.
- **Broadcast Join** — sends the smaller table whole to every executor, avoiding a shuffle of the large table entirely; ideal when one side (like KartMandi's 45,000-row seller dimension) is small enough to comfortably broadcast, typically under a few hundred MB.

**Quiz: Meera broadcasts a table she believes is small, but the job then fails with an out-of-memory error on multiple executors. What is the most likely cause?**
- The table turned out to be too large to fit in memory on every executor simultaneously, and forcing a broadcast made things worse instead of better
- Broadcast joins never fail with out-of-memory errors, so the cause must be unrelated to the join
- `broadcast()` is only a suggestion Spark can freely ignore, so the failure has nothing to do with the hint

> **Answer/explanation:** The first option is correct — a broadcast join copies the entire "small" table into memory on *every single executor*, so if that table is actually large (say, it grew from 45,000 to 45 million rows after a data migration nobody flagged), you multiply that memory cost by the executor count, which can exhaust memory cluster-wide. This is exactly why broadcast joins should only be used on tables you've verified are genuinely small, and why Spark's automatic threshold exists as a safety net. The second option is false — broadcasting an oversized table is one of the most common causes of executor OOM errors in Spark. The third option is technically true in general (Spark can decline an auto-broadcast if a table exceeds the configured threshold), but an explicit `broadcast()` hint forces the broadcast regardless of size, which is precisely what caused this failure.

> **Key takeaways**
>
> - `.explain()` reveals whether a join is a shuffle-heavy `SortMergeJoin` or a shuffle-light `BroadcastHashJoin` — always check it for joins against dimension tables.
> - Broadcasting a small table avoids shuffling the large table entirely, which is often the single biggest speedup available in a Spark pipeline.
> - Only broadcast tables you've verified are genuinely small — broadcasting a table that's actually large can cause out-of-memory failures across the whole cluster.

### 4. Phase 4 — Aggregations and Window Functions

**Business Problem:** KartMandi's seller dashboard needs two things: total revenue per seller per category, and each seller's *rank* within their category — "you're the #3 electronics seller in Pune this week" is a much more actionable number for a seller than a raw revenue figure.

#### 4.1 GroupBy Aggregation

```python
seller_revenue = (
    joined.groupBy("seller_id", "seller_name", "category")
    .agg(
        F.sum("line_total").alias("total_revenue"),
        F.count("order_id").alias("order_count"),
        F.avg("line_total").alias("avg_order_value"),
    )
)

seller_revenue.orderBy(F.desc("total_revenue")).show(5, truncate=False)
```

> **📖 What this does**
>
> `groupBy(...).agg(...)` triggers Spark to shuffle the data so all rows for the same `(seller_id, category)` combination land on the same partition, then computes the sum, count, and average locally within each group — conceptually the same shuffle-then-reduce pattern as MapReduce, just expressed declaratively and executed far more efficiently, including a partial pre-aggregation on each executor before the shuffle (the Spark equivalent of a combiner).

#### 4.2 Ranking Sellers Within Category Using a Window Function

```python
from pyspark.sql.window import Window

category_window = Window.partitionBy("category").orderBy(F.desc("total_revenue"))

ranked_sellers = seller_revenue.withColumn(
    "rank_in_category", F.rank().over(category_window)
)

ranked_sellers.filter(F.col("rank_in_category") <= 3) \
    .orderBy("category", "rank_in_category") \
    .show(30, truncate=False)
```

> **📖 What this does**
>
> A window function computes a value across a *related group of rows* without collapsing them into a single row the way `groupBy` does — here, `Window.partitionBy("category").orderBy(F.desc("total_revenue"))` defines "all sellers in this category, ordered by revenue" as the window, and `F.rank().over(category_window)` assigns each seller their rank within just that category. This is the difference between "what's the total?" (`groupBy`) and "where does each row stand relative to its peers?" (window function) — exactly what a seller-facing leaderboard needs.

**`groupBy` Aggregation vs. Window Function**

- **`groupBy().agg()`** — collapses many rows into one row per group; use when you need a single summary value per key, like total revenue per seller.
- **Window function (`.over()`)** — keeps every row, adding a computed value relative to a defined window of related rows; use when you need each row to know something about its peers, like a rank, a running total, or a percentage of category total.

> **Key takeaways**
>
> - `groupBy().agg()` shuffles data by key and reduces each group to one row — the declarative equivalent of a MapReduce job.
> - Window functions compute per-row values relative to a partition of related rows without collapsing them, essential for rankings and running totals.
> - `F.rank()` gives ties the same rank and skips subsequent ranks (1, 1, 3); `F.dense_rank()` gives ties the same rank without skipping (1, 1, 2) — choose based on what KartMandi's leaderboard should show for tied revenue.

### 5. Phase 5 — Caching and Persistence Strategy

**Business Problem:** The pipeline now computes `ranked_sellers` twice downstream — once to write the full scorecard, once to generate a "top 3 per category" alert email. Without caching, Spark recomputes the entire join and aggregation chain from scratch for each action, doubling the work for no reason.

#### 5.1 Where (and Why) to Cache

```python
# ranked_sellers is used by two separate downstream actions — cache it once here
ranked_sellers.cache()

# Action 1: write the full scorecard
ranked_sellers.write.mode("overwrite").parquet("s3://kartmandi-lake/seller_scorecard_staging/")

# Action 2: generate the top-3-per-category alert — reuses the cached DataFrame,
# does NOT re-run the join and groupBy from scratch
top_sellers = ranked_sellers.filter(F.col("rank_in_category") <= 3)
top_sellers.write.mode("overwrite").json("s3://kartmandi-lake/top_seller_alerts/")

ranked_sellers.unpersist()  # release the cache once both actions are done
```

> **📖 What this does**
>
> `.cache()` (shorthand for `.persist(StorageLevel.MEMORY_AND_DISK)`) marks a DataFrame to be kept in memory (spilling to disk if needed) after it's first computed, instead of being recomputed from the beginning of the lineage on every subsequent action. We cache `ranked_sellers` specifically because it's the last step before *two* separate actions — caching earlier in the chain (like right after the join) would waste memory on data neither downstream step needs in that exact shape; caching later would mean recomputing this exact aggregation and window function twice. `.unpersist()` explicitly frees that memory once we're done, which matters on a shared cluster where other jobs need that capacity.

#### 5.2 Proving the Difference with `.explain()`

```python
# Without cache: two actions on the same lineage each show the full chain from the read
ranked_sellers_uncached = seller_revenue.withColumn("rank_in_category", F.rank().over(category_window))
ranked_sellers_uncached.filter(F.col("rank_in_category") <= 3).explain()
# Physical plan shows the full Scan -> Join -> Aggregate -> Window chain, every time

# With cache: the plan shows an InMemoryTableScan instead of recomputing upstream stages
ranked_sellers.filter(F.col("rank_in_category") <= 3).explain()
```

**Relevant excerpt with caching applied:**

```
== Physical Plan ==
*(1) Filter (rank_in_category <= 3)
+- *(1) InMemoryTableScan [seller_id, seller_name, category, total_revenue, rank_in_category]
      +- InMemoryRelation [...], StorageLevel(disk, memory, deserialized, 1 replicas)
```

> **📖 What this does**
>
> `InMemoryTableScan` in the physical plan is the proof that Spark is reading from the cached result instead of re-executing the read, filter, broadcast join, groupBy, and window function from scratch. Without caching, that entire chain would run twice — once per action — doubling both cluster time and cost for identical work.

**Quiz: Meera caches `clean_orders` right after Phase 2, before the join in Phase 3, reasoning that "earlier is safer." Is this the best caching point for this pipeline?**
- Not necessarily — caching should happen at the point right before data is reused by multiple downstream actions, which here is `ranked_sellers`, not `clean_orders`, which is only consumed once on its way to the join
- Yes — caching as early as possible in any pipeline is always the safest and most efficient choice
- It makes no difference — Spark ignores `.cache()` calls that aren't placed immediately before a `.write()`

> **Answer/explanation:** The first option is correct. `clean_orders` in this pipeline flows into exactly one join and is never independently reused elsewhere, so caching it provides no benefit — it just consumes executor memory holding data that's only read once anyway. `ranked_sellers` is the actual reuse point (two separate write actions depend on it), which is where caching pays off. The second option is a common but costly misconception — indiscriminate caching wastes memory and can push a job to spill to disk or even fail, hurting performance rather than helping it. The third option is false — `.cache()` takes effect regardless of where it's called relative to `.write()`; placement matters for *effectiveness*, not for whether Spark honors the call at all.

> **Key takeaways**
>
> - Cache a DataFrame only at the point where it's reused by multiple downstream actions — caching too early wastes memory on data that's only consumed once.
> - `.explain()` after caching shows `InMemoryTableScan` in place of the recomputed upstream stages — a direct, verifiable proof the cache is working.
> - Always `.unpersist()` a cached DataFrame once you're done with it, especially on a shared cluster.

### 6. Phase 6 — Writing Partitioned Output for BI

**Business Problem:** KartMandi's BI dashboard queries the seller scorecard filtered by category almost every time — "show me electronics sellers," "show me grocery sellers." Writing one giant unpartitioned Parquet file forces every dashboard query to scan the entire dataset.

#### 6.1 Writing Partitioned Parquet

```python
(
    ranked_sellers
    .repartition("category")
    .write
    .mode("overwrite")
    .partitionBy("category")
    .parquet("s3://kartmandi-lake/seller_scorecard/dt=2026-08-12/")
)
```

> **📖 What this does**
>
> `.repartition("category")` first groups the in-memory Spark partitions by category so that when `.partitionBy("category")` writes to disk, each category's data lands in fewer, larger output files instead of being scattered across many small fragments — avoiding a small-files problem on the write side. `partitionBy("category")` on the write itself creates a `category=electronics/`, `category=groceries/`, etc. directory structure, so a BI query filtering `WHERE category = 'electronics'` only reads that one directory's files — the same partition-pruning benefit you'd get from a well-partitioned HDFS layout.

#### 6.2 Verifying with Spark SQL

```python
ranked_sellers.createOrReplaceTempView("seller_scorecard")

spark.sql("""
    SELECT category, seller_name, total_revenue, rank_in_category
    FROM seller_scorecard
    WHERE rank_in_category <= 3
    ORDER BY category, rank_in_category
""").show(30, truncate=False)
```

> **📖 What this does**
>
> `createOrReplaceTempView` registers the DataFrame as a SQL-queryable table for the duration of the Spark session, letting Meera (or an analyst more comfortable in SQL than the DataFrame API) express the exact same logic as a query. Under the hood, both the DataFrame API and Spark SQL compile down to the identical Catalyst logical plan — there's no performance difference between them, only a difference in which syntax is more readable for the task at hand.

**DataFrame API vs. Spark SQL**

- **DataFrame API** — better for pipelines built incrementally in code, with reusable Python functions and easier unit testing of individual transformation steps.
- **Spark SQL** — better for one-off exploratory queries, or when handing a query off to an analyst who thinks in SQL rather than Python; compiles to the exact same execution plan, so there's no "SQL is slower" concern.

##### 🏋️ Hands-On Exercises — Extend the Project

1. Add a `return_rate` metric to the seller scorecard by joining against a `returns` table (returned line-items per seller), and decide — with a comment justifying your choice — whether that join should be a broadcast join.
2. Rewrite the Phase 4 aggregation entirely in Spark SQL using `spark.sql()`, and confirm with `.explain()` that it produces an identical physical plan to the DataFrame API version.
3. Add a second window function that computes each seller's `total_revenue` as a percentage of their category's total revenue (`F.sum("total_revenue").over(Window.partitionBy("category"))`), and use it to flag sellers responsible for more than 15% of a single category's volume.
4. Introduce a deliberately oversized "seller" dimension table (simulate 5 million rows) and re-run the Phase 3 broadcast join to observe the out-of-memory failure firsthand, then fix it by removing the broadcast hint and comparing the resulting `SortMergeJoin` plan.
5. Benchmark the full pipeline's end-to-end runtime with `spark.sql.shuffle.partitions` set to 10, 200, and 800, and record which setting performs best for KartMandi's ~6 million daily order rows and why too many or too few shuffle partitions both hurt performance.

### Apache Spark Project Complete 🎉

KartMandi's seller scorecard pipeline went from a 40-minute Hive-on-MapReduce job to a PySpark pipeline that reads, cleans, joins, aggregates, ranks, and writes partitioned output in well under five minutes — and every step of that speedup is something Meera can now explain with an `.explain()` plan in hand, not just a "trust me, it's faster." Lazy evaluation let Catalyst optimize the full chain, a broadcast join eliminated an unnecessary shuffle of six million rows, window functions delivered per-seller rankings without collapsing the dataset, deliberate caching avoided recomputing the same aggregation twice, and partitioned output keeps every downstream BI query fast as KartMandi's order volume keeps climbing.

> **Karthik** _Senior Data Engineer_
>
> Six weeks ago you were asking me what a shuffle even was. Today you diagnosed a broadcast join OOM from the executor logs before I'd even opened the Spark UI.

> **Priya** _Engineering Lead_
>
> Sellers are seeing yesterday's numbers by 5:15 AM now instead of noon. That's not a technical win, that's sellers trusting the platform enough to keep listing more products with us.

> **Meera** _Junior Data Engineer_
>
> I want to see what this looks like when we stop managing our own cluster entirely and let a managed platform handle it — and figure out what Delta Lake gives us that plain Parquet doesn't.

> **Next: Databricks**
>
> - See how a managed Spark platform handles cluster provisioning, job scheduling, and notebook collaboration for you.
> - Learn Delta Lake's ACID guarantees, time travel, and schema enforcement on top of the Parquet files you just wrote.
> - Carry forward this same seller scorecard pipeline and rebuild it as a Delta Lake table with full version history.
