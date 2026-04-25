# `open(..., "rb")`

## `path.read_bytes()` wrapper

Under the hood, it’s basically:
```py
def read_bytes(path):
    with open(path, "rb") as f:
        return f.read()
```
So:
```py
raw = path.read_bytes()
```
≡
```py
with open(path, "rb") as f:
    raw = f.read()
```

## When you would use `open(..., "rb")`

### 1. Streaming / large files
```py
with open(path, "rb") as f:
    for chunk in iter(lambda: f.read(8192), b""):
        process(chunk)
```
👉 avoids loading entire file into memory


### 2. Partial reads
```py
with open(path, "rb") as f:
    header = f.read(128)
```
### 3. File descriptor control
- buffering behavior
- file locking (advanced)
- interaction with other low-level APIs

### 4. Symmetry with complex write logic

Sometimes you’re already inside a `with open(...)` block for other reasons.