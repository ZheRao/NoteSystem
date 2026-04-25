# Why Serialization before Writes?

Great question — this is one of those small words that hides an important systems boundary.

Let’s make it concrete.

## 🧠 What “serialize” means

**Serialization = convert an in-memory object into a storable/transmittable format**

In your case:
```py
obj = {"company": "A", "token": "abc"}
```
This is a **Python dict in memory**.

You cannot write this directly to a file.

You must convert it into something like:
```
{"company": "A", "token": "abc"}
```
That conversion is **serialization**.

## 🔧 Two-step pipeline (this is the key mental model)

Step 1 — **Python object → JSON string**
```py
json_str = json.dumps(obj)
```
Now:
```
'{"company": "A", "token": "abc"}'
```
👉 This is a **string (text)**

Step 2 — **JSON string → bytes**
```py
data = json_str.encode("utf-8")
```
Now:
```
b'{"company": "A", "token": "abc"}'
```
👉 This is **bytes (what files actually store)**

## ⚠️ Why bytes matter (this is the important part)

Your atomic writer uses:
```py
with open(tmp, "wb") as f:
```
`"wb"` = write binary

That means:
> the file expects bytes, not Python objects, not strings

### ❌ What would be wrong
```py
f.write(obj)        # ❌ invalid
f.write(json_str)   # ❌ also invalid in binary mode
```
### ✅ Correct
```py
f.write(data)  # bytes
```
## 💡 Why I told you to serialize before writing

This is the deeper systems reason:

🔥 **You want failure to happen BEFORE touching the file**
```py
payload = json.dumps(obj).encode("utf-8")
atomic_write_bytes(path, payload)
```
If something is wrong:
- object not serializable
- bad structure
- unexpected type

👉 `json.dumps` fails **before file write begins**

### ❌ Dangerous alternative
```py
with open(tmp, "wb") as f:
    json.dump(obj, f)   # writing directly
```
Problem:
- serialization + writing are mixed
- if something fails mid-way → partial file risk (in non-atomic setups)
- harder to reason about failure boundary

## 🧠 The deeper invariant

> Serialization defines the boundary between “in-memory truth” and “persisted truth.”

You want:
1. fully validate & serialize in memory
2. then commit atomically

This creates a clean separation:

| Phase         | Responsibility         |
| ------------- | ---------------------- |
| Memory        | correctness, structure |
| Serialization | format validity        |
| Write         | durability             |

# `read` instead of `write`

When reading in binary mode, `f.read()` returns raw bytes from disk.

```py
with open(path, "rb") as f:
    raw = f.read()

text = raw.decode("utf-8")
obj = json.loads(text)
```

In this flow:
1. `f.read()` loads the file contents as bytes
2. `.decode("utf-8")` converts bytes into a Python str
3. `json.loads(...)` parses the JSON string into Python objects

## Text Mode Alternative

Opening with `"r"` performs decoding implicitly:
```py
with open(path, "r", encoding="utf-8") as f:
    obj = json.load(f)
```
This is valid, but the decoding step is hidden inside the file object.

## If using `orjson`

Your instinct is good. `orjson` makes this cleaner because it naturally works at the bytes boundary.

Write:

```python
import orjson

payload = orjson.dumps(obj, option=orjson.OPT_INDENT_2)
atomic_write_bytes(path, payload + b"\n")
```

Read:
```py
raw = path.read_bytes()
obj = orjson.loads(raw)
```
That removes the explicit:
```py
.decode("utf-8")
.encode("utf-8")
```
because orjson directly serializes to bytes and deserializes from bytes.
