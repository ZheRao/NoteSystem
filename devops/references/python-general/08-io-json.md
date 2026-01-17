# JSON read/write patterns (text vs bytes)

## Mental model
There are 2 “lanes” for JSON I/O:

1) **Text lane** (human-readable): open files in `"r"` / `"w"` and work with **str**  
2) **Bytes lane** (performance / pipelines): open files in `"rb"` / `"wb"` and work with **bytes**

Key rule: **JSON itself is a text format**, but some libraries (notably `orjson`) let you stay in bytes to avoid expensive decode/encode steps.


## A) Standard library JSON (text lane)

### Read (text → Python object)
```python
import json

with open(path, "r", encoding="utf-8") as f:
    obj = json.load(f)          # file handle -> parses text
```

### Write (Python object → text)
```python
import json

with open(path, "w", encoding="utf-8") as f:
    json.dump(obj, f, indent=4, ensure_ascii=False)
```
Notes:
- `json.load/dump` work with **text streams** (file opened in `"r"`/`"w"`).
- `ensure_ascii=False` keeps Unicode readable (no `\uXXXX` escaping).
- Pretty printing (`indent=4`) increases file size but is great for diffs/debug.

## B) Bytes-oriented JSON with orjson (bytes lane)

### Write bytes directly (Python object → bytes)
```python
import orjson

with open(path, "wb") as f:
    f.write(orjson.dumps(obj, option=orjson.OPT_APPEND_NEWLINE))
```

### Read bytes then parse (bytes → Python object)
```python
import orjson

with open(path, "rb") as f:
    raw = f.read()              # bytes

obj = orjson.loads(raw)         # orjson.loads accepts bytes directly (fast path)
```
Notes:
- `orjson.dumps` returns **bytes** (not str).
- `orjson.loads` accepts **bytes** directly → no `.decode()` needed.

## Decision guide

**Choose TEXT lane if**:
- You want human-readable JSON files
- You’re using stdlib `json` (most compatible)
- You care about pretty diffs (`indent=4`)

**Choose BYTES lane if**:
- You want speed and lower overhead
- You’re writing/reading big JSON blobs
- You’re using `orjson` end-to-end

## C) Hybrid loader: prefer bytes-fast path, fallback to text-required libs

Use this pattern when you want:
- `orjson` if available (fast, accepts bytes)
- fallback to another library that may require **str** (decode step)

```python
def read_bytes(path):
    with open(path, "rb") as f:
        return f.read()

try:
    import orjson as fastjson
    loads = fastjson.loads      # accepts bytes directly
except Exception:
    import ujson as fastjson
    loads = fastjson.loads      # expects str

raw = read_bytes(path)

try:
    doc = loads(raw)            # works for orjson (bytes)
except TypeError:
    text = raw.decode("utf-8")  # required for libs that want str
    doc = loads(text)
```

Note:
- The **TypeError** branch is essentially “this loader doesn’t accept bytes”.
- Decoding large JSON blobs to text can be expensive (extra time + memory), so do it only when needed.