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

---
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

---

**1.2 Install pyenv**

```bash
curl https://pyenv.run | bash
```

---

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

---
## 2. Install Python 3.11 with pyenv

```bash
pyenv install 3.11.9
pyenv versions
```

This:
- Does **not** replace system Python
- Lives under `~/.pyenv/versions/`

---

## 3. Project bootstrap (Poetry-managed)

**3.1 Create Project**

```bash
mkdir project_name && cd project_name
```

---

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

---

## 4. Poetry configuration (clean + preditable)


