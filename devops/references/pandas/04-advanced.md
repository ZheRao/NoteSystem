# Create Mapping based on two columns

```python
mapping = (
    df[["column1", "column2"]]
    .dropna()
    .astype(str)
    .apply(lambda s: s.str.strip())
    .set_index("column1")["column2"]
    .to_dict()
)
```

# JSON Handling

**DataFrame → JSON**

```python
df.to_dict(orient="records")
```

Supported `orient`:
- `records`
- `index`
- `split`
- `dict`
- `list`
- `series`

**Flatten Nested JSON**

```python
pd.json_normalize(data, sep=".", max_level=2)

pd.json_normalize(
    data,
    record_path=["lines"],
    meta=["header_field"],
    record_prefix="line."
)
```


# Interval Matching

```python
idx = pd.IntervalIndex.from_arrays(
    left=intervals["left"],
    right=intervals["right"],
    closed="both"
)

idx.get_indexer(values)
```





# Reshaping

**Melt (Unpivot)**

```python
df.melt(
    id_vars=["location", "currency"],
    var_name="Commodity",
    value_name="Amount"
)
```

**Explode Composite Values**

```python
df["Combined"] = df["Combined"].str.split("+")
df.explode("Combined").reset_index(drop=True)
```


# Advanced Patterns

**Shift**

```python
df["prev"] = df["revenue"].shift(1)
```

**Rank**

```python
df["rank"] = df["score"].rank(ascending=False)
```

**Cumulative**

```python
df["cumsum"] = df["revenue"].cumsum()
```

