# Atomic Write with DataFrames — Pandas & Spark Bytes + Safer Patterns

This document expands the **atomic_write_bytes(dst, data: bytes)** pattern to cases where your “data” is a **DataFrame** (Pandas or Spark).

Key idea:

* Atomic write needs **bytes** (or a temp file path) so it can safely write a `.tmp` and then do an **atomic replace**.
* A DataFrame isn’t bytes — so you must choose a **serialization format**.

## Casution: Scaling Note — Bytes Can Double Your Memory

All “DataFrame → bytes” methods above hold **two big objects** at once:

1. the in-memory DataFrame
2. the in-memory serialized bytes

For large datasets, this can be a killer.

### Safer pattern for very large outputs: atomic write from a temp *file path*

Instead of returning bytes, write directly to a temp file path, then `atomic_replace`.

Example (CSV path-based):

```python
def atomic_write_csv(dst: Path, df: pd.DataFrame, **to_csv_kwargs) -> None:
    ensure_dir(dst.parent)
    tmp = dst.with_name(f".{dst.name}.{uuid.uuid4().hex}.tmp")

    # Write directly to tmp path (no in-memory bytes buffer)
    df.to_csv(tmp, index=False, **to_csv_kwargs)

    # Optional durability: fsync the tmp file
    with open(tmp, "rb") as f:
        os.fsync(f.fileno())

    atomic_replace(tmp, dst)
```

Example (Excel-based):
```python
import os
import uuid
from pathlib import Path
import pandas as pd

def atomic_write_xlsx(
    dst: Path,
    df: pd.DataFrame,
    sheet_name: str = "data",
    engine: str = "openpyxl",
    **to_excel_kwargs,
) -> None:
    """
    Atomically write df to an .xlsx file:
    - writes to a temp file in the same directory
    - fsyncs the temp file
    - atomically replaces dst

    Notes:
    - Keep nulls as blank cells by NOT using na_rep unless you truly want sentinel text.
    - On Windows, replacement can fail if dst is currently open/locked by Power BI/Excel.
    """
    ensure_dir(dst.parent)
    tmp = dst.with_name(f".{dst.name}.{uuid.uuid4().hex}.tmp")

    # Ensure suffix is .xlsx (optional guard)
    if tmp.suffix.lower() != ".xlsx":
        tmp = tmp.with_suffix(".xlsx")

    # Write to temp path
    with pd.ExcelWriter(tmp, engine=engine) as writer:
        df.to_excel(writer, index=False, sheet_name=sheet_name, **to_excel_kwargs)

    # Optional durability: fsync tmp file
    with open(tmp, "rb") as f:
        os.fsync(f.fileno())

    atomic_replace(tmp, dst)
```

This avoids buffering the entire file in memory.


## Pandas DataFrame → bytes

### Option 1 — CSV bytes (StringIO → encode)

```python

def df_to_csv_bytes(df: pd.DataFrame, encoding: str = "utf-8", **to_csv_kwargs) -> bytes:
    buf = io.StringIO()
    df.to_csv(buf, index=False, **to_csv_kwargs)
    return buf.getvalue().encode(encoding)

atomic_write_bytes(Path("out.csv"), df_to_csv_bytes(df, encoding="utf-8"))
```

**Pros**

* Human-readable
* Diff-friendly
* Good interoperability

**Cons / gotchas**

* **Large memory spike**: entire CSV string exists in memory.
* **Types are lossy**: everything becomes text; re-reading requires parsing.
* Encoding matters:
  * `utf-8` is standard.
  * `utf-8-sig` adds a BOM and helps some Excel setups.
* Newlines:
  * Consider `line_terminator="\n"` (or `\r\n` if required).

**When to use**

* Small/medium exports, ad-hoc data exchange, external stakeholders.

---

### Option 2 — Parquet bytes (BytesIO)

```python

def df_to_parquet_bytes(df: pd.DataFrame, **to_parquet_kwargs) -> bytes:
    out = io.BytesIO()
    df.to_parquet(out, index=False, **to_parquet_kwargs)
    return out.getvalue()

atomic_write_bytes(Path("out.parquet"), df_to_parquet_bytes(df))
```

**Pros**

* Compact and fast
* Preserves schema and types better than CSV
* Excellent for downstream analytics

**Cons / gotchas**

* Still a **full in-memory buffer**; can be heavy for big frames.
* Requires a parquet engine (pyarrow or fastparquet).
* Parquet is not human-readable.

**When to use**

* Analytics pipelines, data lakes, internal storage, large datasets.

---

### Option 3 — Pickle bytes (pickle.dumps)

```python
import pickle

data = pickle.dumps(df, protocol=pickle.HIGHEST_PROTOCOL)
atomic_write_bytes(Path("out.pkl"), data)
```

**Pros**

* Extremely easy
* Preserves many Python-specific details

**Cons / gotchas (important)**

* **Security risk**: loading pickle from untrusted sources is unsafe.
* **Portability risk**: tied to Python/Pandas versions; can break across environments.
* Not a stable interchange format.

**When to use**

* Short-lived caches in a controlled environment.

**Avoid when**

* Anything long-term, shared, or cross-environment.

---

### Option 4 — XLSX bytes (ExcelWriter → BytesIO)

```python

def df_to_xlsx_bytes(df: pd.DataFrame, sheet_name: str = "data", **to_excel_kwargs) -> bytes:
    out = io.BytesIO()
    with pd.ExcelWriter(out, engine="openpyxl") as writer:
        df.to_excel(writer, index=False, sheet_name=sheet_name, **to_excel_kwargs)
    return out.getvalue()

atomic_write_bytes(Path("out.xlsx"), df_to_xlsx_bytes(df, sheet_name="data"))
```

**Pros**

* Stakeholder-friendly (Excel)
* Can include multiple sheets, formatting, etc.

**Cons / gotchas**

* Slow and heavy for large data
* Excel row limit (~1,048,576)
* Memory overhead can be significant

**When to use**

* Reporting outputs, finance/audit workflows, business users.

## A Practical Format Decision Rule

* **CSV**: interoperability + readability; accept type loss.
* **Parquet**: internal pipelines; best default for typed data.
* **XLSX**: business/reporting deliverable.
* **Pickle**: cache only; controlled environment; short-lived.


## Spark DataFrame: Important Difference

Spark typically writes **directories**, not single files.

### Why this matters

`df.write.parquet("out.parquet")` produces a folder like:

* `out.parquet/part-00000-...snappy.parquet`
* `_SUCCESS`
* metadata

So you’re not atomically replacing **a single file** — you’re creating **a directory tree**.

## Atomic Write Pattern for Spark (Directory Output)

### Pattern: write to a temp directory, then atomically rename the directory

Conceptually:

1. write to `out.parquet.__tmp__<uuid>/`
2. once Spark finishes successfully, rename temp dir → final dir

**Note:** Directory rename is typically atomic on the same filesystem (same mount), similar to file rename.

Pseudo-structure:

```python

def atomic_write_spark_parquet(df, dst_dir: str, mode: str = "overwrite", **write_kwargs):
    # 1) choose a temp directory adjacent to destination
    tmp_dir = f"{dst_dir}.__tmp__.{uuid.uuid4().hex}"

    # 2) write output directory
    df.write.mode("overwrite").parquet(tmp_dir, **write_kwargs)

    # 3) atomically swap directories (filesystem rename)
    #    - remove old dst_dir if needed
    #    - rename tmp_dir -> dst_dir
    # Implementation depends on environment:
    # - local/HDFS: use filesystem rename
    # - cloud object store: rename may be copy+delete (NOT atomic)
```

### Key environment caveat

* On **HDFS/local POSIX filesystem**, rename is usually atomic.
* On **S3/ADLS/GCS object stores**, “rename” is often implemented as copy+delete, which is **not atomic**.

If you’re on object storage, you typically:

* write to a versioned prefix (run_id)
* then update a small “latest” pointer/manifest

## Spark Single-File Outputs (Use With Care)

Spark can produce a single file by:

* `df.coalesce(1).write...`

But:

* this can bottleneck to one partition
* might be slow for large data

If you truly need one file (e.g., CSV delivery), you can:

* write to temp dir
* find the single `part-*` file
* rename/copy it to a single final filename atomically (filesystem-dependent)

## Recommended Defaults (Pragmatic)

### Pandas

* Default internal storage: **Parquet**
* Default external deliverable: **CSV** or **XLSX**
* Avoid pickle except for caches

### Spark

* Default: write **dataset directories** (Parquet/Delta)
* Implement atomicity as:

  * **temp dir → rename dir** (HDFS/local)
  * **versioned paths + manifest/pointer** (object stores)

## Optional: Freshness / Provenance Metadata

To avoid “dst exists but is stale”, add one of:

1. Embed metadata columns (`extracted_at`, `run_id`, `source_snapshot`)
2. Sidecar manifest file (atomic write) containing:

   * run_id
   * timestamp
   * rowcount
   * schema hash
   * checksum of file

This preserves the atomic integrity guarantee and adds correctness signals.

## Summary

* Atomic write is a **transport guarantee** (integrity), not a business guarantee (freshness).
* Pandas → bytes is convenient but can be memory-heavy.
* For large outputs, prefer path-based temp-file writing.
* Spark writes directories; atomicity must be handled at the directory/prefix level.
* On cloud object stores, use versioned paths + manifests instead of assuming rename is atomic.
