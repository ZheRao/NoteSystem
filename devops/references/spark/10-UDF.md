# User-Define Functions (use only when must)

## 1. Row-wise Python UDF

- `F.udf(py_func, returnType)`
     - Slow, blocks optimizations, serialization overhead

## 2. Pandas UDF
- `pandas_udf(...)`
    - Faster but still has overhead; multiple types: scalar, grouped_agg, map, etc.

## 3. SQL Registration
- `spark.udf.register("name", func, returnType)`
