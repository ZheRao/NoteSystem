# Pandas $\to$ Spark Transformation Cheat Sheet

## Core mindset

**Pandas**

You manipulate data **directly**.
```python
df["x"] = ...
```

**Spark**

You build **column expressions** and return a new DataFrame.
```python
df = df.withColumn("x", ...)
```
That one shift explains most of the syntax difference.

## 1. Reference a column

**Pandas**
```python
df["date"]
```
**Spark**
```python
F.col("date")
```
or sometimes:
```python
df["date"]
```
But for reusable functions, prefer:
```python
F.col(col_name)
```
because it works cleanly with variable column names.

## 2. Create or replace a column

**Pandas**
```python
df["year"] = pd.to_datetime(df["date"]).dt.year
```
**Spark**
```python
df = df.withColumn("year", F.year(F.col("date")))
```

## 3. Convert to date

**Pandas**
```python
df["date"] = pd.to_datetime(df["date"])
```
**Spark**
```python
df = df.withColumn("date", F.to_date(F.col("date")))
```
If timestamp:
```python
df = df.withColumn("date", F.to_timestamp(F.col("date")))
```

## 4. Extract year / month / day

**Pandas**
```python
df["year"] = df["date"].dt.year
df["month"] = df["date"].dt.month
df["day"] = df["date"].dt.day
```
**Spark**
```python
df = df.withColumn("year", F.year(F.col("date")))
df = df.withColumn("month", F.month(F.col("date")))
df = df.withColumn("day", F.dayofmonth(F.col("date")))
```

## 5. Conditional logic

Pandas
```python
df["flag"] = 0
df.loc[df["amount"] > 100, "flag"] = 1
```
Spark
```python
df = df.withColumn(
    "flag",
    F.when(F.col("amount") > 100, 1).otherwise(0)
)
```
For multiple branches:
```python
df = df.withColumn(
    "bucket",
    F.when(F.col("amount") < 0, "negative")
     .when(F.col("amount") == 0, "zero")
     .otherwise("positive")
)
```

## 6. Null handling

**Pandas**
```python
df["name"] = df["name"].fillna("unknown")
```
**Spark**
```python
df = df.withColumn("name", F.coalesce(F.col("name"), F.lit("unknown")))
```
Or for whole DataFrame:
```python
df = df.fillna({"name": "unknown"})
```

## 7. String cleanup

**Pandas**
```python
df["name"] = df["name"].str.strip().str.upper()
```
**Spark**
```python
df = df.withColumn("name", F.upper(F.trim(F.col("name"))))
```
Lowercase:
```python
df = df.withColumn("name", F.lower(F.col("name")))
```

## 8. Concatenate columns

**Pandas**
```python
df["full_name"] = df["first"] + " " + df["last"]
```
**Spark**
```python
df = df.withColumn(
    "full_name",
    F.concat_ws(" ", F.col("first"), F.col("last"))
)
```
`concat_ws` is usually nicer than raw `concat`.



## 9. Rename columns

**Pandas**
```python
df = df.rename(columns={"old": "new"})
```
**Spark**
```python
df = df.withColumnRenamed("old", "new")
```
For many columns:
```python
for old, new in mapping.items():
    df = df.withColumnRenamed(old, new)
```

## 10. Select columns

**Pandas**
```python
df = df[["a", "b", "c"]]
```
**Spark**
```python
df = df.select("a", "b", "c")
```
With dynamic list:
```python
df = df.select(*cols)
```

## 11. Filter rows

**Pandas**
```python
df = df[df["amount"] > 0]
```
**Spark**
```python
df = df.filter(F.col("amount") > 0)
```
or
```python
df = df.where(F.col("amount") > 0)
```

## 12. Drop columns

**Pandas**
```python
df = df.drop(columns=["temp"])
```
**Spark**
```python
df = df.drop("temp")
```
Multiple:
```python
df = df.drop(*cols)
```

## 13. Sort rows

**Pandas**
```python
df = df.sort_values(["date", "amount"], ascending=[True, False])
```
**Spark**
```python
df = df.orderBy(F.col("date").asc(), F.col("amount").desc())
```

## 14. Remove duplicates

**Pandas**
```python
df = df.drop_duplicates()
df = df.drop_duplicates(subset=["id"])
```
**Spark**
```python
df = df.dropDuplicates()
df = df.dropDuplicates(["id"])
```

## 15. Add literal values

**Pandas**
```python
df["source"] = "qbo"
```
**Spark**
```python
df = df.withColumn("source", F.lit("qbo"))
```
Current timestamp:
```python
df = df.withColumn("load_ts", F.current_timestamp())
```
Current date:
```python
df = df.withColumn("load_date", F.current_date())
```

## 16. Group and aggregate

**Pandas**
```python
out = (
    df.groupby("customer", as_index=False)
      .agg({"amount": "sum"})
)
```
**Spark**
```python
out = (
    df.groupBy("customer")
      .agg(F.sum("amount").alias("amount_sum"))
)
```
Multiple aggregations:
```python
out = (
    df.groupBy("customer")
      .agg(
          F.sum("amount").alias("amount_sum"),
          F.avg("amount").alias("amount_avg"),
          F.count("*").alias("row_count")
      )
)
```

## 17. Joins

**Pandas**
```python
out = left.merge(right, on="id", how="left")
```
**Spark**
```python
out = left.join(right, on="id", how="left")
```
With explicit condition:
```python
out = left.join(right, F.col("left.id") == F.col("right.id"), how="left")
```
Though if same column name exists on both sides, plain `on="id"` is cleaner.



## 18. Fiscal year example

**Pandas**
```python
df["date"] = pd.to_datetime(df["date"])
df["fiscal_year"] = df["date"].dt.year
df.loc[df["date"].dt.month >= 11, "fiscal_year"] += 1
```
**Spark**
```python
df = (
    df.withColumn("date", F.to_date(F.col("date")))
      .withColumn(
          "fiscal_year",
          F.year(F.col("date")) +
          F.when(F.month(F.col("date")) >= 11, 1).otherwise(0)
      )
)
```

## 19. Reusable transform template in Spark

This is the pattern you should memorize.
```python
from pyspark.sql import DataFrame as SparkDF
from pyspark.sql import functions as F

def some_transform_spark(df: SparkDF, col_name: str) -> SparkDF:
    if col_name not in df.columns:
        raise KeyError(f"{col_name} is not a column in df")

    return (
        df.withColumn(col_name, F.trim(F.col(col_name)))
          .withColumn("new_flag", F.when(F.col(col_name).isNull(), 0).otherwise(1))
    )
```
Structure:
1. validate columns
2. normalize types if needed
3. build expression with `F.*`
4. attach with `withColumn`
5. return new DataFrame


## 21. Tiny starter toolkit to memorize

For your current phase, master these first:
```python
from pyspark.sql import functions as F
```
and:
```python
F.col
F.lit
F.when
F.coalesce
F.to_date
F.to_timestamp
F.year
F.month
F.dayofmonth
F.trim
F.upper
F.lower
F.concat_ws
F.sum
F.avg
F.count
F.current_timestamp
```
And these DataFrame methods:
```python
withColumn
select
drop
filter
groupBy
agg
join
orderBy
dropDuplicates
withColumnRenamed
```
That gets you very far.





