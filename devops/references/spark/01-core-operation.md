# DataFrame API Cheet Sheet

## 1. Create / Read / Inspect

**Create**
- `spark.read` (csv/json/parquet/jdbc/...)
- `spark.createDataFrame(...)`
- `rdd.toDF(...)`

**Inspect**
- `df.printSchema()`, `df.schema`, `df.dtypes`, `df.columns`
- `df.head(n)`, `df.show(n, truncate=False)`, `df.limit(n)`
- `df.describe(...)`, `df.summary(...)`

## 2. Select / Project / Rename / Drop

**Select**
- `df.select(...)`
- `df.selectExpr("...", "...")` (SQL expressions)

**Add/transform columns**
- `df.withColumn("new", expr)`
- `df.withColumns({"a": exprA, "b": exprB})` *(Spark 3.4+)*
- `df.withColumnRenamed("old", "new")`
- `df.alias("t")` (table alias for joins)

**Drop**
- `df.drop("col1", "col2")`
- `df.distinct()`
- `df.dropDuplicates([...])`


## 3. Filter rows

- `df.filter(condition)`
- `df.where(condition)` - *(alias of filter)*
- Condition can be Column expressions (`F.col("x") > 0`) or SQL strings (`x > 0`)


## 4. Grouping & Aggregation

**Grouping**

- `df.groupBy(...)`
- `df.rollup(...)`
- `df.cube(...)`

**Aggregate**

- `grouped.agg(...)`
- Shorthands: `grouped.count()`, `grouped.sum("x")`, `grouped.avg("x")`, ...


## 5. Joins & Set Ops

**Joins**

- `df.join(other, on=..., how=...)`
- `how`: `inner`, `left`, `right`, `full/outer`, `cross`, `left_semi`, `left_anti`

**Set operations**

- `df.union(other)` (position-based)
- `df.unionByName(other, allowMissingColumns=...)`
- `df.intersect(other)`
- `df.exceptAll(other)` *(keeps duplicates; also `except` exists and drops duplicates)*

## 6. Order / Repartition / Sample

**Order**

- `df.orderBy(...)`, `df.sort(...)`

**Partitions**

- `df.repartition(n)` / `df.repartition(n, "col")`
- `df.repartitionByRange(n, "col1", ...)`
- `df.coalesce(n)` (reduce partitions without full shuffle)

**Sampling**

- `df.sample(withReplacement, fraction, seed=None)`
- `df.randomSplit([w1, w2, ...], seed=None)`

## 7. Missing Data

- `df.na.fill(...)`
- `df.na.drop(...)`
- `df.na.replace(...)`

## 8. Write Path (IO)

- `df.write.format(...).mode(...).option(...).save(path)`
- `.partitionBy("col1", ...)`
- `.bucketBy(n, "col")` + `.sortBy("col")` *(usually with `.saveAsTable(...)`)*

## 9. Data Integrity Check

### 1. Completeness
```python
df.groupBy("year", "month").count().orderBy("year", "month").show()
df2.select("fiscal_year").distinct().collect()
df2.select("fiscal_year").distinct().orderBy("fiscal_year").show()
```
👉 Are any months/year missing?

### 2. Distribution sanity
```python
df.groupBy("month").count().show()
```
👉 Does it look reasonable?

### 3. Range checks
```python
df.agg(F.min("date"), F.max("date")).show()
```
👉 Does time span match expectations?

### 4. Null checks
```python
df.filter(F.col("critical_col").isNull()).count()
```


# Detail #1: `.write()` 

## 1.1. `df.write.format(...).mode(...).option(...).save(path)`

This means:
1. start from a DataFrame `df`
2. access its writer via `.write`
3. tell Spark **what file format** to use with `.format(...)`
4. tell Spark **what to do if data already exists** with `.mode(...)`
5. pass extra write settings with `.option(...)`
6. finally write it to storage with `.save(path)`

## 1.2. partition/repartition

`.repartition(num, column)`
- not a `write` option, must occur before `.write()`
- repartition the df into `num` partitions by `column`
    - ensures records with the same `column` value stay in one partition
- will create `num` files in `path` without human-readable file names, no folder hierarchy
- example without controlling partition count
    ```python
    (
        df.repartition("year", "month")
        .write
        .format("parquet")
        .mode("overwrite")
        .partitionBy("year", "month")
        .save(path)
    )
    ```

### `.partitionBy(column)`
- is a `write` option, must occur after `.write()`
- will create separate human-readable folder hierarchy with names like `fiscal_year=2026`



### `repartition()` vs `coalesce()`
- `repartition(n)`
    - full shuffle
    - redistributes data across cluster
    - good when you want balanced partitions
    - can increase or decrease partition count

- Use when:
    - you need controlled number of output files
    - current partitioning is bad or unknown
    - you want repartitioning by column

- `coalesce(n)`
    - reduces number of partitions
    - avoids full shuffle in many cases
    - cheaper, but less balanced
- Use when:
    - only reducing partition count
    - final output is already reasonably distributed
    - you want fewer files without a big shuffle

- example case of `coalesce()` - Business-friendly CSV export
    ```python
    df.coalesce(1) \
    .write \
    .format("csv") \
    .mode("overwrite") \
    .option("header", "true") \
    .save(path)
    ```


## 1.3. Options

**format options**
```python
df.write.format("parquet")
df.write.format("csv")
df.write.format("json")
df.write.format("orc")
df.write.format("delta")   # if Delta Lake is installed
```

**mode options**
```python
.mode("error")                  # fail if path already exists
.mode("errorifexists")          # fail if path already exists
.mode("append")                 # adds new files into the existing dataset
.mode("overwrite")              # replace existing data at that path
.mode("ignore")                 # if target exists, do nothing
```

**option**

`.option()` configures format-specific behavior or writer behavior.

Pattern:
```python
.option("key", "value")
```
You can chain many:
```python
df.write.format("csv") \
    .option("header", "true") \
    .option("delimiter", ",") \
    .option("quote", '"') \
    .save(path)
```

useful options:
```python
.option("header", "true")       # csv
.option("delimiter", "|")       # csv
.option("quote", '"')           # important when text fields contain commas or quotes
.option("escape", '"')          # important when text fields contain commas or quotes
.option("nullValue", "")        # null handling
.option("emptyValue", "")       # null handling
.option("compression", "snappy")    # parquet optional compression
.option("compression", "gzip")      # common compression for CSV/JSON/Parquet
.option("compression", "snappy")    # common compression for CSV/JSON/Parquet
```

## 1.4. Important Remarks

### `AttributeError: 'PosixPath' object has no attribute '_get_object_id'`

This happens because:

👉 Spark’s `.save()` **does NOT understand Python `pathlib.Path` objects**  
👉 It expects a **plain string**, because it gets passed into the JVM

Under the hood:
- PySpark → converts arguments → sends to Java (Spark engine)
- Java side expects a `String` path
- You passed a Python `Path` object → Py4J tries to treat it like a Java object → fails → `_get_object_id` error


# Detail #2: `.read`

## read specific partitions

```python
from pyspark.sql import functions as F

df = (
    spark.read.parquet(str(pl_path / "spark"))
    .filter(F.col("fiscal_year").isin([2024, 2025, 2026]))
)
```

### Why this is better

Because Spark understands partitioned datasets.

If `fiscal_year` is a partition column, Spark can often do partition pruning, meaning:
- it does not read all partition folders fully
- it only touches the needed ones

So although the code reads from the root path, Spark may intelligently restrict the scan to only:
```
fiscal_year=2024/
fiscal_year=2025/
fiscal_year=2026/
```
That is exactly what partitioning is for.

### Alternative: directly provide multiple paths

You can also read multiple partition paths directly:
```python
df = spark.read.parquet(
    str(pl_path / "spark" / "fiscal_year=2024"),
    str(pl_path / "spark" / "fiscal_year=2025"),
    str(pl_path / "spark" / "fiscal_year=2026"),
)
```
This also works.

**When this is useful**
- when you already know exact partition folders
- when you want to bypass root scan
- when you are working with a hand-picked subset of paths
**Downside**
- less elegant
- more manual path-building
- can get messy if partition selection is dynamic

For most cases, root + filter is cleaner.