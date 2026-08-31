# SQLite (Python) + FastAPI Reference Sheet

> **Status:** first draft — living document.
> **Companion to:** the SQL Syntax Reference Sheet. That one covers what you say *to* the database. This one covers everything wrapped *around* it: the Python driver, the HTTP layer, and the seam between them.
> **Versions tracked:** Python `sqlite3` (3.11+, with 3.12 changes flagged), FastAPI 0.100+, Pydantic **v2**.

---

## Table of contents

| Part | Topic |
|---|---|
| 0 | Conventions and the two-layer model |
| **A** | **Python `sqlite3`** |
| 1 | The DB-API object graph |
| 2 | Connecting and configuring |
| 3 | Executing statements and binding parameters |
| 4 | Reading results — `description`, `row_factory`, fetching |
| 5 | Transactions (the trap-dense part) |
| 6 | Types, adapters, converters |
| 7 | Exceptions |
| 8 | Extending SQLite from Python |
| 9 | Connection lifecycle, threads, WAL |
| **B** | **FastAPI** |
| 10 | The app object and lifespan |
| 11 | Path operations |
| 12 | Parameters: path, query, body, header |
| 13 | Pydantic v2 models |
| 14 | Responses, status codes, errors |
| 15 | Dependency injection |
| 16 | Routers and project structure |
| 17 | Middleware and CORS |
| 18 | Settings and configuration |
| 19 | Testing |
| 20 | Serving |
| **C** | **The seam** |
| 21 | SQLite + FastAPI integration recipes |
| 22 | Traps ledger |
| 23 | Version and compatibility table |

---

## Part 0 — Conventions and the two-layer model

**Conventions**

- `[3.12]` = behaviour introduced or changed in Python 3.12.
- `[trap]` = flagged in the Part 22 ledger.
- Examples use a grain-farm cash-flow domain to stay concrete.

**The two-layer model**

There are two distinct state machines in play and most confusion comes from conflating them:

```
HTTP request  →  FastAPI            →  Python sqlite3        →  SQLite engine
                 (validation,          (Connection, Cursor,     (file, pages,
                  DI, serialization)    transaction control)     WAL, locks)
```

- **FastAPI** owns request/response shape, validation, and dependency lifetimes.
- **`sqlite3`** owns connections, cursors, parameter binding, and — critically — *when a transaction begins and ends*. This is the layer that surprises people, because it inserts transaction control that you didn't write.
- **SQLite** owns locking, durability, and everything in the SQL reference sheet.

A question like "why did my write disappear" is almost always a layer-2 question. "Why is my endpoint slow under load" is usually a layer-2/layer-3 boundary question (blocking driver on an async event loop, or the single-writer lock).

---

# Part A — Python `sqlite3`

## Part 1 — The DB-API object graph

```
sqlite3.connect(...) ──> Connection
                            │
                            ├── .cursor()      ──> Cursor
                            ├── .execute(...)  ──> Cursor   (shortcut: makes one for you)
                            ├── .commit() / .rollback() / .close()
                            └── .row_factory, .isolation_level, .total_changes

Cursor
  ├── .execute(sql, params)      ──> returns self
  ├── .executemany(sql, seq)
  ├── .executescript(sql_text)
  ├── .fetchone() / .fetchmany(n) / .fetchall()
  ├── .description               -- column metadata for the last SELECT
  ├── .rowcount                  -- rows affected by the last DML
  ├── .lastrowid                 -- rowid of the last single INSERT
  └── .row_factory
```

Key relationships:

- A **Connection** is a handle to the database file and the unit of transaction control.
- A **Cursor** is a handle to the results of one statement. It's an iterator.
- `conn.execute(...)` is pure sugar for `conn.cursor().execute(...)` and **returns the cursor**, which is why this chains:

```python
row = conn.execute("SELECT * FROM revisions WHERE id = ?", (rid,)).fetchone()
```

- `description` lives on the **Cursor**, not the Connection. `conn.description` does not exist. Because `conn.execute()` returns a cursor, the practical form is:

```python
cur = conn.execute("SELECT crop_code, amount_cents FROM crop_input_streams")
col_names = [d[0] for d in cur.description]   # ['crop_code', 'amount_cents']
```

`cursor.description` is a tuple of 7-tuples per column: `(name, type_code, display_size, internal_size, precision, scale, null_ok)`. In `sqlite3`, **only `name` is populated**; the other six are always `None`. It's a column-name accessor, not a type introspection API. It is `None` before any `SELECT` has been executed.

### Invariants

1. The Connection is the transaction. The Cursor is the result set.
2. Anything that reads results reads from a Cursor, including `description`.
3. `conn.execute()` returning a cursor is the only reason the one-liner form works — it isn't a separate API.

---

## Part 2 — Connecting and configuring

```python
import sqlite3

conn = sqlite3.connect(
    db_path,                    # str | os.PathLike | ":memory:"
    timeout=5.0,                # seconds to wait on a locked DB before OperationalError
    detect_types=0,             # PARSE_DECLTYPES | PARSE_COLNAMES
    isolation_level="",         # "" = legacy implicit transactions; None = autocommit
    check_same_thread=True,     # forbid use from another thread
    factory=sqlite3.Connection, # custom Connection subclass
    cached_statements=128,
    uri=False,                  # interpret db_path as a URI
)
```

### The `with` trap `[trap]`

```python
with sqlite3.connect(db_path) as conn:
    conn.execute("INSERT INTO t (a) VALUES (1)")
# conn is COMMITTED here — and STILL OPEN. The file handle is not released.
```

`Connection.__exit__` commits on clean exit and rolls back on exception. **It does not close the connection.** This is the single most misremembered fact in the module. It is a *transaction* context manager, not a *resource* context manager.

To get both:

```python
from contextlib import closing

with closing(sqlite3.connect(db_path)) as conn:   # closes
    with conn:                                     # commits / rolls back
        conn.execute("INSERT INTO t (a) VALUES (1)")
```

Or write your own, which is what you want in an application:

```python
from contextlib import contextmanager

@contextmanager
def get_conn(db_path: str, *, write: bool = False):
    conn = sqlite3.connect(db_path, timeout=5.0)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA busy_timeout = 5000")
    try:
        if write:
            with conn:            # transaction scope
                yield conn
        else:
            yield conn
    finally:
        conn.close()
```

### PRAGMAs — which are per-connection and which are persistent

| PRAGMA | Scope | Notes |
|---|---|---|
| `foreign_keys = ON` | **per connection**, resets every time | Must run **outside** a transaction — it's a silent no-op inside one |
| `busy_timeout = 5000` | per connection (ms) | Equivalent to the `timeout=` connect argument |
| `journal_mode = WAL` | **persistent in the file** | Set once at setup, not per connection |
| `synchronous = NORMAL` | per connection | `NORMAL` is the sane default under WAL |
| `cache_size = -64000` | per connection | Negative = KiB |
| `temp_store = MEMORY` | per connection | |

The `foreign_keys` one is worth internalizing: it is off by default, it is per-connection, and every connection factory, pool, script, notebook, and test fixture must set it independently. If it isn't in your connection factory, FK enforcement is not on.

```python
def connect(db_path: str) -> sqlite3.Connection:
    conn = sqlite3.connect(db_path, timeout=5.0)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    return conn
```

### URI connections

```python
conn = sqlite3.connect("file:data/forecast.db?mode=ro", uri=True)     # read-only
conn = sqlite3.connect("file:data/forecast.db?mode=rwc", uri=True)    # read/write/create
conn = sqlite3.connect("file:memdb?mode=memory&cache=shared", uri=True)
```

`mode=ro` is genuinely useful for a read-only serving layer — it makes accidental writes an error rather than a bug.

### Useful module-level attributes

```python
sqlite3.sqlite_version        # '3.45.1' — the ENGINE version; this is the one that matters
sqlite3.threadsafety          # 1 or 3 depending on how the library was compiled
sqlite3.paramstyle            # 'qmark'
```

`sqlite3.sqlite_version` is what you check against feature thresholds (window functions ≥ 3.25, upsert ≥ 3.24, `RETURNING` ≥ 3.35, `STRICT` ≥ 3.37, `FULL JOIN` ≥ 3.39). The old `sqlite3.version` was the module's own version, was always misleading, and was removed in Python 3.14.

### Invariants

1. `with conn:` manages the transaction. `closing(conn)` or `conn.close()` manages the resource. They are different jobs.
2. Every connection starts with foreign keys off. Enforcement belongs in one connection factory that everything routes through.
3. The engine version, not the Python version, determines which SQL features exist.

---

## Part 3 — Executing statements and binding parameters

### The three execute methods

```python
cur = conn.execute(sql, params)             # one statement, one param set
cur = conn.executemany(sql, seq_of_params)  # one statement, many param sets
conn.executescript(sql_script)              # many statements, NO parameters
```

`executescript` issues a `COMMIT` before running the script (in legacy transaction mode) and cannot take parameters. It is for schema files and migrations:

```python
with open("schema.sql") as f:
    conn.executescript(f.read())
```

### Parameter binding — the only safe way

```python
# qmark style — positional
conn.execute(
    "SELECT * FROM crop_input_streams WHERE crop_code = ? AND month_num = ?",
    ("WHEAT", 8),
)

# named style — dict
conn.execute(
    "SELECT * FROM crop_input_streams WHERE crop_code = :crop AND month_num = :m",
    {"crop": "WHEAT", "m": 8},
)
```

Named style is worth defaulting to once a query has more than about three parameters — positional tuples become a counting exercise.

**Never** build SQL with f-strings or `%` formatting from user input. Not for injection reasons alone: bound parameters also let SQLite reuse the prepared statement cache, and they handle quoting and type conversion for you.

Two things that **cannot** be parameterized, because they aren't values:

```python
# ILLEGAL — identifiers are not bindable
conn.execute("SELECT * FROM ?", ("crop_input_streams",))
conn.execute("SELECT * FROM t ORDER BY ?", ("crop_code",))
```

If a table name or sort column must be dynamic, validate it against an allow-list you control, then interpolate:

```python
SORTABLE = {"crop_code", "month_num", "amount_cents"}

def build_order(col: str, direction: str) -> str:
    if col not in SORTABLE:
        raise ValueError(f"unsortable column: {col}")
    return f"ORDER BY {col} {'DESC' if direction == 'desc' else 'ASC'}"
```

The allow-list is the security boundary. An `in` check against a literal set is fine; a regex "looks like an identifier" check is not.

### Dynamic `IN` lists

There's no way to bind a list to a single `?`. Generate the placeholders:

```python
codes = ["WHEAT", "CANOLA", "BARLEY"]
placeholders = ",".join("?" * len(codes))
cur = conn.execute(
    f"SELECT * FROM crop_input_streams WHERE crop_code IN ({placeholders})",
    codes,
)
```

The f-string here is safe because the interpolated text is derived only from `len()`, never from the values themselves. Guard against the empty list — `IN ()` is a syntax error.

### `executemany` for bulk loads

```python
rows = [(rev_id, "WHEAT", "seed", m, cents) for m, cents in monthly.items()]
conn.executemany(
    """INSERT INTO crop_input_streams
       (revision_id, crop_code, category, month_num, amount_cents)
       VALUES (?, ?, ?, ?, ?)""",
    rows,
)
```

Orders of magnitude faster than a Python loop of `execute`, because the statement is prepared once. It accepts any iterable, including a generator, so it streams. `cur.lastrowid` is not meaningful after `executemany` — if you need the generated keys, use `RETURNING` with individual executes, or query them back.

### Cursor result attributes

```python
cur = conn.execute("UPDATE t SET a = 1 WHERE b = ?", (2,))
cur.rowcount     # rows modified by the last DML statement
cur.lastrowid    # rowid of the last successful single-row INSERT
```

`rowcount` is `-1` for `SELECT` statements in `sqlite3` — it is a DML-only signal. To count query results you have to fetch them or run `COUNT(*)`.

### `RETURNING` from Python (SQLite ≥ 3.35)

```python
row = conn.execute(
    "INSERT INTO revisions (dataset_id, status) VALUES (?, 'draft') RETURNING revision_id, created_at",
    (dataset_id,),
).fetchone()
new_id = row["revision_id"]
```

Cleaner than `lastrowid` because it works for multi-column returns, for `UPDATE`/`DELETE`, and it doesn't depend on rowid semantics.

### Invariants

1. Values are bound; identifiers are validated against an allow-list. There is no third option.
2. `executescript` commits first and takes no parameters — it's for schema, never for data.
3. `rowcount` answers "how many rows did I change", never "how many rows did I get".

---

## Part 4 — Reading results: `description`, `row_factory`, fetching

### Default behaviour: tuples

```python
cur = conn.execute("SELECT crop_code, month_num, amount_cents FROM crop_input_streams")
for row in cur:
    print(row)          # ('WHEAT', 8, 125000)  — positional tuple
    print(row[0])
```

Positional access is brittle: adding a column to the `SELECT` list silently shifts every index downstream.

### `conn.row_factory = sqlite3.Row`

```python
conn.row_factory = sqlite3.Row
cur = conn.execute("SELECT crop_code, month_num, amount_cents FROM crop_input_streams")
row = cur.fetchone()

row["crop_code"]      # 'WHEAT'   — access by column name
row["CROP_CODE"]      # 'WHEAT'   — name matching is CASE-INSENSITIVE
row[0]                # 'WHEAT'   — positional access still works
row.keys()            # ['crop_code', 'month_num', 'amount_cents']
len(row)              # 3
list(row)             # ['WHEAT', 8, 125000]  — iterates VALUES, not keys
dict(row)             # {'crop_code': 'WHEAT', 'month_num': 8, 'amount_cents': 125000}
```

What it actually is: a C-implemented row type that keeps a reference to the cursor's `description` and maps names to positions. It is highly memory-efficient compared to a dict per row, supports both access styles, and is hashable and comparable.

What it is **not**: a dict. `sqlite3.Row` has no `.get()`, no `.items()`, no `.values()`, and does not support `**row` unpacking directly. `[trap]`

```python
row.get("crop_code")       # AttributeError
Model(**row)               # TypeError
Model(**dict(row))         # works
```

Note the asymmetry that catches people: iterating a `Row` yields **values**, but `dict(row)` produces a name→value mapping. `list(row)` and `list(dict(row))` return different things.

**Scope:** `row_factory` set on the Connection applies to cursors created afterwards. Set it immediately after connecting, in the connection factory. You can also set it per-cursor for a single query:

```python
cur = conn.cursor()
cur.row_factory = sqlite3.Row
```

### Custom row factories

The factory is any callable `(cursor, row_tuple) -> object`:

```python
def dict_factory(cursor, row):
    return {d[0]: v for d, v in zip(cursor.description, row)}

conn.row_factory = dict_factory
```

This is where `cursor.description` earns its keep — it's how a generic factory learns the column names. Use a plain dict factory when the rows go straight to JSON; use `sqlite3.Row` when you want cheap rows with named access and will convert selectively.

Directly to a Pydantic model:

```python
def model_factory(model_cls):
    def factory(cursor, row):
        cols = [d[0] for d in cursor.description]
        return model_cls(**dict(zip(cols, row)))
    return factory

cur = conn.cursor()
cur.row_factory = model_factory(MonthlyStream)
```

Convenient, but it moves validation errors into the fetch loop where the traceback is less informative. For a serving layer, converting explicitly at the boundary is usually easier to debug.

### Fetch methods

```python
cur.fetchone()      # next row, or None when exhausted
cur.fetchmany(100)  # list of up to n rows; [] when exhausted
cur.fetchall()      # list of ALL remaining rows — loads everything into memory
for row in cur: ... # streaming, lazy — the default choice for large results
cur.arraysize       # default n for fetchmany()
```

`fetchall()` on an unbounded query is how a serving layer runs out of memory. Iterate the cursor, or put a `LIMIT` in the SQL.

### Getting column names without fetching rows

```python
cur = conn.execute("SELECT * FROM crop_input_streams LIMIT 0")
cols = [d[0] for d in cur.description]
```

`LIMIT 0` executes the statement — so the column list reflects the real query, including expression aliases — without materializing rows. Useful for schema-conformance checks against a declared data contract.

### Invariants

1. Positional row access couples your Python to the column order of the `SELECT`. Name-based access via `row_factory` removes an entire class of silent bugs.
2. `sqlite3.Row` is a mapping-*like* sequence, not a dict. Convert with `dict(row)` at the boundary where you need dict semantics.
3. `cursor.description` is the only source of column names, and it exists only after execution.

---

## Part 5 — Transactions in Python `sqlite3`

This is the densest trap area in the module, because **the driver opens transactions you didn't write**.

### Legacy transaction control (the default, all versions)

With `isolation_level=""` (the default):

- Before an `INSERT`/`UPDATE`/`DELETE`/`REPLACE`, the driver implicitly issues `BEGIN` if no transaction is open.
- It does **not** implicitly begin before `SELECT`, DDL (`CREATE`/`DROP`/`ALTER`), or most PRAGMAs.
- Nothing is committed until you call `conn.commit()` (or exit a `with conn:` block).

Consequences:

```python
conn = sqlite3.connect(db)
conn.execute("INSERT INTO t (a) VALUES (1)")
conn.close()          # NO COMMIT — the insert is silently rolled back  [trap]
```

```python
conn.execute("CREATE TABLE t (a)")     # DDL: no implicit BEGIN, auto-committed
conn.execute("INSERT INTO t VALUES (1)")   # implicit BEGIN here
# ...crash...                          # insert lost, table survives
```

And the one that matters for schema setup:

```python
conn.execute("INSERT INTO t (a) VALUES (1)")   # opens a transaction
conn.execute("PRAGMA foreign_keys = ON")       # SILENT NO-OP inside a transaction  [trap]
```

### `isolation_level` values

| Value | Behaviour |
|---|---|
| `""` (default) | implicit `BEGIN` before DML; you call `commit()` |
| `"DEFERRED"` | same as `""`, explicit |
| `"IMMEDIATE"` | implicit `BEGIN IMMEDIATE` — takes the write lock at statement start |
| `"EXCLUSIVE"` | implicit `BEGIN EXCLUSIVE` |
| `None` | **autocommit** — no implicit transactions at all; you write `BEGIN` yourself |

`isolation_level=None` is the honest mode. It gives you exactly the transaction boundaries you wrote:

```python
conn = sqlite3.connect(db, isolation_level=None)
conn.execute("PRAGMA foreign_keys = ON")     # now reliably outside a transaction
conn.execute("BEGIN IMMEDIATE")
try:
    conn.execute("UPDATE ...")
    conn.execute("INSERT ...")
    conn.execute("COMMIT")
except Exception:
    conn.execute("ROLLBACK")
    raise
```

`BEGIN IMMEDIATE` deserves its own note. In deferred mode, a transaction that starts by reading takes a shared lock and then tries to upgrade to a write lock on its first write. If two connections both do this, neither can upgrade and one gets `SQLITE_BUSY` immediately — `busy_timeout` does **not** help, because backing off cannot resolve it. Any transaction that will write should start with `BEGIN IMMEDIATE`.

### `[3.12]` The `autocommit` attribute

Python 3.12 added a PEP 249-conformant mode:

```python
conn = sqlite3.connect(db, autocommit=False)   # proper transactional mode
conn = sqlite3.connect(db, autocommit=True)    # true autocommit
conn = sqlite3.connect(db, autocommit=sqlite3.LEGACY_TRANSACTION_CONTROL)  # old behaviour
```

With `autocommit=False`, a transaction is open at all times and `commit()`/`rollback()` start a new one — no implicit-BEGIN guessing. The legacy constant is the current default and remains so for compatibility. Worth adopting for new code on 3.12+; check `sys.version_info` before relying on it in anything shared.

### The `with conn:` block, precisely

```python
with conn:
    conn.execute("UPDATE ...")
    conn.execute("INSERT ...")
# exits normally -> COMMIT
# raises          -> ROLLBACK, exception propagates
# either way      -> connection stays OPEN
```

Nesting `with conn:` does **not** give you nested transactions — the inner block commits everything. SQLite's nesting primitive is `SAVEPOINT`:

```python
conn.execute("SAVEPOINT sp1")
try:
    conn.execute("...")
    conn.execute("RELEASE sp1")
except Exception:
    conn.execute("ROLLBACK TO sp1")
    conn.execute("RELEASE sp1")
    raise
```

### Invariants

1. Closing a connection with uncommitted work discards it. There is no implicit commit on close.
2. PRAGMAs that configure the connection must run before any transaction opens — which in legacy mode means before your first DML statement.
3. A transaction that intends to write should announce it with `BEGIN IMMEDIATE`. Deferred-then-upgrade is the SQLite deadlock.
4. `with conn:` is one transaction, no matter how deeply you nest it.

---

## Part 6 — Types, adapters, converters

### The default type mapping

| Python | SQLite |
|---|---|
| `None` | `NULL` |
| `int` | `INTEGER` |
| `float` | `REAL` |
| `str` | `TEXT` |
| `bytes` | `BLOB` |

Everything else raises `InterfaceError` unless you register an adapter. Note what's absent: `bool`, `datetime`, `date`, `Decimal`, `UUID`. `bool` works only because it's an `int` subclass — it round-trips as `0`/`1`, so a value written as `True` comes back as `1`. `[trap]`

### Adapters (Python → SQLite)

```python
import sqlite3, datetime
from decimal import Decimal

sqlite3.register_adapter(datetime.date, lambda d: d.isoformat())
sqlite3.register_adapter(datetime.datetime, lambda dt: dt.isoformat(" "))
sqlite3.register_adapter(Decimal, str)
```

Or define `__conform__` on your own classes. Adapters are **global to the process**, not per-connection.

`[3.12]` The built-in `date`/`datetime` adapters and converters are **deprecated**. On 3.12+ you'll get a `DeprecationWarning` when relying on them, and they'll eventually be removed. Register your own explicitly — which is better anyway, since the built-ins had no timezone handling.

### Converters (SQLite → Python)

Converters need `detect_types` to be enabled:

```python
sqlite3.register_converter("DATE", lambda b: datetime.date.fromisoformat(b.decode()))
sqlite3.register_converter("BOOLEAN", lambda b: b != b"0")

conn = sqlite3.connect(db, detect_types=sqlite3.PARSE_DECLTYPES)
```

- `PARSE_DECLTYPES` — looks at the **declared column type** in the schema and applies the matching converter. Only works for plain columns from a real table; an expression or aggregate has no declared type.
- `PARSE_COLNAMES` — looks for a type hint in the column *alias*: `SELECT ts AS "ts [DATE]"`. Works for expressions too.
- Combine with `|`.

Converter names are matched case-insensitively, and the converter receives `bytes`, always — decode it yourself.

### The pragmatic position

Converters are global, invisible at the call site, and silently inert on expressions and aggregates. For a serving layer, explicit conversion at the boundary is usually the better trade:

- store ISO-8601 `TEXT` for timestamps and parse in the Pydantic model,
- store integer minor units for money and divide in the response model,
- store `0`/`1` for booleans and let Pydantic coerce to `bool`.

Pydantic v2 already does all three coercions natively, which makes the model the natural conversion boundary. Registering converters *and* using Pydantic means two conversion layers with different failure modes.

### Invariants

1. Only five Python types round-trip without configuration. Everything else is a decision you have to make explicitly.
2. `bool` survives the write and does not survive the read. Convert at the model boundary.
3. `PARSE_DECLTYPES` reads the schema, not the value. Any computed column bypasses it.

---

## Part 7 — Exceptions

```
Exception
 └── sqlite3.Warning
 └── sqlite3.Error
      ├── InterfaceError        -- misuse of the driver itself
      └── DatabaseError
           ├── DataError            -- value out of range, bad type
           ├── OperationalError     -- locked DB, missing table, disk I/O, bad SQL at runtime
           ├── IntegrityError       -- constraint violation (FK, UNIQUE, CHECK, NOT NULL)
           ├── InternalError
           ├── ProgrammingError     -- wrong number of bindings, closed cursor, bad SQL syntax
           └── NotSupportedError
```

The two you handle by name:

```python
try:
    conn.execute("INSERT INTO revisions (dataset_id, status) VALUES (?, 'published')", (ds,))
except sqlite3.IntegrityError as exc:
    # UNIQUE / FK / CHECK / NOT NULL violation
    ...
except sqlite3.OperationalError as exc:
    # 'database is locked', 'no such table', 'disk I/O error'
    ...
```

`IntegrityError` is your constraint system reporting a caught bug — usually a 409 or 422 at the HTTP layer. `OperationalError` is an environment or contention problem — usually a 503, or a retry.

Distinguishing *which* constraint fired requires string inspection, since SQLite doesn't expose a structured error code through the standard attributes:

```python
msg = str(exc)
if "FOREIGN KEY constraint failed" in msg: ...
if "UNIQUE constraint failed: revisions.dataset_id" in msg: ...
```

Fragile, but the message does include the table and column for `UNIQUE`, `CHECK`, and `NOT NULL` violations, which is enough to build a useful error response. `[3.11]` `sqlite3.Error` also carries `.sqlite_errorcode` and `.sqlite_errorname` (e.g. `SQLITE_CONSTRAINT_UNIQUE`) — prefer those where available, they're structured and stable.

```python
except sqlite3.IntegrityError as exc:
    if exc.sqlite_errorname == "SQLITE_CONSTRAINT_FOREIGNKEY":
        ...
```

### Invariants

1. `IntegrityError` means the database refused a write your code should have prevented. It's a bug report, not an outage.
2. `OperationalError: database is locked` means contention or a missing `busy_timeout`, not corruption.
3. Never catch bare `sqlite3.Error` at a boundary where you'd swallow a `ProgrammingError` — that's your own SQL being wrong.

---

## Part 8 — Extending SQLite from Python

### User-defined scalar functions

```python
conn.create_function("cents_to_dollars", 1, lambda c: None if c is None else c / 100.0)
conn.execute("SELECT cents_to_dollars(amount_cents) FROM crop_input_streams")
```

Signature: `create_function(name, narg, func, *, deterministic=False)`. Pass `deterministic=True` when the function is pure — it's required if you want to use the function in an index expression or a partial index predicate, and it lets the planner cache results.

`narg=-1` accepts any number of arguments.

### User-defined aggregates

```python
class WeightedAvg:
    def __init__(self):
        self.num = 0.0
        self.den = 0.0
    def step(self, value, weight):
        if value is not None and weight:
            self.num += value * weight
            self.den += weight
    def finalize(self):
        return self.num / self.den if self.den else None

conn.create_aggregate("weighted_avg", 2, WeightedAvg)
```

### Collations

```python
conn.create_collation("NUMERIC_TEXT", lambda a, b: (int(a) > int(b)) - (int(a) < int(b)))
conn.execute("SELECT * FROM t ORDER BY code COLLATE NUMERIC_TEXT")
```

### The cost

Every call crosses the C↔Python boundary. A UDF inside a `WHERE` on a million-row scan is dramatically slower than the equivalent expression in SQL. Use them for logic that genuinely can't be expressed in SQL, not for convenience — and never as a substitute for a sargable predicate.

### Backup and dump

```python
# online backup — works on a live database, no locking of the source for readers
with sqlite3.connect("backup.db") as dest:
    conn.backup(dest, pages=0, progress=None)

# textual dump — schema + INSERTs, for diffing or version control
with open("dump.sql", "w") as f:
    for line in conn.iterdump():
        f.write(line + "\n")
```

`conn.backup()` is the correct way to snapshot a live SQLite database. Copying the file with `shutil.copy` while a writer is active can produce a corrupt copy, especially under WAL.

### Introspection

```python
conn.execute("SELECT name, sql FROM sqlite_master WHERE type = 'table'").fetchall()
conn.execute("PRAGMA table_info('crop_input_streams')").fetchall()
conn.execute("PRAGMA foreign_key_list('crop_input_streams')").fetchall()
conn.execute("PRAGMA index_list('crop_input_streams')").fetchall()
conn.execute("PRAGMA foreign_key_check").fetchall()     # [] means clean
conn.execute("PRAGMA integrity_check").fetchall()       # [('ok',)] means clean
conn.total_changes                                      # rows changed since connect
```

`PRAGMA foreign_key_check` and `integrity_check` returning empty/`ok` make good CI assertions after a migration or a bulk load.

---

## Part 9 — Connection lifecycle, threads, WAL

### Thread safety

`sqlite3` connections default to `check_same_thread=True`, which raises if a connection is used from a thread other than the one that created it. This is a Python-level guard, not a SQLite limitation.

Options, in order of preference:

1. **One connection per thread / per request.** Simplest and correct. SQLite connections are cheap to open — this is not Postgres, there's no network handshake or server-side session.
2. `check_same_thread=False` **plus** your own lock. Only if you're deliberately sharing one connection and serializing access yourself.
3. `threading.local()` holding a connection per thread.

`sqlite3.threadsafety` reports the underlying library's mode: `1` = module-level serialization only, `3` = fully serialized, connections shareable. Python 3.11+ reports `3` on typical builds.

### WAL mode

```python
conn.execute("PRAGMA journal_mode = WAL")      # persistent; set once at setup
conn.execute("PRAGMA synchronous = NORMAL")    # per connection
```

WAL changes the concurrency story substantially:

- Readers do not block the writer; the writer does not block readers.
- There is still exactly **one writer at a time**, database-wide.
- The database becomes three files: `db`, `db-wal`, `db-shm`. Deployment, backup, and `.gitignore` all need to know this.
- WAL does not work well over network filesystems (NFS, some container volume drivers).

For a read-heavy serving layer over a periodically-rebuilt SQLite file — a forecast API over a batch-built database — WAL plus short read connections is close to ideal, and the write contention that dominates SQLite discussions mostly doesn't apply.

### Rebuild-and-swap deployments

When the database is a build artifact rather than a mutable store, the safest publish is an atomic rename:

```python
import os, tempfile, shutil

tmp = build_database_to_temp()          # write to forecast.db.tmp
os.replace(tmp, "forecast.db")          # atomic on POSIX, same filesystem
```

Existing open connections continue to see the old inode until they reconnect — which means the serving layer needs a way to cycle connections after a publish, or it will serve stale data indefinitely. Options: open a fresh connection per request (simplest), or hold a generation counter that a lifespan-managed pool checks.

### Invariants

1. Opening a SQLite connection is cheap. Per-request connections are a legitimate default, not a compromise.
2. One writer, always. Concurrency planning for SQLite is about *write* serialization, not connection count.
3. Under WAL, the database is three files. Anything that copies, backs up, or ships the database must handle all three — or use `conn.backup()`.

---

# Part B — FastAPI

## Part 10 — The app object and lifespan

```python
from fastapi import FastAPI

app = FastAPI(
    title="GrowLytics API",
    version="0.3.0",
    description="Agricultural cash-flow forecast serving layer",
    summary="Short one-liner shown in the docs",
    docs_url="/docs",              # None to disable Swagger UI
    redoc_url="/redoc",            # None to disable ReDoc
    openapi_url="/openapi.json",   # None to disable the schema entirely
    root_path="/api",              # when mounted behind a proxy prefix
    lifespan=lifespan,             # see below
    default_response_class=JSONResponse,
    dependencies=[Depends(verify_key)],   # applied to EVERY route
)
```

`title` and `version` are not decoration — they populate the generated OpenAPI document, which is what `/docs` renders and what client generators consume. Bump `version` when the contract changes.

In production, `docs_url=None, redoc_url=None, openapi_url=None` if the API isn't public.

### Lifespan — startup and shutdown

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    app.state.db_path = settings.db_path
    verify_schema(settings.db_path)
    yield
    # shutdown
    ...

app = FastAPI(lifespan=lifespan)
```

Everything before `yield` runs once at startup; everything after runs at shutdown. This replaces `@app.on_event("startup")` / `@app.on_event("shutdown")`, which are deprecated.

Good uses: validating that the database file exists and matches the expected schema version, warming caches, opening a shared read pool. Bad use: opening one long-lived SQLite connection to share across all requests — see Part 21.

`app.state` is a plain namespace for app-scoped objects. Reachable in handlers via `request.app.state`.

---

## Part 11 — Path operations

```python
@app.get("/forecasts/{forecast_id}",
         response_model=ForecastOut,
         status_code=200,
         tags=["forecasts"],
         summary="Fetch one forecast",
         description="Longer text; the docstring is used if this is omitted",
         response_description="The forecast envelope",
         responses={404: {"model": ErrorOut}},
         deprecated=False)
def get_forecast(forecast_id: int) -> ForecastOut:
    """Markdown here shows up in the docs when `description` is not set."""
    ...
```

Methods: `@app.get`, `.post`, `.put`, `.patch`, `.delete`, `.head`, `.options`, `.trace`. Also `@app.api_route(path, methods=["GET", "POST"])`.

### Route matching order

Routes are matched **in declaration order**, first match wins. Static segments must be declared before parameterized ones that could shadow them:

```python
@app.get("/forecasts/summary")       # must come FIRST
def summary(): ...

@app.get("/forecasts/{forecast_id}") # otherwise "summary" is parsed as forecast_id
def get_one(forecast_id: int): ...
```

With `forecast_id: int` the shadowed case would 422 rather than silently misroute — but with `str` it fails silently. Declare specific routes first regardless. `[trap]`

### `def` vs `async def` — the decision that matters most with SQLite

| You write | FastAPI runs it | Use when |
|---|---|---|
| `async def` | on the event loop | all I/O in the handler is `await`-able |
| `def` | in a threadpool (anyio, ~40 workers) | the handler does **blocking** I/O |

`sqlite3` is a **blocking** driver. Calling it inside `async def` blocks the event loop and stalls every other request in the process.

```python
# WRONG — blocks the event loop for the duration of the query
@app.get("/forecasts")
async def list_forecasts():
    return conn.execute("SELECT ...").fetchall()

# RIGHT — FastAPI runs this in a threadpool, the loop stays free
@app.get("/forecasts")
def list_forecasts():
    return conn.execute("SELECT ...").fetchall()
```

The rule: **plain `def` for anything touching `sqlite3`.** This is the opposite of the instinct that "async is faster", and it's the single highest-impact FastAPI decision in a SQLite-backed service. `[trap]`

Threadpool handlers plus per-request connections also compose correctly — each thread gets its own connection, so `check_same_thread` never fires.

---

## Part 12 — Parameters: path, query, body, header

FastAPI infers the source from the signature: a name in the path template is a path param; a Pydantic model is the body; anything else scalar is a query param.

### The modern `Annotated` style

```python
from typing import Annotated
from fastapi import Query, Path, Body, Header, Depends

@app.get("/streams")
def list_streams(
    crop_code: Annotated[str, Query(min_length=2, max_length=8, pattern=r"^[A-Z]+$")],
    month: Annotated[int | None, Query(ge=1, le=12)] = None,
    limit: Annotated[int, Query(ge=1, le=500)] = 100,
    offset: Annotated[int, Query(ge=0)] = 0,
    tags: Annotated[list[str], Query()] = [],       # ?tags=a&tags=b
):
    ...
```

`Annotated[T, Query(...)]` is the current idiom and is preferred over the older `limit: int = Query(100, ge=1)` default-value style — it keeps the type separate from the metadata, works with non-FastAPI tooling, and lets you reuse the annotation:

```python
Limit = Annotated[int, Query(ge=1, le=500)]

def list_streams(limit: Limit = 100): ...
```

### Path parameters

```python
@app.get("/forecasts/{forecast_id}/months/{month_num}")
def get_month(
    forecast_id: Annotated[int, Path(ge=1)],
    month_num: Annotated[int, Path(ge=1, le=12)],
): ...

@app.get("/files/{file_path:path}")     # :path captures slashes
def read_file(file_path: str): ...
```

Path params are always required. Validation failures return 422 with a structured error body — you get range checking for free rather than writing `if month_num < 1: raise`.

### Body

```python
class RevisionIn(BaseModel):
    dataset_id: int
    label: str

@app.post("/revisions")
def create(payload: RevisionIn): ...                     # model -> body

@app.post("/revisions")
def create(payload: RevisionIn, dry_run: bool = False):  # model -> body, scalar -> query
    ...

@app.post("/x")
def x(a: Annotated[int, Body()], b: Annotated[int, Body()]):
    ...   # forces scalars into the body as {"a": 1, "b": 2}

@app.post("/y")
def y(payload: Annotated[RevisionIn, Body(embed=True)]):
    ...   # expects {"payload": {...}} instead of {...}
```

### Headers, cookies, forms, files

```python
from fastapi import Cookie, Form, File, UploadFile

user_agent: Annotated[str | None, Header()] = None      # maps User-Agent (underscore->hyphen)
session: Annotated[str | None, Cookie()] = None
name: Annotated[str, Form()]                            # needs python-multipart
upload: Annotated[UploadFile, File()]
```

### Validation constraints reference

| Applies to | Constraints |
|---|---|
| numbers | `gt`, `ge`, `lt`, `le`, `multiple_of` |
| strings | `min_length`, `max_length`, `pattern` |
| any | `default`, `alias`, `title`, `description`, `deprecated`, `include_in_schema`, `examples` |

---

## Part 13 — Pydantic v2 models

```python
from pydantic import BaseModel, Field, ConfigDict, field_validator, model_validator, computed_field

class StreamOut(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,     # allow construction from objects with attributes
        populate_by_name=True,    # accept field name as well as alias
        extra="forbid",           # reject unknown keys (default is "ignore")
        frozen=False,
        str_strip_whitespace=True,
    )

    crop_code: str = Field(..., min_length=2, max_length=8)
    month_num: int = Field(..., ge=1, le=12)
    amount_cents: int = Field(..., alias="amountCents")
    note: str | None = None

    @computed_field
    @property
    def amount(self) -> float:
        return self.amount_cents / 100

    @field_validator("crop_code")
    @classmethod
    def upper(cls, v: str) -> str:
        return v.upper()

    @model_validator(mode="after")
    def check_consistency(self):
        if self.month_num > 6 and self.amount_cents < 0:
            raise ValueError("negative amounts not allowed in H2")
        return self
```

`Field(...)` — the literal ellipsis — marks a field required with no default.

### v1 → v2 method renames

| Pydantic v1 | Pydantic v2 |
|---|---|
| `.dict()` | `.model_dump()` |
| `.json()` | `.model_dump_json()` |
| `Model.parse_obj(d)` | `Model.model_validate(d)` |
| `Model.parse_raw(s)` | `Model.model_validate_json(s)` |
| `.copy()` | `.model_copy()` |
| `Config` class | `model_config = ConfigDict(...)` |
| `orm_mode = True` | `from_attributes=True` |
| `@validator` | `@field_validator` |
| `@root_validator` | `@model_validator` |
| `Model.schema()` | `Model.model_json_schema()` |

`model_dump()` options worth knowing: `exclude_none=True`, `exclude_unset=True`, `by_alias=True`, `mode="json"` (produces JSON-safe primitives rather than Python objects).

### Separate models per direction

```python
class StreamBase(BaseModel):
    crop_code: str
    month_num: int
    amount_cents: int

class StreamCreate(StreamBase):
    pass                             # what the client may send

class StreamOut(StreamBase):
    stream_id: int                   # server-assigned, never accepted as input
    created_at: datetime
```

Separating input and output models is what stops a client from setting `stream_id` or `created_at`. A single shared model always ends up either accepting fields it shouldn't or exposing fields it shouldn't.

### Constructing models from `sqlite3.Row`

```python
row = conn.execute("SELECT ... WHERE id = ?", (i,)).fetchone()
StreamOut.model_validate(dict(row))    # dict() is required — Row is not a mapping to Pydantic
```

`from_attributes=True` does **not** help here: `sqlite3.Row` exposes columns by *subscript*, not by attribute. `dict(row)` is the conversion. `[trap]`

---

## Part 14 — Responses, status codes, errors

### Status codes

```python
from fastapi import status

@app.post("/revisions", status_code=status.HTTP_201_CREATED)
def create(...): ...

@app.delete("/revisions/{rid}", status_code=status.HTTP_204_NO_CONTENT)
def delete(rid: int) -> None: ...
```

Use the `status` constants rather than integer literals — they document themselves and typo to a `NameError` instead of a wrong response.

### `response_model`

```python
@app.get("/streams", response_model=list[StreamOut], response_model_exclude_none=True)
def list_streams(): 
    return [dict(r) for r in cur.fetchall()]     # dicts are fine; FastAPI validates them
```

`response_model` does three jobs: it validates the outgoing data, it **filters out any field not declared on the model**, and it documents the response in OpenAPI. The filtering is a security property — a `SELECT *` that picks up an internal column won't leak it if the response model doesn't declare it.

Options: `response_model_exclude_none`, `response_model_exclude_unset`, `response_model_include={...}`, `response_model_exclude={...}`, `response_model_by_alias`.

The return type annotation (`-> list[StreamOut]`) also works as a response model in recent FastAPI and is the tidier form; `response_model=` wins if both are present.

### Errors

```python
from fastapi import HTTPException

raise HTTPException(status_code=404, detail="Forecast not found")
raise HTTPException(status_code=409, detail={"error": "duplicate", "key": code},
                    headers={"X-Error": "duplicate"})
```

Custom exception classes plus a handler keep HTTP concerns out of the data layer:

```python
class RevisionNotFound(Exception):
    def __init__(self, rid: int): self.rid = rid

@app.exception_handler(RevisionNotFound)
def handle_missing(request: Request, exc: RevisionNotFound):
    return JSONResponse(status_code=404, content={"detail": f"revision {exc.rid} not found"})
```

This is the pattern that lets a repository/query module raise domain errors while the HTTP layer decides on status codes. Mapping table for a SQLite-backed API:

| Condition | Status |
|---|---|
| row not found | 404 |
| validation failed (FastAPI does this) | 422 |
| `IntegrityError` — UNIQUE violation | 409 Conflict |
| `IntegrityError` — FK / CHECK violation | 422 or 400 |
| `OperationalError: database is locked` | 503 + `Retry-After` |
| unexpected `sqlite3.Error` | 500 |

### Response classes

```python
from fastapi.responses import JSONResponse, ORJSONResponse, PlainTextResponse, StreamingResponse, FileResponse

@app.get("/export.csv")
def export():
    def rows():
        for r in conn.execute("SELECT ..."):
            yield ",".join(map(str, r)) + "\n"
    return StreamingResponse(rows(), media_type="text/csv")
```

`StreamingResponse` over a cursor iterator is the right shape for large exports — it never materializes the result set. Note the connection must outlive the generator, which a `yield`-based dependency handles correctly.

Returning a `Response` object directly bypasses `response_model` validation and serialization entirely.

---

## Part 15 — Dependency injection

```python
from fastapi import Depends
from typing import Annotated

def get_db() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(settings.db_path, timeout=5.0)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    try:
        yield conn
    finally:
        conn.close()

DB = Annotated[sqlite3.Connection, Depends(get_db)]

@app.get("/streams")
def list_streams(db: DB, limit: int = 100):
    return [dict(r) for r in db.execute("SELECT ... LIMIT ?", (limit,))]
```

The `DB = Annotated[...]` alias is the idiom worth adopting: declare the dependency once, use `db: DB` in every handler.

### Dependency semantics

- **`yield` dependencies** run setup before the handler, cleanup after the response is generated. This is the correct shape for connections, transactions, and file handles.
- **Caching:** within a single request, the same dependency callable is called **once** and the result reused. `Depends(get_db, use_cache=False)` opts out. This means two handlers' worth of sub-dependencies share one connection per request, which is what you want.
- **Nesting:** dependencies can depend on dependencies, arbitrarily deep.
- **Class dependencies:** any callable works, including a class (its `__init__` signature becomes the dependency's parameters).
- **Route/router/app-level:** `dependencies=[Depends(f)]` on a decorator, router, or the app runs `f` for its effect without injecting the value — for auth checks and the like.
- **Overrides in tests:** `app.dependency_overrides[get_db] = get_test_db`.

`get_db` here is a plain `def` generator, which FastAPI correctly runs in the threadpool alongside a `def` handler.

### A write-scoped variant

```python
def get_db_write() -> Iterator[sqlite3.Connection]:
    conn = connect(settings.db_path)
    try:
        conn.execute("BEGIN IMMEDIATE")
        yield conn
        conn.execute("COMMIT")
    except Exception:
        conn.execute("ROLLBACK")
        raise
    finally:
        conn.close()
```

Transaction-per-request for writes. Because cleanup runs *after* the response is produced, a handler that returns successfully commits, and any exception — including an `HTTPException` — rolls back.

---

## Part 16 — Routers and project structure

```python
# routers/streams.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/streams",
    tags=["streams"],
    dependencies=[Depends(verify_key)],
    responses={404: {"description": "Not found"}},
)

@router.get("/")
def list_streams(db: DB): ...

@router.get("/{stream_id}")
def get_stream(stream_id: int, db: DB): ...
```

```python
# main.py
from fastapi import FastAPI
from routers import streams, revisions

app = FastAPI(title="GrowLytics API", version="0.3.0", lifespan=lifespan)
app.include_router(streams.router)
app.include_router(revisions.router, prefix="/v1")   # prefixes compose
```

A layout that scales without ceremony:

```
app/
  main.py            -- FastAPI(), middleware, include_router
  config.py          -- Settings (pydantic-settings)
  db.py              -- connect(), get_db(), get_db_write()
  models.py          -- Pydantic request/response models
  queries.py         -- SQL + row->dict functions; NO FastAPI imports
  routers/
    streams.py
    revisions.py
  schema.sql
```

The load-bearing rule is that `queries.py` imports nothing from FastAPI. It takes a connection and parameters, returns rows or raises domain errors. That keeps the SQL testable without an HTTP client and makes the layer reusable from batch scripts.

---

## Part 17 — Middleware and CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],   # explicit list, or ["*"]
    allow_origin_regex=r"https://.*\.example\.com",
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    max_age=600,
)
```

`allow_origins=["*"]` and `allow_credentials=True` are **mutually incompatible** — the browser rejects a wildcard origin on a credentialed request. If you need cookies or `Authorization`, enumerate the origins. `[trap]`

CORS applies to browser callers only; it is not a security control, it's a browser policy declaration. Vite dev servers, preview builds, and production all need to be listed or matched.

Custom middleware:

```python
@app.middleware("http")
async def add_timing(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = f"{time.perf_counter() - start:.4f}"
    return response
```

Middleware ordering: the last one added is the outermost. Middleware runs for every request including 404s, unlike dependencies.

---

## Part 18 — Settings and configuration

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="GROWLYTICS_",
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    db_path: str = "data/forecast.db"
    cors_origins: list[str] = ["http://localhost:5173"]
    read_only: bool = True
    log_level: str = "INFO"

settings = Settings()      # reads GROWLYTICS_DB_PATH, GROWLYTICS_READ_ONLY, ...
```

Precedence, highest first: init arguments → environment variables → `.env` file → field defaults.

Nested settings via `__` delimiter: `GROWLYTICS_DB__PATH` maps to `settings.db.path` with `env_nested_delimiter="__"`.

For testability, prefer injecting settings as a cached dependency over importing a module-level singleton:

```python
from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()

# in tests: app.dependency_overrides[get_settings] = lambda: Settings(db_path=":memory:")
```

---

## Part 19 — Testing

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_list_streams():
    r = client.get("/streams", params={"crop_code": "WHEAT"})
    assert r.status_code == 200
    assert r.json()[0]["cropCode"] == "WHEAT"
```

`TestClient` runs the app in-process — no server, no network — and triggers lifespan when used as a context manager:

```python
with TestClient(app) as client:      # lifespan startup/shutdown run
    ...
```

### Test database fixture

```python
import pytest, sqlite3

@pytest.fixture
def db_path(tmp_path):
    path = tmp_path / "test.db"
    conn = sqlite3.connect(path)
    conn.executescript(open("app/schema.sql").read())
    conn.executescript(open("tests/fixtures/seed.sql").read())
    conn.close()
    return str(path)

@pytest.fixture
def client(db_path):
    def _get_db():
        conn = sqlite3.connect(db_path)
        conn.row_factory = sqlite3.Row
        conn.execute("PRAGMA foreign_keys = ON")
        try:
            yield conn
        finally:
            conn.close()

    app.dependency_overrides[get_db] = _get_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

A file-backed `tmp_path` database is usually better than `:memory:` here: an in-memory database is per-connection, so anything that opens a second connection sees an empty database. If you do want in-memory, use a shared-cache URI and hold one connection open for the test's duration.

---

## Part 20 — Serving

```bash
uvicorn app.main:app --reload                          # development
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4   # production-ish
```

```python
# programmatic
import uvicorn
uvicorn.run("app.main:app", host="127.0.0.1", port=8000, reload=True)
```

`--reload` is development only — it watches files and restarts. `--workers N` forks N processes.

**Workers and SQLite:** N worker processes means N independent sets of connections against one database file. Reads scale fine under WAL. Writes still serialize on the single database write lock, so `busy_timeout` matters more as worker count rises. For a read-mostly serving layer over a batch-built file, multiple workers are a straightforward win; for a write-heavy workload, they mostly manufacture lock contention.

Behind a reverse proxy, set `--root-path /api` (or `root_path=` on the app) so the generated OpenAPI URLs are correct.

---

# Part C — The seam

## Part 21 — SQLite + FastAPI integration recipes

### The baseline: per-request connection via DI

```python
# db.py
import sqlite3
from typing import Iterator, Annotated
from fastapi import Depends
from .config import settings

def connect(read_only: bool = True) -> sqlite3.Connection:
    if read_only:
        conn = sqlite3.connect(f"file:{settings.db_path}?mode=ro", uri=True, timeout=5.0)
    else:
        conn = sqlite3.connect(settings.db_path, timeout=5.0)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA busy_timeout = 5000")
    return conn

def get_db() -> Iterator[sqlite3.Connection]:
    conn = connect(read_only=True)
    try:
        yield conn
    finally:
        conn.close()

def get_db_write() -> Iterator[sqlite3.Connection]:
    conn = connect(read_only=False)
    try:
        conn.execute("BEGIN IMMEDIATE")
        yield conn
        conn.execute("COMMIT")
    except Exception:
        conn.execute("ROLLBACK")
        raise
    finally:
        conn.close()

DB = Annotated[sqlite3.Connection, Depends(get_db)]
DBWrite = Annotated[sqlite3.Connection, Depends(get_db_write)]
```

Why per-request rather than a shared connection: SQLite connections are cheap to open, per-request connections sidestep `check_same_thread` entirely under the threadpool, and each request gets clean transaction state. A single app-wide connection shared across threadpool workers requires `check_same_thread=False` plus your own lock, and serializes all reads for no benefit.

### Rows to response models

```python
# queries.py — no FastAPI imports
def fetch_streams(conn, crop_code: str, limit: int, offset: int) -> list[dict]:
    cur = conn.execute(
        """SELECT stream_id, crop_code, month_num, amount_cents
           FROM crop_input_streams
           WHERE crop_code = :crop
           ORDER BY month_num, stream_id
           LIMIT :limit OFFSET :offset""",
        {"crop": crop_code, "limit": limit, "offset": offset},
    )
    return [dict(r) for r in cur]

# routers/streams.py
@router.get("/", response_model=list[StreamOut])
def list_streams(db: DB, crop_code: str, limit: int = 100, offset: int = 0):
    return fetch_streams(db, crop_code, limit, offset)
```

Returning plain dicts and letting `response_model` validate them is the least-ceremony path. The model still filters unexpected columns and documents the contract.

The `ORDER BY month_num, stream_id` tiebreaker is not decoration — without a total order, `LIMIT/OFFSET` pagination can repeat and skip rows.

### Mapping SQLite errors to HTTP

```python
from fastapi import Request
from fastapi.responses import JSONResponse
import sqlite3

@app.exception_handler(sqlite3.IntegrityError)
def integrity_handler(request: Request, exc: sqlite3.IntegrityError):
    name = getattr(exc, "sqlite_errorname", "")
    if name == "SQLITE_CONSTRAINT_UNIQUE":
        return JSONResponse(409, {"detail": "resource already exists"})
    if name == "SQLITE_CONSTRAINT_FOREIGNKEY":
        return JSONResponse(422, {"detail": "referenced record does not exist"})
    return JSONResponse(400, {"detail": "constraint violation"})

@app.exception_handler(sqlite3.OperationalError)
def operational_handler(request: Request, exc: sqlite3.OperationalError):
    if "locked" in str(exc):
        return JSONResponse(503, {"detail": "busy, retry"}, headers={"Retry-After": "1"})
    raise exc      # anything else is a real bug — let it 500 loudly
```

Deliberately do **not** blanket-map every `OperationalError` to a friendly message. `no such table` is a deployment bug and should be loud.

### Not found

```python
@router.get("/{stream_id}", response_model=StreamOut)
def get_stream(stream_id: int, db: DB):
    row = db.execute(
        "SELECT stream_id, crop_code, month_num, amount_cents FROM crop_input_streams WHERE stream_id = ?",
        (stream_id,),
    ).fetchone()
    if row is None:
        raise HTTPException(404, f"stream {stream_id} not found")
    return dict(row)
```

`fetchone()` returning `None` is the only "not found" signal you get. Every single-row lookup needs this check — forgetting it produces `TypeError: 'NoneType' object is not subscriptable` from somewhere further down. `[trap]`

### Aggregate endpoints and the NULL-sum problem

```python
row = db.execute(
    "SELECT COALESCE(SUM(amount_cents), 0) AS total FROM crop_input_streams WHERE crop_code = ?",
    (crop_code,),
).fetchone()
return {"total_cents": row["total"]}
```

`SUM` over zero rows is `NULL` in SQL, which arrives in Python as `None`, which then fails validation against `total_cents: int`. `COALESCE` in the SQL is the right fix — not `or 0` in Python, which would also swallow a legitimate zero from a different code path.

### Avoiding N+1

```python
# BAD: one query per crop
for crop in crops:
    rows = db.execute("SELECT ... WHERE crop_code = ?", (crop,)).fetchall()

# GOOD: one query, group in Python
placeholders = ",".join("?" * len(crops))
rows = db.execute(f"SELECT crop_code, ... WHERE crop_code IN ({placeholders})", crops).fetchall()

grouped = defaultdict(list)
for r in rows:
    grouped[r["crop_code"]].append(dict(r))
```

SQLite queries are cheap relative to a network round trip, so N+1 hurts less here than with a client/server database — but it still turns a single index scan into N statement preparations, and it becomes a real problem as N grows with the data.

### Money at the boundary

Store integer cents, convert once, in the response model:

```python
class StreamOut(BaseModel):
    amount_cents: int = Field(exclude=True)      # kept for computation, not serialized

    @computed_field
    @property
    def amount(self) -> float:
        return self.amount_cents / 100
```

Keeping the integer as the stored and transported form, with the division happening at exactly one place, means no accumulated float error and no ambiguity about units in the API contract.

### Schema-version guard at startup

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    conn = connect(read_only=True)
    try:
        version = conn.execute("PRAGMA user_version").fetchone()[0]
        if version != EXPECTED_SCHEMA_VERSION:
            raise RuntimeError(f"schema version {version}, expected {EXPECTED_SCHEMA_VERSION}")
        missing = verify_expected_tables(conn)
        if missing:
            raise RuntimeError(f"missing tables: {missing}")
    finally:
        conn.close()
    yield

app = FastAPI(lifespan=lifespan)
```

`PRAGMA user_version` is a free integer slot in the database header — the standard place to stamp a schema version. Failing loudly at startup beats failing per-request with `no such column`.

### Streaming a large export

```python
@router.get("/export.csv")
def export(db: DB):
    def rows():
        cur = db.execute("SELECT crop_code, month_num, amount_cents FROM crop_input_streams")
        yield ",".join(d[0] for d in cur.description) + "\n"
        for r in cur:
            yield ",".join(str(v) for v in r) + "\n"
    return StreamingResponse(rows(), media_type="text/csv",
                             headers={"Content-Disposition": 'attachment; filename="streams.csv"'})
```

Note `cur.description` supplying the header row — the column names come from the query itself, so they can't drift out of sync with the data.

The generator is consumed *after* the handler returns, so the connection must still be open. A `yield`-based dependency guarantees that, because its `finally` runs after the response completes. A `with closing(connect()) as conn:` inside the handler would **not** — the connection would be closed before the first row is streamed. `[trap]`

### Health check

```python
@app.get("/health")
def health(db: DB):
    db.execute("SELECT 1").fetchone()
    return {"status": "ok", "sqlite": sqlite3.sqlite_version}
```

A health check that doesn't touch the database only proves the process is running.

---

## Part 22 — Traps ledger

| # | Trap | Why | Fix |
|---|---|---|---|
| 1 | `with sqlite3.connect(...)` leaves the connection open | `__exit__` commits, doesn't close | `contextlib.closing()`, or a factory with `try/finally` |
| 2 | Writes vanish after the script ends | no implicit commit on `close()` | `conn.commit()` or `with conn:` |
| 3 | `PRAGMA foreign_keys = ON` does nothing | silently ignored inside an open transaction | run it immediately after connecting, before any DML |
| 4 | FK constraints not enforced at all | pragma is per-connection and off by default | put it in the one connection factory |
| 5 | `conn.description` is `AttributeError` | `description` is on the Cursor | `conn.execute(...).description` |
| 6 | `row.get(...)` / `Model(**row)` fails | `sqlite3.Row` is not a dict | `dict(row)` |
| 7 | `list(row)` gives values, `dict(row)` gives pairs | Row iterates values | be explicit about which you want |
| 8 | `bool` comes back as `int` | SQLite has no boolean type | coerce in the Pydantic model |
| 9 | `SQLITE_BUSY` immediately, despite `busy_timeout` | deferred transaction upgrading to write | `BEGIN IMMEDIATE` for write transactions |
| 10 | Whole API stalls under light load | blocking `sqlite3` call inside `async def` | use plain `def` handlers |
| 11 | `TypeError: 'NoneType' is not subscriptable` | `fetchone()` returned `None` | check for `None`, raise 404 |
| 12 | Endpoint returns `None` for a total | `SUM` of zero rows is NULL | `COALESCE(SUM(x), 0)` in the SQL |
| 13 | `/forecasts/summary` hits the `{id}` route | routes match in declaration order | declare static paths first |
| 14 | CORS fails only with credentials | `allow_origins=["*"]` + `allow_credentials=True` is invalid | enumerate origins |
| 15 | Streaming response yields nothing / closed cursor | connection closed before the generator ran | use a `yield` dependency, not a `with` inside the handler |
| 16 | In-memory test DB looks empty | `:memory:` is per-connection | file in `tmp_path`, or shared-cache URI |
| 17 | `lastrowid` is wrong after a bulk insert | not meaningful after `executemany` | `RETURNING`, or query back |
| 18 | `rowcount` is `-1` after a SELECT | it's a DML-only signal | `COUNT(*)` or fetch |
| 19 | `DeprecationWarning` on datetime columns | default adapters deprecated in 3.12 | register your own adapters |
| 20 | Stale data served after a database rebuild | open connections hold the old inode | per-request connections, or cycle on publish |
| 21 | `sqlite3.version` looks far too low | that's the module version, not the engine | `sqlite3.sqlite_version` |
| 22 | Table name won't bind as a parameter | identifiers aren't values | allow-list, then interpolate |

---

## Part 23 — Version and compatibility table

| Feature | Requires |
|---|---|
| `sqlite3.Error.sqlite_errorcode` / `.sqlite_errorname` | Python 3.11 |
| `Connection.autocommit` attribute | Python 3.12 |
| Default datetime adapters deprecated | Python 3.12 |
| `sqlite3.version` removed | Python 3.14 |
| `Connection.backup()` | Python 3.7 |
| `create_function(..., deterministic=True)` | Python 3.8, SQLite ≥ 3.8.3 |
| Window functions | SQLite ≥ 3.25 |
| `FILTER` on aggregates | SQLite ≥ 3.30 |
| `UPDATE ... FROM` | SQLite ≥ 3.33 |
| `RETURNING` | SQLite ≥ 3.35 |
| `ALTER TABLE DROP COLUMN` | SQLite ≥ 3.35 |
| CTE `MATERIALIZED` hints | SQLite ≥ 3.35 |
| `STRICT` tables | SQLite ≥ 3.37 |
| `->` / `->>` JSON operators | SQLite ≥ 3.38 |
| `RIGHT` / `FULL JOIN`, `IS DISTINCT FROM` | SQLite ≥ 3.39 |
| `lifespan=` (replacing `on_event`) | FastAPI 0.93+ |
| `Annotated` dependency style | FastAPI 0.95+ |
| Pydantic v2 support | FastAPI 0.100+ |
| Return-type annotation as response model | FastAPI 0.89+ |

Check the engine at runtime rather than assuming: `sqlite3.sqlite_version_info >= (3, 35, 0)`.

---

## Appendix A — Conventions worth keeping

- One `connect()` factory. Every connection in the codebase — app, scripts, tests, notebooks — goes through it, so pragmas and `row_factory` can't drift.
- `queries.py` imports nothing from FastAPI. SQL is testable without HTTP.
- Named parameters (`:name`) once a query has more than three bindings.
- `dict(row)` at the boundary; `sqlite3.Row` everywhere inside.
- Separate `*In` and `*Out` Pydantic models. Never one shared model.
- Plain `def` handlers for anything that touches SQLite.
- `status.HTTP_*` constants over integer literals.
- Every single-row fetch is followed by a `None` check.
- Every `ORDER BY` used for pagination ends with a unique tiebreaker column.
- Stamp `PRAGMA user_version` on every schema change and assert it at startup.

## Appendix B — To add as this evolves

- Authentication: `OAuth2PasswordBearer`, API-key dependencies, scopes
- `BackgroundTasks` vs. a real task queue
- Structured logging and request IDs via middleware
- OpenAPI customization: `openapi_extra`, custom `operation_id`, client generation
- WebSockets and Server-Sent Events for live forecast updates
- `aiosqlite` — what it does and doesn't buy you over `def` handlers
- Migration tooling for SQLite (hand-rolled `user_version` steps vs. Alembic)
- Connection pooling if per-request connections ever stop being enough
- `sqlite3.Connection.set_authorizer` / `set_progress_handler` for guardrails
- FTS5 from Python
- Profiling: `set_trace_callback`, timing middleware, `EXPLAIN QUERY PLAN` in tests