# Schema Enforcement Playbook (Silver Boundary)
*A reusable guideline for pandas today and Spark tomorrow.*

## Philosophy
- **Silver is a contract boundary.** Silver’s job is to take messy raw extracts and produce a **standardized, machine-trustworthy** dataset.
- **Schema ≠ config.** Config is runtime (paths, overwrite, env). Schema is a **versioned data contract** (columns/types/nullability/constraints).
- **Fail fast at the boundary.** If the contract is broken, stop early and loudly. Don’t let garbage leak into Gold.
- **Keep Silver boring.** No business logic. Only normalization, coercion, validation, and safe writes.

## Recommended Repo Placement
- `json_configs/contracts/<table>.contract.json` → **source-of-truth contract** (engine-agnostic)
- `json_configs/contracts/<table>.mapping.json` → **raw header → canonical name mapping**
- `src/.../common/contract_loader.py` → load+parse contract JSON into dataclasses
- `src/.../common/enforce.py` → enforce contract on pandas df + compile to Spark schema later
- `src/.../silver/<table>.py` → table-specific pipeline: normalize → map → enforce → write

## Contract Format (JSON)
### Minimum fields (engine-agnostic)
A good contract specifies:
- `required_columns` (must exist)
- `columns[]` with `{ name, type, nullable }`
- coercion failure thresholds: `max_bad_fraction`, `max_bad_count`
- policy: `allow_extra_columns`

Example skeleton:
```json
{
  "table": "sales_order",
  "version": 1,
  "allow_extra_columns": false,
  "required_columns": ["so_id", "order_date"],
  "columns": [
    { "name": "so_id", "type": "string", "nullable": false },
    { "name": "order_date", "type": "date", "nullable": false }
  ],
  "coercion": { "max_bad_fraction": 0.001, "max_bad_count": 10 }
}
```

Mapping file skeleton:
```json
{
  "table": "sales_order",
  "mapping": {
    "SOID": "so_id",
    "Order Date": "order_date"
  }
}
```

## Enforcement Pipeline (Raw → Silver)

Apply in this exact order:

### Step 0 — Define Canonical Names
- Canonical convention: `lower_snake_case`
- Schema is defined ONLY in canonical names.

### Step 1 — Read Raw (I/O boundary)
- Read CSV with deterministic options (encoding, delimiter, NA policy).
- **No column renames, no business logic**.

### Step 2 — Normalize + Map Columns
- Normalize raw headers (trim/strip, lower, replace spaces → `_`).
- Apply mapping (raw variants → canonical).
- Outcome: stable column set in canonical naming.

### Step 3 — Presence Checks (Missing columns)

**Rule**: missing required columns → error.

Checklist:
- Verify all `required_columns` exist.
- Decide how to handle extras:
    - `allow_extra_columns=false` → error (strict)
    - `allow_extra_columns=true` → warn + keep (practical early on)

### Step 4 — Type Coercion (Deterministic)

For each column in contract:
- Coerce to target type (string/int/float/decimal/date/timestamp/bool).
- Count coercion failures.

**Important definition**:

A “coercion failure” is a value that was **non-null before** coercion and becomes **null after** coercion.

### Step 5 — Coercion Failure Thresholds

**Rule**: type coercion failures above threshold → error.

Use both:
- `max_bad_count`
- `max_bad_fraction`

Fail if either triggers:
- `bad_count > max_bad_count` OR `bad_count / total_rows > max_bad_fraction`

This prevents silent “half your column became null” disasters.

### Step 6 — Nullability Checks

For every `nullable=false` column:
- If any null remains → error (or allow tiny tolerance if reality demands).

### Step 7 — Write Silver (Atomic, Idempotent)
- Partition path (e.g., `silver/<dataset>/YYYY/MM/DD/<file>.csv`)
- Write atomically: temp → rename
- Support `overwrite=True/False`
- Return a structured `IngestResult` (rows written, path, warnings)

```python
from __future__ import annotations
from dataclasses import dataclass
from pathlib import Path
from typing import List

@dataclass
class IngestResult:
    dataset: str
    raw_files: List[Path]
    written_files: List[Path]
    rows_total: int
    notes: List[str]
```

## What Silver Enforces vs Gold Enforces

### Silver enforces (mechanical truth)
- column existence
- types + parseability
- nullability
- “safe to compute on”
- reproducible, idempotent writes

### Gold enforces (semantic truth)
- business filters (cancelled/voided)
- classifications and mappings (customer type, product group)
- KPI definitions
- reconciliation rules

**Litmus test**:  
If Finance could argue about it, it belongs in **Gold**, not Silver.

## Logging + Failure Hygiene

Always report:
- which raw file(s)
- row count
- which columns failed coercion and how many
- where output was written

Error messages should include:
- missing columns list
- coercion failure counts and thresholds
- a hint to inspect raw values (top offenders)

## Practical Defaults (Good starting policy)
- `allow_extra_columns = true` for early exploration, then flip to `false` once stable.
- IDs default to `string` (avoid leading-zero bugs).
- Dates parsed with `errors="coerce"` + strict thresholds.
- `max_bad_fraction = 0.001` (0.1%) and `max_bad_count = 10` as initial guardrails.
- Non-nullable columns: default strict (`0` tolerated).

## “One Function” Contract for Every Table

Every Silver table should expose exactly one orchestration-friendly entrypoint:
- `standardize_<table>_to_silver(raw_df, contract, mapping, paths, ...) -> IngestResult`

Everything else stays internal helpers.

## Future-Proofing for Spark (Design Now)

If you keep contracts in JSON and canonical column names:
- pandas enforcement = `apply_contract_pandas(df, contract)`
- Spark enforcement later = `compile_to_structtype(contract)` + `spark.read.schema(...)`  
    Same contract, different engine.

## Checklist: Done Means Done
- contract JSON exists + versioned
- mapping JSON exists (raw variants covered)
- normalize → map → validate required columns
- coerce types + compute coercion failures
- enforce thresholds
- enforce nullability
- atomic write + idempotent overwrite behavior
- returns structured result metadata