# Transmit Rows

## Row-based

```python
yield Row(id=..., val=...)
```

- Column names inferred from field names
- Schema inference may trigger a small job


## Tuple-based (recommended)

Define schema

```python
schema = StructType([
    StructField("id",   StringType(),   False),
    StructField("val",  IntegerType(),  True),
    ...
])
```

Yield tuples (order matters)

```python
def f(it):
    for r in it:
        yield (_id, _val, ...)

df = spark.createDataFrame(rdd, schema=schema)
```

Or yield from dictionary

```python
rows["id"] = ...
rows["val"] = ...
yield row
```

- No schema inference
- Faster
- Predictable types

# Spark Schema

## 🧠 1. What a Spark Schema actually is

Think of a Spark DataFrame as:
> RDD of Rows + Schema

The schema is not optional metadata. It is:
- the **contract**
- the **column order**
- the **type system**
- the **execution hint for Spark**

### 🔑 Core idea

A schema is a **tree structure**:
```
StructType
  ├── StructField (col1)
  ├── StructField (col2)
  └── StructField (col3)
```
Each field = one column

## 🧩 2. Core building blocks

✅ `StructType` → **the whole schema**
```python
from pyspark.sql.types import StructType

schema = StructType([...])
```
- Represents the entire row structure
- Think: **table definition**

### ✅ StructField → one column
```python
from pyspark.sql.types import StructField, StringType

StructField("col_name", StringType(), True)
```
Arguments:
1. name → `"col_name"`
2. type → `StringType()`
3. nullable → `True` / `False`

### ✅ Primitive types (most common)
```python
from pyspark.sql.types import *

StringType()
IntegerType()
DoubleType()
BooleanType()
TimestampType()
DateType()
```
👉 In your current system:
> everything = `StringType()` (good starting point)

## 🧱 3. Minimal working example

from pyspark.sql.types import StructType, StructField, StringType
```python
schema = StructType([
    StructField("col1", StringType(), True),
    StructField("col2", StringType(), True),
    StructField("col3", StringType(), True),
])
```
Use it:
```python
df = spark.createDataFrame(rdd, schema=schema)
```
👉 Spark will:
- NOT infer schema
- TRUST your definition
- enforce column order

### ⚠️ Critical: column order comes from schema

Even if your data is:
```python
Row(col2="b", col1="a")
```
Spark will output:
```
col1 | col2
```
👉 Schema order wins

## 🧠 4. Why this matters for YOU

You are doing:
> heterogeneous → canonical schema

So:
- schema = **global contract**
- data = **mapped into it**

👉 This is exactly what `StructType` is for

## ⚡ 5. Use case pattern

**Build schema dynamically**
```python
from pyspark.sql.types import StructType, StructField, StringType

schema = StructType([
    StructField(col, StringType(), True)
    for col in columns
])
```
**Feed normalized rows**
```python
from pyspark.sql import Row

Row(**record)
```
**Create**
```python
df = spark.createDataFrame(rdd, schema=schema)
```
## 🧠 6. Important behavior details (this is where people get burned)

### 🔥 1. Missing keys → null
```python
schema = ["a", "b"]

Row(a=1)
```
→ becomes:
```
a=1, b=None
```
### 🔥 2. Extra keys → ignored (or error depending on version)

If your Row has fields not in schema:
- Spark may drop them
- or error

👉 Best practice: align before

### 🔥 3. Type mismatch

If schema says:
```python
IntegerType()
```

but data is `"abc"` → 💥

👉 That’s why your “all string first” approach is smart

### 🔥 4. Nullable = False
```python
StructField("col1", StringType(), False)
```
- null values will cause failure

👉 You are correct to use True

## 🧩 7. Nested schemas (important for future)

Schemas can be nested:
```python
StructType([
    StructField("id", StringType(), True),
    StructField("meta", StructType([
        StructField("source", StringType(), True),
        StructField("timestamp", StringType(), True)
    ]), True)
])
```
👉 This becomes:
```
id | meta.source | meta.timestamp
```
## 🧠 8. Schema introspection (very useful)
```python
df.printSchema()
```
Example output:
```
root
 |-- col1: string (nullable = true)
 |-- col2: string (nullable = true)
 ```
### Access programmatically
```python
df.schema
```
or:
```python
for field in df.schema:
    print(field.name, field.dataType)
```

## ⚡ 9. Convert schema to/from JSON (advanced but useful)
```python
schema.json()
```
or:
```python
StructType.fromJson(json_obj)
```
👉 useful for:
- saving schema
- reusing across jobs

## 🧠 10. Big picture (this is the important upgrade)

You are moving from:
> “data-driven structure” (inferSchema)

to:
> “schema-driven system”

That means:
- you control structure
- you control consistency
- Spark becomes execution engine, not decision maker

## 🎯 Practical rules for your system
**Always do:**
- explicit schema
- nullable = True
- StringType first
- normalize before DataFrame

**Avoid:**
- schema inference
- relying on dict structure
- mixing types early

## 🧠 Final compression

Think of Spark like this:
| Concept         | Your system equivalent |
| --------------- | ---------------------- |
| StructType      | global schema          |
| StructField     | column definition      |
| Row             | normalized record      |
| createDataFrame | execution              |

# Schema Evolution - Mutliple DFs

Good. This is the **exact next layer** you need — and it connects directly to what you’re building.

We’re going to cover:

1. **What schema evolution actually means in Spark**
2. **How to merge schemas (your real problem)**
3. **How Spark handles missing columns**
4. `unionByName` **mechanics (VERY important)**
5. **How to design your pipeline around this**

## 🧠 1. What “schema evolution” really is

In your case:

You have multiple datasets like:
```
File A → [col1, col2, col3]
File B → [col2, col3, col4]
File C → [col1, col4]
```
You want:
```
[col1, col2, col3, col4]
```
👉 That is **schema evolution = union of schemas**

## 🔥 2. The core invariant

> ***All DataFrames must share the same schema before combining***

Spark does NOT magically align different schemas unless you tell it how.

## ⚠️ 3. The WRONG way (very common bug)
```python
df1.union(df2)
```
This is position-based:
```
df1: [col1, col2]
df2: [col2, col1]
```
👉 Spark will match:
```
col1 ↔ col2  ❌ WRONG
col2 ↔ col1  ❌ WRONG
```

## ✅ 4. The RIGHT way: unionByName
```python
df1.unionByName(df2)
```
Now Spark matches:
```
by column name ✔️
```
🔥 Even better: allow missing columns
```python
df1.unionByName(df2, allowMissingColumns=True)
```
This does:
- aligns columns by name
- fills missing columns with `null`

### Example

**df1**:
```
col1 col2
 a    b
```
**df2**:
```
col2 col3
 x    y
```
**Result**:
```
col1 col2 col3
 a    b    null
 null x    y
```
👉 This is EXACTLY your normalization goal

## 🧠 5. What Spark is doing internally

When you use:
```python
unionByName(..., allowMissingColumns=True)
```
Spark effectively does:
1. compute superset of columns
2. add missing columns with null
3. reorder columns
4. union

👉 That’s the same system you’re building manually

## 🔥 6. So why not just use `unionByName`?

Good question — this is where your case is different.

**You are NOT starting with DataFrames**

You are starting with:
- raw files
- inconsistent schemas
- custom parsing logic

👉 So you must:
- normalize BEFORE DataFrame creation

## 🧩 7. Two valid architectures

### ✅ Option A — your current approach (recommended)
```
raw → normalize → create DF (full schema)
```
Pros:
- full control
- deterministic
- safe

### ⚡ Option B — Spark-native evolution
```
file → df (local schema)
→ unionByName(all dfs)
```
Pros:
- simpler
Cons:
- less control
- harder debugging
- relies on Spark behavior

## 🧠 8. Schema merging logic (important)

You asked earlier about building schema dynamically.

Here’s the real pattern:

### Step 1 — collect all schemas
```python
all_columns = set()

for file in files:
    cols = extract_column_meta_one_file(file)
    all_columns.update(cols)
```
### Step 2 — define order

You must choose:

**Option 1 (recommended)**:
```python
columns = sorted(all_columns)
```
**Option 2 (better for you)**:
- preserve first-seen order
```python
columns = []
seen = set()

for file in files:
    for col in extract_column_meta_one_file(file):
        if col not in seen:
            columns.append(col)
            seen.add(col)
```
👉 This gives:
- deterministic
- meaningful order

### Step 3 — build schema
```python
schema = StructType([
    StructField(col, StringType(), True)
    for col in columns
])
```
## 🔥 9. Key subtlety: column order stability

Even though Spark can align columns:

👉 YOU should still maintain stable order

Why?

- Parquet schema consistency
- reproducibility
- debugging
- downstream joins

## 🧠 10. Schema evolution vs schema enforcement

Evolution:
> expand schema when new columns appear

Enforcement:
> reject unexpected columns

You are doing:
> controlled evolution

That’s ideal.

## ⚡ 11. Advanced: schema merge function (very useful)

You can formalize this:
```python
def merge_columns(existing, new_cols):
    seen = set(existing)
    for col in new_cols:
        if col not in seen:
            existing.append(col)
            seen.add(col)
    return existing
```
👉 This becomes a reusable primitive

🧠 12. Where this connects to your future

This exact pattern is:
- Spark schema evolution
- Delta Lake schema merge
- data warehouse ingestion
- feature store alignment

And even deeper:
> mapping multiple representations into a stable internal space

That’s the same pattern as:
- embeddings
- representation learning
- your future architecture ideas

## 🎯 Final mental model
You are building:
```
Local schemas (variable, messy)
        ↓
Schema merge (union)
        ↓
Canonical schema (stable)
        ↓
Row normalization
        ↓
Spark execution
```

# Type Evolution + Conflict Resolution + Schema Versioning

Yep — now we get into the part where Spark stops being “just a dataframe library” and starts looking like a **schema-governed execution system**.

The three next layers are:
1. **type evolution** — when a column is not always the same type
2. **conflict resolution** — when the same column name may not mean the same thing
3. **schema versioning** — when the canonical schema itself changes over time

## 1. Type evolution

This is the problem:
```python
file_a: revenue = "123.45"
file_b: revenue = "98"
file_c: revenue = ""
file_d: revenue = "N/A"
```
At the raw ingestion stage, the same logical column may appear as:
- decimal-looking strings
- integer-looking strings
- blanks
- junk placeholders

That is why your instinct to start with all `StringType()` is very strong. 
Spark schemas are explicit trees of `StructType` containing `StructField`(name, dataType, nullable), 
and `createDataFrame` accepts an explicit schema instead of inferring one.

### Good ingestion rule

For messy multi-report ingestion:
> Bronze/raw normalized layer = all strings, nullable=True

Then later:
> Silver/typed layer = cast selected columns intentionally

That avoids early breakage from mixed source behavior.

**Why this is better**

Because type evolution is usually not a parsing problem. It is a **semantic decision**:
- should `"0012"` stay string or become int?
- should `"2026-03-24"` become date?
- should `""` mean null?
- should `"1,234.50"` be cleaned before cast?

So the durable pattern is:
```
raw_df   -> explicit all-string schema
typed_df -> select + clean + cast
```

**Example**
```python
from pyspark.sql import functions as F

typed_df = (
    raw_df
    .withColumn("revenue", F.when(F.col("revenue") == "", None).otherwise(F.col("revenue")))
    .withColumn("revenue", F.col("revenue").cast("double"))
)
```

## 2. Conflict resolution

This is nastier than type evolution.

Type evolution says:
> same meaning, messy representation

Conflict resolution says:
> same column name, possibly different meaning

Example:
```
"amount"
```
might mean:
- gross revenue in one report
- net amount in another
- quantity-adjusted amount somewhere else

This is not a Spark syntax problem. This is a **data contract problem**.

### The rule

Never assume:
> same header text = same semantic field

You need a resolution layer, for example:
- report type → canonical mapping
- source system → canonical mapping
- known rename dictionary
- forbidden collisions list

Good practical pattern

Have an external config like:
```python
COLUMN_CANONICAL_MAP = {
    ("sales_report", "amount"): "gross_amount",
    ("returns_report", "amount"): "return_amount",
}
```
Then your local schema extraction becomes:
```python
def canonicalize(local_cols, report_type):
    return [
        COLUMN_CANONICAL_MAP.get((report_type, col), col)
        for col in local_cols
    ]
```
That way:
- Spark handles structure
- **you handle meaning**

This is one of the biggest hidden ETL invariants.

## 3. Schema versioning

This is when your canonical schema changes over time.

Example:
- v1 had `customer`
- v2 splits into `customer_id` and `customer_name`
- v3 adds `source_file`
- v4 deprecates `legacy_code`

If you do not version this somehow, then old data and new data become hard to compare.

### Simple versioning approach

Keep:
- a persisted canonical column list
- a schema version id
- migration notes

Example:
```python
SCHEMA_V1 = ["a", "b", "c"]
SCHEMA_V2 = ["a", "b", "c", "d"]
```
And store with outputs:
- schema version
- ingestion run timestamp
- source/report type

**Why this matters**

Because “new column appeared” is not just a technical event. It is:
- sometimes acceptable evolution
- sometimes upstream change
- sometimes source corruption

Versioning lets you distinguish:
- intentional schema growth
- accidental drift

## 4. Spark-native alignment vs your normalization-first pattern

Spark’s `unionByName(..., allowMissingColumns=True)` is useful when you already have DataFrames with partially different schemas, 
because Spark aligns by name and fills missing fields with null.

But your case is deeper than that.

You are not just combining DataFrames. You are doing:
```
raw heterogeneous files
-> local header extraction
-> local row decoding
-> canonical field mapping
-> explicit global schema
-> DataFrame creation
```
That is why your manual normalization is still the better architecture.

Use Spark’s alignment features as a convenience later, not as the foundation of meaning.

## 5. Production-grade pipeline shape

This is the durable version of what you’re building:

**Phase A — discovery**
- scan files
- extract local headers
- canonicalize names
- merge into superset schema

**Phase B — contract build**
- define `StructType([StructField(..., StringType(), True), ...])`
- persist schema snapshot/version

**Phase C — normalization**
- each file yields rows
- each row is interpreted with local header order
- each row is written into canonical dict/Row

**Phase D — typed refinement**
- clean placeholders
- cast target columns intentionally
- add metadata fields like source/report/schema_version

**Phase E — validation**
- unknown columns?
- duplicate canonical names?
- row/header length mismatch?
- expected key columns missing?

That is a real ingestion engine.

## 6. The mental model that will help you most

There are three distinct layers:

**Layer 1: Structural schema**

Handled by Spark:
- `StructType`
- `StructField`
- nullable/type/order

**Layer 2: Canonical naming**

Handled by your logic:
- merge columns
- rename conflicting fields
- stabilize global schema

**Layer 3: Semantic typing**

Handled by downstream transforms:
- cast to numeric/date/bool
- define business meaning
- validate assumptions

When people mix these three layers together, systems become fragile.

When they separate them, systems become durable.

7. My recommendation for you

For your current stage, I would do this:
- **Raw canonical DataFrame**
    - every column `StringType()`
    - every field nullable
    - plus metadata columns like `source_file`, `report_type`, `schema_version`

- **Typed curated DataFrame**
    - cast only known stable columns
    - keep problematic columns as string until understood

- **Config-driven conflict resolution**
    - report-type-based canonicalization
    - explicit handling for ambiguous columns

That is the right level of engineering for where your system is.

8. Tiny example skeleton
```python
from pyspark.sql import Row
from pyspark.sql.types import StructType, StructField, StringType

def build_schema(columns):
    return StructType([
        StructField(col, StringType(), True)
        for col in columns
    ])

def normalize_row(global_columns, local_columns, values):
    if len(local_columns) != len(values):
        raise ValueError(
            f"Mismatch: {len(local_columns)} columns vs {len(values)} values"
        )

    record = dict.fromkeys(global_columns, None)

    for key, value in zip(local_columns, values):
        record[key] = value

    return Row(**record)
```
That is already a strong primitive.
