# `rsync` — Efficient, Incremental File Synchronization

**Basic command**

```bash
rsync -av src/ dest/
```

Copies files from `src` to `dest` efficiently, transferring only what changed and preserving metadata where possible

## 1. Core flags

`-a` — **archive mode**

A bundle of options equivalent to:
- recursive copy
- preserve timestamps
- preserve permissions
- preserve owner/group
- preserve symlinks
- preserve device/special files

Note (WSL + `/mnt/c`)
Some metadata (owner/group, permissions) cannot be fully preserved on Windows-mounted filesystems.  
This is expected and harmless.

`-v` — **verbose**

Prints each file as it is processed.

## 2. Trailing slash semantics (critical)

This is one of the most important `rsync` rules.

- `src/`  
    Copy *contents* of `src` into `dest`

```text
dest/fileA
dest/subdir/...
```

- `src`  
    Copy the *directory itself* into `dest`

```text
dest/src/fileA
dest/src/subdir/...
```

## 3. Common usage patterns

- Copy contents of `src` into `dest` (create `dest` if needed)

```bash
rsync -av src/ dest/
```

- Preview what would happen (no changes)

```bash
rsync -av --dry-run src/ dest/
```

- Show overall progress (very useful for large trees)

```bash
rsync -av --info=progress2 src/ dest/
```


## 4. File-level copy behavior

- Copy a single file

```bash
rsync -av src/file1 dest/
```

Result:

```text
dest/file1
```

- Copy a file nested in a directory **without recreating the directory tree**

```bash
rsync -av src/folder1/file1 dest/
```

Result:

```text
dest/file1
```

(`rsync` does *not* create intermediate directories unless you ask it to)


## 5. How `rsync` decides whether to copy a file

**Default behavior — "quick check"**

By default, `rsync` skips files when:
- file size is the same **and**
- modification time is the same

This is fast and usually sufficient.

`--ignore-times`
-

Disables the quick check shortcut.

Bahvior:
- Forces `rsync` to examine file contents even if size + mtime match
- Uses rolling checksums **only when the quick cehck would have skipped**
- Transfers only the changed blocks if difference exist

Use when:
- timestamps are unreliable
- files are regenerated in-place
- content changes but mtime does not

`--checksum`
-

Strongest correctness guarantee.

Behavior:
- Always computes full checksums for every file on both sides
- Ignores size and timestamp entirely
- Transfers file only if content differs

Tradeoff:
- Slight CPU cost on both sides
- Excellent for small files (JSON, CSV, configs)
- Overkill for large binaries

# `wslpath` — Windows ↔ Linux Path Translation

`wslpath` converts paths between Windows and WSL formats reliably.

**Why use it**
- portable
- robust
- bi-directional
- avoids hardcoding `/mnt/c/...`

## 1. Common patterns

**1) Windows → Linux (use Linux tools on a Windows folder)**

```bash
SRC_WIN='C:\Users\Zhe Rao\Documents\src'
SRC_LNX="$(wslpath -u "$SRC_WIN")"

rsync -av --info=progress2 "$SRC_LNX/" "$HOME/src_copy/"
```

**2) Linux → Windows (pass Linux paths to Windows programs)**

```bash
DST_LNX="$HOME/src_copy"
DST_WIN="$(wslpath -w "$DST_LNX")"

explorer.exe "$DST_WIN"
```

**3) UNC network shares**

```bash
UNC='\\fileserver\teamshare\project'
UNC_LNX="$(wslpath -u "$UNC")"
# -> /mnt/unc/fileserver/teamshare/project

ls "$UNC_LNX"
```
