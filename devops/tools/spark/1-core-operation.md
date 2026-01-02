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
