# Mistake #1: `.replace('0','')` → Eliminating Leading `0` in `month`

❌ **What went wrong**
```python
date_split[1].replace('0','')
```
This means:
- `"01"` → `"1"` ✅ (looks right)
- `"10"` → `"1"` ❌ (wrong)
- `"12"` → `"12"` (fine)

So it’s doing a **global replace**, not a “leading zero trim”.

Results:  
The scarier insight is the one you already saw: **row counts alone are weak safeguards**. If one file is skipped and another is read twice, 
total volume can still look “reasonable enough,” especially when duplicated rows are zero-amount or otherwise low-salience. 
That means validation cannot rely on a single aggregate.

✅ **Correct approach**
```python
f"{date_split[0]}_{int(date_split[1])}"
```
Why this is best:
- `"01"` → `1`
- `"10"` → `10`
- no edge cases
- no string hacks

👉 This is the one you should default to.

## Lesson #1

### 🔑 A simple control mechanism (this is the real tool)

When you see something like:
- dates
- numbers
- IDs
- timestamps

Run this quick internal check:

👉 **“Am I treating structured data as a string?”**

If yes → force this switch:
> “Can I convert this into a typed representation first?”

That’s it.

### Why you sometimes get it right and sometimes don’t

Because your brain has two competing optimization strategies:

**Fast path (what just happened)**
- continue current pattern
- minimal context switching
- lower cognitive load

**Correct path**
- step back
- reframe data type
- slightly higher cognitive cost

Under flow, the brain prefers fast path.

## Lesson #2: how I caught it when I was examining the data for another concern

### 🔥 What you actually did (this is high-level behavior)

Your problem was:
> “Why is the data incomplete?”

That is **not a distributed computation problem**.
That is a **distribution inspection problem**.

And you pivoted to:
```python
df["month"].value_counts()
```
That is exactly the right tool for that question.

### 🧠 This is a key system invariant

> **Use Spark to move data. Use Pandas to understand data.**

Spark is good at:
- large-scale transformations
- joins, aggregations, pipelines
- distributed execution

Pandas is good at:
- seeing patterns instantly
- sanity checks
- distributions
- anomalies
- quick iteration

You just naturally switched layers.

That’s a big deal.

### 🔁 Turn this into a repeatable playbook

Next time you feel:
- “this doesn’t make sense”
- “Spark is acting weird”
- “data seems off”

Do this immediately:
```python
sample = df.limit(10000).toPandas()
```
Then:
```python
sample["col"].value_counts()
sample.describe()
sample.head()
```

### 🧠 Even better: Spark-native equivalent (for scale)

Once you **know what to look for**, you can go back to Spark:
```python
df.groupBy("month").count().orderBy("month").show()
```
But the key is:
- **Pandas to discover**
- **Spark to scale**

## Reflection

### 🧠 Why this feels terrifying

Because you just realized:
> systems can look correct locally and be wrong globally

Everything can be:
- syntactically valid
- logically consistent
- runs without errors

…and still be:
> silently wrong

That’s the most dangerous class of bugs.

### 🔁 What actually scales (this is key)

You cannot scale correctness by:
> checking every line of code

You scale correctness by:
> enforcing invariants at the system level

Like you just did.

## 🔧 Turn this into a system (this is your upgrade)

Instead of relying on intuition alone, formalize it.

For any pipeline, always check:

### 1. Completeness
```python
df.groupBy("year", "month").count().orderBy("year", "month").show()
```
👉 Are any months missing?

### 2. Distribution sanity
```python
df.groupBy("month").count().show()
```
👉 Does it look reasonable?

### 3. Range checks
```python
df.agg(F.min("date"), F.max("date")).show()
```
👉 Does time span match expectations?

### 4. Null checks
```python
df.filter(F.col("critical_col").isNull()).count()
```
### 5. Cross-field consistency

Example:
- fiscal_year vs date
- transaction type vs amount sign
- etc.

# Mistake #2: Uncontrolled Overwrite 

**What is happening is**:
- your **raw file selection logic** is based on calendar-quarter storage boundaries (files starts with Jan, Apr, Jul, Oct)
- your **business partition key** is `fiscal_year`
- because fiscal year starts in **November**, reading the raw files needed for FY2024/FY2025/FY2026 necessarily pulls in some records that belong to **FY2023**
- then your write step partitions by `fiscal_year`
- so unless you explicitly filter the final DataFrame, Spark will also overwrite `fiscal_year=2023`

**Your fix is the right fix**
```python
df2 = df2.filter(F.col("fiscal_year").isin(scope))
```

## The deeper invariant

You now have three different scopes:

1. **Raw read scope**  
Which files must be read so no needed records are missed
2. **Transformation scope**  
Which rows are needed to compute/derive fields correctly
3. **Write scope**  
Which final business partitions are allowed to be updated

Those three scopes are often **not identical**.

That is the real lesson here.

Your raw read scope must be **wider** than your write scope because of boundary overlap.

### The operational rule

In systems like this, the write layer should usually be protected by an explicit contract:
> “Even if upstream extraction reads extra rows for correctness, only these target business partitions may be published.”

So yes, the final filter is the right guardrail.

## I would make it stricter than just filtering

Filtering is necessary, but I would also add an assertion so accidental leakage becomes impossible to miss.

Example:
```python
from pyspark.sql import functions as F

allowed = set(scope)

actual = {
    row["fiscal_year"]
    for row in df2.select("fiscal_year").distinct().collect()
}

unexpected = actual - allowed
if unexpected:
    print(f"[WARN] extra fiscal_year values present before final write filter: {sorted(unexpected)}")

df2 = df2.filter(F.col("fiscal_year").isin(scope))

post = {
    row["fiscal_year"]
    for row in df2.select("fiscal_year").distinct().collect()
}

if post - allowed:
    raise ValueError(f"Write dataframe still contains unexpected fiscal years: {sorted(post - allowed)}")
```
Even better: fail hard before write if the post-filter result is wrong.

## 🔥 What just happened (objectively)

You encountered a **silent data corruption class bug**:
- No errors
- No crashes
- Output “looked valid”
- But semantics were wrong (FY2023 overwritten unintentionally)

This is one of the **highest-risk categories** in data systems.

You didn’t just fix a bug — you detected a **failure mode**.

## 🧠 Why this bug is especially dangerous

Because it sits at the intersection of:
- time boundaries (calendar vs fiscal)
- storage chunking (quarterly files)
- derived fields (`fiscal_year`)
- distributed writes (Spark partitioning)

This is exactly where systems break in production.

Not junior-level bugs. This is **real-world failure terrain**.

## ⚠️ The uncomfortable truth

> If you hadn’t caught it, it *would* have silently corrupted historical data.

- No alert
- No exception
- No obvious signal

Just gradually wrong outputs.

That’s not hypothetical — that’s exactly how these systems fail in industry.

## 🧭 But here’s the important reframing

This is not a sign you’re vulnerable.

This is proof that:
> You are now operating at a level where correctness is your responsibility, not the framework’s.

Spark will not save you here.

And that’s by design.

## 🧱 What you just unlocked (this is the real value)

You’ve discovered a **class of invariant you must enforce explicitly**:
> **“Write scope must be explicitly bounded and verified.”**

This is now part of your system design DNA.

## 🛡️ Turn this fear into a permanent guardrail

Right now you have:
```python
df2 = df2.filter(F.col("fiscal_year").isin(scope))
```
Good.

But not enough for long-term safety.
```python
Add a hard invariant check (non-negotiable)
actual = {r["fiscal_year"] for r in df2.select("fiscal_year").distinct().collect()}
expected = set(scope)

if actual != expected:
    raise ValueError(
        f"Write scope mismatch.\nExpected: {sorted(expected)}\nActual: {sorted(actual)}"
    )
```
This converts:

- ❌ silent corruption  
→ into
- ✅ immediate failure

## 🧠 The deeper pattern (this is the real lesson)

You just encountered:
> **Boundary mismatch between data extraction and business semantics**

This pattern will repeat in many forms:
- timezone vs date partitions
- late-arriving data
- incremental loads
- slowly changing dimensions
- API pagination boundaries
- window-based aggregations

This wasn’t a one-off bug.

This is a **category**.