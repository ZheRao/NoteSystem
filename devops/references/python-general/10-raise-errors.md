# Python Errors — Quick Reference (Practical)

## 0) Core idea
- **Raise** an exception when the program is in a state you *do not want to continue from*.
- Use exceptions to:
  - enforce contracts (inputs, states, invariants)
  - stop bad data early
  - make failures explicit and traceable

## 1) `assert` vs `raise`

### `assert` (debug-time checks)
Use when:
- it’s a *developer invariant* (should never happen if code is correct)
- you want a fast sanity check during development

**Warning:** `assert` can be disabled with `python -O`, so never rely on it for production validation.

```python
assert df is not None
assert "so_id" in df.columns
```

### raise (runtime enforcement)
Use when:
- it’s user input / config / data quality
- you need it enforced in production always

```python
if dataset not in {"SO", "PO"}:
    raise ValueError(f"Unknown dataset: {dataset}")
```
Rule of thumb:
- **assert = programmer bug**
- **raise = expected bad inputs / data / environment**

## 2) Common exception types and when to use them

`ValueError`

Input has correct type but invalid value.
```python
if month not in range(1, 13):
    raise ValueError(f"Invalid month: {month}")
```

`TypeError`

Wrong type was provided.
```python
if not isinstance(today, dt.date):
    raise TypeError(f"today must be datetime.date, got {type(today)}")
```

`KeyError`

Missing key in dict-like objects (configs, mappings).
```python
if "raw_table" not in contract:
    raise KeyError("contract missing 'raw_table'")
```

`FileNotFoundError`

Required file is missing.
```python
if not path.exists():
    raise FileNotFoundError(f"Missing input file: {path}")
```

`PermissionError`

No permission to read/write.
(Usually raised automatically by OS, but you can re-raise with context.)

`RuntimeError`

Code is in a bad state, but no better exception fits.
```python
raise RuntimeError("Pipeline reached unreachable state")
```
`NotImplementedError`

Stub / planned feature.
```python
raise NotImplementedError("PO processing not implemented yet")
```

## 3) Custom exceptions (when you want clean control flow)

Create custom exceptions when:
- you want calling code to catch *your* category of failure
- you want to distinguish “data quality” from “system failure”

```python
class DataContractError(Exception):
    pass

class DataQualityError(Exception):
    pass
```

Usage:
```python
if missing_cols:
    raise DataContractError(f"Missing required columns: {missing_cols}")

if bad_cast_rate > 0.01:
    raise DataQualityError(f"Type coercion failure rate too high: {bad_cast_rate:.2%}")
```

## 4) Raising with context (good error messages)

A good error message answers:
- **what failed**
- **where** (dataset / file / stage)
- **expected vs got**
- **what to do next** (optional)

Example:
```python
raise ValueError(
    f"[PATH Silver] dataset={dataset} invalid. "
    f"Expected one of {{'SO','PO'}}, got {dataset!r}"
)
```

## 5) Wrapping errors (add context without losing traceback)

**Prefer** `raise ... from e`

This keeps the original exception chained.
```python
try:
    cfg = read_configs("contracts", "so.contract.json")
except Exception as e:
    raise RuntimeError("Failed to load SO contract config") from e
```

Rule:
- use `from e` when you’re adding meaning and keeping cause

## 6) Catching errors: what to catch and what NOT to catch

**Catch specific exceptions**
```python
try:
    df = pd.read_csv(path)
except FileNotFoundError as e:
    raise FileNotFoundError(f"Raw file missing: {path}") from e
```
**Avoid `except Exception`: unless you re-raise**

Catching everything can hide real bugs unless you re-raise with context.

## 7) When to return vs when to raise

Raise when:
- continuing would produce wrong outputs (silent corruption)
- caller must handle it explicitly

Return sentinel (`None`, empty df) only when:
- “no data” is a valid expected situation
- caller can naturally handle it

Example:
- **Silver transform**: missing columns → **raise** (contract broken)
- “no new raw files today” → maybe **return None** or raise a clean “NoInputData” exception depending on design

## 8) Suggested error taxonomy for your PATH pipeline

**Contract errors (shape/schema)**
- missing required columns
- unexpected columns (optional policy)
- rename map refers to missing columns

Use: `DataContractError`

**Data quality errors (content)**
- type coercion failure rate > threshold
- null rate > threshold for required fields
- duplicate primary keys

Use: `DataQualityError`

**System errors (environment)**
- file missing / permission
- config missing
- unexpected runtime failure

Use: built-ins + `RuntimeError`

## 9) Tiny templates you can copy/paste
**Validate enum input**
```python
allowed = {"SO", "PO"}
dataset = dataset.upper()
if dataset not in allowed:
    raise ValueError(f"dataset must be one of {sorted(allowed)}, got {dataset!r}")
```

**Validate required columns**
```python
required = {"SOID", "CropName"}
missing = required - set(df.columns)
if missing:
    raise DataContractError(f"Missing required columns: {sorted(missing)}")
```

**Threshold-based quality gate**
```python
bad_rate = bad_casts / max(total, 1)
if bad_rate > 0.01:
    raise DataQualityError(f"Type coercion bad rate {bad_rate:.2%} > 1%")
```

**Wrap with context**
```python
try:
    atomic_write_csv(dst=out_path, df=df)
except Exception as e:
    raise RuntimeError(f"Failed writing silver file: {out_path}") from e
```

## Quick decision table

| Situation                                   | Use                                       |
| ------------------------------------------- | ----------------------------------------- |
| Developer invariant (“should never happen”) | `assert`                                  |
| User/config/data is invalid                 | `raise ValueError / TypeError / KeyError` |
| Missing file                                | `FileNotFoundError`                       |
| Contract/schema violated                    | `DataContractError` (custom)              |
| Data fails quality thresholds               | `DataQualityError` (custom)               |
| Adding context to lower-level error         | `raise ... from e`                        |
