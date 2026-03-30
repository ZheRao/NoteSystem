# Advanced Read

Yes — there is a better way, and this is where **directory layout matters more than filename tricks**.

For Pandas, I would strongly recommend moving away from:
```
pl_2024.parquet
pl_2025.parquet
```
and instead using a **partition-like folder structure**, such as:
```
pl/
  fiscal_year=2024/
    data.parquet
  fiscal_year=2025/
    data.parquet
```
That gives you a much cleaner long-term contract.

## Why this is better

Because filenames like `pl_2025.parquet` force you to manage discovery logic manually:
- list files
- parse fiscal year from filename
- select matching files
- concatenate them

Whereas partition-style folders make the storage shape itself carry meaning.

That aligns much better with:
- Spark
- Parquet dataset readers
- future filtering
- engine-agnostic design

## In plain Pandas

Basic `pandas.read_parquet()` usually reads:
- one file, or
- sometimes a directory of parquet files, depending on backend

But Pandas itself is **not** as naturally “dataset-oriented” as Spark.

So if you want robust multi-file reads, the best approach is usually one of these two.

## Option 1: Manual file discovery + pd.concat()

Works anywhere, very explicit:
```python
from pathlib import Path
import pandas as pd

root = Path("pl")

files = sorted(root.glob("fiscal_year=*/data.parquet"))

df = pd.concat(
    [pd.read_parquet(f) for f in files],
    ignore_index=True
)
```
If you only want a subset:
```python
scope = {2024, 2025}

files = [
    root / f"fiscal_year={fy}" / "data.parquet"
    for fy in scope
    if (root / f"fiscal_year={fy}" / "data.parquet").exists()
]

df = pd.concat(
    [pd.read_parquet(f) for f in files],
    ignore_index=True
)
```
This is simple and dependable.

## Option 2: Use PyArrow dataset reading

This is the more scalable and more “Spark-like” approach.
```python
import pyarrow.dataset as ds
import pandas as pd

dataset = ds.dataset("pl", format="parquet", partitioning="hive")
table = dataset.to_table()
df = table.to_pandas()
```
If your folders are like:
```
pl/fiscal_year=2024/data.parquet
pl/fiscal_year=2025/data.parquet
```
then `partitioning="hive"` lets PyArrow infer `fiscal_year` from the folder name.

That is very nice, because the partition column can be reconstructed automatically even if it is not physically stored in each file.

You can also filter before loading:
```python
import pyarrow.dataset as ds

dataset = ds.dataset("pl", format="parquet", partitioning="hive")

table = dataset.to_table(
    filter=ds.field("fiscal_year").isin([2024, 2025])
)

df = table.to_pandas()
```
That is the closest thing to Spark-style partition pruning in this ecosystem.

## Which structure should you choose?

I would choose:
```
pl/
  fiscal_year=2024/
    data.parquet
  fiscal_year=2025/
    data.parquet
```
not:
```
pl/
  fiscal_year_2024/
    pl.parquet
```
and definitely not just:
```
pl_2024.parquet
pl_2025.parquet
```
## Why `fiscal_year=2024` is best

Because this is the standard **Hive-style partition layout**.

That means:
- Spark understands it naturally
- PyArrow understands it naturally
- many tools recognize it
- the storage contract is self-describing

So this:
```
fiscal_year=2024/
```
is much better than:
```
fiscal_year_2024/
```
because the latter is just a string in a folder name, while the former is a real partition convention.

## Practical recommendation

For now, I would do this:

**Writing**
```python
for fy, chunk in df.groupby("fiscal_year", sort=False):
    out = pl_path / f"fiscal_year={fy}"
    out.mkdir(parents=True, exist_ok=True)
    chunk.to_parquet(out / "data.parquet", index=False)
```

**Reading**
```python
from pathlib import Path
import pandas as pd

files = sorted((pl_path).glob("fiscal_year=*/data.parquet"))
df = pd.concat([pd.read_parquet(f) for f in files], ignore_index=True)
```
Then later, if you want more elegance and filtering efficiency, switch the read path to `pyarrow.dataset`.

## The deeper architectural point

You are not trying to make Pandas imitate Spark mechanically.

You are trying to make both engines obey the same storage invariant:
> data is persisted as a partitioned dataset, where fiscal year is the replaceable unit.

That is the real win.

**Lean in — core growth**: use Hive-style folders now. That choice will age much better than encoding partition meaning in filenames.