
# Branching & PR Workflow

## 1. Inspect Branches

```bash
git branch --show-current   # current branch
git branch                  # local branches
git branch -a               # include remote branches
```

Visualize history:
```bash
git log --oneline --decorate --grph --all -n 20
```

## 2. Keep Base Branch Up-to-Date

**2.1 Standard Approach:**
```bash
git switch main
git pull
```

If `main` changed while you were developing:
```bash
git switch feature/new-branch
git merge main
```

**2.2 Alternative: Use Remote-Tracking Branch**

Without switching to `main`:
```bash
git fetch origin
git switch feature/new-branch
git merge origin/main
# or
git rebase origin/main`
```

## 3. Other Common Branch Operations

**3.1 Create a New Branch**

```bash
git switch -c feature/new-branch
```

Naming convention:
- `feature/...`
- `fix/...`
- `chore/...`
Makes PRs easier to scan and review

**3.2 Review Changes**

```bash
git diff
```

**3.3 Push Branch**

```bash
git push -u origin feature/new-branch
```

After the first push, future pushes only need
```bash
git push
```

**3.4 Cleanup After PR Merge**

```bash
git switch main
git pull
git branch -d feature/new-branch                # delete local branch
git push origin --delete feature/new-branch     # delete remote branch
```


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



