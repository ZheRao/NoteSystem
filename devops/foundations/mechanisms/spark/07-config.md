# Initialization Configuration vs. Runtime Configuration

## 🧭 System-level insight (Lean in — core growth)

This is actually deeper than Spark — it's a **universal system design pattern**:

Every complex system has:
- **Initialization configuration (immutable-ish)**
- **Runtime configuration (mutable)**

You’ll see this everywhere:

- Databases (startup flags vs session variables)
- Kubernetes (pod spec vs runtime env)
- ML systems (model architecture vs inference params)

## ⚡ Practical rule (use this going forward)

✅ **Put in `builder.config()` if**:
- It's part of your **platform definition**
- You want **consistency across jobs**
- You don’t want accidental overrides

✅ **Use `spark.conf.set()` if**:
- It’s **job-specific**
- You’re experimenting
- You need dynamic behavior

# which Spark configs are safe to change at runtime vs not

Perfect — this is one of those “**small surface, huge leverage**” topics. If you internalize this, you’ll avoid a *ton* of silent Spark bugs.

I’m going to give you:

1. ✅ A practical classification (what you can vs can’t change)
2. 🧠 A mechanical reason (so it sticks, not memorized)
3. ⚡ A battle-tested cheat sheet
4. 🔍 A debugging heuristic you’ll actually use

## 🧠 The Real Model (don’t memorize configs — understand this)

Every Spark config falls into one of these:

### 🧱 1. Infrastructure (IMMUTABLE after start)

> Defines the physical execution engine

- Executors
- Memory
- Cores
- Serialization

👉 Changing these after startup is like:
> “Changing CPU count while program is running”

❌ Doesn’t work

### ⚙️ 2. Execution Planning (SEMI-MUTABLE)

> Affects how Spark plans jobs

- Shuffle partitions
- Join strategy
- Broadcast thresholds

👉 Works **only if set before the action is triggered**

⚠️ Subtle bugs live here

### 🧪 3. SQL / Runtime Behavior (FULLY MUTABLE)

> Affects query behavior, not infrastructure

- Partition overwrite mode
- Case sensitivity
- ANSI mode

✅ Safe to change anytime (before the operation)

## 📊 The Cheat Sheet (what you actually need)

### 🧱 IMMUTABLE (builder.config ONLY — do NOT rely on runtime set)
```python
spark.executor.memory
spark.executor.cores
spark.driver.memory
spark.master
spark.serializer
spark.executor.instances
```
🔥 Rule:
> If it affects cluster/resources → **builder only**

### ⚙️ SEMI-MUTABLE (set BEFORE action)
```python
spark.sql.shuffle.partitions
spark.sql.autoBroadcastJoinThreshold
spark.sql.adaptive.enabled
spark.sql.adaptive.coalescePartitions.enabled
```
⚠️ Gotcha:
```python
df = spark.read.parquet(...)

spark.conf.set("spark.sql.shuffle.partitions", "10")

df.groupBy(...).count().show()
```
👉 Works (because plan happens at action)

BUT:
```python
df2 = df.groupBy(...).count()   # plan may already be partially locked

spark.conf.set("spark.sql.shuffle.partitions", "10")

df2.show()
```
👉 Might NOT behave as expected

### 🧪 FULLY MUTABLE (safe, your category)
```python
spark.sql.sources.partitionOverwriteMode
spark.sql.caseSensitive
spark.sql.ansi.enabled
spark.sql.execution.arrow.pyspark.enabled
```
👉 These are query **behavior toggles**

🔬 Your Case — Why it’s SAFE
```python
spark.sql.sources.partitionOverwriteMode = dynamic
```
This controls:
> “When writing partitions, do we overwrite all or just touched ones?”

It does NOT affect:
- cluster
- execution plan complexity
- memory layout

👉 So Spark allows runtime changes

## ⚠️ The Dangerous Zone (this is where people get burned)

🧨 Silent Failure Pattern
```python
df = spark.read.parquet(...)

df2 = df.groupBy("x").count()

spark.conf.set("spark.sql.shuffle.partitions", "10")

df2.show()
```
You think:
> “I set partitions to 10”

Reality:
> Execution plan may already be fixed

## 🧠 Why this happens (important insight)

Spark has **lazy evaluation**, but:
> Logical plan can start forming earlier than you think

So config timing matters relative to:
- transformation definition
- action trigger

## 🧭 Golden Rules (you’ll actually use these)

### Rule 1 — Platform configs → builder
```python
builder.config(...)
```
Examples:
- overwrite mode (your case)
- default shuffle partitions
- adaptive execution

### Rule 2 — Job tuning → set early
```python
spark.conf.set(...)
```
BUT:

👉 Set it **before defining transformations if unsure**

### Rule 3 — Never rely on mid-pipeline changes

❌ Avoid:
```python
df = ...
df2 = df.join(...)

spark.conf.set(...)

df2.write(...)
```
👉 This creates **non-deterministic behavior**

## 🔍 Debugging Heuristic (this will save you hours)

When something feels off:

**Step 1**
```python
print(spark.conf.get("your.config"))
```
**Step 2**

Check execution plan:
```python
df.explain(True)
```
Look for:
- number of partitions
- join strategy


## 🧠 Deep Insight (Lean in — core growth)

What you're touching here is:
> **The boundary between declarative logic and execution engine**

Spark is not just:
> “run my code”

It is:
> “build a plan → optimize → execute”

Configs influence **different layers of that pipeline**

## ⚡ My recommendation for YOUR system (very important)

Given you're building a **long-term data platform**:

**Lock these in builder**:
```python
DEFAULT_CONF = {
    "spark.sql.sources.partitionOverwriteMode": "dynamic",
    "spark.sql.adaptive.enabled": "true",
    "spark.sql.shuffle.partitions": "200",  # or tuned value
}
```
**Never rely on runtime mutation for**:
- core platform behavior
- write semantics

## 🧭 Final Mental Model

```
builder.config → defines the physics of your system
spark.conf.set → tweaks behavior within those physics
```