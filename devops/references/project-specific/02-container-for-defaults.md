# Situation #1

> **Need default argument values for CSV read with Pandas**

**1. Define in `/common/config.py`**

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

**2. Import in `/io/file_read.py`**

```python
cfg = csvReadConfig()
pd.read_csv(path, encoding=cfg.encoding, sep=cfg.delimiter, dtype=cfg.dtype,
            na_values=cfg.na_values, keep_default_na=cfg.keep_default_na)
```

**Appendix: `from dataclasses import dataclass`**
---

> A **dataclass** is a Python feature that auto-generates boring boilerplate for classes that mostly store data.

Without `@dataclass`, you'd have to write the constructor yourself:

```python
class csvReadConfig:
    def __init__(self, encoding="utf-8-sig", delimiter=",", dtype=None, ...):
        self.encoding = encoding
        ...
```

With `@dataclass`, Python writes that for you atuomatically

**What the decorator means**

A **decorator** is just "a function that takes something and returns a modified version of it."

So:

```python
@dataclass(...)
class csvReadConfig:
    ...
```

**means**: "Create this class ... then pass it through `dataclass(...)` which adds methods and behaviors"

Specifically, `@dataclass` auto-adds things like:
- `__init__` (constructor)
- `__repr__` (nice printing)
- `__eq__` (comparisons)  
    *(and more, depending on options)*

`frozen=True`  
This is important, it makes the object **immutable** after creation

So:

```python
cfg = csvReadConfig()       # works
cfg.encoding = "latin1"     # ❌ raises dataclasses.FrozenInstanceError
```

And you must:

```python
cfg2 = csvReadConfig(encoding="latin1")
```

**Appendix 2: `from __future__ import annotations`**
---

**The problem it solves**  
In Python, type hints are normally evaluated “right away” (historically). That can cause issues when you reference types that don’t exist yet, or you want forward references.

**What it changes**  
With `from __future__ import annotations`, Python stores annotations as strings (postponed evaluation).

Example without `__future__`:

```python
def f(x: "CsvReadConfig"): ...
```

With `__future__`, you can do:

```python
def f(x: CsvReadConfig): ...
```