# `repartition` and file layout

## Key rule

```python
df.repartition(n, "fiscal_year")
```

- All rows with the same `fiscal_year` go to **exactly one Spark partition**
- With `.write.partitionBy("fiscal_year")`:
    - **≤1 file per fiscal_year per write**


## Case Study (3 fiscal years, 20 original partitions)

- **No repartition** (20 partitions)
    - Each partition may contain mixed years
    - Each partition writes its own file
    - → 20 files where each could contain records from any year

- `.repartition(3, "fiscal_year")`
    - 3 Spark partitions
    - Each year → one partition
    - → 3 files, 1 file per year

- `.repartition(2, "fiscal_year")`
    - One partition holds 1 year
    - One partition holds 2 years
    - → 2 files, 1 file contains 2 years

- `.repartition(4, "fiscal_year")`
    - 4 partitions, 3 keys
    - 1 partition empty
    - → 4 files, 1 file is empty

# Spark Shuffle

Good — this is exactly the direction you want to go.  
This is where the “toy → real system” boundary becomes very concrete.

## 🔥 What is a Spark shuffle (intuitively)

> A shuffle is when Spark has to move data across machines to regroup it.

In pandas:
- everything is already in one place → grouping is local

In Spark:
- data is split across executors → grouping often requires repartitioning

## 🔹 Step-by-step: what happens during `groupBy` / `Window`

Let’s say your data is distributed like this:
```
Partition 1:
Airdrie (CC), 202602, 120
Waldeck (CC), 202602, 90

Partition 2:
Airdrie (F),  202602, 80
Waldeck (F),  202602, 60
```
Now you run:
```python
groupBy("master_location", "year_month_key")
```
### Step 1 — Hash the keys

Spark computes:
```
hash("Airdrie", 202602) → Partition X
hash("Waldeck", 202602) → Partition Y
```

### Step 2 — Shuffle (data movement 🚨)

Rows are physically moved:
```
Partition X:
Airdrie (CC), 120
Airdrie (F),  80

Partition Y:
Waldeck (CC), 90
Waldeck (F),  60
```
👉 Now each group is colocated

### Step 3 — Compute locally

Now Spark can safely compute:
```
Airdrie → sum = 200
Waldeck → sum = 150
```

### Step 4 — (for window) broadcast back

For window functions:
```python
F.sum(...).over(Window.partitionBy(...))
```
Spark:
- keeps row-level structure
- attaches the computed group value to each row

## 🔹 Why shuffle is expensive

Because it involves:

### 1. Network I/O
- data moves between machines

### 2. Disk I/O
- intermediate data may spill to disk

### 3. Serialization
- data must be encoded/decoded

### 4. Synchronization barrier
- all tasks must finish before next stage

👉 This is where real systems slow down or break

## 🔹 This is what the LinkedIn post meant
> “AI writes clean code… production doesn’t care”

Your pandas code:
```python
groupby(...).transform(...)
```
Looks trivial.

But in Spark:
> That one line = distributed shuffle + recomputation + coordination

## 🔹 Common failure modes (this is gold)

❌ **1. Data skew**

Example:
```
Airdrie → 10 million rows
Waldeck → 100 rows
```
After shuffle:
```
Partition X → huge (slow)
Partition Y → tiny (fast)
```
👉 One machine becomes bottleneck

❌ **2. Too many small partitions**
- overhead dominates
- scheduler inefficiency

❌ **3. Too few partitions**
- not enough parallelism
- large memory pressure per executor

❌ **4. Repeated shuffles**
```python
df.groupBy(...).agg(...)
df.groupBy(...).agg(...)
df.groupBy(...).agg(...)
```

👉 each one = full shuffle

❌ **5. Joining large tables without partition alignment**

👉 triggers another shuffle

## 🔹 How to think about it (this is the upgrade)

In pandas, you think:
> “what transformation do I want?”

In Spark, you must also think:
> “what data movement does this require?”

## 🔹 Window vs groupBy + join (real difference)

Both require shuffle, but:

**Window**:
- one shuffle
- compute + attach in one step
**groupBy + join**:
- shuffle for groupBy
- shuffle again for join (often)

👉 window is often cheaper

## 🔹 Visual mental model

Think of your data as:
```
[Machine A]  [Machine B]  [Machine C]
```
Before group:
```
mixed keys everywhere
```
After shuffle:
```
Machine A → all Airdrie
Machine B → all Waldeck
```
👉 grouping requires **reorganizing the entire dataset**

## 🔹 Why your current system hasn’t hit this yet

Because:
- your data size is still manageable
- pandas hides distribution
- no network / cluster constraints

That’s why:
> everything feels smooth… until suddenly it doesn’t

## 🔹 Practical advice (important for your trajectory)

When you move to Spark-scale thinking:

**Always ask**:
1. Does this operation cause a shuffle?
2. How large are the groups?
3. Are keys evenly distributed?
4. Can I reuse grouped results?
5. Can I reduce data before grouping?

🔚 Blunt truth

The difference between:
> “clean code”  
> and  
> “production-ready system”

is often:
> understanding where the data moves

Not just what the code does.

## 🚀 Lean in — core growth

You just crossed into one of the most important system intuitions:
> Computation is cheap. Data movement is expensive.

Everything in distributed systems flows from this.
