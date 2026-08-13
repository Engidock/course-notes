# 📊 Matplotlib Project Mastery

> **👋 Hey Fresher — Read This First!**

> Matplotlib is Python's original charting library — every data analyst, scientist, and engineer uses it to turn raw numbers into visual stories. Without visualisation, a table of 80,000 rows tells you nothing at a glance. A well-made chart makes the trend obvious in 3 seconds. This document uses **short code blocks** — each one covers exactly one concept — with a plain-English explanation right beside it. No 200-line notebooks. One chart type at a time, always explained simply.

> **Company in this project:** VisualEdge — a Bengaluru-based analytics dashboard startup. They serve 12 FMCG clients who need weekly sales charts, regional comparisons, and trend lines sent every Monday morning. Right now, their analyst Priya exports CSV data and manually builds charts in Excel every Sunday night. Your lead is Arjun. You are about to replace her 4-hour Sunday ritual with a clean, reproducible Matplotlib pipeline that runs in under 10 seconds.

#### What You Will Learn and Build in This Project

You will build a complete Matplotlib visualisation pipeline for VisualEdge — from a basic line chart to a multi-panel dashboard — learning every core concept along the way, each explained with a real business reason.

Figure & Axes, Line Chart, Bar Chart, Scatter Plot, Histogram, Pie Chart, Subplots, Styling, Annotations, Save & Export

> **📦 Phase 1 — Foundations**

> Install Matplotlib, understand Figure vs Axes, plot your first line chart, and learn the two-interface system (pyplot vs OO API).

> Build bar charts, scatter plots, histograms, and pie charts — each with real VisualEdge sales data and clear business context.

> Titles, labels, legends, colours, fonts, gridlines, annotations, and custom styles — make your charts look production-ready.

> Multi-panel dashboards with subplots, tight layouts, saving figures as PNG/PDF, and integration with Pandas DataFrames.

### 1. Phase 1 — Matplotlib Foundations

**Business Problem:** VisualEdge's client reports are built by hand in Excel every Sunday. Each chart takes 20 minutes of clicking. Arjun needs a Python script that regenerates all 8 charts in 10 seconds whenever the CSV data is updated — consistent, branded, and ready to embed in PDF reports.

**Scene 1 — VisualEdge Office | Sunday Night Crisis**

> **Priya** _Data Analyst — VisualEdge_
> 
> Arjun, I've been here since 6 PM. The client just sent a corrected CSV — the July sales numbers changed. That means I have to redo all 8 Excel charts from scratch. By hand. Again. I submit every Monday at 9 AM and I still have 4 charts left.

> **Arjun** _Lead Engineer — VisualEdge_
> 
> Riya, your first task: write a Matplotlib script that loads the CSV and generates all 8 charts automatically. When the client sends a corrected file, Priya runs one command and all charts are regenerated in under 10 seconds — same style, same layout, same filenames. Zero manual work.

#### 1.1 Install Matplotlib

```bash
# Install matplotlib
pip install matplotlib

# Verify installation
import matplotlib
print(matplotlib.__version__)
```

**📖 Why matplotlib?**

Matplotlib is the foundation of almost every Python charting library — Seaborn, Pandas `.plot()`, and even parts of Plotly build on top of it. Learn Matplotlib once and you understand the chart ecosystem.  
  
You will also use `matplotlib.pyplot`, aliased as `plt` — just like Pandas is `pd`, Matplotlib is `plt` by universal convention.

```
3.9.2
```

#### 1.2 The Two Core Objects — Figure and Axes

Understanding Figure vs Axes is the single most important concept in Matplotlib. Every confusion beginners have traces back to not knowing the difference.

┌──────────────────────────────────────────────────────────────┐ │ FIGURE (the canvas) │ │ fig = plt.figure() or fig, ax = plt.subplots() │ │ │ │ ┌────────────────────────────────────────────────────┐ │ │ │ AXES (the actual chart) │ │ │ │ ax = fig.add_subplot() or _, ax = plt.subplots()│ │ │ │ │ │ │ │ ▲ Y-Axis label ax.set_ylabel("Sales ₹") │ │ │ │ │ │ │ │ │ │ ╭──╮ ╭──╮ │ │ │ │ │ │ │ ╭──│ │──╮ ← plot / bar / scatter │ │ │ │ │ │ │ │ │ │ │ drawn on the Axes │ │ │ │ └───┴──┴─┴──┴──┴──┴──► X-Axis │ │ │ │ ax.set_xlabel("Month") │ │ │ │ ax.set_title("Monthly Sales") │ │ │ │ ax.legend() ax.grid(True) │ │ │ └────────────────────────────────────────────────────┘ │ │ │ │ fig.savefig("chart.png") ← saves the entire Figure │ └──────────────────────────────────────────────────────────────┘ Figure = the full white canvas / window. One figure can hold many Axes. Axes = one chart panel. Holds the plot, axis labels, title, legend. pyplot = plt.plot() is a shortcut — it draws on the "current" Axes.

> **💡 Think of it like this**

> A **Figure** is like a blank PowerPoint slide — the canvas. An **Axes** is a chart box placed on that slide. One slide can have multiple chart boxes (subplots). When you call `plt.plot()` directly, Matplotlib quietly creates a Figure and an Axes for you. When you create them yourself with `fig, ax = plt.subplots()`, you have full control — that's the professional way.

#### 1.3 Your First Line Chart — The Beginner Way (pyplot)

```python
import matplotlib.pyplot as plt

months  = ["Jan", "Feb", "Mar", "Apr"]
sales   = [120, 145, 132, 178]

plt.plot(months, sales)
plt.show()
```

**📖 plt.plot() and plt.show()**

`plt.plot(x, y)` draws a line chart on the current Axes.  
  
`plt.show()` renders the figure in a window or notebook. Without it, nothing appears.  
  
This is the **pyplot (quick) interface**. Perfect for a first experiment. For production charts, you'll use the **OO (object-oriented) API** — shown next — which gives you full control.

#### 1.4 The Professional Way — Object-Oriented API

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots()

ax.plot(months, sales)
ax.set_title("Monthly Sales")
ax.set_xlabel("Month")
ax.set_ylabel("Sales (₹ Lakhs)")

plt.show()
```

**📖 fig, ax = plt.subplots()**

`plt.subplots()` returns two objects: the **Figure** and the **Axes**.  
  
You call methods on `ax` directly — `ax.plot()`, `ax.set_title()`, `ax.set_xlabel()`. This is cleaner, explicit, and the correct way for any chart you want to customise or save.  
  
Every experienced Matplotlib user writes charts this way.

> **✅ Phase 1 Key Takeaways**

> - Figure = canvas. Axes = chart. One Figure can hold multiple Axes (subplots).
> - `plt.plot(x, y)` is the quick way. `fig, ax = plt.subplots()` is the production way.
> - Always call `plt.show()` — otherwise the chart won't render.
> - Call methods on `ax` (not `plt`) when using the OO API — `ax.set_title()`, `ax.set_xlabel()`.

### 2. Phase 2 — Chart Types

**Business Problem:** VisualEdge needs 4 different chart types for their Monday reports: a line chart for trends, a bar chart for regional comparisons, a scatter for correlation analysis, and a histogram for distribution of order values. Each chart tells a different story about the same data.

**Scene 2 — VisualEdge Standup | "The Wrong Chart Problem"**

> **Client Email (paraphrased by Arjun)** _Feedback from VisualEdge client — FMCG company_
> 
> Your sales chart for Q3 uses a pie chart to show monthly trends. Pie charts can't show change over time — the client's MD couldn't understand if July was better or worse than June. He wants a line chart for trends, a bar chart for comparisons, and a scatter only when you're showing two variables together. Use the right chart for the right question.

> **Arjun** _Lead Engineer — VisualEdge_
> 
> Riya, chart selection is not a styling decision — it's a communication decision. Line = trend over time. Bar = comparison between categories. Scatter = relationship between two numbers. Histogram = distribution of one number. Pie = part of a whole (rarely useful). Learn all four this week.

#### 2.1 Line Chart — Trends Over Time

```
months = ["Jan","Feb","Mar","Apr","May","Jun"]
sales  = [120, 145, 132, 178, 160, 195]

fig, ax = plt.subplots()
ax.plot(months, sales,
        marker="o",
        color="#1976d2",
        linewidth=2)
ax.set_title("H1 Monthly Sales")
ax.set_ylabel("₹ Lakhs")
plt.tight_layout()
plt.show()
```

**📖 ax.plot() key parameters**

`marker="o"` — draws a circle dot at each data point. Other options: `"s"` (square), `"^"` (triangle), `"D"` (diamond).  
  
`color=` — accepts hex colours, names like `"red"`, or shortcodes like `"b"` (blue).  
  
`linewidth=` — thickness of the line. Default is 1.5.  
  
`plt.tight_layout()` — auto-adjusts padding so labels don't get cut off. Always call it before `show()`.

📈 Output: Line Chart

A smooth line connecting Jan–Jun with blue circle markers at each month. Clear upward trend visible at a glance — exactly what the client's MD asked for to understand monthly performance.

#### 2.2 Bar Chart — Comparing Categories

```
regions = ["North", "South", "East", "West"]
revenue = [340, 520, 290, 410]

fig, ax = plt.subplots()
ax.bar(regions, revenue,
       color=["#1976d2","#e84d0e","#388e3c","#7b1fa2"])
ax.set_title("Q2 Revenue by Region")
ax.set_ylabel("Revenue (₹ Lakhs)")
plt.tight_layout()
plt.show()
```

**📖 ax.bar() — vertical bar chart**

`ax.bar(categories, values)` creates a vertical bar chart — the best chart for comparing quantities across categories (regions, products, banks).  
  
Passing a **list of colours** applies a different colour to each bar — great for making regions visually distinct in client reports.  
  
For a horizontal bar chart (better for long category names), use `ax.barh()` instead.

📊 Output: Bar Chart

Four bars in different colours showing South as the top region (₹520L) and East as the lowest (₹290L). The client can compare all four regions in one second — impossible with a data table.

#### 2.3 Horizontal Bar Chart — Long Category Names

```
products = ["Shampoo Pro", "Face Wash X",
              "Body Lotion", "Sunscreen SPF"]
units    = [4200, 3800, 5100, 2900]

fig, ax = plt.subplots()
ax.barh(products, units,
        color="#11567f")
ax.set_title("Units Sold by Product")
ax.set_xlabel("Units Sold")
plt.tight_layout()
plt.show()
```

**📖 ax.barh() — horizontal bar chart**

When your category names are long (product names, city names, bank names), vertical bars cause label overlap. `barh()` flips the chart — categories on the Y-axis, values on the X-axis.  
  
The only parameter swap: `set_xlabel()` instead of `set_ylabel()` for the value axis. Everything else is identical to `bar()`.

#### 2.4 Scatter Plot — Relationship Between Two Variables

```python
import numpy as np

# Ad spend vs revenue for 30 cities
ad_spend = np.random.randint(50, 500, 30)
revenue  = ad_spend * 2.5 + np.random.normal(0,40,30)

fig, ax = plt.subplots()
ax.scatter(ad_spend, revenue,
           color="#e84d0e", alpha=0.7)
ax.set_xlabel("Ad Spend (₹K)")
ax.set_ylabel("Revenue (₹K)")
ax.set_title("Ad Spend vs Revenue")
plt.show()
```

**📖 ax.scatter() — show correlations**

Use a scatter plot when you want to see if two numeric variables are related (correlated). Each dot is one data point — one city, one transaction, one store.  
  
`alpha=0.7` — transparency (0=invisible, 1=solid). Use 0.5–0.8 when dots overlap.  
  
If the dots form a diagonal cloud going up-right, there's a positive correlation (more ad spend → more revenue). If it's a horizontal blob, there's no correlation — a critical insight.

#### 2.5 Histogram — Distribution of One Variable

```
# Order values for 1000 transactions
order_values = np.random.exponential(
    scale=500, size=1000)

fig, ax = plt.subplots()
ax.hist(order_values,
        bins=30,
        color="#388e3c",
        edgecolor="white")
ax.set_title("Order Value Distribution")
ax.set_xlabel("Order Value (₹)")
ax.set_ylabel("Frequency")
plt.show()
```

**📖 ax.hist() — understand distributions**

A histogram groups numerical values into bins and counts how many fall in each. It answers: "Are most orders small or large? Is the distribution skewed?"  
  
`bins=30` — number of equal-width buckets. More bins = more detail but noisier. Start with 20–30 for most datasets.  
  
`edgecolor="white"` — adds a white border between bars so they're visually separated.  
  
For VisualEdge: a right-skewed histogram tells the client that most orders are small, but a few high-value orders drive most revenue — an actionable insight about where to focus.

#### 2.6 Pie Chart — Parts of a Whole (Use Sparingly)

```
categories = ["Skin Care", "Hair Care",
               "Body Care", "Others"]
shares     = [42, 28, 20, 10]

fig, ax = plt.subplots()
ax.pie(shares,
       labels=categories,
       autopct="%1.1f%%",
       startangle=90)
ax.set_title("Revenue by Category")
plt.show()
```

**📖 ax.pie() — show proportions**

`autopct="%1.1f%%"` — automatically adds percentage labels inside each slice (e.g. "42.0%"). The format string controls decimal places.  
  
`startangle=90` — rotates the first slice to start at the top (12 o'clock position). Looks cleaner than the default 3 o'clock start.  
  
**Fresher tip:** Pie charts are *only* appropriate when showing parts of a whole (market share, budget breakdown). Never use pie for trends over time — use a line chart.

Chart Type

Function

Use When

ax.plot()

Line Chart

Trends over time — monthly sales, stock prices

ax.bar()

Bar Chart

Comparing values across categories — regions, products

ax.barh()

Horizontal Bar

Same as bar() but for long category names

ax.scatter()

Scatter Plot

Relationship between two numeric variables

ax.hist()

Histogram

Distribution / spread of one numeric variable

ax.pie()

Pie Chart

Part-of-whole proportions (use sparingly)

> **✅ Phase 2 Key Takeaways**

> - Line chart → trend over time. Bar chart → category comparison. Scatter → correlation. Histogram → distribution.
> - `marker=`, `color=`, `linewidth=` are the three most-used `ax.plot()` parameters.
> - `alpha=` controls transparency — useful in scatter plots when dots overlap.
> - `bins=` in `ax.hist()` controls granularity — start with 20–30 bins.
> - Pie charts are a last resort — most data stories are better told with a bar or line chart.

### 3. Phase 3 — Styling & Control

**Business Problem:** VisualEdge's charts look like default Python output — grey background, tiny font, no brand colours. The client's CMO wants charts that match VisualEdge's brand palette, have readable axis labels, annotate key data points, and look professional enough to embed in board-level PDF reports without further editing.

**Scene 3 — VisualEdge Client Review | "These Look Like Student Homework"**

> **Client CMO** _CMO — FMCG Client_
> 
> The data is correct but these charts look unprofessional. Grey background, no company colours, the axis labels are tiny, and there's no annotation on the peak month. I can't send these to the board. Can your team make them look like the charts in our annual report?

> **Arjun** _Lead Engineer — VisualEdge_
> 
> Riya, this week is about styling. White background, larger fonts, branded colours, gridlines, and annotations on the key point. Once we set a consistent style, every chart we produce automatically looks the same — no manual adjustments per chart.

#### 3.1 Labels, Title, and Legend

```
fig, ax = plt.subplots()

ax.plot(months, sales,
        label="2024 Sales")
ax.plot(months, target,
        label="Target",
        linestyle="--")

ax.set_title("Sales vs Target",
             fontsize=16)
ax.set_xlabel("Month")
ax.set_ylabel("₹ Lakhs")
ax.legend()
plt.show()
```

**📖 Labels, legend, and linestyle**

When you plot multiple lines on the same Axes, the legend tells the viewer which line is which. Three steps:  
  
1. Add `label="Series Name"` to each `ax.plot()` call.  
2. Call `ax.legend()` — Matplotlib auto-places the legend where it covers the least data.  
  
`linestyle="--"` — dashed line. Options: `"-"` (solid), `"--"` (dashed), `":"` (dotted), `"-."` (dash-dot).  
  
`fontsize=16` in `set_title()` makes the title larger and more prominent.

#### 3.2 Colours, Gridlines & Background

```
fig, ax = plt.subplots(
    figsize=(10, 5))

ax.set_facecolor("#f8f9fa")
fig.patch.set_facecolor("white")

ax.grid(True,
       linestyle="--",
       alpha=0.6,
       color="#cccccc")

ax.spines["top"].set_visible(False)
ax.spines["right"].set_visible(False)
```

**📖 Professional chart styling**

`figsize=(10, 5)` — chart dimensions in inches (width, height). Use `(10, 5)` for landscape, `(6, 6)` for square.  
  
`ax.set_facecolor()` — Axes plot area colour. `fig.patch.set_facecolor()` — the outer Figure background.  
  
`ax.grid()` — adds horizontal/vertical reference lines. `alpha=0.6` makes them subtle. Light gridlines help readers trace values without dominating the chart.  
  
`ax.spines["top"].set_visible(False)` — removes the border box from the top and right. This modern "open" style looks cleaner and more editorial.

#### 3.3 Annotations — Point to What Matters

```
# Mark the peak month with an annotation
peak_month = "Jun"
peak_value = 195
ax.annotate(
    "Peak: ₹195L",
    xy=(peak_month, peak_value),
    xytext=(4, 210),
    arrowprops={"arrowstyle": "->",
                 "color": "red"},
    fontsize=11,
    color="red"
)
```

**📖 ax.annotate() — draw attention**

Annotations tell the viewer exactly where to look and what it means. For board reports, the client doesn't want to find the peak — they want it pointed out.  
  
`xy=` — the data point to point at (the arrow tip).  
  
`xytext=` — where the label text appears (the arrow tail).  
  
`arrowprops=` — configures the arrow style. `"->"` gives a clean directional arrow.  
  
Annotations are the difference between a chart that shows data and a chart that tells a story.

#### 3.4 Custom Tick Labels & Axis Formatting

```python
import matplotlib.ticker as mticker

# Format y-axis as ₹ values
ax.yaxis.set_major_formatter(
    mticker.FuncFormatter(
        lambda x, _: f"₹{x:,.0f}"
    )
)

# Rotate x-axis labels 45 degrees
plt.xticks(rotation=45,
           ha="right")
```

**📖 Axis formatters and label rotation**

`FuncFormatter(lambda x, _: f"₹{x:,.0f}")` — a custom function applied to every tick. This converts `195000` into `₹1,95,000` on the axis — readable by the client without needing a separate label.  
  
`plt.xticks(rotation=45, ha="right")` — rotates the x-axis labels 45 degrees so they don't overlap. `ha="right"` (horizontal alignment) aligns the end of each label to its tick position — prevents labels from sliding left.

#### 3.5 Apply a Built-in Style Sheet

```python
# Apply a style BEFORE creating figures
plt.style.use("seaborn-v0_8-whitegrid")

# See all available styles
print(plt.style.available)
```

**📖 Style sheets — one line, consistent look**

Matplotlib ships with dozens of pre-built style sheets. One call at the top of your script applies the style to every chart in the script.  
  
Popular styles:  

`"seaborn-v0_8-whitegrid"` — clean white with gridlines, professional  

`"ggplot"` — grey background, colourful (R-style)  

`"fivethirtyeight"` — bold, editorial-style  

`"dark_background"` — dark theme for dashboards  
  

            For VisualEdge's client reports, `seaborn-v0_8-whitegrid` is the standard — it looks clean, printed or on screen.

> **✅ Phase 3 Key Takeaways**

> - `figsize=(w, h)` in `plt.subplots()` controls chart size in inches.
> - Hide top and right spines for a modern, editorial look.
> - `ax.annotate()` turns a chart into a story — point to peaks, drops, targets.
> - `ax.grid()` with low alpha adds subtle reference lines — never let grid dominate the data.
> - `plt.style.use()` applies a consistent look to every chart — use it once at the top of your script.

### 4. Phase 4 — Subplots & Export

**Business Problem:** VisualEdge's Monday report has 4 charts on one page — a trend line, regional bar chart, distribution histogram, and market share pie. Right now, Priya builds each in a separate Excel file. The client wants a single dashboard image that shows all four together, sized to fit on one printed A4 page.

**Scene 4 — VisualEdge Dashboard Sprint**

> **Arjun** _Lead Engineer — VisualEdge_
> 
> Riya, the final deliverable is a single PNG file: 4 charts in a 2×2 grid. Top-left: monthly trend line. Top-right: regional bar chart. Bottom-left: order value histogram. Bottom-right: category pie chart. The whole dashboard generates from one Python script when the CSV updates. That's what we're building today.

#### 4.1 Subplots — Multiple Charts in One Figure

```
# 2x2 grid of charts
fig, axes = plt.subplots(
    nrows=2,
    ncols=2,
    figsize=(14, 10)
)

# Unpack axes for convenience
ax1 = axes[0, 0]  # top-left
ax2 = axes[0, 1]  # top-right
ax3 = axes[1, 0]  # bottom-left
ax4 = axes[1, 1]  # bottom-right
```

**📖 plt.subplots(nrows, ncols)**

`plt.subplots(2, 2)` creates a 2-row, 2-column grid of Axes and returns them as a 2D numpy array. You index into it with `axes[row, col]` using zero-based indexing.  
  
The `figsize=(14, 10)` makes the whole figure large enough so each subplot has enough room — critical for readable axis labels in a grid.  
  
You can now draw on each subplot independently: `ax1.plot(...)`, `ax2.bar(...)`, etc.

#### 4.2 Build the Full 2×2 Dashboard

```python
# ── Complete 2x2 Dashboard for VisualEdge ─────────────────────
import matplotlib.pyplot as plt
import numpy as np

plt.style.use("seaborn-v0_8-whitegrid")

# Data
months      = ["Jan","Feb","Mar","Apr","May","Jun"]
sales       = [120, 145, 132, 178, 160, 195]
regions     = ["North", "South", "East", "West"]
revenue     = [340, 520, 290, 410]
orders      = np.random.exponential(500, 1000)
categories  = ["Skin", "Hair", "Body", "Other"]
shares      = [42, 28, 20, 10]

# Create figure
fig, axes = plt.subplots(2, 2, figsize=(14, 10))
ax1, ax2, ax3, ax4 = axes.flatten()

# Top-left: Line chart
ax1.plot(months, sales, marker="o", color="#1976d2", linewidth=2)
ax1.set_title("H1 Monthly Sales Trend")
ax1.set_ylabel("₹ Lakhs")

# Top-right: Bar chart
ax2.bar(regions, revenue,
        color=["#1976d2","#e84d0e","#388e3c","#7b1fa2"])
ax2.set_title("Q2 Revenue by Region")
ax2.set_ylabel("Revenue (₹L)")

# Bottom-left: Histogram
ax3.hist(orders, bins=30, color="#388e3c", edgecolor="white")
ax3.set_title("Order Value Distribution")
ax3.set_xlabel("Order Value (₹)")

# Bottom-right: Pie chart
ax4.pie(shares, labels=categories,
        autopct="%1.1f%%", startangle=90)
ax4.set_title("Revenue by Category")

# Finalize
fig.suptitle("VisualEdge Weekly Dashboard",
             fontsize=18, fontweight="bold")
plt.tight_layout()
plt.show()
```

> **axes.flatten()** — converts the 2D `[[ax1, ax2], [ax3, ax4]]` array into a flat list `[ax1, ax2, ax3, ax4]`. Unpacking a flat list is cleaner than indexing `axes[0,0]` repeatedly.  
**fig.suptitle()** — adds a super-title spanning the entire Figure (above all subplots). Different from `ax.set_title()` which titles only one subplot.  
**plt.tight_layout()** — auto-adjusts spacing between subplots so titles and labels don't overlap. *Always* call it in multi-chart figures.

#### 4.3 Plotting Directly from a Pandas DataFrame

```python
import pandas as pd

df = pd.read_csv("sales_report.csv")

# Method 1: Pass df columns to Matplotlib
fig, ax = plt.subplots()
ax.bar(df["region"], df["revenue"])

# Method 2: df.plot() with ax= argument
df.plot(x="region", y="revenue",
        kind="bar", ax=ax)
```

**📖 Pandas + Matplotlib integration**

In real projects, your data comes from a Pandas DataFrame — not a hand-typed list.  
  
**Method 1** — pass DataFrame columns as X and Y to `ax.plot()` / `ax.bar()`. This is pure Matplotlib and gives you full control.  
  
**Method 2** — use Pandas' built-in `df.plot()`. Internally, it calls Matplotlib. Pass `ax=ax` to draw on a specific Axes (essential for subplots).  
  
Both methods produce identical charts. Method 1 is more explicit — use it when styling matters.

#### 4.4 Save Figures to File — PNG, PDF, SVG

```
# Save as high-res PNG (for reports)
fig.savefig("dashboard.png",
            dpi=300,
            bbox_inches="tight")

# Save as PDF (for print-ready reports)
fig.savefig("dashboard.pdf",
            bbox_inches="tight")

# Save as SVG (for web / editable vector)
fig.savefig("dashboard.svg")
```

**📖 fig.savefig() — the export step**

`dpi=300` — dots per inch. 72 dpi is screen resolution (blurry when printed). 150 dpi is acceptable. **300 dpi is print-ready** — always use this for client reports.  
  
`bbox_inches="tight"` — trims whitespace around the figure and ensures labels/titles aren't cut off at the edges. Almost always required.  
  
Save as **PNG** for email/Slack attachments. **PDF** for print and board reports. **SVG** if the client wants to edit the chart in Illustrator or embed on a website.

> **✅ Phase 4 Key Takeaways**

> - `plt.subplots(nrows, ncols)` creates a grid of Axes — index with `axes[row, col]` or flatten with `axes.flatten()`.
> - `fig.suptitle()` adds a title for the entire dashboard — different from `ax.set_title()` per subplot.
> - Always call `plt.tight_layout()` before `savefig()` — prevents label clipping.
> - `fig.savefig("name.png", dpi=300, bbox_inches="tight")` is the production export command.
> - Pass `ax=ax` when using `df.plot()` inside a subplot grid — otherwise Pandas creates a new Figure.

### 5. Phase 5 — Real-World Patterns & Common Mistakes

**Business Problem:** VisualEdge's new engineer Riya writes code that produces charts without errors — but the charts have invisible labels, wrong colours, or missing legends that only appear in some environments. These are the real bugs that production chart pipelines hit.

#### 5.1 Adding Value Labels on Bars

```
bars = ax.bar(regions, revenue)

# Add value label above each bar
for bar in bars:
    height = bar.get_height()
    ax.text(
        bar.get_x() + bar.get_width() / 2,
        height + 5,
        f"₹{height}",
        ha="center",
        fontsize=10
    )
```

**📖 Adding data labels to bars**

Bar charts are clearer when the exact value appears above each bar — the reader doesn't need to trace to the Y-axis.  
  
`bar.get_height()` — the value of the bar (its top Y coordinate).  
`bar.get_x() + bar.get_width()/2` — the horizontal centre of the bar.  
`height + 5` — place the text 5 units above the bar top.  
`ha="center"` — horizontally centre the text above the bar.  
  
This loop pattern — iterate over returned bar objects — is the standard Matplotlib idiom for adding labels.

#### 5.2 Dual-Axis Chart — Two Different Scales

```
fig, ax1 = plt.subplots()

# Left axis: Revenue (₹ Lakhs)
ax1.bar(months, sales, color="#1976d2",
        alpha=0.6, label="Revenue")
ax1.set_ylabel("Revenue (₹L)", color="#1976d2")

# Right axis: Growth Rate (%)
ax2 = ax1.twinx()
growth = [0, 21, -9, 35, -10, 22]
ax2.plot(months, growth, color="#e84d0e",
         marker="o", label="Growth %")
ax2.set_ylabel("Growth (%)", color="#e84d0e")
```

**📖 ax.twinx() — two Y-axes on one chart**

When you want to overlay two datasets that have very different scales (revenue in ₹ lakhs vs growth rate in %), a dual-axis chart lets you show both on the same X-axis without one dataset being invisible.  
  
`ax1.twinx()` creates a new Axes (`ax2`) that shares the same X-axis but has an independent right Y-axis.  
  
Always colour the axis labels to match the corresponding data series — it's the only way the reader knows which Y-axis belongs to which line.

#### 5.3 plt.close() — Prevent Memory Leaks in Scripts

```
for region in ["North", "South", "East"]:
    fig, ax = plt.subplots()
    ax.plot(months, region_data[region])
    ax.set_title(f"{region} Sales")
    fig.savefig(f"{region}_chart.png",
               dpi=150,
               bbox_inches="tight")
    plt.close(fig)  # ← CRITICAL
```

**📖 plt.close(fig) — always close in loops**

When your script generates many charts in a loop (one per region, one per client, one per week), each `plt.subplots()` call creates a new Figure object in memory. Without `plt.close(fig)`, they all stay open.  
  
With 50+ charts, this triggers a Matplotlib warning: *"More than 20 figures have been opened."* In large scripts, it can consume gigabytes of RAM.  
  
Rule: always call `plt.close(fig)` immediately after `savefig()` in any loop that generates multiple charts.

##### ⚠️ The 3 Most Common Matplotlib Bugs

**Bug 1 — Calling plt.show() inside a loop:** Call `plt.show()` only once at the end of a script, or not at all when saving to file. Calling it inside a loop creates new figures mid-loop.

**Bug 2 — Mixing plt. and ax. calls:** If you create `fig, ax = plt.subplots()`, use `ax.set_title()` — not `plt.title()`. Mixing the two works sometimes, fails silently in subplots.

**Bug 3 — Forgetting bbox_inches="tight" in savefig:** Axis labels and titles are cut off in the saved image. Always include `bbox_inches="tight"`.

### 6. Quick Reference — Full API Cheat Sheet

Function / Property

What It Does

Common Values

plt.subplots(r,c)

Create Figure + Axes grid

figsize=(10,6), sharex=True

ax.plot(x,y)

Line chart

marker="o", color="#hex", linewidth=2

ax.bar(x,y)

Vertical bar chart

color=[], width=0.6, edgecolor="white"

ax.barh(y,x)

Horizontal bar chart

color=, height=0.6

ax.scatter(x,y)

Scatter plot

s=50 (size), alpha=0.7, c=colour

ax.hist(data)

Histogram

bins=30, edgecolor="white"

ax.pie(values)

Pie chart

labels=, autopct="%1.1f%%", startangle=90

ax.set_title()

Chart title

fontsize=14, fontweight="bold"

ax.set_xlabel/ylabel()

Axis labels

fontsize=12

ax.legend()

Auto legend from labels

loc="upper right", fontsize=10

ax.grid()

Gridlines

linestyle="--", alpha=0.5

ax.annotate()

Arrow + text on chart

xy=, xytext=, arrowprops={}

ax.text(x,y,s)

Plain text label

ha="center", fontsize=10

ax.twinx()

Second Y-axis (right side)

—

ax.spines[x].set_visible(False)

Hide chart border

"top","right","left","bottom"

fig.suptitle()

Title for entire figure

fontsize=18, y=1.02

fig.savefig()

Export to file

dpi=300, bbox_inches="tight"

plt.tight_layout()

Fix overlapping subplots

pad=1.5

plt.style.use()

Apply a global theme

"seaborn-v0_8-whitegrid", "ggplot"

plt.close(fig)

Free figure memory

Use in loops

### 7. Quizzes & Q&A

**Quiz: ❓ Quiz 1 — You want to create a 3-column layout of 3 charts in one row. What is the correct plt.subplots() call?**

- A) plt.subplots(3)
- B) plt.subplots(1, 3)
- C) plt.subplots(3, 1)
- D) plt.subplots(cols=3)

> **Answer/explanation:** ✅ **Answer: B — plt.subplots(1, 3).** The signature is `plt.subplots(nrows, ncols)`. One row, three columns = `(1, 3)`. Option A creates 3 rows (1 column each). Option C creates 3 rows in 1 column. Option D is invalid — there is no `cols=` keyword argument.

**Quiz: ❓ Quiz 2 — Your chart's title is cut off at the top when you open the saved PNG. What do you add to fix it?**

- A) ax.set_title(pad=20)
- B) bbox_inches="tight" in savefig()
- C) fig.set_size_inches(20, 20)
- D) plt.margins(0.5)

> **Answer/explanation:** ✅ **Answer: B — bbox_inches="tight".** This tells Matplotlib to calculate the bounding box of all elements (including titles and labels) and include them in the saved image. Without it, labels outside the default figure boundaries get clipped. Option A adjusts the pad between the title and the plot area, but won't fix clipping in the saved file.

**Quiz: ❓ Quiz 3 — You use a for loop to generate 50 charts and save each as a PNG. After chart 21, Matplotlib prints a warning about too many open figures. What should you add?**

- A) plt.show() after each savefig()
- B) plt.close(fig) after each savefig()
- C) fig.clear() before each loop
- D) plt.clf() at the start of the script

> **Answer/explanation:** ✅ **Answer: B — plt.close(fig).** Matplotlib keeps all figures in memory until explicitly closed. In a loop generating many charts, you must call `plt.close(fig)` right after saving to release memory. `plt.show()` renders a figure but doesn't close it. `plt.clf()` clears the current figure's contents but doesn't close the Figure object.

##### 🙋 Fresher Q&A — Questions You Will Definitely Have

**Q: Q: What's the difference between plt.plot() and ax.plot()? They both seem to draw a line chart.**

A: They produce the same chart for a simple script. `plt.plot()` is the shortcut — it implicitly draws on the "current active Axes." `ax.plot()` is explicit — you're calling it on a specific Axes object you created. For a single chart, either works. For subplots, you *must* use `ax.plot()` — otherwise Matplotlib doesn't know which subplot to draw on. Professional Matplotlib code always uses the `ax.` style.

**Q: Q: When I run plt.show() in a script, the window pops up and everything freezes until I close it. Is that normal?**

A: Yes — `plt.show()` is a blocking call. It opens an interactive window and pauses the script until you close the window. This is expected when running `.py` files. In Jupyter notebooks, `plt.show()` is not needed — charts render inline automatically. For production scripts that only need to save charts (no interactive window), just call `fig.savefig()` without `plt.show()`.

**Q: Q: I added a label= parameter to ax.plot() but ax.legend() shows nothing. Why?**

A: You must call `ax.legend()` *after* all your `ax.plot()` calls — not before. The legend collects all label= values from all series drawn so far. Also double-check spelling: the parameter must be `label=` (singular), not `labels=`. One character difference, no error message, but a blank legend.

**Q: Q: What does tight_layout() actually do? Can I always add it?**

A: `plt.tight_layout()` automatically adjusts the padding around and between subplots so that axis labels, tick labels, and titles don't overlap. Without it, in multi-subplot figures, titles commonly overlap the axis labels of the row above. Yes — calling it is harmless for single-panel charts too. Make it a habit: always call `plt.tight_layout()` immediately before `plt.show()` or `fig.savefig()`.

##### ✅ Matplotlib Best Practices for Production Charts

- Always use `fig, ax = plt.subplots()` — never rely on the implicit current-figure state in scripts.
- Set `plt.style.use()` once at the top of every script — never configure fonts/colours per chart.
- Always pass `dpi=300, bbox_inches="tight"` in `savefig()` — no exceptions for client-facing charts.
- Call `plt.close(fig)` immediately after `savefig()` in loops — prevents memory leaks in automated scripts.
- Use `ax.annotate()` on the one data point the chart is about — a chart without a point of focus tells no story.
- Always label both axes — a chart with an unlabelled Y-axis is meaningless to anyone who didn't write it.
- Prefer `ax.barh()` over `ax.bar()` when category names are longer than 6 characters — prevents label overlap.
- Use `alpha=` in scatter plots when dots overlap — solid dots at full opacity hide the data density pattern.

##### 🧪 Practice Exercise — Build the VisualEdge Dashboard Yourself

1. Create a `sales.csv` with columns: month, region, product, units_sold, revenue, ad_spend. Add 6 months × 4 regions of data with realistic FMCG numbers (₹50K–₹600K revenue range).
2. Load with `pd.read_csv()`. GroupBy month and sum revenue. Plot as a line chart with markers. Annotate the peak month with `ax.annotate()`.
3. GroupBy region and sum revenue. Plot as a horizontal bar chart (`ax.barh()`). Add value labels at the end of each bar using `ax.text()`.
4. Plot the distribution of units_sold as a histogram with 20 bins. Add a vertical line at the mean using `ax.axvline(df["units_sold"].mean(), color="red", linestyle="--")`.
5. Combine all 3 charts into a `1×3` subplot grid using `plt.subplots(1, 3, figsize=(18, 5))`. Add a suptitle "VisualEdge Monthly Report".
6. Apply the `"seaborn-v0_8-whitegrid"` style. Remove top and right spines from all 3 Axes. Save as `dashboard.png` at 300 dpi.

### Matplotlib Project Complete 🎉

You have built VisualEdge's complete Matplotlib visualisation pipeline — from your first `plt.plot()` to a branded 4-chart dashboard exported as a print-ready PNG. You now know Figure vs Axes, all 6 chart types, professional styling, annotations, subplots, Pandas integration, and the production export workflow.

> **Arjun**
> 
> "Three weeks ago, Priya spent every Sunday night manually building 8 Excel charts and emailing them one by one. Last Monday, she ran one Python script. 10 seconds later, all 8 charts were saved at 300 dpi, consistently styled, ready to paste into the PDF report. That's what Matplotlib does — it takes a manual, error-prone visual workflow and turns it into a reproducible, version-controlled pipeline."

> **Priya**
> 
> "The moment everything clicked for me: understanding that Figure is the canvas and Axes is the chart. Once I knew that, I stopped calling plt.title() on subplots and wondering why the wrong chart got the title. Everything in Matplotlib traces back to: which Axes object am I drawing on right now?"

> **Riya**
> 
> "And the chart the client called 'student homework'? That was the best lesson. Correct data, wrong chart type, default style, missing labels — no one can read it. The chart is the communication. Technical accuracy is only half the job. The other half is making it readable in 3 seconds by someone who didn't write the code."
