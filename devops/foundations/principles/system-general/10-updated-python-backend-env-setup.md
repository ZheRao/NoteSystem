# Python Backend Environment Setup & Troubleshooting

A reusable reference for moving a Python/FastAPI backend between development computers (Linux and macOS), using **Git**, **pyenv**, and **Poetry**.

## 1. Responsibilities at a glance

| Tool or file | Purpose |
| --- | --- |
| Git | Synchronizes tracked source code and project configuration. |
| `pyenv` | Installs and selects Python interpreter versions. |
| `.python-version` | Selects the Python version for the directory (when using pyenv). |
| `pyproject.toml` | Declares project metadata, Python compatibility, dependencies, and Poetry settings. |
| `poetry.lock` | Records resolved dependency versions for reproducible installation. |
| Poetry | Resolves/installs dependencies and manages virtual environments. |
| `.venv/` | Local, machine-specific virtual environment; **do not commit**. |

**Important:** `.python-version` selects an interpreter but does not install it. `poetry.lock` pins dependencies but does not replace a compatible Python interpreter.

## 2. First-time setup on another computer

From the repository root:

```bash
git pull
cd backend
cat .python-version
pyenv versions
python --version
```

Check the Python constraint in `pyproject.toml`. Install the version required by the project if it is missing:

```bash
pyenv install <version-from-.python-version>
python --version
```

If pyenv reports `python-build: definition not found`, first check `pyenv install --list`. For a Git-based pyenv installation at `~/.pyenv`, update it with:

```bash
git -C ~/.pyenv pull
pyenv install --list | grep 3.13
```

Other pyenv installations (e.g. package managers) must be updated using their own installation method. Python builds on Linux may require compiler and development libraries.

### Prefer an in-project virtual environment

From `backend/`:

```bash
poetry config virtualenvs.in-project true --local
poetry config virtualenvs.in-project
```

The `--local` setting is saved to `poetry.toml` in the current project directory. Decide whether to track this project-specific configuration in Git; it is separate from `pyproject.toml`.

If no old environment is selected, create/install normally:

```bash
poetry install
poetry env info --path
```

The reported path should end in `backend/.venv`.

If Poetry continues selecting a pre-existing cached environment, explicitly create a local one using the pyenv-selected interpreter:

```bash
python -m venv .venv
poetry env use "$(pwd)/.venv/bin/python"
poetry install
```

Before running `python -m venv`, verify that `python --version` matches the intended project version. If Poetry still selects a different environment, inspect `poetry env info` and the `VIRTUAL_ENV` environment variable; deactivate any unrelated active environment and retry. Do not delete environments blindly.

## 3. Daily workflow

```bash
git pull
cd backend
poetry install
```

Run commands without activating the environment:

```bash
poetry run python --version
poetry run uvicorn main:app --reload
```

The Uvicorn command assumes `main.py` exports an `app` object and Uvicorn is installed; adjust the module path to match the project.

Or activate the local environment for an interactive shell:

```bash
source .venv/bin/activate
which python
# Work normally...
deactivate
```

On Windows PowerShell, activation uses `.venv\Scripts\Activate.ps1` instead.

**Note:** `source .venv/bin/activate` works only when `.venv` exists inside the current project directory. `poetry run` works regardless of whether Poetry stores the environment locally or in its cache.

## 4. Troubleshooting reference

### A. Poetry cannot invoke the pyenv shim

Example:

```text
Command '['/home/<user>/.pyenv/shims/python', '-Ic',
'import sys; print(sys.executable)']' returned non-zero exit status 1.
```

The shim is an entry point used by pyenv, not the actual interpreter. Check:

```bash
pyenv versions
pyenv version
pyenv which python
python --version
poetry env info
```

A missing selected Python version is one possible cause, not the only one. Confirm the underlying pyenv error before changing configuration.

### B. `python-build: definition not found: <version>`

The installed pyenv/python-build may not know that version. Update pyenv using the appropriate installation method, then check `pyenv install --list` again. Verify the requested version is real and supported.

### C. `No file/folder found for package backend`

By default, `poetry install` installs dependencies **and** attempts to install the current project as a Python package. A small FastAPI application with a `main.py` but no package directory may not be packaged.

For dependency management only, use either:

```bash
poetry install --no-root
```

Or, for a deliberate non-package application, set under the existing `[tool.poetry]` table in `pyproject.toml`:

```toml
[tool.poetry]
package-mode = false
```

Then use `poetry install`. If the project is meant to be distributed as an installable package, instead configure its package layout correctly. Do not create a second `[tool.poetry]` table if one already exists.

### D. `No dependencies to install or update`, but no `.venv/`

This usually means Poetry has already installed dependencies in a **centralized** environment, commonly under:

```text
~/.cache/pypoetry/virtualenvs/
```

Find the environment actually selected:

```bash
poetry env info
poetry env info --path
```

No visible `.venv/` does **not** mean no virtual environment exists.

### E. Enabling in-project environments did not move the old environment

`poetry config virtualenvs.in-project true --local` changes the preferred location for environment creation; it does **not** migrate an existing environment. Inspect the selected environment, then create a new local `.venv` if necessary. Confirm that `poetry env info --path` points to the new location.

### F. `poetry env remove --all` did not remove the cached environment

That command removes environments Poetry recognizes for the current project; it is not a guaranteed cleanup of every directory in Poetry's global cache. A cached environment may not be associated with the current project or interpreter selection. **Inspect before deleting.**

## 5. Safe cleanup of an unused centralized environment

First confirm the new local environment is functional:

```bash
poetry env info --path
poetry run python --version
poetry run python -c "import fastapi; print(fastapi.__version__)"
```

The first command should point to `backend/.venv`.

Then inspect candidate cached environments:

```bash
ls -ld ~/.cache/pypoetry/virtualenvs/*
```

Identify the exact old directory by name and, ideally, by checking its interpreter and installed packages. Do not assume that every directory named `backend-*` is disposable: another checkout could use it.

Once verified that a specific cached environment is unused, remove **only that exact directory**:

```bash
rm -ri ~/.cache/pypoetry/virtualenvs/<verified-old-environment-name>
```

Never run a broad removal against the entire `virtualenvs` directory. Re-run the local environment checks afterward.

## 6. Version control and portability

Typically **commit**:

- `.python-version` (if the team standardizes on pyenv)
- `pyproject.toml`
- `poetry.lock`
- Backend source and tests
- Optionally `poetry.toml` if the project intentionally shares the in-project Poetry preference

Typically **ignore**:

```gitignore
.venv/
__pycache__/
*.py[cod]
```

Do not commit interpreter binaries or machine-specific virtual environments. Each computer installs its own Python interpreter and dependencies from the committed project files.

## 7. Quick verification checklist

```bash
cd backend
python --version
poetry config virtualenvs.in-project
poetry env info --path
poetry run python --version
poetry run python -c "import fastapi; print(fastapi.__version__)"
```

Expected: the Python version is compatible with the project, the Poetry environment points to `backend/.venv`, and FastAPI imports successfully.

---

**Key mental model:** `pyenv` chooses **which Python**; Poetry chooses **which environment and dependencies**; Git carries **source and reproducible configuration**, not the installed environment itself.
