# 1. Common Filtering

## Unique Values

```python
df["col"].unique()
```

## Column/Cell Access

```python
df.iloc[:, 1:6]
df.loc[:, ["col1", "col2"]]

df["col1"]
df.col1

df.at[row_label, column_name]        ## fastest calar access, e.g. df.at[0, "col1"]

df["col"].max()             # generate max value
df["col"].idxmax()          # generate the first row_label of the max value occurrence, ignore NaN, if all NaN, returns NaN
```

### `.at` vs. `.loc`
- `.loc` → general-purpose (can return Series/DataFrame)
- `.at` → optimized for single value access

👉 Faster + more explicit intent

## Conditional Filtering

```python
df.loc[df["sales"] >= 5000, ["itemID", "sales"]]
df[(df["sales"] >= 5000) & (df["cost"] == 30)]
df[df.duplicated(subset=["col1", "col2"], keep="first")]
```

## `.query()` Syntax

```python
df.query('countery == "USA" and state == "New York"')
```

## String Filtering (No Regex)

```python
df[df["name"].str.contians("fire", case=False, regex=False, na=False)]
```

**clean multi-regex slicing**

```Python
mask_feedlot = df["Class"].str.contains(
    r"\bfeedlot\b",
    case=False,
    na=False,
)

mask_cow = (
    df["Class"].str.contains(
        r"\bcow\s*[/-&]?\s*calf\b",
        case=False,
        na=False,
    )
)

mask_classified = mask_feedlot | mask_cow

df_class_feedlot = df.loc[mask_feedlot].copy()
df_class_cow = df.loc[mask_cow].copy()
df = df.loc[~mask_classified].copy()
```
Why this is better:
- each rule is named
- you only evaluate each regex once
- the exclusion logic becomes explicit
- much easier to debug counts like:
    ```python
    mask_feedlot.sum()
    mask_cow.sum()
    mask_classified.sum()
    ```

# 2. Membership & Null Checks

```python
df[df["country"].isin(["USA", "UK"])]

df[~df["col"].isna()]
df["col"].notna()
```
