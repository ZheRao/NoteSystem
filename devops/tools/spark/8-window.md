# Window Functions

## 1. Define a window

```python
w = Window.partitionBy("key").orderBy("timestamp")
```

## 2. Ranking / navigation

- `F.row_number`, `F.rank`, `F.dense_rank`, `F.ntile`
- `F.lag`, `F.lead`
- `F.first`, `F.last`, `F.nth_value`

## 3. Aggregation over frames

- `F.sum`, `F.avg`, `F.min`, `F.max`, `F.count` over:
    - `w.rowsBetween(...)`
    - `w.rangeBetween(...)`

Use cases: time-series, previous-value logic, cumulative metrics, QC
