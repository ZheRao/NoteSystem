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

- `.partitionBy("col1", ...)`
- `.bucketBy(n, "col")` + `.sortBy("col")` *(usually with `.saveAsTable(...)`)*

### `df.write.format(...).mode(...).option(...).save(path)`

This means:
1. start from a DataFrame `df`
2. access its writer via `.write`
3. tell Spark **what file format** to use with `.format(...)`
4. tell Spark **what to do if data already exists** with `.mode(...)`
5. pass extra write settings with `.option(...)`
6. finally write it to storage with `.save(path)`

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
