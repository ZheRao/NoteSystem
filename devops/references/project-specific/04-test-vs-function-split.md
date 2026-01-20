# Test Coverage and Function Splitting

**The goal of tests is not coverage.**

It’s **confidence per hour**.

So no — you should **not** try to write unit tests for every helper function *just because it exists*.

## What to test (PATH silver edition)

### 1) Test the “business invariants” first (highest ROI)

These are the things that, if broken, silently corrupt downstream truth:

**Core invariants for `process_raw_table` pipeline**

- **Column contract applied correctly**
    - excluded columns removed
    - renames applied
    - final columns match expected set/order (if you care)

- **Path partitioning is correct**
    - given `today`, output goes to `silver_root/YYYY/MM/DD/<dataset>.csv`

- **Output is a faithful write**
    - what you wrote is what you intended (read it back and compare)

These are *pipeline tests* (integration-ish), but still cheap if you use a temp dir.

---
### 2) Unit test “pure logic” helpers (cheap + stable)

If a helper is deterministic and has no I/O, it’s basically free to test.

Examples:
- `_resolve_silver_path(...)` ✅ perfect unit test candidate
- any “transform df using contract dict” function ✅

---
### 3) Don’t unit-test I/O wrappers deeply (unless they’re risky)

Stuff like `atomic_write_csv`, `read_data`, config loaders:

- You **do** want one or two tests that verify the contract:
    - atomic write actually replaces atomically (or at least “write then file exists and content correct”)
    - `read_data` respects prefix+date logic

But you don’t need 20 tests for every edge case *yet*.

**Rule**: test I/O wrappers only at the “public promise” level.

## Your “where does it end?” question (the real one)

Function splitting ends when you hit one of these boundaries:

### Boundary A — “Can I test the logic without touching disk/network?”

If you can’t, splitting helps.

Right now `process_raw_table` is hard to unit test because it:
- reads configs
- reads raw csv
- transforms
- writes

Even if helpers exist, they’re still *wired together* inside one function.

---
### Boundary B — “Does this sub-step have a name and an invariant?”

If yes, it deserves a function (often).  
Example: **standardize columns using contract** is absolutely a named invariant.

---
### Boundary C — “Would I ever want to reuse this in another pipeline?”

If yes, split it.

---
### Boundary D — “Is the function mostly orchestration?”

Orchestration functions are allowed to be “long-ish”. Their job is glue.

So you don’t need infinite splitting. You just want:
- **one orchestration function** that calls
- **a few testable building blocks**.

## The crystalized testing strategy (what I’d do in your repo)

### Tier 0 — Smoke test (1 test, huge value)

A temp directory + a tiny raw csv + contract json (or dict) + config stub.

Assert:
- output file exists at expected partition path
- output df columns match expected
- maybe row count unchanged

This catches 80% of “oops” bugs.

### Tier 1 — Pure unit tests (fast)
- `_resolve_silver_path`
- `apply_contract(df, contract)` (you should extract this)
    - drop + rename results

### Tier 2 — I/O contract tests (minimal)
- `atomic_write_csv` writes and replaces correctly (1 test)
- `read_data` selects correct file given prefix+date (1–2 tests)

That’s it. Stop.

## A very practical compromise: split *only one* thing now

Keep `process_raw_table` as orchestration, but extract exactly one pure function:

```python
def standardize_raw_table(df: pd.DataFrame, contract: dict) -> pd.DataFrame:
    df = df.drop(columns=contract["raw_table"]["excludes"])
    df = df.rename(columns=contract["raw_table"]["column_rename"])
    return df
```
Why this one?
- It’s the **most important invariant**
- It becomes a **beautiful unit test**
- It keeps `process_raw_table` readable

Everything else can stay as-is for now.

## The mental model to reuse forever
- **Orchestration code**: test via 1–2 pipeline smoke tests.
- **Pure logic**: unit test heavily (cheap, stable).
- **I/O helpers**: test only their promises, lightly.