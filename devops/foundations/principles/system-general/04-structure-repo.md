# Editable Install

**Goal**
- You havea  repo with a real package (e.g., `path_adapter/`)
- You can run: `pip install -e .`
- Imports work immediately after code changes (no reinstall)

## 1. Preparation

**Recommended repo structure**

This avoids import weirdness and makes packaging cleaner.

```css
path-data-adapter/
  pyproject.toml
  README.md
  src/
    path_adapter/
      __init__.py
      ingest.py
      transform.py
  tests/
    test_smoke.py
```

Key points:
- Your importable packages lives in `src/path_adapter/`
- The folder `path_adapter/` must have `__init__.py`

---

**Minimal `pyproject.toml` (setuptools)**

Create `pyproject.toml` at repo root:

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "path-data-adapter"
version = "0.1.0"
description = "PATH data adapter"
readme = "README.md"
requires-python = ">=3.10"
dependencies = []

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]
```

That's enough for editable installs.

## 2. Editable Install Steps

From the repo root:

```bash
python -m venv .venv
# activate:
# Windows (PowerShell): .venv\Scripts\Activate.ps1
# Linux/macOS: source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -e .
```

Test it:

```bash
python -c "import path_adapter; print(path_adapter.__file__)"
```

You should see a path pointing into your repo (under `src/...`)

---

**Optional but very useful: dev extras**

If you want `pip install -e ".[dev]"`:

```toml
[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]
```

Then:

```bash
python -m pip install -e ".[dev]"
pytest
```

---

**Optional: define a CLI command**

If you want to run `path-adapter ...` from the terminal:

In `pyproject.toml`:

```toml
[project.scripts]
path-adapter = "path_adapter.__main__:main"
```

Create `src/path_adapter/__main__.py`:

```python
def main():
    print("hello from PATH adapter")
```

Reinstall editable (or it might already be picked up):

```bash
python -m pip install -e .
path-adapter
```
