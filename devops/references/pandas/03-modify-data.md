# 1. Simple Operations

**Sorting**

```python
df.sort_values(["col1", "col2"], ascending=[False, True])
```

**Column Calculations**

```python
df["price"] = np.where(df["item_type"] == "expensive", 999.99, 1.99)
df["total_price"] = df["units_sold"] * df["cost"]
```

**Type Conversion**

```python
df = df.astype({"col1": type1, "col2": type2})
```

**Clip Values**

```python
df["Amount"] = df["Amount"].clip(lower=0)
```

# 2. Date & Time

**Parse Dates**

```python
df["date"] = pd.to_datetime(df["date"], format="%Y-%m-%d", errors="coerce")
```

Common formats:
- `%m/%d/%Y %I:%M %p`
- `%Y-%m-%d %H:%M:%S.%f`
- `%Y-%m-%dT%H:%M:%S`
- `%Y-%m-%d %H:%M:%S%z`

**Date Arithmetic**

```python
df["date_diff"] = pd.to_timedelta(df["date_diff"])
df["days_from_monday"] = pd.to_timedelta(df["date"].dt.weekday, unit="D")
df["date_plus_1"] = df["date"] + pd.Timedelta(days=1)
```

**Extract Parts**

```python
df["year"] = df["date"].dt.year
df["month_name"] = df["date"].dt.month_name()
```

# 3. String Manipulation

```python
df["first_name"] = df["name"].str.split(" ").str[0]
df["col"].str.zfill(2)
df["col"].str.slice(3, 5)
df["col"].str.startswith("A")
```

# 4. Column Management & Copying

```python
df = df.drop(columns=["col1", "col2"])
df = df.rename(columns={"old": "new"})
df = df.add_prefix("col_")

df["str_col"] = df["str_col"].replace({"old": "new"})
```

```python
df2 = df1.copy(deep=True)
```

# 5. Merging & Concatenation

```python
pd.merge(df1, df2, on="key", how="left", suffixes=["_df1", "_df2"])
pd.concat([df1, df2], ignore_index=True)
```

# 6. Index Management

```python
df.set_index("column")
df.reset_index(drop=True)
```

# 7. Aggregation

**Value Counts**

```python
df["col"].value_counts()
```

**GroupBy**

```python
df.groupby("col1").agg({
    "col2": "sum",
    "col3": "mean",
    "col4": "prod"
}).reset_index()
```

Concatenation per group:

```python
df.groupby("id", as_index=False).agg(
    combined=("text_col", " ".join)
)
```

**Pivot**

```python
df.pivot_table(
    index="row",
    columns="col",
    values="val",
    aggfunc="sum",
    fill_value=0
)
```

# 8. Missing Values

**Inspect**

```python
df.isna().sum()
```

**Fill**

```python
df.fillna({"col1": df["col1"].mean(), "col2": 0}, limit=20)
df["col"].interpolate()
```

Mapping-based fill:

```python
mapping = df["key"].map(map_series)
df["target"] = df["target"].fillna(mapping)
```

**Drop**

```python
df.dropna(subset=["col1", "col2"], how="all")
```
