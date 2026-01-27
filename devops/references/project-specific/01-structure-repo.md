# Structure Overview: `src/project_name`

```text
└── src
    └── path_adapter
        ├── __main__.py                 # production-callable interface
        ├── __init__.py                 # formalize and simplify import path
        ├── json_configs                # contracts
        │   ├── __init__.py
        │   ├── contracts
        │   │   ├── so.contract.json
        │   │   └── so.mapping.json
        │   └── io
        │       └── paths.json
        ├── pipelines                   # high-, thin-layer orchestration, feeds into __main__.py
        │   ├── full_refresh.py
        │   ├── po_pipeline.py
        │   └── so_pipeline.py
        ├── bronze                      # actual architecture
        │   ├── _helpers.py
        │   ├── extract_po.py
        │   └── extract_so.py
        ├── silver
        │   ├── _helpers.py
        │   └── so.py
        ├── gold
        │   ├── _helpers.py
        │   ├── datetable.py
        │   └── so.py
        └── utils                       # common methods across entire system
            ├── _helpers.py
            ├── config.py
            └── filesystem.py
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

  utils/
    __init__.py
    config.py               # dataclasses for paths/options
    models.py               # IngestResult, TableResult, etc.
    logging.py
```

`utils/config.py`
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
class csvReadConfig:
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

from path_adapter.utils.config import csvReadConfig

def read_configs(config_type:str, name:str) -> dict:
    """
    Reads and return configurations stored in json_configs/config_type/name, e.g., json_configs/io/path.win.json
    """
    p = files("path_adapter.json_configs").joinpath(f"{config_type}/{name}")
    return json.loads(p.read_text())

def read_csv_from_path(
    csv_path: Path,
    cfg: csvReadConfig,
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
from path_adapter.utils.config import csvReadConfig
from path_adapter.io.file_read import read_csv_from_path
from path_adapter.io.file_read import read_configs

__all__ = [
    "csvReadConfig",
    "read_csv_from_path",
    "read_configs"
]
```

*During Import*
---
```python
from path_adapter import (
    csvReadConfig,
    read_csv_from_path
)
```


## 2. Config & State: `src/project_name/json_configs`

Structure:

```text
json_config/
├── __init__.py             # enable import
├── contracts
│   ├── so.contract.json    # included/excluded columns from raw csv, column rename, memo, schema contract
│   └── so.mapping.json     # business logic mappings, customer_type, location_mapping, ...
└── io
    └── paths.win.json      # path configs

state/                      # machine-written (gitignored)
  pipeline_meta.json        # last_load_date, watermark, last_fx_rate_date, etc.
  checkpoints/
  locks/
```

Location for placement ("production-proof" approach)

```text
src/path_adapter/
  __init__.py
  json_config/
    io/paths.win.json
```

Load it:

```python
from importlib.resources import files
import json

def read_paths(name="paths.win.json") -> dict:
    p = files("path_adapter.json_config").joinpath(f"io/{name}")
    return json.loads(p.read_text())
```
