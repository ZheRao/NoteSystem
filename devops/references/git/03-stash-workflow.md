# Syncing `main` when you have **uncommitted** local work (stash workflow)

## 1. The problem
You’re on `main` locally, you have **uncommitted** edits (or new files), and `origin/main` has new commits (pushed by someone else).
You want to pull remote changes first, then commit + push your local work.

Direct `git pull` can fail with messages like:
- “Your local changes would be overwritten by merge”
because Git may need to update your working tree, and it won’t risk overwriting your uncommitted work.

## 2. The safe solution: stash → pull → unstash

**2.1 Temporarily save your local uncommitted work**:

```bash
git stash -u
```

**2.2 Sync with GitHub:**

```bash
git pull --rebase
# or: git pull
```

**2.3 Restore your work:**

```bash
git stash pop
```

**2.4 Now commit + push normally:**

```bash
git add -A
git commit -m "..."
git push
```

## 3. What is `git stash`?

`git stash` saves your **working directory change** (and usually staged changes too) into an internal “stash stack”, and resets your working tree back to match `HEAD` (your last commit).  
This gives you a clean state to pull/rebase/checkout without losing your local progress.

Think of it as:

- “pause my current work”
- “go do a clean operation”
- “resume my work”

`git stash -u` (**important**)
---

By default, `git stash` does **NOT** include untracked files (new files you created that aren’t `git add`’d yet).

`-u` means **include untracked files** too.

**Example**

If you created:

- `notes/tmp.md` (new file, untracked)
- edited `src/a.py` (tracked file)

Then:

- `git stash` would stash `src/a.py` edits, but leave `notes/tmp.md` in your folder.
- `git stash -u` stashes both.

> There’s also `-a` (“all”) which includes ignored files too, but that’s usually not what you want.

`git stash pop`
---

`git stash pop` does two things:

1. applies the most recent stash onto your current working tree
2. removes that stash entry from the stash stack

So it’s like: “restore my work and delete the saved copy.”

**Alternative**: `git stash apply`

`apply` restores but does not delete the stash.  
Use it when you want a “safety copy” until you’re sure things are good.
