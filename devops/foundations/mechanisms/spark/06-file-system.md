
# Understand How Spark Handles Files With `Overwrite` Option

Key question:
- If I overwrite `fiscal_year=2026,` does Spark really overwrite it even though the new parquet file name is different?

## The core answer

Yes — **if Spark is overwriting that partition**, it does not need to reuse the same `part-0000...parquet` filename.

Because Spark’s overwrite logic is not:
> “replace file with same name”

It is more like:
> “replace the dataset content in the target scope”

So the important unit is **not the parquet filename**, but the **target path / partition scope**.

## What happens conceptually

Suppose you currently have:
```
spark/
  fiscal_year=2026/
    part-00000-oldid.snappy.parquet
```
Then you write new 2026 data.

Spark may produce:
```
spark/
  fiscal_year=2026/
    part-00000-newid.snappy.parquet
```
The filename changed, but that is okay, because Spark can delete old contents and commit new contents.

The overwrite contract is about **final visible contents**, not filename continuity.

## Important distinction: overwrite can behave in two different scopes

This is where it gets subtle.

### Case A — overwrite the whole table path

If you do:
```python
df_2026.write \
    .format("parquet") \
    .mode("overwrite") \
    .partitionBy("fiscal_year") \
    .save(path)
```
then depending on Spark settings, this may mean:
- overwrite the entire target dataset
- not just `fiscal_year=2026`

That means other partitions like 2024 and 2025 might be removed too.

So **plain overwrite is dangerous** if your intent is “replace only one or a few partitions.”

### Case B — dynamic partition overwrite

If configured properly, Spark can **overwrite only the partitions present in the incoming DataFrame**.

That is usually what you want for your use case.

For example, if the incoming DataFrame only contains:
```
fiscal_year = 2026
```
then Spark overwrites only:
```
fiscal_year=2026/
```
and leaves other partitions untouched.

## The config that matters

For partitioned overwrite, the important Spark config is:
```python
spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")
```
Then a write like:
```python
df_2026.write \
    .format("parquet") \
    .mode("overwrite") \
    .partitionBy("fiscal_year") \
    .save(path)
```
will overwrite only the partitions that appear in `df_2026`.

Without dynamic mode, overwrite is often effectively **table-level overwrite**.

## So what actually gets deleted?

If dynamic partition overwrite is active, and your input contains only 2026, Spark effectively treats it like:
- remove old files under `fiscal_year=2026/`
- write new files under `fiscal_year=2026/`
- leave `fiscal_year=2024/`, `fiscal_year=2025/`, etc. alone

It **does not care** that the new filename differs from the old one.

That is normal.

## Mental model

In normal Python/local filesystem thinking:
- overwrite often means “same path, same filename gets replaced”

In Spark dataset thinking:
- overwrite means “replace the contents of the target dataset scope”

That scope may be:
- the whole table path
- specific partitions, if dynamic partition overwrite is enabled

Practical recommendation for your case

If you want to refresh only selected fiscal years, do this:
```python
spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

(
    df_filtered.write
      .format("parquet")
      .mode("overwrite")
      .partitionBy("fiscal_year")
      .save(str(pl_path / "spark"))
)
```
where df_filtered contains only the fiscal years you want to replace.

Example:
```python
df_filtered = df2.filter("fiscal_year = 2026")
```
or multiple years:
```python
df_filtered = df2.filter("fiscal_year in (2024, 2025, 2026)")
df_filtered = df2.filter(F.col("fiscal_year").isin(scope))
```
Then only those partitions are replaced.

## Very blunt warning

If you do partitioned overwrite work often, and you do **not** explicitly control overwrite mode, you are one bad run away from wiping the entire table.

That is not theoretical. That is one of the classic Spark footguns.