# Atomic File Write — Integrity & Durability Mechanism

## Purpose

Atomic file writing is a filesystem pattern that guarantees **data integrity**: readers will see **either the old complete file or the new complete file**, never a partially written file.

This mechanism is foundational for ETL pipelines, system outputs, checkpoints, and any workflow where corrupted or half-written files are unacceptable.

## Core Invariant

> **If the final filename exists, its contents are complete.**

This invariant holds even if:

* the process crashes mid-write
* the machine loses power
* the file is rewritten multiple times per day

Atomic write guarantees **integrity**, not **freshness**.

## The Atomic Write Pattern

### Canonical Implementation

```python

def atomic_write_bytes(dst: Path, data: bytes) -> None:
    """
    Write bytes atomically: write temp file in same directory then rename.
    """
    ensure_dir(dst.parent)
    tmp = dst.with_name(f".{dst.name}.{uuid.uuid4().hex}.tmp")
    with open(tmp, "wb") as f:
        f.write(data)
        f.flush()
        os.fsync(f.fileno())
    atomic_replace(tmp, dst)
```

## Step-by-Step Mechanics

### 1. Write to a Unique Temporary File

```python
tmp = dst.with_name(f".{dst.name}.{uuid.uuid4().hex}.tmp")
```

* A UUID ensures **no filename collision**, even with concurrent writers.
* Temporary files isolate incomplete writes from readers.
* If the process crashes here, only a `.tmp` file remains.

**Result:**

* Readers still see the previous valid `dst` file.

---

### 2. Write Bytes (User-Space)

```python
f.write(data)
```

* Data is written into Python / libc buffers first.
* No durability or visibility guarantees yet.

---

### 3. Flush Python Buffers → OS

```python
f.flush()
```

* Forces Python to hand buffered bytes to the OS.
* Data may still reside in OS page cache.

**Guarantee:** bytes have reached the kernel.

---

### 4. fsync — OS Cache → Disk

```python
os.fsync(f.fileno())
```

* Forces the OS to commit file contents to stable storage.
* Protects against crashes and power loss.

**Guarantee:** file contents are durable.

---

### 5. Atomic Replace (Rename)

```python
atomic_replace(tmp, dst)
```

Under the hood (POSIX):

```c
rename(tmp, dst);
```

Filesystem guarantee:

* Directory entry switch is **atomic**.
* Readers see **either** old `dst` **or** new `dst`.
* No intermediate or partial state exists.

**Key insight:**

> Atomic replace swaps a pointer, not bytes.

## Atomic Replace vs Regular Write

### Regular Write (Unsafe)

```python
open(dst, "wb").write(data)
```

Observed by readers:

* empty file
* partially written file
* corrupted file if crash occurs

### Atomic Replace (Safe)

Observed by readers:

* old complete file
* instant switch
* new complete file

Never partial.

## What Atomic Write Guarantees

✅ File contents are never partially visible

✅ Crashes do not corrupt the final file

✅ Safe repeated overwrites of the same filename

✅ Readers always see a valid version

## What Atomic Write Does NOT Guarantee

❌ Freshness (latest run vs last successful run)

❌ Ordering between concurrent writers

❌ Audit history

❌ Protection against logical data errors

## Rewriting the Same File Multiple Times

Safe pattern:

* v2 exists as `dst`
* v3 writes `.tmp`
* v3 crashes → `dst` still points to v2
* v3 succeeds → atomic replace switches to v3

**Invariant holds:** `dst` is always complete.

## Freshness & Versioning Patterns (Optional)

To detect *which* version is present, add one of:

1. Embedded metadata (`extracted_at`, `run_id`)
2. Sidecar manifest file (`.json`)
3. Versioned filenames + `latest` pointer

Atomic write handles **integrity**; these handle **meaning**.

## Advanced Durability (Optional)

For maximum safety on crash-prone systems:

* `fsync(tmp)` before rename
* `fsync(dst.parent)` after rename

This ensures the rename itself survives power loss.

## Mental Model

* `write()` → Python buffer
* `flush()` → OS kernel
* `fsync()` → disk
* `rename()` → atomic pointer swap

## One-Line Law

> **Prepare bytes privately, then atomically swap the filename pointer.**

This is the core of atomic file writing.
