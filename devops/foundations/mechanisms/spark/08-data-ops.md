# `window` Functions

Yes — Spark absolutely has an equivalent, but it looks different and this difference is very important.
> In pandas: `groupby(...).transform(...)`  
> In Spark: window functions (`Window`)

## 🔹 The Spark equivalent

Your pandas code:
```python
df["inventory_total"] = (
    df.groupby(["master_location", "year_month_key"])["inventory_count"]
      .transform("sum")
)
```
**Spark version**:
```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

window_spec = Window.partitionBy("master_location", "year_month_key")

df = df.withColumn(
    "inventory_total",
    F.sum("inventory_count").over(window_spec)
)

df = df.withColumn(
    "perc",
    F.col("inventory_count") / F.col("inventory_total")
)
```

## 🔹 Mental mapping (VERY important)

| Concept              | Pandas              | Spark                     |
| -------------------- | ------------------- | ------------------------- |
| Group rows           | `groupby(...)`      | `Window.partitionBy(...)` |
| Compute group metric | `.transform("sum")` | `F.sum(...).over(window)` |
| Broadcast to rows    | automatic           | automatic via window      |
| Keep row count       | yes                 | yes                       |


👉 Window = Spark’s version of “**group + broadcast back**”

## 🔹 Why Spark uses “window” instead of “transform”

This is where things become **non-toy**.

Pandas world:
- all data is in memory
- there is a global index
- easy to “map back” results

Spark world:
- data is distributed across machines
- no global index
- rows are partitioned

So Spark needs a different abstraction:
> “For each row, look at all rows in its partition and compute something”

That’s exactly what a **window** is.

## 🔹 What `Window.partitionBy(...)` actually means
```python
window_spec = Window.partitionBy("master_location", "year_month_key")
```
This tells Spark:
> “Group rows by these keys, but DO NOT collapse them — keep them as rows”

Then:
```python
F.sum("inventory_count").over(window_spec)
```
means:

“For each row, compute the sum of inventory_count within its group”

## 🔹 Same example

Input:
```
Airdrie (CC), 202602, 120
Airdrie (F),  202602, 80
```
After window:
```
inventory_total = 200 for both rows
```
Exactly like pandas transform.

## 🔹 Why this matters for you

You mentioned earlier:
> “I don’t know when a system stops being a toy”

This is one of those boundaries.

Pandas solution
```python
groupby(...).transform(...)
```
Spark solution
```python
Window.partitionBy(...).over(...)
```
👉 Same idea, different execution model

🔹 Important difference (this is critical)

In pandas:
- grouping is cheap
- everything is local

In Spark:
- grouping = **shuffle (expensive)**
- data moves across machines

So:
> Window operations are powerful, but not “free”

## 🔹 Alternative in Spark (agg + join)

You can also do your earlier approach:
```python
total_df = (
    df.groupBy("master_location", "year_month_key")
      .agg(F.sum("inventory_count").alias("inventory_total"))
)

df = df.join(
    total_df,
    on=["master_location", "year_month_key"],
    how="left"
)
```
This is equivalent to your pandas `agg + merge`.

## 🔹 When to use which in Spark

**Use window when**:
- you need row-level output (like percentages)
- you want cleaner logic
- you avoid extra joins

**Use groupBy + join when**:
- you reuse the aggregated table elsewhere
- you want to cache intermediate results
- performance tuning requires it

## 🔹 Deep system insight (this is the real lesson)

In pandas:
> grouping = logical operation

In Spark:
> grouping = **data movement problem**

That’s why:
- pandas has `transform`
- Spark has `window`

Same concept, different constraints.
