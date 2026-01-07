# Publish Gold Tables from WSL to Windows before Azure 

**Problem at hand**:
- Systems developed in Linux
- No connector available to Power BI
- Gold tables have to be syned to SharePoint so Power BI could grab them
- Files must be transferred from Linux to Windows for SharePoint syncing

---

key idea: **seperate compute from publish**
- **Pipeline (WSL, Spark)**: writes `gold/<table_name>/...` as Parquet in WSL filesystem (fast, reliable)
- **Publisher (Pandas)**: reads Parquet → writes Excel to a Windows/SharePoint folder using **temp → rename**
- **Bash**: kicks publisher(s) concurrently (not "move files" necessarily — better to publish directly to final dir atomically)

**Why I'd avoid "async move" of finished `.xlsx`**

Moving can still collide with sync/locks and you still need atomicity. If you instead have the publisher write 
**temp in the final folder** and rename, you get the atomic publish without an extra hop.

---

**Minimal architecture that's safe and reusable**

**1) One Python script: `publish_gold.py`**

Responsibilities:
- discover which tables to publish (by config or directory scan)
- read parquet (PyArrow/Polars/Pandas)
- write Excel to `DEST_DIR/<table>.xlsx._tmp_`
- close file
- rename to `DEST_DIR/<table>.xlsx`
- optional: write `<table>.ready`

**2) One bash wrapper: `publish_all.sh`**
- calls the script for multiple tables concurrently (or the script does concurrency itself)
- logs success/failure clearly

This becomes one "module" you reuse across projects, not repeated glue.

---

**Practical notes for Excel output**

- Use **one sheet per table**
- If a table might exceed Excel limits (≈1 million rows), fail loudly or split sheets
- Use `xlsxwriter` engine for speed/reliability

---

**Code template**

`publish_gold.py`

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import os
import shutil
import time
from pathlib import Path

import pandas as pd


def atomic_replace(src: Path, dst: Path, retries: int = 5, backoff_s: float = 0.3) -> None:
    """
    Replace dst with src as safely as possible on Windows-mounted paths.
    """
    dst.parent.mkdir(parents=True, exist_ok=True)

    last_err = None
    for i in range(retries):
        try:
            # If dst exists, replace it (atomic-ish on same filesystem).
            # shutil.move uses os.rename when possible.
            shutil.move(str(src), str(dst))
            return
        except Exception as e:
            last_err = e
            time.sleep(backoff_s * (2 ** i))

    raise RuntimeError(f"Failed to publish {dst.name} after {retries} retries") from last_err


def parquet_to_excel(parquet_path: Path, out_xlsx: Path, sheet_name: str = "data") -> None:
    # Read parquet
    df = pd.read_parquet(parquet_path)

    # Excel row limit safety (optional)
    if len(df) > 1_048_576:
        raise ValueError(f"{parquet_path} has {len(df):,} rows > Excel limit")

    tmp = out_xlsx.with_suffix(out_xlsx.suffix + ".__tmp__")

    # Write to a temp file FIRST
    with pd.ExcelWriter(tmp, engine="xlsxwriter") as writer:
        df.to_excel(writer, index=False, sheet_name=sheet_name)

    # Then atomically publish
    atomic_replace(tmp, out_xlsx)

    # Optional ready marker
    ready = out_xlsx.with_suffix(out_xlsx.suffix + ".ready")
    ready.write_text(f"ok\n", encoding="utf-8")


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--gold-parquet", required=True, help="Path to gold parquet (file or directory)")
    ap.add_argument("--out-dir", required=True, help="Windows/SharePoint-synced output directory")
    ap.add_argument("--name", required=True, help="Output base name (without extension)")
    ap.add_argument("--sheet", default="data")
    args = ap.parse_args()

    gold = Path(args.gold_parquet)
    out_dir = Path(args.out_dir)
    out_xlsx = out_dir / f"{args.name}.xlsx"

    # Support either a single parquet file or a parquet directory produced by Spark
    parquet_file = gold
    if gold.is_dir():
        # Spark output directory: read the directory as parquet dataset
        # pandas.read_parquet can read a directory if pyarrow is installed.
        parquet_file = gold

    parquet_to_excel(parquet_file, out_xlsx, sheet_name=args.sheet)
    print(f"Published: {out_xlsx}")


if __name__ == "__main__":
    main()

```

---

`publish_all.sh` (run multiple in parallel)

```bash
#!/usr/bin/env bash
set -euo pipefail

OUT_DIR="/mnt/c/Users/you/SharePointSync/GoldExports"

python3 publish_gold.py --gold-parquet "/home/you/pipeline/gold/projectA" --out-dir "$OUT_DIR" --name "projectA" &
python3 publish_gold.py --gold-parquet "/home/you/pipeline/gold/projectB" --out-dir "$OUT_DIR" --name "projectB" &
python3 publish_gold.py --gold-parquet "/home/you/pipeline/gold/projectC" --out-dir "$OUT_DIR" --name "projectC" &

wait
echo "All publishes complete."

```

This gives you:
- Spark stays in WSL filesystem (sale)
- Publishing to SharePoint folder is atomic (safe for `.xlsx`)
- Concurrency is easy (fast)
- Only one script to reuse (not painful)

---

If you want the *cleanest* "no repeated commands" UX, you can make the Python script accept a small
YAML/JSON config of tables to publish and run them with a `--jobs N` worker pool 

Then your bash becomes literally:

```bash
python3 publish_gold.py --config publish.yaml --jobs 4
```
