# `importlib.resources` — accessing package data safely

**The problem this solves**

In Python projects, we often need to load **non-Python files** (JSON configs, templates, SQL, etc.) that ship *with* a package.

Naive approaches like:

```python
open("config/io/paths.win.json")
```

break when:
- code is executed from a different working directory
- the package is installed as a wheel
- the package is zipped (no real filesystem path)

`importlib.resources` solves this by providing a **package-aware**, **installation-safe** way to access bundled data.

**Key concept: *resources are not guaranteed to be files***
---

When a package is installed:
- source installs → files exist on disk
- wheels / zipapps → files may live **inside an archive**

Therefore:
- `open(path)` ❌ is unsafe
- `importlib.resources` ✅ abstracts over both cases

**Minimal working example**
---

Project layout

```text
src/
  path_adapter/
    __init__.py
    configs/
      __init__.py
      io/
        paths.win.json
```

The directory **must be importable**:
- under `src/`
- with `__init__.py`

**Loading a JSON resource (recommended)**
---

```python
from importlib.resources import files
import json

ref = files("path_adapter.configs").joinpath("io/paths.win.json")
data = json.loads(ref.read_text(encoding="utf-8"))
```

Why this works:
- `files()` locates the package wherever it’s installed
- `read_text()` works even if the resource lives inside a wheel

**Why not `open(ref)` even if it looks like a Path?**
---

Sometimes `ref` **prints as a** `PosixPath` during development. This is an implementation detail.

⚠️ Do not rely on this.

In production:

- `ref` may represent a virtual path inside a zip
- `open(ref)` may fail

**Rule**:

> If you didn’t explicitly create the path yourself, don’t open() it.

**When you *do* need a real filesystem path**

Some tools (Spark, C extensions, CLI tools) require a literal path.

Use `as_file()`:

```python
from importlib.resources import files, as_file
import json

ref = files("path_adapter.configs").joinpath("io/paths.win.json")

with as_file(ref) as path:      # guaranteed real Path
    with open(path, "r", encoding="utf-8") as f:
        data = json.load(f)
```

This:
- extracts the resource to a temp location if needed
- cleans it up automatically

**Packaging requirement (important)**
---

Non-`.py` files must be explicitly included in the package.

Example (`pyproject.toml`, setuptools):

```toml
[tool.setuptools.package-data]
path_adapter = ["configs/**/*.json"]
```

Without this:
- editable installs may work
- built wheels may silently miss data ❌

**Mental model (worth remembering)**
---

| Thing             | Meaning                    |
| ----------------- | -------------------------- |
| `files(pkg)`      | Locate package resources   |
| `ref.read_text()` | Safe read, archive-aware   |
| `ref.open()`      | File-like access           |
| `as_file(ref)`    | Force real filesystem path |
| `open()`          | Only for paths you created |

**When to use this pattern**
---

Use `importlib.resources` for:

- config templates
- default mappings
- embedded SQL
- reference data
- anything that must ship with code

Do **not** use it for:

- runtime state
- user-editable configs
- secrets