# Pandas `groupby()`

Great question — and this is one of those things where once it “clicks,” a lot of Pandas becomes much clearer.

You’ve only seen:
```python
df.groupby(...).agg(...)
```
But that’s just one use case (aggregation).  
`groupby` is actually much more general.

## 🧠 What groupby really is

At a high level:
> `df.groupby(col)` = a way to partition rows into buckets based on `col`

It does **NOT** immediately compute anything.

Instead, it creates a **lazy grouping object** that knows:
- which rows belong to each group
- how to retrieve them efficiently

## 🔍 Think of it like this

Given:
```python
df
```
```
index | fiscal_year | value
0     | 2024        | 10
1     | 2025        | 20
2     | 2024        | 30
3     | 2025        | 40
```
When you do:
```python
g = df.groupby("fiscal_year")
```
You now have something like:
```
{
  2024 → rows [0, 2]
  2025 → rows [1, 3]
}
```
But nothing is computed yet — it’s just organized.

## 🧱 What is inside groupby?

Internally, it stores:

### 1. Group keys
```python
[2024, 2025]
```
### 2. Row indices per group
```python
{
  2024: [0, 2],
  2025: [1, 3]
}
```
### 3. Reference to original DataFrame

So when you ask for a group, it does:
> “give me rows [0, 2] from df”

## 🧪 How you can actually use it


### 1. Iterate over groups (this is what you need)
```python
for fy, chunk in df.groupby("fiscal_year"):
    print(fy)
    print(chunk)
```
Output:
```
2024
  rows 0, 2

2025
  rows 1, 3
```
👉 This is why it's perfect for your use case:
> “for each fiscal year → write a file”


### 2. Get a specific group
```python
g = df.groupby("fiscal_year")

g.get_group(2024)
```
### 3. Apply a function to each group
```python
df.groupby("fiscal_year").apply(lambda x: x.head(1))
```
### 4. Transform (important for your earlier question)
```python
df["total"] = df.groupby("fiscal_year")["value"].transform("sum")
```
This:
- computes sum per group
- **broadcasts back to original rows**

### 5. Aggregation (what you already know)
```python
df.groupby("fiscal_year").agg({"value": "sum"})
```

## 🔥 Key mental shift
`groupby` is NOT aggregation

It is:
> a partitioned view of your DataFrame

Aggregation is just one operation you can perform on that partition.

## ⚙️ Why it’s more efficient than your loop

Your loop:
```python
df[df["fiscal_year"] == fy]
```
→ scans entire column every time

groupby:
```python
df.groupby("fiscal_year")
```
→ builds mapping once, then reuses it

🧠 Mapping to your system thinking

This is actually very aligned with your architecture:

| Pandas concept         | Your system               |
| ---------------------- | ------------------------- |
| `groupby`              | partitioning              |
| group                  | partition                 |
| iteration over groups  | partition-wise processing |
| `partitionBy` in Spark | same idea                 |

So this:
```python
for fy, chunk in df.groupby("fiscal_year"):
```
is basically:
> “for each partition, process independently”

Exactly like Spark.

## ⚠️ Subtle details (important)

### 1. `groupby` does NOT copy data immediately

Chunks are views (or lightweight slices)

→ efficient

### 2. Order is not guaranteed unless specified
```python
df.groupby("fiscal_year", sort=False)
```
Use this to preserve input order (usually what you want)

### 3. Multi-column grouping works
```python
df.groupby(["fiscal_year", "location"])
```
Now keys become tuples:
```
(2024, "Airdrie")
```

## 🧠 The “aha” insight

You were thinking:
> “how do I repeatedly filter df into smaller pieces?”

But groupby says:
> “what if the DataFrame was already organized into those pieces?”

That’s the real shift.

## ✅ For your exact use case

Best pattern:
```python
df_scope = df[df["fiscal_year"].isin(scope)]

for fy, chunk in df_scope.groupby("fiscal_year", sort=False):
    chunk.to_parquet(pl_path / f"pl_{fy}.parquet", index=False)
```

## Final takeaway
> `groupby` is a partitioning mechanism, not an aggregation function

Once you see it that way:
- `.agg()` = summarize partitions
- `.transform()` = broadcast within partitions
- iteration = process partitions independently