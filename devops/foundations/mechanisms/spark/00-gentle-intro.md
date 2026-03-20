# How to build Spark functions systematically

When writing a Spark transform, use this checklist:

### Step 1: validate input columns
```python
if date_col not in df.columns:
    raise KeyError(...)
```
### Step 2: normalize types if needed
```python
df = df.withColumn(date_col, F.to_date(F.col(date_col)))
```
### Step 3: build column expressions
```python
expr = F.year(F.col(date_col)) + F.when(...).otherwise(...)
```
### Step 4: attach result with withColumn
```python
df = df.withColumn("fiscal_year", expr)
```
### Step 5: return the new DataFrame
```python
return df
```
That is the clean Spark pattern.

# Pandas → Spark Scenario

This **is** a bit more abstraction, but it is the **right kind** of abstraction:  
not “clever framework abstraction,” just **separating engine-specific mechanics from stable transform intent**.

And yes — let’s do a quick Spark tutorial focused exactly on what you need.

## Core Invariant

***Use Pandas to discover invariants. Use Spark to enforce them at scale.***

## The core mental shift for Spark

If you already know Pandas, the biggest change is:

**Pandas**

You usually think:
```python
df["new_col"] = ...
```

**Spark**

You should think:
```python
df = df.withColumn("new_col", ...)
```
Spark DataFrames are effectively **immutable**.  
You do not modify them in place.  
Each transformation returns a **new DataFrame**.

#### That is the first invariant.

## 1. The three core things you need for Spark transforms

Most simple Spark transformation functions are built from these:

### A. `F.col(...)`

Reference a column by name.
```python
from pyspark.sql import functions as F

F.col("order_date")
```
This means: “the Spark column named order_date”.

### B. `df.withColumn(...)`

Create or replace a column.
```python
df = df.withColumn("year", F.year(F.col("order_date")))
```
This either:
- creates `year` if it does not exist
- or replaces it if it already exists

### C. Spark SQL functions from `pyspark.sql.functions`

Examples:
```python
F.year(...)
F.month(...)
F.to_date(...)
F.when(...).otherwise(...)
F.lit(...)
F.upper(...)
F.trim(...)
```
These are the building blocks.

## 2. Your exact fiscal year example in Spark

Let’s say you have:
```python
df.columns
# ["transaction_date", ...]
```
and you want to create `fiscal_year` where months >= 11 belong to next year.

### Step 1: import functions
```python
from pyspark.sql import functions as F
```

### Step 2: make sure the date column is a real date
```python
df = df.withColumn("transaction_date", F.to_date(F.col("transaction_date")))
```
Why:
- if the column is string, `year()` and `month()` may behave inconsistently
- converting once makes downstream logic cleaner

### Step 3: create fiscal year
```python
df = df.withColumn("fiscal_year", F.year(F.col("transaction_date")))
```
Now `fiscal_year` starts as normal calendar year.

### Step 4: adjust based on cutoff month
```python
df = df.withColumn(
    "fiscal_year",
    F.when(
        F.month(F.col("transaction_date")) >= 11,
        F.col("fiscal_year") + 1
    ).otherwise(F.col("fiscal_year"))
)
```
That means:
- if month >= 11, add 1
- else leave it alone

## 4. Even cleaner version

You do not actually need to create `fiscal_year` twice.

You can compute it directly:
```python
from pyspark.sql import DataFrame as SparkDF
from pyspark.sql import functions as F

def _create_fiscal_year_spark(
    df: SparkDF,
    date_col: str,
    cut_off: int = 11
) -> SparkDF:
    if date_col not in df.columns:
        raise KeyError(f"{date_col} is not a column in df")

    return (
        df.withColumn(date_col, F.to_date(F.col(date_col)))
          .withColumn(
              "fiscal_year",
              F.year(F.col(date_col)) +
              F.when(F.month(F.col(date_col)) >= cut_off, 1).otherwise(0)
          )
    )
```
This is usually how I would write it.

## 5. How to think in Spark expressions

This is the real tutorial piece.

In Spark, you are not writing row-by-row Python logic.

You are building a **column expression graph**.

For example:
```python
F.year(F.col(date_col))
```
means:
- take column `date_col`
- extract its year

And:
```python
F.when(F.month(F.col(date_col)) >= cut_off, 1).otherwise(0)
```
means:
- compute month from `date_col`
- compare it to `cut_off`
- return 1 if true, 0 otherwise

Then:
```python
F.year(F.col(date_col)) + ...
```
means:
- add those two expressions together

So Spark code is often:

**expression-building**, not imperative mutation.

#### That is the second invariant.

## 6. Equivalent mapping: Pandas vs Spark

Here is a useful translation table.

### Create a new column

**Pandas**
```python
df["year"] = pd.to_datetime(df["date"]).dt.year
```

**Spark**
```python
df = df.withColumn("year", F.year(F.col("date")))
```

### Conditional logic

**Pandas**
```python
df.loc[df["month"] >= 11, "fiscal_year"] += 1
```
**Spark**
```python
df = df.withColumn(
    "fiscal_year",
    F.when(F.col("month") >= 11, F.col("fiscal_year") + 1)
     .otherwise(F.col("fiscal_year"))
)
```
### String cleanup

**Pandas**
```python
df["name"] = df["name"].str.strip().str.upper()
```
**Spark**
```python
df = df.withColumn("name", F.upper(F.trim(F.col("name"))))
```
## 7. The 80/20 Spark transformation toolkit

For your project, learn these first:
```python
from pyspark.sql import functions as F
```
and become comfortable with:
```python
F.col()
F.lit()
F.when().otherwise()
F.to_date()
F.to_timestamp()
F.year()
F.month()
F.dayofmonth()
F.trim()
F.upper()
F.lower()
F.concat_ws()
F.coalesce()
F.isnull()
F.count()
F.sum()
F.avg()
```
And these DataFrame methods:
```python
df.select(...)
df.withColumn(...)
df.drop(...)
df.filter(...)
df.groupBy(...).agg(...)
df.join(...)
df.orderBy(...)
```
If you master those, you can do a lot.

Lean in — core growth.

## 8. A tiny tutorial example from scratch

Suppose:
```python
data = [
    ("2024-10-15",),
    ("2024-11-01",),
    ("2025-01-20",),
]
```
Create DataFrame:
```python
df = spark.createDataFrame(data, ["transaction_date"])
```
Transform:
```python
from pyspark.sql import functions as F

out = (
    df.withColumn("transaction_date", F.to_date(F.col("transaction_date")))
      .withColumn(
          "fiscal_year",
          F.year(F.col("transaction_date")) +
          F.when(F.month(F.col("transaction_date")) >= 11, 1).otherwise(0)
      )
)
```
Show result:
```python
out.show()
```
Expected idea:
```
+----------------+-----------+
|transaction_date|fiscal_year|
+----------------+-----------+
|      2024-10-15|       2024|
|      2024-11-01|       2025|
|      2025-01-20|       2025|
+----------------+-----------+
```

## 10. Common beginner mistakes in Spark

These are worth knowing early.

### Mistake 1: using `df.column_name` with dynamic names

Wrong:
```python
df.date_col
```
Correct:
```python
F.col(date_col)
```
or
```python
df[date_col]
```

### Mistake 2: forgetting reassignment

Wrong:
```python
df.withColumn("x", ...)
return df
```

Correct:
```python
df = df.withColumn("x", ...)
return df
```
Unless you chain directly.

### Mistake 3: writing Python `if` instead of Spark `when`

Wrong:
```python
if F.month(F.col(date_col)) >= 11:
    ...
```
That does not work because that is a Spark column expression, not a Python boolean.

Correct:
```python
F.when(F.month(F.col(date_col)) >= 11, ...)
```

### Mistake 4: assuming operations happen immediately

Spark is lazy.

This:
```python
df = df.withColumn(...)
```
does not execute right away.

Execution usually happens at actions like:
```python
df.show()
df.count()
df.collect()
df.write...
```
#### That is the third invariant.

## 12. My blunt recommendation

For your repo, do not build a grand abstraction system yet.

Use this rule:
- one public transform name
- one Pandas implementation
- one Spark implementation
- dispatch by `isinstance`
- write tiny tests for both

That is enough structure to scale without becoming architecture theater.

## 13. Tiny practice exercise for you

A good next step would be to implement these both in Pandas and Spark:
- `create_fiscal_month`
- `normalize_string_column`
- `add_load_timestamp`
- `select_and_rename_columns`

Those four will force you to internalize the Spark style quickly.

Lean in — core growth