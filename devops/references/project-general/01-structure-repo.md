# Structure Overview: `src/project_name`

```text
├── __init__.py
├── common
│   └── config.py
├── io
│   ├── file_read.py
│   └── file_write.py
├── json_configs
│   ├── __init__.py
│   ├── contracts
│   │   ├── so.contract.json
│   │   └── so.mapping.json
│   └── io
│       └── paths.win.json
└── silver
    └── sales_order.py
```

## 1. Importable Scripts: `src/project_name/module_type/module_name`

```text
src/path_adapter/
  __init__.py

  io/                       # filesystem boundaries: OneDrive/local, atomic write, globbing
    __init__.py
    fs.py
    discovery.py

  silver/                   # raw -> standardized tables
    __init__.py
    sales_orders.py
    invoices.py
    schema.py               # canonical schemas + dtype maps
    cleaning.py             # reusable “boring” cleaning helpers

  gold/                     # standardized -> business tables / marts
    __init__.py
    sales_metrics.py
    customer_mix.py
    mappings.py             # reference data joins / rules

  common/
    __init__.py
    config.py               # dataclasses for paths/options
    models.py               # IngestResult, TableResult, etc.
    logging.py
```

`common/config.py`
---
- Holds **typed config objects** (paths, CSV read options, silver write options)
- so the pipeline is driven by explicit inputs instead of notebook globals.
- It’s necessary for reproducibility, testing, and future orchestration (config becomes the contract).
```python
from __future__ import annotations
from dataclasses import dataclass
from pathlib import Path
from typing import Optional, Dict

@dataclass(frozen=True)
class CsvReadConfig:
    encoding: str = "utf-8-sig"
    delimiter: str = ","
    dtype: Optional[Dict[str, str]] = None
    na_values: tuple[str, ...] = ("", "NA", "N/A", "null", "NULL")
    keep_default_na: bool = True
```

`io/file_read.py`
---
- Implements the **I/O boundary**
- read a raw CSV from OneDrive/local path and return a DataFrame with consistent parsing settings.
- It’s necessary to keep “getting bytes into a DataFrame” separate from transformations, so storage quirks don’t leak into Silver logic.
```python
from __future__ import annotations
from pathlib import Path
from typing import Optional
import pandas as pd

from path_adapter.common.config import CsvReadConfig

def read_csv_from_path(
    csv_path: Path,
    cfg: CsvReadConfig,
) -> pd.DataFrame:
    """
    Pure I/O boundary: reads CSV from a path and returns a DataFrame.
    No business logic, no column renames (keep it dumb).
    """
    if not csv_path.exists():
        raise FileNotFoundError(f"CSV not found: {csv_path}")

    df = pd.read_csv(
        csv_path,
        encoding=cfg.encoding,
        sep=cfg.delimiter,
        dtype=cfg.dtype,
        na_values=list(cfg.na_values),
        keep_default_na=cfg.keep_default_na,
        low_memory=False,
    )
    return df
```

`__init__.py`
---
- Defines the **public API** of your package 
- by re-exporting the few functions/configs you want to be easy to import (`from path_adapter import ...`). 
- It’s necessary so notebooks + orchestrators don’t reach into internal module paths (which makes refactors painful).
```python
from path_adapter.common.config import CsvReadConfig
from path_adapter.io.file_read import read_csv_from_path

__all__ = [
    "CsvReadConfig",
    "read_csv_from_path",
]
```

*During Import*
---
```python
from path_adapter import (
    CsvReadConfig,
    read_csv_from_path
)
```















## 2. Config & State: `src/project_name/json_configs`

Structure:

```text
config/
  README.md

  env/                      # “what this run is” (dev/stage/prod, local, etc.)
    dev.json
    prod.json

  io/                       # “where stuff is”
    paths.local.json
    paths.wsl.json
    paths.prod.json

  contracts/
    bronze.json
    silver.json
    gold.json
    expectations.json   # null rules, allowed ranges, uniqueness, etc.

  connectors/
    qbo.json          # base URLs, scopes, realm-id presence rules, pagination strategy
    fx_rates.json     # provider name, base currency, refresh cadence

  domain/                   # “business decisions / logic”
    companies.json          # canonical company list + ids
    mappings/
      account_map.json
      class_map.json
      location_map.json
    rules/
      exclusions.json       # ignore accounts/classes/vendors/etc.
      normalization.json    # rename rules, standardization, etc.

  runtime/                  # knobs that change behavior but aren’t “business logic”
    logging.json
    spark.json
    performance.json

state/                      # machine-written (gitignored)
  pipeline_meta.json        # last_load_date, watermark, last_fx_rate_date, etc.
  checkpoints/
  locks/
```

Location for placement ("production-proof" approach)

```text
src/path_adapter/
  __init__.py
  resources/
    config/io/paths.win.json
```

Load it:

```python
from importlib.resources import files
import json

def read_paths(name="paths.win.json") -> dict:
    p = files("path_adapter.resources").joinpath(f"config/io/{name}")
    return json.loads(p.read_text())
```
