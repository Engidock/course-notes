# 🐼 Pandas Project Mastery

> **👋 Hey Fresher — Read This First!**

> Pandas is the tool every data analyst, data engineer, and data scientist uses to work with structured data in Python. Without Pandas, processing even a simple CSV file means writing dozens of lines of raw Python loops. Pandas solves all of that — loading, cleaning, filtering, aggregating, and transforming data takes just one or two lines. This document uses **short code blocks** — each one covers exactly one concept — with a plain-English explanation right beside it. No 50-line notebooks to get lost in. One idea at a time, always explained simply.

> **Company in this project:** DataPay — an Indian fintech analytics team in Bengaluru. They receive raw CSV exports of payment transactions from 5 partner banks every day. Right now, their analyst Meera is copy-pasting data in Excel, manually removing duplicates, and emailing summary tables to the CTO. Your lead is Rohan. You are about to replace Meera's 3-hour Excel marathon with a 20-line Pandas pipeline.

#### What You Will Learn and Build in This Project

You will build a complete Pandas data pipeline for DataPay — from loading raw CSVs to producing a clean, aggregated, merged report — learning every core Pandas concept along the way, each explained with a real business reason.

Series & DataFrame, Load & Inspect, Clean Data, Filter & Select, GroupBy & Agg, Merge & Join, Reshape, String Ops, DateTime, Export & Plot

> **📦 Phase 1 — Foundations**

> Install Pandas, understand Series vs DataFrame, load CSVs, inspect shape and dtypes, and navigate rows and columns.

> Handle missing values, remove duplicates, fix data types, rename columns, and filter out bad rows.

> Filter with conditions, GroupBy aggregations, merge multiple DataFrames, pivot tables, and string/date operations.

> Chain operations, apply functions, export to CSV/Excel, and plot summary charts — a complete end-to-end data pipeline.

### 1. Phase 1 — Pandas Foundations

**Business Problem:** Meera's Excel file has 80,000 payment rows from 5 banks. Excel crashes every time she applies a filter. Pandas loads 80,000 rows in under a second, processes them without crashing, and every step is reproducible code — not hidden mouse clicks.

**Scene 1 — DataPay Analytics Room | Day of the Excel Crash**

> **Meera** _Data Analyst — DataPay_
> 
> Rohan, Excel just crashed again. That's the third time today. I had the pivot table almost done — 80,000 rows from HDFC, ICICI, and Axis Bank combined. The CTO wants the daily payment summary by 5 PM and I've lost everything.

> **Rohan** _Lead Data Engineer — DataPay_
> 
> Priya, your first task: replace Meera's Excel workflow with a Pandas pipeline. Same data, same output — but reproducible, crash-proof, and fast enough to run every morning in 3 seconds on a laptop. Start by just loading the CSV correctly.

#### 1.1 Install Pandas

```bash
# Install pandas and openpyxl (for Excel)
pip install pandas openpyxl

# Verify the installation
import pandas as pd
print(pd.__version__)
```

**📖 Why alias as pd?**

By convention, everyone imports Pandas as `pd`. This means instead of writing `pandas.read_csv()` every time, you write `pd.read_csv()`. You'll see `pd` in every data engineering codebase on the planet.  
  
`openpyxl` is required whenever you read or write `.xlsx` Excel files. Install it once, forget about it.

```
2.2.1
```

#### 1.2 The Two Core Structures — Series and DataFrame

Everything in Pandas is built on two objects. Understanding them clearly makes every other concept easy.

```
  SERIES                              DATAFRAME
  (one column, like a list             (a full table — many Series
   with labels)                         sharing the same index)

  Index  Value                          Index  transaction_id  amount   bank     status
  ─────  ──────                         ─────  ──────────────  ──────   ────     ──────
    0    "TXN001"                          0    TXN001          5000     HDFC     success
    1    "TXN002"                          1    TXN002          1200     ICICI    failed
    2    "TXN003"                          2    TXN003          8500     Axis     success
    3    "TXN004"                          3    TXN004          200      HDFC     success

  pd.Series(["TXN001", ...])             pd.read_csv("payments.csv")

  ► A Series is one column.              ► A DataFrame is the whole table.
  ► Every row has an index label.        ► Each column IS a Series.
  ► Supports fast maths, filters.        ► Columns share the same index.
```

> **💡 Think of it like this**

> A **Series** is like one column in an Excel sheet — a vertical list of values with row numbers on the left. A **DataFrame** is the full Excel sheet — rows and columns. In Pandas, every column of a DataFrame is a Series. Once you know this, the whole library clicks.

#### 1.3 Creating a Series and DataFrame from Scratch

```python
# A Series — one column of data
amounts = pd.Series([5000, 1200, 8500, 200])
print(amounts)
```

**📖 pd.Series()**

Pass a Python list and Pandas creates a Series. The left column (0, 1, 2, 3) is the **index** — auto-generated, but you can customise it.  
  
Series support vectorised operations: `amounts * 1.18` multiplies every value by 1.18 (GST) in one line — no loops needed.

```
0    5000
1    1200
2    8500
3     200
dtype: int64
```

```python
# A DataFrame — built from a dict
data = {
    "transaction_id": ["TXN001", "TXN002", "TXN003"],
    "amount":         [5000, 1200, 8500],
    "bank":           ["HDFC", "ICICI", "Axis"],
    "status":         ["success", "failed", "success"]
}
df = pd.DataFrame(data)
print(df)
```

**📖 pd.DataFrame() from a dict**

Pass a Python dictionary where **keys are column names** and **values are lists of data**. Every list must be the same length.  
  
In practice you almost never create DataFrames manually — you load them from CSV, Excel, or databases. But building one from a dict is perfect for testing and understanding the structure.

transaction_id

amount

bank

status

0

TXN001

5000

HDFC

success

1

TXN002

1200

ICICI

failed

2

TXN003

8500

Axis

success

#### 1.4 Load a Real CSV File

```
# Load the DataPay daily transactions file
df = pd.read_csv("payments.csv")
```

**📖 pd.read_csv()**

This is the most-used function in Pandas. One line loads an entire CSV file — no matter how large — into a DataFrame in memory.  
  
Common useful parameters:  

`sep=` — delimiter character (default comma)  

`encoding=` — e.g. `"utf-8"` or `"latin-1"`  

`parse_dates=["date"]` — auto-convert date columns  

`nrows=100` — load only first 100 rows (great for previewing huge files)

#### 1.5 Inspect Your Data — The First 5 Commands You Always Run

```
df.head()        # first 5 rows
df.tail(3)      # last 3 rows
df.shape # (rows, columns)
df.info()        # dtypes + null counts
df.describe()    # stats: mean, min, max...
```

**📖 Always Inspect Before You Analyse**

**head()** — Preview the first rows. Confirm columns loaded correctly.  
  
**shape** — Quick sanity check: did all 80,000 rows load? Are there 12 columns or 13?  
  
**info()** — Reveals which columns are objects (strings) vs numbers, and how many null values each column has. This is where you discover cleaning problems.  
  
**describe()** — Instant stats: mean transaction amount, min, max. Great for spotting data anomalies.

```
# df.info() output
RangeIndex: 80000 entries, 0 to 79999
Data columns (total 6 columns):
 #   Column          Non-Null Count  Dtype
 0   transaction_id  80000 non-null  object
 1   date            79850 non-null  object   ← 150 nulls!
 2   amount          80000 non-null  float64
 3   bank            80000 non-null  object
 4   status          79990 non-null  object   ← 10 nulls
 5   customer_id     80000 non-null  int64
dtypes: float64(1), int64(1), object(4)
```

> **✅ Phase 1 Key Takeaways**

> - Series = one column. DataFrame = full table. Every DataFrame column is a Series.
> - `pd.read_csv()` loads any CSV in one line — no loops, no file handling.
> - Always run `head()`, `shape`, and `info()` before anything else.
> - `info()` exposes null counts and data types — your first clue about what needs cleaning.

### 2. Phase 2 — Data Cleaning

**Business Problem:** DataPay's raw CSVs from 5 banks are messy. Some dates are missing. Some amounts are entered as "5,000" (string with comma, not a number). The bank name column has "hdfc", "HDFC", "Hdfc" — three spellings for the same bank. Any analysis on dirty data produces wrong reports. Cleaning comes first, always.

**Scene 2 — DataPay Analytics Room | The Wrong Report Incident**

> **Rohan** _Lead Data Engineer — DataPay_
> 
> Priya, the CTO just flagged our HDFC total. We reported ₹2.3 crore but the bank says it's ₹3.1 crore. The issue? Half of HDFC's rows are labelled "hdfc" in lowercase. Our groupby only counted "HDFC". We missed 26% of transactions. This is a data cleaning failure — not a code failure.

#### 2.1 Handle Missing Values

```
# Check how many nulls per column
df.isnull().sum()
```

**📖 isnull().sum()**

`isnull()` returns a DataFrame of True/False — True where there's a missing value. `.sum()` counts all the Trues per column.  
  
This gives you a count of missing values per column in one line — your data quality dashboard.

```
transaction_id      0
date              150
amount              0
bank                0
status             10
customer_id         0
dtype: int64
```

```
# Drop rows where date is missing
df = df.dropna(subset=["date"])

# Fill missing status with "unknown"
df["status"] = df["status"].fillna("unknown")
```

**📖 dropna() vs fillna()**

**dropna(subset=[...])** — Remove rows only where the specified columns are null. Without `subset`, it drops any row that has even one null anywhere — often too aggressive.  
  
**fillna(value)** — Replace nulls with a sensible default. For payment status, `"unknown"` is better than deleting the row, because the transaction itself is valid — just the status tag is missing.

#### 2.2 Fix Data Types

```
# "amount" loaded as string: "5,000"
df["amount"] = (
    df["amount"]
    .str.replace(",", "")   # remove commas
    .astype(float)         # convert to number
)

# Convert date column to datetime
df["date"] = pd.to_datetime(df["date"])
```

**📖 astype() and pd.to_datetime()**

When a column that should be a number loads as a string (because of commas, currency symbols, or spaces), you first clean the string, then call `.astype(float)` to convert.  
  
**pd.to_datetime()** converts string dates like "2024-01-15" or "15-Jan-2024" into proper datetime objects. Once converted, you can extract year, month, day — or filter by date range — in one line.

#### 2.3 Remove Duplicates

```python
# Check for duplicate transaction IDs
print(df.duplicated(subset=["transaction_id"]).sum())

# Remove them — keep first occurrence
df = df.drop_duplicates(subset=["transaction_id"])
```

**📖 duplicated() and drop_duplicates()**

Banks sometimes send the same transaction twice in their daily export (a known integration issue). Without deduplication, your totals are inflated.  
  
`duplicated(subset=)` flags rows where the specified column has a value seen before. `drop_duplicates()` removes them, keeping the first occurrence by default.

```
247   ← 247 duplicate transaction_id rows found and removed
```

#### 2.4 Standardise String Values

```python
# Fix inconsistent bank names
df["bank"] = df["bank"].str.strip().str.upper()

# Check unique values after cleaning
print(df["bank"].unique())
```

**📖 str.strip().str.upper()**

`str.strip()` removes leading/trailing whitespace — "HDFC " and " HDFC" both become "HDFC".  
  
`str.upper()` converts to uppercase — "hdfc", "Hdfc", "HDFC" all become "HDFC".  
  
Chaining these two calls in one line is a standard cleaning step for any text category column. After this, your GroupBy totals will be correct.

```
['HDFC' 'ICICI' 'AXIS' 'SBI' 'KOTAK']   ← clean, consistent
```

#### 2.5 Rename Columns

```
# Rename messy column names from raw CSV
df = df.rename(columns={
    "Txn ID":    "transaction_id",
    "Amt (INR)": "amount",
    "Bank Name": "bank"
})
```

**📖 rename(columns={})**

Raw CSVs from banks often have inconsistent column names — spaces, special characters, abbreviations. Always standardise column names to lowercase with underscores (snake_case) at the start of your pipeline.  
  
This means `df["amount"]` always works, and you never get a KeyError because someone added a trailing space to a column name.

> **✅ Phase 2 Key Takeaways**

> - `isnull().sum()` is your data quality report — run it immediately after loading.
> - Use `dropna(subset=)` to be surgical about which rows to drop. Never use plain `dropna()` unless you're sure every column must be complete.
> - Always convert date columns with `pd.to_datetime()` — raw strings can't be sorted or filtered by date range.
> - `str.strip().str.upper()` on category columns before any GroupBy — prevents silent grouping errors.

### 3. Phase 3 — Filtering and Selecting Data

**Business Problem:** The CTO asks: "Show me all HDFC transactions above ₹10,000 that failed in January." In Excel, Meera would click 4 filter dropdowns. In Pandas, it's one line of code — and it runs in milliseconds on 80,000 rows.

#### 3.1 Select a Single Column

```
# Get the "amount" column as a Series
df["amount"]

# Get multiple columns as a DataFrame
df[["amount", "bank", "status"]]
```

**📖 Single vs Double Brackets**

`df["amount"]` — single brackets → returns a **Series** (one column).  
  
`df[["amount", "bank"]]` — double brackets, a list inside → returns a **DataFrame** (multiple columns).  
  
This trips up every beginner. Remember: if you pass a list, you get a DataFrame back. If you pass a string, you get a Series.

#### 3.2 Filter Rows with Conditions

```
# Only HDFC transactions
hdfc = df[df["bank"] == "HDFC"]

# Transactions above ₹10,000
high_value = df[df["amount"] > 10000]
```

**📖 Boolean Indexing**

`df["bank"] == "HDFC"` creates a Series of True/False — one value per row. Passing that inside `df[...]` keeps only the rows where the condition is True.  
  
This is called **boolean indexing** and it's the heart of Pandas filtering. No SQL WHERE clause needed.

#### 3.3 Multiple Conditions — AND / OR

```
# HDFC AND failed AND amount > ₹10,000
mask = (
    (df["bank"]   == "HDFC")    &
    (df["status"] == "failed")   &
    (df["amount"] >  10000)
)
result = df[mask]
```

**📖 & for AND, | for OR**

In Pandas, you use `&` (not `and`) and `|` (not `or`) to combine conditions. Each condition must be wrapped in parentheses.  
  
Storing the combined condition in a variable called `mask` is a best practice — it keeps your filtering logic readable when you have 3 or more conditions.

#### 3.4 loc and iloc — Precise Row/Column Selection

```
  df.loc[ row_label , column_name ]     ← label-based
  df.iloc[ row_number, column_number ]  ← position-based (like Python list indices)

  df.loc[0, "amount"]          → value at row index 0, column "amount"
  df.loc[0:4, "bank":"status"] → rows 0-4, columns bank through status (inclusive)

  df.iloc[0, 2]                → row 0, column 2 (0-indexed)
  df.iloc[0:5, :]              → rows 0-4, all columns

  Rule of thumb:
  ► Use loc  when you know the column NAME  (almost always)
  ► Use iloc when you need a specific row NUMBER position
```

```
# Get specific rows and columns by label
df.loc[0:4, ["transaction_id", "amount"]]

# Get first 10 rows, first 3 columns by position
df.iloc[:10, :3]
```

**📖 loc vs iloc**

`loc` is for humans — you refer to rows and columns by their names/labels. Readable and safe.  
  
`iloc` is for machines — you use integer positions (like Python list indexing). Useful when you need the nth row regardless of what the index says.  
  
A common mistake: after filtering, row indices are not contiguous (0,1,2,3...) any more. `iloc[0]` always means "first row of the current DataFrame", while `loc[0]` means "the row whose index label is 0" — which might not exist after filtering.

### 4. Phase 4 — GroupBy and Aggregation

**Business Problem:** The CTO's daily report needs: total transaction amount per bank, count of failed transactions per bank, and average transaction size per bank. In Excel this is a 5-minute pivot table. In Pandas it's 3 lines — and it runs automatically every morning via a cron job.

#### 4.1 GroupBy — Split, Apply, Combine

```
  HOW GROUPBY WORKS — Split, Apply, Combine

  Original DataFrame                After groupby("bank").sum("amount")
  ──────────────────                ───────────────────────────────────
  bank    amount                    bank    amount
  HDFC    5000        SPLIT →       HDFC    5200       ← 5000 + 200
  ICICI   1200        APPLY sum     ICICI   1200
  HDFC     200        COMBINE       AXIS    8500
  AXIS    8500

  Step 1 — SPLIT:   Group rows by unique bank values
  Step 2 — APPLY:   Run the aggregation (sum/count/mean/max) on each group
  Step 3 — COMBINE: Stack the results into a new, smaller DataFrame
```

```
# Total amount per bank
df.groupby("bank")["amount"].sum()
```

**📖 groupby().agg()**

Read this as: "group the DataFrame by bank name, then for each group compute the sum of the amount column."  
  
The result is a Series where the index is the bank names and the values are the totals — exactly what you'd get from an Excel pivot table, in one line.

```
bank
AXIS     32,150,000
HDFC     91,200,000
ICICI    45,600,000
KOTAK    18,900,000
SBI      61,750,000
Name: amount, dtype: float64
```

#### 4.2 Multiple Aggregations at Once

```
# Count transactions AND sum amounts per bank
summary = df.groupby("bank").agg(
    total_amount   = ("amount", "sum"),
    num_txns       = ("transaction_id", "count"),
    avg_amount     = ("amount", "mean"),
    max_amount     = ("amount", "max")
).reset_index()
```

**📖 Named Aggregations**

`agg(new_column_name = ("source_column", "function"))` — this pattern lets you compute multiple stats in one GroupBy call and give each result a clean name.  
  
`reset_index()` converts the bank names from the index back into a regular column, so the result is a flat DataFrame you can export to CSV or send to the CTO.

bank

total_amount

num_txns

avg_amount

max_amount

0

AXIS

32,150,000

4,230

7,601

499,000

1

HDFC

91,200,000

18,400

4,956

750,000

2

ICICI

45,600,000

9,800

4,653

600,000

#### 4.3 GroupBy with Multiple Keys

```
# Failed transactions per bank per month
df["month"] = df["date"].dt.to_period("M")

failures = (
    df[df["status"] == "failed"]
    .groupby(["month", "bank"])
    ["transaction_id"].count()
    .reset_index()
)
```

**📖 Multi-Key GroupBy**

Pass a list of column names to `groupby()` to group by multiple dimensions simultaneously.  
  
`dt.to_period("M")` converts a datetime column to a "Month Period" — e.g., `2024-01`. This lets you group by month without worrying about specific day values.  
  
The pattern here is: filter first (only failed rows), then group — this is faster than grouping everything and then filtering the groups.

### 5. Phase 5 — Merging DataFrames

**Business Problem:** DataPay has two files: `payments.csv` (transaction data) and `customers.csv` (customer info). To answer "which customer tier generates the most failed transactions?", we need to join them — exactly like a SQL JOIN.

```
  MERGE — Like a SQL JOIN

  payments.csv                customers.csv
  ────────────                ─────────────
  customer_id  amount         customer_id  name     tier
  1001         5000           1001         Amit     Gold
  1002         1200      ──►  1002         Priya    Silver
  1003         8500           1003         Rahul    Platinum

  After merge on "customer_id":
  customer_id  amount  name   tier
  1001         5000    Amit   Gold
  1002         1200    Priya  Silver
  1003         8500    Rahul  Platinum
```

```
customers = pd.read_csv("customers.csv")

# Inner join — only matching rows
merged = pd.merge(
    df,
    customers,
    on="customer_id",
    how="left" # keep all payment rows
)
```

**📖 pd.merge() — The Four Join Types**

`how="inner"` — Only rows that match in both DataFrames. Drops unmatched.  
  
`how="left"` — All rows from the left (payments), matched rows from right (customers). Unmatched customers get NaN. **Most common in data pipelines.**  
  
`how="right"` — All rows from the right.  
  
`how="outer"` — All rows from both, NaN where no match.

> **💡 Merge vs Join vs Concat — When to use which?**

> **pd.merge()** — Join two DataFrames on a common column (like SQL JOIN). Use this most of the time.  

**df.join()** — Join on the index, not a column. Shortcut for index-based merges.  

**pd.concat()** — Stack DataFrames vertically (more rows) or horizontally (more columns). Use when you have 5 bank CSV files to combine into one.

#### 5.1 Concatenate Multiple Files

```python
import glob

# Load all 5 bank CSV files at once
files = glob.glob("bank_data/*.csv")
all_dfs = [pd.read_csv(f) for f in files]
df = pd.concat(all_dfs, ignore_index=True)
```

**📖 pd.concat() — Stack Files Together**

`glob.glob("*.csv")` finds all CSV files matching a pattern. The list comprehension reads each one into a DataFrame. `pd.concat()` stacks them vertically — like Excel's "Append" feature but for any number of files.  
  
`ignore_index=True` resets the row index to 0,1,2,3... in the combined DataFrame. Without it, each file's original indices are preserved — causing duplicate index values.

### 6. Phase 6 — String and DateTime Operations

**Business Problem:** DataPay needs to extract the month from transaction dates, parse customer email domains to find which corporate clients transact most, and categorise transaction notes using keyword search. All of this requires string and datetime operations.

#### 6.1 DateTime Operations with .dt

```
# Extract date parts from a datetime column
df["year"]    = df["date"].dt.year
df["month"]   = df["date"].dt.month
df["weekday"] = df["date"].dt.day_name()

# Filter: only January 2024 transactions
jan = df[(df["date"].dt.year  == 2024) &
          (df["date"].dt.month == 1)]
```

**📖 The .dt Accessor**

When a column is datetime type, `.dt` unlocks a set of date/time properties: `.year`, `.month`, `.day`, `.hour`, `.day_name()`, `.is_weekend`, and many more.  
  
This is why converting your date column with `pd.to_datetime()` during cleaning matters — without it, `.dt` doesn't exist and date filtering requires slow string parsing.

#### 6.2 String Operations with .str

```
# Extract domain from email column
df["domain"] = (
    df["email"]
    .str.split("@")
    .str.get(1)
)

# Find transactions mentioning "refund"
refunds = df[df["note"].str.contains("refund", case=False, na=False)]
```

**📖 The .str Accessor**

Just like `.dt` for datetimes, `.str` unlocks string methods for an entire column at once — no loops.  
  
`str.contains("refund", case=False)` — case-insensitive keyword search across 80,000 rows in milliseconds. `na=False` prevents it from crashing on null values in the notes column.  
  
`str.split("@").str.get(1)` — splits every email on "@" and takes the second part (index 1), which is the domain.

### 7. Phase 7 — Apply Functions and Creating New Columns

**Business Problem:** DataPay needs to categorise each transaction as "micro" (<₹1K), "standard" (₹1K–₹50K), or "high-value" (>₹50K) for risk scoring. No single Pandas method does this — it requires a custom function applied row-by-row.

#### 7.1 Create a New Column — Vectorised (Fast)

```
# Add a GST column — vectorised, instant
df["gst_amount"] = df["amount"] * 0.18
# Flag high-value transactions
df["is_high_value"] = df["amount"] > 50000
```

**📖 Vectorised Column Creation**

Arithmetic and comparison operations on a column automatically apply to every row. This is **vectorisation** — Pandas does the loop internally in compiled C code, making it 100x faster than a Python for-loop.  
  
Always prefer vectorised operations over `apply()` when the logic can be expressed with standard operators (+, -, *, /, ==, >).

#### 7.2 apply() — For Custom Logic

```python
# Custom risk category function
def categorise(amount):
    if amount < 1000:  return "micro"
elif amount < 50000: return "standard"
else:                  return "high_value"
df["risk_tier"] = df["amount"].apply(categorise)
```

**📖 apply() — Call a Function on Every Row**

`apply(function)` calls your function once per value in the column. The return value becomes the new column's value for that row.  
  
**Note:** `apply()` is slower than vectorised operations because Python calls the function 80,000 times. Use it only when the logic requires if/elif/else or calls that can't be vectorised. For this specific case, `pd.cut()` is even faster (see below).

#### 7.3 pd.cut() — Bin Numeric Values (Faster Alternative)

```
# Bin amount into categories — faster than apply()
df["risk_tier"] = pd.cut(
    df["amount"],
    bins   = [0, 1000, 50000, float("inf")],
    labels = ["micro", "standard", "high_value"]
)
```

**📖 pd.cut() — Vectorised Binning**

`pd.cut()` divides a numeric column into labelled intervals — exactly like if/elif/else buckets, but vectorised.  
  
`bins` defines the edges. `labels` names each interval. The result is a Categorical column — memory-efficient and sortable in bin order rather than alphabetically.

### 8. Phase 8 — Pivot Tables and Reshaping

**Business Problem:** The CTO wants a table: banks as rows, months as columns, total transaction amount as values. This is an Excel pivot table — and Pandas has it built in.

#### 8.1 Pivot Table

```
pivot = pd.pivot_table(
    df,
    values = "amount",
    index  = "bank",
    columns= "month",
    aggfunc= "sum",
    fill_value=0
)
```

**📖 pd.pivot_table()**

`values` — the column to aggregate.  
`index` — what becomes the rows.  
`columns` — what becomes the columns.  
`aggfunc` — how to aggregate (sum, mean, count).  
`fill_value=0` — replace NaN with 0 for months where a bank had no transactions.  
  
The result is a 2D summary table ready to export to Excel for the CTO.

#### 8.2 melt() — Unpivot / Wide to Long

```
# Wide format: one column per month
# Long format: one row per bank+month
long_df = pivot.reset_index().melt(
    id_vars  ="bank",
    var_name ="month",
    value_name="total_amount"
)
```

**📖 melt() — Wide to Long**

Pivot tables are in "wide" format — one column per month. Most charting libraries and databases prefer "long" format — one row per observation.  
  
`melt()` unpivots: takes column headers and turns them into row values. `id_vars` are the columns to keep as-is; everything else gets "melted" into two columns — one for the old column name, one for the value.

### 9. Phase 9 — Export and Visualise

**Business Problem:** The pipeline is complete. Now we need to send the CTO a clean Excel report, save a CSV for the database team, and plot a bar chart of transactions by bank for the weekly presentation.

#### 9.1 Export to CSV and Excel

```
# Save to CSV
summary.to_csv("daily_report.csv", index=False)

# Save to Excel — multiple sheets
with pd.ExcelWriter("report.xlsx") as writer:
    summary.to_excel(writer, sheet_name="Summary", index=False)
    pivot.to_excel(writer, sheet_name="By Month")
```

**📖 to_csv() and to_excel()**

`index=False` — Don't write the DataFrame's row index (0,1,2,3) as a column in the output. Almost always what you want.  
  
`ExcelWriter` as a context manager lets you write multiple DataFrames to different sheets in the same workbook — something Excel users always need but find painful to automate. Pandas does it in 3 lines.

#### 9.2 Quick Plots with Pandas + Matplotlib

```python
import matplotlib.pyplot as plt

# Bar chart: total amount per bank
summary.set_index("bank")["total_amount"].plot(
    kind="bar", title="Total Transactions by Bank",
    color="steelblue", figsize=(10,5), rot=0
)
plt.tight_layout()
plt.savefig("bank_summary.png", dpi=150)
```

**📖 df.plot() — Built-in Charts**

Every Pandas Series and DataFrame has a `.plot()` method that wraps Matplotlib. `kind=` accepts: `"bar"`, `"barh"`, `"line"`, `"hist"`, `"box"`, `"pie"`, `"scatter"`.  
  
For production-quality charts, switch to `seaborn` or `plotly`. But for quick exploratory plots during analysis, Pandas' built-in `.plot()` is fast and requires no extra code.

### 10. Phase 10 — The Complete DataPay Pipeline

**Business Problem:** Combine everything into a single, readable pipeline that Meera can run every morning — replacing her 3-hour Excel session with a 3-second script.

#### 10.1 The Full Pipeline — Start to Finish

```python
import pandas as pd
import glob

# ── STEP 1: LOAD ──────────────────────────────────────
files = glob.glob("bank_data/*.csv")
df = pd.concat([pd.read_csv(f) for f in files], ignore_index=True)

# ── STEP 2: CLEAN ─────────────────────────────────────
df.rename(columns={"Txn ID":"transaction_id", "Amt (INR)":"amount", "Bank Name":"bank"}, inplace=True)
df.dropna(subset=["date"], inplace=True)
df.drop_duplicates(subset=["transaction_id"], inplace=True)
df["bank"]   = df["bank"].str.strip().str.upper()
df["amount"] = df["amount"].str.replace(",", "").astype(float)
df["date"]   = pd.to_datetime(df["date"])
df["status"].fillna("unknown", inplace=True)

# ── STEP 3: ENRICH ────────────────────────────────────
df["month"]     = df["date"].dt.to_period("M")
df["risk_tier"] = pd.cut(df["amount"], bins=[0,1000,50000,float("inf")], labels=["micro","standard","high_value"])

# ── STEP 4: ANALYSE ───────────────────────────────────
summary = df.groupby("bank").agg(
    total_amount = ("amount", "sum"),
    num_txns     = ("transaction_id", "count"),
    avg_amount   = ("amount", "mean")
).reset_index()

# ── STEP 5: EXPORT ────────────────────────────────────
summary.to_csv("daily_report.csv", index=False)
print("✅ Report generated. Rows processed:", len(df))
```

> **This is the complete DataPay pipeline.** 20 lines replace Meera's 3-hour Excel session. It loads all 5 bank CSV files, cleans and deduplicates 80,000 rows, enriches with month and risk tier, produces a grouped summary, and exports to CSV — all in under 3 seconds. Every step is a concept from earlier phases. This is the payoff.

### 11. Quick Reference — Most-Used Pandas Commands

Command

What it does

pd.read_csv("file.csv")

Load a CSV file into a DataFrame

df.head() / df.tail()

Preview first / last 5 rows

df.shape

Tuple of (rows, columns)

df.info()

Column types and null counts

df.describe()

Summary stats: mean, min, max, std

df.isnull().sum()

Count missing values per column

df.dropna(subset=["col"])

Drop rows where column is null

df["col"].fillna(value)

Replace nulls with a default value

df.drop_duplicates(subset=)

Remove duplicate rows

df.rename(columns={})

Rename columns

df["col"].astype(float)

Change a column's data type

pd.to_datetime(df["col"])

Convert strings to datetime

df[df["col"] == value]

Filter rows by condition

df.loc[rows, cols]

Select by label

df.iloc[rows, cols]

Select by position

df.groupby("col").agg()

Group and aggregate

pd.merge(df1, df2, on=, how=)

Join two DataFrames

pd.concat([df1, df2])

Stack DataFrames vertically

df["col"].str.upper()

String operations on a column

df["col"].dt.month

Extract date parts

df["col"].apply(func)

Apply a custom function

pd.cut(df["col"], bins=)

Bin numeric values into categories

pd.pivot_table(df, ...)

Create a pivot table

df.melt(id_vars=)

Wide to long format

df.to_csv("file.csv")

Export to CSV

df.to_excel("file.xlsx")

Export to Excel

df.plot(kind="bar")

Quick bar/line/histogram chart

**Quiz: ❓ Quiz 1 — You load a CSV and find the "amount" column has dtype object. What's the most likely cause and fix?**

- A) The column has too many values — use df["amount"].astype(int)
- B) Some values have commas or currency symbols — clean with str.replace() then astype(float)
- C) The CSV file is corrupted — reload it
- D) Pandas can't handle large numbers — use numpy instead

> **Answer/explanation:** ✅ **Answer: B.** When a numeric column loads as object (string), it almost always means some values contain non-numeric characters — commas like "5,000", currency symbols like "₹5000", or spaces. Fix: `df["amount"].str.replace(",", "").str.replace("₹", "").astype(float)`. This is one of the most common real-world Pandas cleaning steps.

**Quiz: ❓ Quiz 2 — What's wrong with this code? df[df["bank"] == "HDFC" and df["amount"] > 10000]**

- A) You should use df.loc[] instead of df[]
- B) Python's "and" doesn't work on Series — use & with parentheses around each condition
- C) The condition order is wrong — amount must come first
- D) Nothing is wrong

> **Answer/explanation:** ✅ **Answer: B.** Python's built-in `and` and `or` operators don't work on Pandas Series — they can only evaluate single True/False values. For multiple conditions in Pandas, use `&` (AND) and `|` (OR), and wrap each condition in parentheses: `df[(df["bank"] == "HDFC") & (df["amount"] > 10000)]`.

**Quiz: ❓ Quiz 3 — You groupby "bank" and get bank names as the index. You want them as a regular column. What do you call?**

- A) df.set_index("bank")
- B) df.reset_index()
- C) df.reindex()
- D) df.index.to_list()

> **Answer/explanation:** ✅ **Answer: B — reset_index().** After a groupby().agg(), the grouping column becomes the index. `reset_index()` moves it back to a regular column, giving you a flat DataFrame with a clean 0,1,2... index. This is a standard last step in any groupby chain before exporting or displaying results.

##### 🙋 Fresher Q&A — Questions You Will Definitely Have

**Q: Q: What's the difference between df.drop_duplicates() and df.isnull().sum()?**

A: They solve different problems. `drop_duplicates()` removes rows that are exact copies of each other (or match on a key column). `isnull().sum()` counts missing/NaN values — rows that exist but have blank fields. You need both in every cleaning pipeline — they fix different categories of dirty data.

**Q: Q: When should I use apply() vs a vectorised operation?**

A: If your logic can be written as arithmetic (+, -, *, /) or a comparison (==, >, <), use vectorised — it's 10–100x faster. Only reach for `apply()` when your logic requires if/elif/else branches or calls external libraries per row. For binning numbers, `pd.cut()` is vectorised and faster than `apply()` with if/else.

**Q: Q: Why does df.info() show "object" dtype for my date column even though the values look like dates?**

A: Pandas doesn't automatically convert date strings — it loads them as plain text (object). You must explicitly convert: `df["date"] = pd.to_datetime(df["date"])`. After this, the dtype becomes `datetime64[ns]` and you can use the `.dt` accessor to extract year, month, day, weekday, etc.

**Q: Q: What does inplace=True do? Should I use it?**

A: `inplace=True` modifies the DataFrame in place instead of returning a new one. It saves one line (no `df =` assignment). However, many experienced Pandas users avoid it because it can cause subtle bugs in complex pipelines and it's harder to debug. Both styles are correct — the assignment style (`df = df.dropna(...)`) is safer and more readable, especially when learning.

##### ✅ Pandas Best Practices for Real Data Pipelines

- Always run `info()` and `isnull().sum()` immediately after loading data — before any analysis.
- Standardise column names to snake_case at the top of every pipeline — prevents KeyErrors later.
- Always pass `ignore_index=True` when using `pd.concat()` — prevents duplicate index issues.
- Prefer vectorised operations over `apply()` whenever possible — it's dramatically faster on large datasets.
- Always use `dropna(subset=[...])` instead of plain `dropna()` — surgical, not aggressive.
- Run `str.strip().str.upper()` on any text category column before groupby — prevents silent grouping errors.
- Convert date columns with `pd.to_datetime()` at the cleaning step — never work with date strings.
- Use `index=False` in `to_csv()` and `to_excel()` — the index column is almost never useful in output files.

##### 🧪 Practice Exercise — Build the DataPay Pipeline Yourself

1. Create a `payments.csv` with columns: transaction_id, date, amount, bank, status, customer_id. Add 20 rows with intentional issues: 2 duplicates, 3 null dates, mixed-case bank names, one amount as "5,000" string.
2. Load the CSV with `pd.read_csv()` and run `info()` — note how the issues appear.
3. Clean the data: rename columns, drop null dates, remove duplicates, standardise bank names, fix the amount column, convert date to datetime.
4. Add a "risk_tier" column using `pd.cut()` with thresholds: micro <₹1K, standard ₹1K–₹50K, high_value >₹50K.
5. GroupBy bank and compute: total amount, transaction count, average amount, percentage failed.
6. Export the summary to `daily_report.csv` and plot a bar chart of total amount by bank.

### Pandas Project Complete 🎉

You have built DataPay's complete Pandas data pipeline — from pd.read_csv on day one to loading, cleaning, filtering, groupby aggregation, merging, string and datetime operations, pivot tables, apply functions, and export. You now speak the language of every data team on the planet.

> **Rohan**
> 
> "Three weeks ago, Meera spent 3 hours every morning opening five Excel files, removing duplicates by hand, copy-pasting into a master sheet, and building a pivot table — only to see Excel crash and lose it all. This morning, the pipeline ran in 2.8 seconds. Same data. Same output. But reproducible, automated, and crash-proof. That's what Pandas does — it takes manual, error-prone Excel work and turns it into reliable, versioned code."

> **Meera**
> 
> "The moment that changed everything for me: understanding that every column of a DataFrame is a Series, and that operations like str.upper() or dt.month apply to the entire column at once — no loops. Once I saw that, the whole library made sense. Pandas doesn't replace your thinking; it removes the friction between your question and your answer."

> **Priya**
> 
> "And the HDFC groupby error? That was the best lesson. Wrong data in, wrong answers out — no error message, no warning, just a silently incorrect report. Always clean first. Always check unique values before groupby. Always use info() before analysis. Pandas is powerful, but data quality is your responsibility."
