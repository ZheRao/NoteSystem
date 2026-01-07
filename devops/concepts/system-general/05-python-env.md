# Python Environments — Clean Mental Model & Workflow

**Big Picture**

| Layer           | Tool        | Responsibility                                  |
| --------------- | ----------- | ----------------------------------------------- |
| OS              | `apt`       | System dependencies (C libs, compilers)         |
| Python versions | `pyenv`     | Install & select **Python interpreters**        |
| Project env     | `poetry`    | Create & manage **virtual environments + deps** |
| App sandbox     | `pipx`      | Install **CLI tools** in isolation              |
| Notebooks       | `ipykernel` | Bridge venv ↔ Jupyter/VS Code                   |

> Rule of thumb  
> - **pyenv chooses which Python**
> - **Poetry chooses which packages**
> - **ipykernel exposes that environment to tools**

## 1. Install `pyenv` (Python version manager)

**1.1 System build dependencies (Ubuntu/WSL)**

Required so Python can compile cleanly

```bash
sudo apt update
sudo apt install -y \
  build-essential libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev \
  wget curl llvm libncurses5-dev libncursesw5-dev \
  xz-utils tk-dev libffi-dev liblzma-dev \
  python3-openssl git
```

If you skip this → `pyenv install` fails later

**1.2 Install pyenv**

```bash
curl https://pyenv.run | bash
```

**1.3 Shell initialization (idempotent)**

Append **once**, safely:

```bash
grep -q 'PYENV_ROOT' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
EOF
```

Key ideas:
- `grep -q ... ||` → append only if missing
- `<<'EOF'` → literal block, no variable expansion now
- `pyenv init` → installs shims & shell hooks

Apply immediately:

```bash
source ~/.bashrc
# or
exec $SHELL
```

Verify:

```bash
pyenv --version
```

## 2. Install Python 3.11 with pyenv

```bash
pyenv install 3.11.9
pyenv versions
```

This:
- Does **not** replace system Python
- Lives under `~/.pyenv/versions/`

## 3. Project bootstrap (Poetry-managed)

**3.1 Create Project**

```bash
mkdir project_name && cd project_name
```

**3.2 Lock Python version locally**

```bash
pyenv local 3.11.9
```

Creates `.python-version`
→ Any shell in this directory uses Python 3.11.9.

Sanity checks:

```bash
python -V
pyenv which python
```

## 4. Poetry configuration (clean + preditable)

**4.1 Force venv inside repo**

```bash
poetry config --local virtuaenv.in-project true
```

Results:
- `.venv/` lives in repo
- No mystery global environments
- Easy cleanup & inspection

Files to remember:
- `poetry.toml` → **Poetry behavior**
- `pyproject.toml` → **Projet metadata + deps**

**4.2 Initialize `pyproject.toml`**

```bash
poetry init -n \
  --name "mf_qbo_spark_refactor" \
  --description "Spark refactor experimentation for QBO ETL" \
  --license "MIT"
```

Notes:
- `-n` avoids wizard (fast, scriptable)
- License string ≠ License file (create separately if needed)

## 5. Create & bind the virtual environment

**5.1 Tell Poetry which Python to use**

```bash
poetry env use python3.11
```

This:
- Uses pyenv's local Python
- Creates `.venv/` if missing

**5.2 Install dependencies (initially none)**

```bash
poetry install
```

## 6. Dependency management

**6.1 Add runtime deps**

```bash
poetry add pyspark==3.5.1 pyarrow pandas orjson rich
```

**6.2 Add dev-only deps**

```bash
poetry add -D ipykernel
```

Why `-D`:
- Keeps runtime env lean
- Dev tooling is optional by design

## 7. Jupyter / VS Code integration (ipykernel)

**7.1 Register kernel**

```bash
poetry run python -m ipykernel install --user \
  --name mf-qbo-spark \
  --display-name "mf-qbo-spark (poetry)"
```

Important details:
- `poetry run` → ensures **correct venv**
- `--name` → internal ID (must be unique)
- `--display_name` → what humans see

Kernel location:

```bash
~/.local/share/jupyter/kernels/<name>/kernel.json
```

**7.2 Manage kernels**

```bash
jupyter kernelspec list
jupyter kernelspec uninstall old_kernel_name
```

## 8. Daily usage patterns
| Task        | Command                          |
| ----------- | -------------------------------- |
| Run script  | `poetry run python script.py`    |
| Open shell  | `poetry shell`                   |
| Add dep     | `poetry add <pkg>`               |
| Update lock | `poetry update`                  |
| Rebuild env | `rm -rf .venv && poetry install` |


## 9. Failure diagnostics

**"Wrong Python version"**

```bash
python -V
pyenv version
poetry env info
```

**"imports work in shell but not notebook"**

→ Kernel registered from wrong env
Fix: re-run `ipykernel install` via `poetry run`

**"Poetry ignoring pyenv"**

→ You forgot `pyenv local` **before** `poetry env use`

## 10. Conceptual map

```css
apt
 └─ pyenv
     └─ python 3.11.x
         └─ poetry
             └─ .venv/
                 └─ ipykernel
                     └─ Jupyter / VS Code
```

Each layer is **replaceable, isolated,** and **debuggable**.
