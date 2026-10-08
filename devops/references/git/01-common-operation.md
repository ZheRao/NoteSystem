

# Common Daily Operations

## 1. Stop Tracking/Ignore Files

**1.1 Untrack a File** (while keep it locally)

```bash
git rm --cached path/to/file
git commit -m "stop tracking file"
```

**1.2 Ignore Files** (`.gitignore`)

```text
# config
.env
*.env

# Python
__pycache__/
*.pyc
.venv/
```

## 2. Other Common Operations

**2.1 Stage All**

```bash
git add -A      # includes deletes
```

**2.2 Undo Changes**

Unstage files:
```bash
git restore --staged .
```

Discard local changes:
```bash
git restore .
```

**2.3 Erase all changes after last `git pull`**

```bash
# 1. Update your local references with the remote server
git fetch origin

# 2. Reset your current branch to match the remote branch exactly
git reset --hard origin/<branch-name>

# 3. Clean up any untracked or new local files/directories
git clean -fd
```



