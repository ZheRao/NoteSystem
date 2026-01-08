# Schema handling

## Row-based

```python
yield Row(id=..., val=...)
```

- Column names inferred from field names
- Schema inference may trigger a small job


## Tuple-based (recommended)

Define schema

```python
schema = StructType([
    StructField("id",   StringType(),   False),
    StructField("val",  IntegerType(),  True),
    ...
])
```

Yield tuples (order matters)

```python
def f(it):
    for r in it:
        yield (_id, _val, ...)

df = spark.createDataFrame(rdd, schema=schema)
```

Or yield from dictionary

```python
rows["id"] = ...
rows["val"] = ...
yield row
```

- No schema inference
- Faster
- Predictable types
