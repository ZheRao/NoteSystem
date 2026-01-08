# String Functions

## 1. Common Functions

**Case**
- `F.lower`, `F.upper`, `F.initcap`

**Trim**
- `F.trim`, `F.ltrim`, `rtrim`

**Length/substring**
- `F.length`
- `F.substr(col, pos, len)` *(1-indexed pos in Spark SQL semantics)*

**Replacement**
- `F.regexp_replace`
- `F.translate`

**Match/extract**
- `F.regexp_extract`
- `like`, `rlike`

**Concat**
- `F.concat`, `F.concat_ws`

**Pad**
- `F.lpad`, `F.rpad`
