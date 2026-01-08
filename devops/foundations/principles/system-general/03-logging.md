# Python Logging

> Python logging is a **name-based, hierarchical message routing system** where log records propagate upward until they encounter handlers.

## 1. What a logger is

**What it is**

- A named singleton returned by `logging.getLogger(name)`
- Identity is **only the string name**
- Same name → same logger object

**What it is not**

- Not tied to files
- Not tied to modules
- Not passed around (usually)
- Not ordered by creation time

## 2. Logger hierarchy (parent/child)

- No physical tree exists in code
- Hirarchy is derived from **dot-separated name prefixes**

Example:

```text
"sandbox.partition.3"
└── parent: "sandbox.partition"
    └── parent: "sandbox"
        └── parent: root
```
- Parents are determined by progressively removing the last segment

## 3. Propagation

When calling `logger.info(...)`
1. Logger creates a `LogRecord`
2. Sends record to its own handlers
3. If `propagate=True` (default)
    - Record is sent to parent logger
4. Continues until:
    - Root logger is reached, or
    - `propagate=False`


## 4. Root logger

- `logging.getLogger()` returns the root logger
- Root is the final catch-all
- Once root handlers are configured:
    - `logging.getLogger(__name__)` **just works**
    - All child loggers propagate into root unless stopped

## 5. Handlers (where logs actually go)

- Examples:
    - `FileHandler`
    - `StreamHandler`

- Handlers:
    - Decide *where* logs go
    - Decide *how* logs are formatted
    - Can be attached at **any level** of the hierarchy

**important consequence**
- Multiple loggers writing to the same handler  
    → same output destination
