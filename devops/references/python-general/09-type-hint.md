# Python Type Hints — Practical Gold Standard

## 1. Mental Model: What Type Hints *Are* (and Are Not)

* Type hints **do nothing at runtime** (with very rare exceptions like `TypedDict` + runtime validators).
* They are **contracts for humans + tools**:

  * static type checkers (mypy, pyright, pyre)
  * IDEs (VS Code / Pylance)
  * future readers (including future me)
* The value is not correctness alone — it is **early error detection, schema clarity, and intent locking**.

Think of type hints as:

> *Executable documentation that can fail fast — before production.*

## 2. `TypedDict` — Schema for Dictionaries

### What it is

`TypedDict` (from `typing`) lets you describe the **exact shape** of a dictionary:

* required keys
* value types per key
* optional keys (explicitly)

At runtime, it is still a **plain `dict`** — zero overhead.

### Why it matters

Using `dict[str, str]` is *structurally weak*:

* any key is allowed
* missing keys are not caught
* typos silently pass type checking

`TypedDict` turns a loose bag of strings into a **contract**.

### Example

```python
from typing import TypedDict

class MetaDatum(TypedDict):
    Name: str
    Value: str
```

This guarantees:

* both keys exist
* both values are `str`
* key typos are caught by the type checker

## 3. Optional Keys in `TypedDict`

### Option A — `NotRequired` (preferred, explicit)

```python
from typing import TypedDict, NotRequired

class Column(TypedDict):
    ColTitle: str
    ColType: ColType
    MetaData: list[MetaDatum]
    Description: NotRequired[str]
```

**Semantics**:

* most keys are required
* specific keys are explicitly optional
* best choice for stable schemas that evolve

---

### Option B — `total=False` (all keys optional)

```python
class PartialColumn(TypedDict, total=False):
    ColTitle: str
    ColType: ColType
    MetaData: list[MetaDatum]
```

**Semantics**:

* every key is optional
* useful for partial updates / patches
* dangerous if used as a primary schema

## 4. `Literal` — Value-Level Constraints

### What it is

`Literal` restricts a value to a **finite set of allowed constants**.

This is stronger than a type like `str` or `int`.

### Example

```python
from typing import Literal

ColType = Literal["Date", "Text", "Number"]
```

Now:

* ✅ `"Date"`
* ❌ `"Datetime"`
* ❌ `"Numbers"`

Type checkers will reject invalid values.

### When to use `Literal`

* small enums
* config values
* schema fields
* places where typos are expensive

Use `Literal` instead of `Enum` when:

* no behavior is attached
* you only need validation + clarity

## 5. Container Types: `list[T]` vs `Sequence[T]`

### `list[T]`

Means:

* *specifically* a Python list
* mutable
* `.append()`, `.sort()` expected

Use when:

* you mutate the container
* the function *requires* a list

---

### `Sequence[T]`

Means:

* any ordered, indexable container
* list, tuple, etc.
* **read-only intent**

Use when:

* you only iterate / index
* you want API flexibility

⚠️ **Important gotcha**:

```python
Sequence[str]
```

also accepts:

```python
"ABC"  # strings are sequences of characters
```

Defensive check if needed:

```python
if isinstance(companies, str):
    raise TypeError("Expected a sequence of strings, not a single string")
```

## 6. Why `dict[str, str]` Is Usually a Smell

It may be **technically correct**, but it loses intent.

Problems:

* no required keys
* no schema
* no typo protection
* no evolution safety

Prefer:

* `TypedDict` for schemas
* `dataclass` for structured objects

Rule of thumb:

> If a dict has a *meaning*, it deserves a schema.

## 7. `TypedDict` vs `dataclass`

| Use case                     | Choose                    |
| ---------------------------- | ------------------------- |
| Pure data contract           | `TypedDict`               |
| Immutable config             | `@dataclass(frozen=True)` |
| Behavior + methods           | `@dataclass`              |
| JSON / dict interoperability | `TypedDict`               |

In pipeline systems:

* **configs** → `dataclass`
* **task / message contracts** → `TypedDict`

## 8. Design Principle (Gold Rule)

> **Type hints should encode intent, not just appease the checker.**

If a type:

* prevents a real class of bugs
* clarifies ownership or responsibility
* locks down a contract boundary

…it is worth writing.

If it doesn’t — it’s noise.

## 9. Recommended Usage Pattern (Systems Work)

* External boundaries → strongest types (`TypedDict`, `Literal`)
* Internal glue code → flexible types (`Sequence`, generics)
* Runtime validation only at edges
* Never rely on memory for schema — **encode it**

*This document is a living reference. Update it when a new pattern earns its place.*
