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

df.at[0, "col1"]        ## fastest calar access
```

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

# 2. Membership & Null Checks

```python
df[df["country"].isin(["USA", "UK"])]

df[~df["col"].isna()]
df["col"].notna()
```
