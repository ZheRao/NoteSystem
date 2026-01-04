# Higher-order Functions (arrays/maps)

These keep us in "pure column land" (Catalyst-optimizable) instead of Python UDFs

- `F.transform(arr, lambda x: ...)`
- `F.filter(arr, lambda x: ...)`
- `F.aggregate(arr, zero, lambda acc, x: ...)`
- `F.exists(arr, lambda x: ...)`
- `F.forall(arr, lambda x: ...)`
- `F.reduce(arr, lambda acc, x: ...)` *(availability varies; sometimes `aggregate` is the workhorse)
