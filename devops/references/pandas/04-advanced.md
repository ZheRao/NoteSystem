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

**use `map` to apply different fx_rates**

```python
rates = {"CAD": 1.0, "USD": fx_rate, "EUR": eur_rate}
df["AmountCAD"] = df["Amount"] * df["Currency"].map(rates).fillna(1.0)
```

**Applying a dictionary mapping onto a pandas column**
```python
df["new"] = df["old"].map(mapping_table).fillna("Unsuccessful")
```

## how Pandas `.map()` is different from Python `map()`

|Feature 	|Pandas series`.map()`	                    |Built-in Python `map()`                        |
|-----------|-------------------------------------------|-----------------------------------------------|
|Object	    |Called *on* a Pandas Series.	            |A standalone function.                         |
|Input Arg	|Function, **Dictionary**, or Series.	    |**Function** only.                             |
|Returns	|A new Pandas Series.	                    |A "map object" (an iterator).                  |
|Missing    |Values	Returns `NaN` if a key is missing.	|Throws an error (if using a dict in a function)|

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

# `itertuples()` — Row Iteration Pattern

`itertuples()` converts each row into a lightweight tuple-like object.

Example:

```python
for row in revisions.itertuples(index=False):
    print(row)
```

Output:

```python
Pandas(input_revision_id='2026_01', revision_num=1)
Pandas(input_revision_id='2026_02', revision_num=2)
```

Each row becomes an object with attributes:

```python
row.input_revision_id
row.revision_num
```

instead of repeated dataframe lookups.

### Important detail: `index=False`

Without:

```python
itertuples(index=False)
```

you get:

```python
Pandas(Index=0, input_revision_id='2026_01', revision_num=1)
```

Usually you don't want dataframe index mixed into business logic rows.

So:

```python
index=False
```

keeps iteration cleaner.

# Collect → Concat Once

### What actually happens internally

DataFrames are not dynamically growing arrays.

Each `concat` typically creates a NEW dataframe.

So this:

```python
A = concat(A, B)
```

roughly means:

1. allocate new memory
2. copy old dataframe
3. copy new dataframe
4. discard old dataframe

Every iteration repeats this.

### Better Pattern: Collect → Concat Once

Instead:

```python
frames = []

for ...:
    frames.append(input_v2)

versioned_inputs = pd.concat(frames, ignore_index=True)
```

### Why this is better

Now:
1. each dataframe is created independently
2. list append is cheap
3. concat happens ONE time

Pandas can:
- calculate final shape once
- allocate memory once
- copy data once

This is much more scalable.




