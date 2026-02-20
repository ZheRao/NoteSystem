# 1. Simple Operations

### Sorting

```python
df.sort_values(["col1", "col2"], ascending=[False, True])
```

### Column Calculations

```python
df["price"] = np.where(df["item_type"] == "expensive", 999.99, 1.99)
df["total_price"] = df["units_sold"] * df["cost"]
df["total_price"] = df["total_price"].round(2)
```

### Type Conversion

```python
df = df.astype({"col1": type1, "col2": type2})
```

### Clip Values

```python
df["Amount"] = df["Amount"].clip(lower=0)
```

# 2. Date & Time

### Parse Dates

```python
df["date"] = pd.to_datetime(df["date"], format="%Y-%m-%d", errors="coerce")
```

Common formats:
- `%m/%d/%Y %I:%M %p`
- `%Y-%m-%d %H:%M:%S.%f`
- `%Y-%m-%dT%H:%M:%S`
- `%Y-%m-%d %H:%M:%S%z`


**Given** `date_year`, `date_month` as names, `date.day` as columns, composing a date column with
```python
df["Date"] = pd.to_datetime(
    df["date_year"].astype(str) + " " +
    df["date_month"] + " " +
    df["date_day"].astype(str),
    format="%Y %B %d"
)
```

### common related codes

**Year**

| Code | Meaning                |
| ---- | ---------------------- |
| `%Y` | 4-digit year (`2026`)  |
| `%y` | 2-digit year (`26`) ⚠️ |

**Month**
| Code | Meaning                | Example   |
| ---- | ---------------------- | --------- |
| `%B` | Full month name        | `January` |
| `%b` | Abbreviated month name | `Jan`     |
| `%m` | Month number (01–12)   | `01`      |

**Days**
| Code | Meaning              |
| ---- | -------------------- |
| `%d` | Day of month (01–31) |



### Date Arithmetic

```python
df["date_diff"] = pd.to_timedelta(df["date_diff"])
df["days_from_monday"] = pd.to_timedelta(df["date"].dt.weekday, unit="D")
df["date_plus_1"] = df["date"] + pd.Timedelta(days=1)
```


### Extract Parts

```python
df["year"] = df["date"].dt.year
df["month_name"] = df["date"].dt.month_name()
```

# 3. String Manipulation

```python
df["first_name"] = df["name"].str.split(" ").str[0]
df["col"].str.slice(3, 5)
df["col"].str.startswith("A")
df["col"].str.capitalize()

# before - 1, 3, 10, 7
df["col"] = df["col"].astype(str).str.zfill(2)
# after -"01", "03", "10", "07"
```

### Efficient Extraction and Concatenation of String Parts

```python
parts = df["Account"].str.split(" ", n=2, expand=True)
df["Account_new"] = parts[0] + parts[1]
```

**Quick Notes**

Why `.str[0]` works on a list object if I do `df["Account"].str.split().str[0]`?
1. `df["Account"].str.split()` is a Series.
2. Each element inside the Series is a Python list.
3. `.str[...]` works on each element of the Series.
4. It applies __getitem__(0) to every element.

`.str.split()` vs `expand=True`
1. without `expand=True`, you get
    ```text
    0    ['4000', 'Revenue', 'Domestic']
    1    ['5000', 'Expense', 'Office']
    ```
    - Each row is a Python list
    - The Series dtype is object
    - You're storing Python objects inside Pandas
2. with `expand=True`, you get
    ```text
             0         1          2
        0   4000   Revenue   Domestic
        1   5000   Expense     Office
    ```
    - This is a **DataFrame**, not a Series.
    - No Python lists stored
    - Each token is stored as a column
    - Structured tabular representation
    - Much more Pandas-native
    - Vectorized operations are cleaner
3. differences
    - **Lists inside cells = semi-structured data.**  
        Columns = structured data.  
        Structured representations scale better.
    - **Memory Layout**  
        Columns are contiguous arrays & Pandas can operate column-wise efficiently   
        vs.  
        Each cell is a pointer to a Python list & Operations involve Python-level indexing
    - **Cleaner Downstream Logic**
        `parts.str[0] + parts.str[1]`  
        vs.  
        `parts[0] + parts[1]`

What `n=2` does
- Perform at most 2 splits → produce at most 3 columns.
- If you only care about first two tokens, this avoids unnecessary work.
- It’s a small optimization, but philosophically correct.


# 4. Column Management & Copying

```python
df = df.drop(columns=["col1", "col2"])
df = df.rename(columns={"old": "new"})
df = df.add_prefix("col_")

df["str_col"] = df["str_col"].replace({"old": "new"})           # replace whole string
df["str_col"] = df["str_col"].str.replace("substring_old", "substring_new", regex=False)    # replace substring
# or
df["str_col"] = df["str_col"].replace("substring_old", "substring_new", regex=True)    # replace substring
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

### Value Counts

```python
df["col"].value_counts()
```

### GroupBy

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

### Pivot

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

### Inspect

```python
df.isna().sum()
```

### Fill

```python
df.fillna({"col1": df["col1"].mean(), "col2": 0}, limit=20)
df["col"].interpolate()
```

Mapping-based fill:

```python
mapping = df["key"].map(map_series)
df["target"] = df["target"].fillna(mapping)
```

### Drop

```python
df.dropna(subset=["col1", "col2"], how="all")
```
