# Column Domain-Specific Language

## 1. Common Column Expressions

**1.1 Build Columns**

- `F.col("x")`
- `F.lit(123)`
- `F.expr("sql_expression")`


**1.2 Operators**

- Arithmetic
    - `+ - * / %`

- Comparison
    - `== != > < >= <=`

- Boolean
    - `& | ~` (and/or/not)
    - Always use parentheses: `(a > 3) & (b < 3)`

**1.3 Advanced utilities**

- Basics
    - `F.abs`, `F.round`, `F.bround`, `F.floor`, `F.ceil`, `F.rint`

- Exp/log
    - `F.exp`, `F.expm1`, `F.log`, `F.log10`, `F.log1p`

- Trig
    - `F.sin`, `F.cos`, `F.tan`, `F.asin`

- Bit
    - `F.shiftleft`, `F.shiftright`
    - `F.bit_and`, `F.bit_or`, `F.bit_xor`

- Utility
    - `F.greatest`, `F.least`
    - `F.rand`, `F.randn`
    - `F.format_number`
    - `F.monotonically_increasing.id()`


## 2. Conditionals & Null Logic

**CASE WHEN**

- `F.when(cond, v1).otherwise(v2)`

**Null helpers**

- `F.coalesce(c1, c2, ...)` (first non-null)
- `F.ifnull(a, b)` *(~Spark 3.5; availability vary by build)*
- `F.nullif(a, b)` (null if equal)


## 3. Filters

- `col.isNull()`, `col.isNotNull()`
- `col.between(a, b)`
- `col.isin(list_or_args)`
- `col.like("pat%")`, `col.rlike("regex")`


