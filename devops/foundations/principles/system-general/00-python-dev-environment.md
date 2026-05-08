# Python Development Environment — Unified Architecture & Workflow

> A clean, layered, reproducible Python development setup for Ubuntu/WSL.
>
> Designed for:
>
> * long-term maintainability
> * fast recovery
> * minimal environment corruption
> * isolated projects
> * scalable engineering workflows
> * defensive infrastructure practices

## 0. Core Philosophy

### System layers

| Layer               | Tool              | Responsibility                            |
| ------------------- | ----------------- | ----------------------------------------- |
| OS                  | `apt`             | System libraries, compilers, tooling      |
| Python Runtime      | `pyenv`           | Install & select Python interpreters      |
| Project Environment | `poetry` / `venv` | Manage project dependencies               |
| CLI Sandbox         | `pipx`            | Isolated installation of Python CLI tools |
| Notebook Bridge     | `ipykernel`       | Connect environments to Jupyter / VS Code |
| Source Control      | `git`             | Versioned code + reproducibility          |

## 1. Architectural Mental Model

### Correct architecture

```text
Ubuntu system Python
    ↓
bootstrap only

pyenv Python 3.x
    ↓
project .venv
    ↓
project dependencies
    ↓
VS Code / Jupyter / scripts
```


### Critical principle

> System Python is NOT your development environment.

Ubuntu internally relies on system Python for:

* package management
* system utilities
* OS tooling

You should:

* leave it mostly untouched
* never install project dependencies into it

All real project work should happen inside:

* pyenv-managed interpreters
* isolated virtual environments

## 2. High-Level Workflow

### One-time machine setup

```text
Ubuntu packages
    ↓
pyenv
    ↓
pipx
    ↓
Poetry
    ↓
VS Code / tooling
```

### New project setup

```text
git clone
    ↓
pyenv local <version>
    ↓
poetry install
    ↓
configure .env
    ↓
run project
```

### Recovery workflow

```text
reinstall machine tooling
    ↓
clone repo
    ↓
restore .env
    ↓
poetry install
    ↓
continue working
```

## 3. One-Time Ubuntu / WSL Setup

### 3.1 Update Ubuntu

```bash
sudo apt update && sudo apt upgrade -y
```

Optional cleanup:

```bash
sudo apt autoremove
sudo apt autoclean
```

## 4. Essential System Packages

Install core tooling + Python build dependencies.

```bash
sudo apt install -y \
  build-essential \
  curl \
  git \
  pkg-config \
  unzip \
  wget \
  llvm \
  tk-dev \
  xz-utils \
  zlib1g-dev \
  libbz2-dev \
  libssl-dev \
  libffi-dev \
  libreadline-dev \
  libsqlite3-dev \
  liblzma-dev \
  libncurses5-dev \
  libncursesw5-dev \
  libxml2-dev \
  libxmlsec1-dev \
  python3-openssl
```

**Why these matter**

These packages allow:

* Python compilation
* wheel compilation
* C-extension builds
* SSL support
* compression support
* SQLite support
* terminal support

Without them:

* `pyenv install` may fail
* `pip install` may fail for source builds

## 5. System Python (Bootstrap Only)

Ubuntu already contains system Python.

Optional bootstrap packages:

```bash
sudo apt install -y python3 python3-venv python3-pip
```

These are useful for:

* bootstrapping
* emergency recovery
* compatibility

But:

> do NOT use system Python for project dependency management.

## 6. Git Setup

**Configure Git identity**

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

Verify:

```bash
git config --list
```

## 7. SSH Setup (GitHub/GitLab)

**Generate SSH key**

```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

Default location:

```text
~/.ssh/id_ed25519
```

**Show public key**

```bash
cat ~/.ssh/id_ed25519.pub
```

Add this key to:

* GitHub
* GitLab
* Bitbucket


**Test SSH authentication**

```bash
ssh -T git@github.com
```

## 8. Install pyenv

### 8.1 Install pyenv

```bash
curl https://pyenv.run | bash
```

### 8.2 Shell initialization

Append once to `~/.bashrc`:

```bash
grep -q 'PYENV_ROOT' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - bash)"
EOF
```

Apply changes:

```bash
source ~/.bashrc
```

### 8.3 Verify pyenv

```bash
pyenv --version
which pyenv
```

## 9. Install Python via pyenv

**Install Python 3.11.9**

```bash
pyenv install 3.11.9
```

List versions:

```bash
pyenv versions
```

**Set global default**

```bash
pyenv global 3.11.9
```

Verify:

```bash
python --version
which python
```

Expected:

```text
Python 3.11.9
~/.pyenv/shims/python
```

### Why pyenv Exists

Without pyenv:

```text
machine-wide Python version conflicts
```

Example:

* Project A requires Python 3.10
* Project B requires Python 3.11
* Ubuntu tools require system Python

pyenv isolates interpreters cleanly per project.

## 10. Install pipx

**Install**

```bash
sudo apt install -y pipx
pipx ensurepath
```

Restart shell:

```bash
source ~/.bashrc
```

### pip vs pipx

| Tool           | Purpose                                    |
| -------------- | ------------------------------------------ |
| `pip install`  | Install into active environment            |
| `pipx install` | Install standalone CLI app in isolated env |

Use:

* `pip` → project dependencies
* `pipx` → developer tools

## 11. Install Global Python CLI Tools

**Poetry**

```bash
pipx install poetry
```

**Jupyter**

```bash
pipx install --include-deps jupyter
```

**Useful optional tools**

```bash
pipx install ruff
pipx install black
pipx install httpie
```

## 12. Java Setup (PySpark)

**Install OpenJDK 17**

```bash
sudo apt install -y openjdk-17-jdk
```

**Configure JAVA_HOME**

Append to `~/.bashrc`:

```bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
echo 'export PATH="$JAVA_HOME/bin:$PATH"' >> ~/.bashrc
```

Apply:

```bash
source ~/.bashrc
```

Verify:

```bash
java -version
```

## 13. VS Code Setup

Install:

* VS Code
* Python extension (WSL extension if using WSL)
* Pylance


**Open project**

```bash
code .
```

**Select interpreter**

```text
Ctrl + Shift + P
Python: Select Interpreter
```

Choose:

```text
<repo>/.venv/bin/python
```

## 14. Project Bootstrap Workflow

**Clone repo**

```bash
git clone <repo-url>
cd <repo-name>
```

**Lock Python version locally**

```bash
pyenv local 3.11.9
```

Creates:

```text
.python-version
```

**Configure Poetry**

```bash
poetry config --local virtualenvs.in-project true
```

This creates:

```text
.venv/
```

inside the repo.

Benefits:

* predictable
* inspectable
* disposable
* isolated

**Create environment**

```bash
poetry env use python3.11
```

**Install dependencies**

```bash
poetry install
```

## 15. Dependency Management

**Runtime dependencies**

```bash
poetry add pandas pyspark pyarrow orjson rich
```

**Development dependencies**

```bash
poetry add -D ipykernel pytest
```

# 16. Environment Variables & Secrets

**Never hardcode secrets**

Use:

* `.env`
* `.env.example`

**Typical workflow**

```bash
cp .env.example .env
```

Then fill in actual values.

**Example**

```env
QBO_CLIENT_ID=
QBO_CLIENT_SECRET=
DATABASE_URL=
```

**Rules**

Commit:

* `.env.example`

Never commit:

* `.env`
* credentials
* API secrets
* service account files

## 17. Jupyter / VS Code Integration

**Register kernel**

```bash
poetry run python -m ipykernel install --user \
  --name qbo-env \
  --display-name "qbo-env (poetry)"
```

**List kernels**

```bash
jupyter kernelspec list
```

**Remove old kernels**

```bash
jupyter kernelspec uninstall old_kernel
```

## 18. Daily Workflow

**Open shell**

```bash
poetry shell
```

**Run script**

```bash
poetry run python script.py
```

**Add dependency**

```bash
poetry add package_name
```

**Update dependencies**

```bash
poetry update
```

**Rebuild environment**

```bash
rm -rf .venv
poetry install
```

## 19. Recovery Workflow

If environment is destroyed:

**Reinstall tooling**

```bash
apt packages
pyenv
pipx
poetry
```

**Rebuild project**

```bash
git clone ...
cd repo

pyenv install 3.11.9
pyenv local 3.11.9

poetry install

cp .env.example .env
```

## 20. Failure Diagnostics

**Wrong Python version**

```bash
python -V
pyenv version
pyenv which python
```

**Poetry using wrong interpreter**

```bash
poetry env info
```

Fix:

```bash
poetry env use python3.11
```

**pyenv install fails**

Usually missing:

* build dependencies
* SSL libs
* zlib libs

Revisit Section 4.

**Imports work in shell but not notebook**

Wrong kernel registered.

Re-register kernel:

```bash
poetry run python -m ipykernel install --user ...
```

**VS Code unresolved imports**

Usually:

* wrong interpreter selected
* `.venv` missing
* Poetry env not activated

## 21. Layered Failure Model

| Failure                | Likely Layer          |
| ---------------------- | --------------------- |
| gcc missing            | Ubuntu / apt          |
| pyenv install fails    | build dependencies    |
| wrong Python version   | pyenv                 |
| import errors          | Poetry / venv         |
| notebook issues        | ipykernel             |
| VS Code issues         | interpreter selection |
| package compile errors | system libraries      |
| git auth fails         | SSH configuration     |


## 22. What NOT To Do

**Never**

```bash
sudo pip install ...
```

**Never**

Install project dependencies into:

* system Python
* global Python

**Never**

Commit:

* `.venv`
* `.env`
* credentials

**Never**

Rely on globally installed libraries.

**Never**

Mix:

* Conda
* Poetry
* system pip

casually without clear boundaries.

## 23. Final Operational Philosophy

> Every layer should be:
>
> * isolated
> * replaceable
> * inspectable
> * reproducible
> * minimally coupled

If this architecture is followed correctly:

* environments become disposable
* recovery becomes fast
* debugging becomes layered
* scaling becomes safer
* machine rebuilds stop being catastrophic
