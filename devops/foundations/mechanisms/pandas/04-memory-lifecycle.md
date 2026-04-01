# Memory Lifecycle for `filter -> merge -> concat -> groupby`

Here is the right mental model.

## Memory lifecycle map for a Pandas pipeline

Suppose your pipeline is:
```python
df1 = df[df["type"] == "split"].copy()
df2 = df1.merge(perc_map, on=["Location", "inventory_year_month_key"], how="left")
df3 = pd.concat([df_offset, df2], ignore_index=True)
df4 = df3.groupby(["Location", "inventory_year_month_key"], as_index=False).agg(...)
```
The key idea:

**Pandas usually materializes a new object at each major reshaping step**.

So memory usage is not one steady line. It rises and falls in waves.

### Stage 0: original input

You start with:
- `df`
- `perc_map`

Memory roughly holds:
- full `df`
- full `perc_map`

### Stage 1: filter / copy
```python
df1 = df[df["type"] == "split"].copy()
```
Now memory holds:
- `df`
- `perc_map`
- `df1`

If `df1` is much smaller than `df`, fine.  
If it is almost same size as `df`, then you are already paying a lot.

Important:

If you do `.copy()`, assume new memory is allocated for that subset.


### Stage 2: merge
```python
df2 = df1.merge(perc_map, ...)
```
This is one of the big spikes.

During this step, memory roughly holds:

- `df`
- `perc_map`
- `df1`
- temporary merge machinery
- `df2` result

So merge peak is often:

**old left + right + new output + temp helpers**

This is why merge feels expensive.

Afterward, if `df1` is dead, you should let it go:
```python
del df1
```
Then memory drops to:
- `df`
- `perc_map`
- `df2`

### Stage 3: concat
```python
df3 = pd.concat([df_offset, df2], ignore_index=True)
```
Another spike.

During concat, memory roughly holds:
- `df`
- `perc_map`
- `df_offset`
- `df2`
- `df3`

So concat peak is often:

**all inputs + new combined output**

Afterward, if `df_offset` and `df2` are dead:
```python
del df_offset, df2
```
Memory drops to:
- `df`
- `perc_map`
- `df3`

### Stage 4: groupby + agg
```python
df4 = df3.groupby(...).agg(...)
```
This can also be a big spike, depending on cardinality and aggregation complexity.

During this step, memory roughly holds:
- `df`
- `perc_map`
- `df3`
- groupby internals
- `df4`

Afterward:
```py
del df3
```
Memory drops to:
- `df`
- `perc_map`
- `df4`

Visual intuition
```
Think of memory like this:

start:         df + perc_map

filter:        df + perc_map + df1

merge peak:    df + perc_map + df1 + temp + df2
after merge:   df + perc_map + df2

concat peak:   df + perc_map + df_offset + df2 + df3
after concat:  df + perc_map + df3

groupby peak:  df + perc_map + df3 + temp + df4
after groupby: df + perc_map + df4
```
So the peaks happen at the transitions.

The most important distinction

## There are two different memory problems:

### 1. Peak memory

This is the maximum memory needed while an operation is running.

Affected most by:
- merge
- concat
- groupby
- sort
- duplicate big intermediates

### 2. Residual memory

This is memory still occupied after the step is done.

Affected by:
- keeping old variables alive
- unnecessary inspection frames
- long functions with many intermediates
- not deleting dead objects in large pipelines

A lot of confusion comes from mixing these two.

## What `del` actually helps with

`del` helps with **residual memory**, not the spike during the operation.

Example:
```python
df2 = df1.merge(...)
del df1
```
This does **not** make merge cheaper.  
It just means once merge is done, old `df1` can disappear sooner.

So:
- `del` = lifecycle cleanup
- algorithm/schema changes = real peak-memory optimization

## Best production pattern

A good pattern is:
```py
df_split = df_split.merge(
    perc_map[["Location", "inventory_year_month_key", "pct"]],
    on=["Location", "inventory_year_month_key"],
    how="left",
)

df = pd.concat([df_offset, df_split], ignore_index=True)

del df_offset, df_split
```
Why this is good:
- overwrite when old version is dead
- only bring needed columns into merge
- delete dead intermediates after concat

## Highest-yield memory tactics, ranked

This is the part that matters most.

### 1. Reduce participating columns

Huge win.

Instead of:
```python
df_split.merge(perc_map, ...)
```
do:
```python
df_split.merge(
    perc_map[["Location", "inventory_year_month_key", "pct"]],
    ...
)
```
Do not carry unused columns through expensive operations.

### 2. Reduce participating rows before merge/concat

Also huge.

If only some rows need enrichment, do not merge all rows.

### 3. Rebind variables when old version is dead

Good hygiene.
```python
df_split = df_split.merge(...)
```
instead of keeping `df_split2`, `df_split3`, etc. in production.

### 4. Delete dead intermediates after major materialization steps

Helpful in large pipelines.
```python
del df_offset, df_split
```
after concat if they are truly dead.

### 5. Use smaller dtypes

Very real win.

Examples:
- `category` for repeated strings when appropriate
- `int32` instead of `int64` if safe
- `float32` instead of `float64` if precision allows

Be careful, but this matters.

### 6. Prefer `map` over `merge` when the relationship is truly simpler

Sometimes much cheaper.

If you are just adding one value column from a unique key mapping, `map` can be lighter than a full merge.

## A clean lifecycle style

This is a good style for production pipelines:
```python
df_split = df.loc[mask_split].copy()
df_offset = df.loc[mask_offset].copy()

df_split = df_split.merge(
    perc_map[["Location", "inventory_year_month_key", "pct"]],
    on=["Location", "inventory_year_month_key"],
    how="left",
)

df = pd.concat([df_offset, df_split], ignore_index=True)
del df_offset, df_split
```
This style is good because:
- names reflect current truth
- dead objects are not kept around
- schema entering merge is controlled
- lifecycle is easy to reason about

What not to obsess over

Do not over-focus on:
- whether rebinding saves everything
- whether `del` instantly returns memory to OS
- tiny naming differences

Those are secondary.

The big architectural questions are:
- How many rows are entering this operation?
- How many columns are entering this operation?
- Is this materializing a whole new table?
- Am I keeping old intermediates alive longer than needed?

That is the compounding instinct.

## Blunt summary

Use this mental model:
- `merge` and `concat` create new materialized DataFrames
- peak memory happens while old inputs and new output coexist
- `del` only helps after the operation
- overwriting variable names is good hygiene, not magic
- the biggest wins come from shrinking rows, columns, and unnecessary intermediates

A very good engineering question to ask yourself is:

**where is the first place in the pipeline that I can make the table smaller before the next materializing step?**

That question compounds far beyond Pandas.