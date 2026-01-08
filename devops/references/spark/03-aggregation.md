# Aggregation Functions

## 1. Core

- `F.count`, `F.countDistinct`
- `F.sum`, `F.avg`, `F.min`, `F.max`
- `F.first`, `F.last`


## 2. Approx/Quantiles

- `F.approx_count_distinct`
- `F.approx_percentile(col, p)`


## 3. Stats

- `F.stddev`, `F.stddev_pop`, `F.stddev_samp`
- `F.var_pop`, `F.var_samp`
- `F.corr`
- `F.covar_pop`, `F.covar_samp`


## 4. Advanced Grouping

- SQL `GROUPING SETS`, `ROLLUP`, `CUBE`
- DataFrame-side: `rollup/cube` + `agg`, or do it via SQL for full control
