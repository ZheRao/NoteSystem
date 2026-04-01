# Python Import & Module Mechanics

> This document explains **what actually happens** when Python imports a module, why names behave the way they do, and how this affects type hints, contracts, and system design.

This is a **foundational mechanism**, not a usage guide.


## 1. The Core Truth (Anchor Statement)

> **A Python module is the atomic unit of import.**
>
> Python always loads and executes the *entire module*. Import statements only control **which names are bound into the caller’s namespace**.

If this sentence is understood, most import-related confusion disappears.


## 2. What Happens When You Import a Module

Example:

```python
from task_scheduler import flatten_job_scheduler
```

Python performs the following steps:

1. Locate `task_scheduler.py`
2. Execute the file **top to bottom**:

   * imports
   * constants
   * class definitions
   * function definitions
3. Create a **module object** with its own namespace (`module.__dict__`)
4. Bind **only the requested name(s)** (`flatten_job_scheduler`) into the caller’s namespace

**Important**: The module is fully loaded *before* any name is returned.


## 3. Two Namespaces You Must Distinguish

### 3.1 Module namespace (internal)

Inside `task_scheduler.py`:

```python
FlattenTask
flatten_job_scheduler
_some_helper
```

All of these names exist in the **module’s namespace** once the module is imported.

---

### 3.2 Importer namespace (external)

In the importing file:

```python
flatten_job_scheduler   # available
FlattenTask             # NOT available unless explicitly imported
```

This is the key distinction:

> **Unimported names still exist — they are simply not bound in the caller’s namespace.**

## 4. Why `from module import x` Does *Not* Partially Load Code

Python does **not** load individual functions or classes in isolation.

Reasons:

* decorators must run
* class inheritance must resolve
* module-level constants must exist
* circular imports must stabilize

Therefore:

> **Modules are executed as wholes. Names are optional exports.**

## 5. How This Relates to Type Hints

### Key rule

> **Type annotations are resolved in the module where they are written.**

Example:

```python
# task_scheduler.py
class FlattenTask(TypedDict): ...

def flatten_job_scheduler(...) -> list[FlattenTask]: ...
```

When you import only the function:

```python
from task_scheduler import flatten_job_scheduler
```

This works because:

* the module was fully executed
* `FlattenTask` exists inside the module
* the function’s annotations already reference it

You only need to import `FlattenTask` **if you want to reference it yourself**:

```python
from task_scheduler import flatten_job_scheduler, FlattenTask

tasks: list[FlattenTask] = flatten_job_scheduler(...)
```

## 6. Runtime vs Static Enforcement (Critical Distinction)

* `TypedDict`, `list[T]`, `Literal`, etc. **do not enforce constraints at runtime**
* They exist for:

  * static type checkers (mypy, pyright)
  * IDEs
  * human readers

At runtime:

* a `TypedDict` is just a `dict`
* no automatic validation occurs

If runtime enforcement is needed, it must be added explicitly (validators, pydantic, assertions).

## 7. `from __future__ import annotations`

When enabled:

* annotations are stored as **strings**
* evaluation is deferred
* import cycles are easier to manage

This reduces runtime coupling, but **does not change** the import mechanics described above.

## 8. Practical Design Implications (Systems Work)

* Put schemas and contracts in the same module (or imported there)
* Do not fear importing only functions — the module is still fully loaded
* Keep runtime behavior separate from static contracts
* Avoid relying on memory: encode contracts in types

## 9. One-Line Mental Model (Carry This)

> **Python imports modules, not symbols. Symbols are just names you choose to bind.**

