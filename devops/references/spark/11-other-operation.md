# Broadcasting Variables

**Concept**
- Broadcast objects are sent **once per executor**
- All tasks on that executor reuse it

**Usage**

```python
col_map_bc = sc.broadcast(col_map)

rdd2 = rdd.mapPartitions(
    lambda it: flatten_partition(it, col_map_bc.value)
)

col_map_bc.unpersist()
```


# `TaskContext`

**What it is**
- Runtime metadata for the **currently executing task**
- Available inside:
    - `mapPartitions`
    - `map`
    - `foreachPartition`

**Access**

```python
from pyspark import TaskContext

ctx = TaskContext.get()
```

- Returns `None` on driver
- Returns context inside executor task

**Useful fields**

```python
ctx.partitionId()       # partition index
ctx.attemptNumber()     # retry count
ctx.stageId()           # stage ID
```

Other
- `ctx.getLocalProperty(key)`
- `ctx.resources()` (GPU/resource scheduling)

**Typical usage**

```python
def process_partition(it):
    ctx = TaskContext.get()
    pid = ctx.partitionId()

    for row in it:
        yield row
```
