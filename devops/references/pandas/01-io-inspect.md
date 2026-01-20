# DataFrame Creation & IO

## Create DataFrame

```python
df = pd.DataFrame(
    data=[[1,2,3], [4,5,6]],
    columns=["col1", "col2", "col3"],
    index=["row1", "row2"]
)
```

## Read Files

```python
pd.read_csv(path, dtype={"col": "string"}, usecols=columns)
pd.read_parquet(path)       # storage-efficient
pd.read_excel(path, sheet_name="sheet1", dtype={"col": "string"})
pd.read_json(path)
```

## Empty DataFrame With Same Schema

```python
empty_df = df.head(0).copy(deep=True)

new_row = {"col1": v1, "col2": v2}
empty_df.loc[len(empty_df)] = new_row
```

# Inspecting DataFrames

```python
df.columns
list(df.index)
df.index.name

df.info()
df.describe()

df.shape
df.size

df.head(n)
df.tail(n)
df.sample(n, random_state=seed)

df1.equals(df2) # compare if two data frames are exact same (values + index + column labels + dtypes)
```


# Writing Output

**CSV**

```python
df.to_csv(path, index=False)
```

**Excel**

```python
with pd.ExcelWriter("file.xlsx") as writer:
    df1.to_excel(writer, sheet_name="Sheet1", index=False)
    df2.to_excel(writer, sheet_name="Sheet2", index=False)
```

