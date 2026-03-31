# Assignment with Matched Index

This is a **great question**, and this is exactly the mental model you need to fully “unlock” Pandas.

Let’s go step by step and remove the confusion completely.

## 🧠 Core answer (short version)
```python
df.loc[mask, "col"]
```
returns:
> a Series containing only the rows where `mask == True`, with their original index preserved

NOT:
- ❌ not full length with nulls
- ❌ not reindexed to 0..N

## 🔍 What it actually looks like

Suppose:
```python
df = pd.DataFrame({
    "A": [10, 20, 30, 40],
    "Memo": ["a", "b", "c", "d"]
}, index=[0, 1, 2, 3])

mask = df["A"] > 20
```
So:
```python
mask = [False, False, True, True]
```
**Now evaluate**:
```python
df.loc[mask, "Memo"]
```
You get:
```python
2    c
3    d
Name: Memo, dtype: object
```

**Key observations**:
- length = 2 (only True rows)
- index = [2, 3] (original indices preserved)

## 🔥 THIS is the key mechanism

> Index alignment is how Pandas “remembers everything.”

Pandas does NOT rely on:
- row order
- implicit position
- hidden mapping

It relies on:
> explicit index labels

## 🧩 So how does assignment work?

When you do:
```python
df.loc[mask, "Memo"] = some_series
```
Pandas does:
1. Identify target rows via mask → indices `[2, 3]`
2. Look at RHS (`some_series`)
3. Align RHS by **index labels**
4. Assign values to matching indices

Example
```python
s = pd.Series(["X", "Y"], index=[2, 3])

df.loc[mask, "Memo"] = s
```
Result:
```python
0    a
1    b
2    X
3    Y
```

## ⚠️ Important subtlety

If you do:
```python
s = pd.Series(["X", "Y"])  # index = [0, 1]
df.loc[mask, "Memo"] = s
```
Now Pandas tries to align:
- target indices: `[2, 3]`
- RHS indices: `[0, 1]`

No match → you get:
```python
2    NaN
3    NaN
```
💥 This is a classic bug source.

## 🧠 Back to your case

When you do:
```python
original_key = df.loc[mask, "inventory_year_month_key"].copy()
```
That Series has:
- only masked rows
- original indices

Then:
```python
note = "(" + original_key.astype(str) + ...
```
`note` keeps the same indices.

Then:
```python
df.loc[mask, "Memo"] = note + df.loc[mask, "Memo"].fillna("")
```
Both sides:
- have same masked indices
- align perfectly

So Pandas knows exactly:
- which row gets which memo
- without copying the full dataframe

## 🧬 Mental model upgrade

Think of Pandas as:
> a dictionary of columns keyed by index labels

NOT:
> a 2D array with positions

## 🧠 Final intuition (this is the one to keep)

When you write:
```python
df.loc[mask, "col"] = something
```
You are saying:
> “For rows with indices X, update column col with values aligned by index”

NOT:
> “Replace the Nth to Mth rows”

## 🚀 Why this matters for your trajectory

This concept directly maps to:
- SQL → joins on keys
- Spark → row identity via partition + key
- distributed systems → key-based computation

👉 Lean in — core growth

# Unindexed List

When RHS is a **list / array (no index)**:
```python
df.loc[mask, "col"] = some_list
```
Pandas switches from **label alignment** → to **positional assignment**
> It assigns values in order to the selected rows.

## 🔍 Example
```python
mask = df["A"] > 20   # indices [2, 3]

df.loc[mask, "Memo"] = ["X", "Y"]
```
Pandas interprets this as:
- first True row (index 2) → `"X"`
- second True row (index 3) → `"Y"`

Result:
```
2 → X
3 → Y
```
## ⚠️ Length must match

Yes — this will fail:
```python
df.loc[mask, "Memo"] = ["X"]   # ❌ too short
df.loc[mask, "Memo"] = ["X", "Y", "Z"]  # ❌ too long
```
Error like:
> ValueError: Must have equal len keys and value

Because Pandas has no index to align — it relies purely on **position count**.