# 1. Persistent storage

Here's experiment A broken into notebook cells. Each cell does one thing, and each has a prediction to write down *before* you run it.

## How `:memory:` fits in

`sqlite3.connect()` takes one argument that says *where the database lives*:

- `sqlite3.connect("experiment.db")` opens, or creates, a **file** on disk.
- `sqlite3.connect(":memory:")` creates a **brand-new database in RAM** that belongs to that connection.

So `:memory:` isn't an option you add to a file connection; it replaces the filename. Each call to `connect()` returns an independent connection object, so "two connections" just means calling it twice and keeping both in separate variables.

## Cell 0: setup and instruments

```python
import os, sys, sqlite3

print("Python:", sys.version)
print("SQLite library:", sqlite3.sqlite_version)

DB_PATH = "experiment_a.db"
if os.path.exists(DB_PATH):
    os.remove(DB_PATH)          # start every run from a clean slate
print("DB file will be at:", os.path.abspath(DB_PATH))

def tables(conn):
    rows = conn.execute("SELECT name FROM sqlite_master WHERE type = 'table'").fetchall()
    return [r[0] for r in rows]
```

`tables()` is a tiny helper so every later cell can ask "what tables does this connection see?" in one call. `sqlite_master` is SQLite's own catalog table, so that query is SQLite behavior, not something the Python module adds. Note it as SQLite-specific: other databases expose their catalogs differently.

## Cell 1: create, commit, close

> Prediction: after `CREATE TABLE`, is `in_transaction` True or False?

```python
conn = sqlite3.connect(DB_PATH)
print("isolation_level:", repr(conn.isolation_level))
print("autocommit:", getattr(conn, "autocommit", "n/a (Python < 3.12)"))

print("in_transaction before CREATE:", conn.in_transaction)
conn.execute("CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT NOT NULL)")
print("in_transaction after CREATE:", conn.in_transaction)

conn.commit()
conn.close()
print("File exists after close:", os.path.exists(DB_PATH), "| size:", os.path.getsize(DB_PATH), "bytes")
```

`isolation_level`, `autocommit` and `in_transaction` are all properties of the **Python module**, not of SQLite. Mark them that way in your notes.

## Cell 2: reconnect and inspect

> Prediction: does the new connection see the table?

```python
conn = sqlite3.connect(DB_PATH)
print("Tables after reconnect:", tables(conn))
conn.close()
```

This answers "does closing a connection destroy the database?" for the file case.

## Cell 3: same thing without `commit()`

> Prediction: if you never call `commit()`, does the table survive a close and reopen? Base it on what Cell 1 showed for `in_transaction`.

```python
os.remove(DB_PATH)

conn = sqlite3.connect(DB_PATH)
conn.execute("CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT NOT NULL)")
print("in_transaction after CREATE:", conn.in_transaction)
conn.close()                    # no commit

conn = sqlite3.connect(DB_PATH)
print("Tables after reopen (no commit):", tables(conn))
conn.close()
```

If the result surprises you, look back at Cell 1's `in_transaction` output. The two together tell you whether the Python module opened a transaction around the `CREATE TABLE`.

## Cell 4: two connections to the same file

> Prediction: do two connections to the same file see the same tables?

```python
f1 = sqlite3.connect(DB_PATH)
f2 = sqlite3.connect(DB_PATH)

f1.execute("CREATE TABLE IF NOT EXISTS clients (id INTEGER PRIMARY KEY, name TEXT NOT NULL)")
f1.commit()

print("f1 sees:", tables(f1))
print("f2 sees:", tables(f2))

f1.close()
f2.close()
```

This is the baseline to compare against in the next cell.

## Cell 5: two `:memory:` connections

> Prediction: does `m2` see the table created through `m1`? After `m1` closes and you reconnect, is the table still there?

```python
m1 = sqlite3.connect(":memory:")
m1.execute("CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT NOT NULL)")
print("m1 sees:", tables(m1))

m2 = sqlite3.connect(":memory:")
print("m2 sees:", tables(m2))

m1.close()
m1 = sqlite3.connect(":memory:")
print("m1 after close + reconnect sees:", tables(m1))

m1.close()
m2.close()
```

Comparing Cells 4 and 5 answers "what changes when using `:memory:`?" Concretely: is "the database" the same thing for two file connections as for two memory connections?

## Afterwards

Under the cells, write down three observations:

1. What survives a close, for a file and for memory.
2. Whether the table survived without `commit()`, and what `in_transaction` showed. That tells you whether `CREATE TABLE` ran inside a transaction in this mode.
3. What `:memory:` means for testing. It gives you a fresh, isolated database on every `connect()`, which is useful for test fixtures. But anything that needs two connections to the *same* database, like experiments B and F, can't use plain `:memory:` and has to use a file.

For later reference (tier 3): SQLite does support a *shared* in-memory database through a special URI (`file:...?mode=memory&cache=shared` with `uri=True`), but you don't need it for these labs. A file per experiment is simpler and behaves like the real thing.

Bring back your observations from Cells 1 and 3 in particular. They determine how you'll handle transactions explicitly in the rest of the experiments.

# 2. Visibility across connections

Those observations are accurate, with one value worth double-checking before B, because B depends on it.

## Check: was `autocommit` really `True`?

On Python 3.12+, the default value of `conn.autocommit` is **`-1`**, the constant `sqlite3.LEGACY_TRANSACTION_CONTROL`, not `True`. These are three different modes:

| `autocommit` | What the module does |
|---|---|
| `-1` (legacy, the default) | Implicit `BEGIN` before INSERT/UPDATE/DELETE; never before SELECT or CREATE |
| `True` | Never issues `BEGIN`; every statement commits on its own |
| `False` | A transaction is always open; `commit()` and `rollback()` immediately open the next one |

For `CREATE TABLE`, legacy and `True` look identical: no transaction, and the table persists without `commit()`. That's what you saw. They **diverge on INSERT**: legacy wraps it in a transaction, `True` doesn't. So if you saw `-1`, your conclusion "nothing needs commit" would be wrong for experiment B. Re-print it once to be sure.

A small extra check for the `False` mode: look at `in_transaction` *before* the `CREATE TABLE` too, right after connecting. "A transaction is always open" may already show there.

## Recommendation: use `autocommit=False` for the rest of Lab 05

- **It matches the other databases' drivers.** psycopg (PostgreSQL) and pyodbc (Azure SQL) both default to "autocommit off": a transaction is open until you explicitly commit or roll back. Code written against that model ports; code relying on legacy mode's implicit rules doesn't.
- **Transaction boundaries become explicit in your own code.** You decide them, rather than reconstructing which statements Python wrapped.
- **DDL is transactional too.** Your observation 2 shows a failed schema setup can be rolled back like anything else.

Mark this in your notes as a `sqlite3`-specific setting. The *concept* ("explicit commit, explicit rollback") is portable; the parameter name isn't. **(Lesson candidate: pick the driver mode that matches the target databases' default transaction model, so the transaction boundaries you test are the ones production will have.)**

## Experiment B, cell by cell

### Cell 0: clean setup

```python
DB_PATH = "experiment_b.db"
if os.path.exists(DB_PATH):
    os.remove(DB_PATH)

setup = sqlite3.connect(DB_PATH, autocommit=False)
setup.execute("CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT NOT NULL)")
setup.commit()
setup.close()

def names(conn):
    return [r[0] for r in conn.execute("SELECT name FROM clients").fetchall()]
```

### Cell 1: two connections

```python
c1 = sqlite3.connect(DB_PATH, autocommit=False, timeout=1)
c2 = sqlite3.connect(DB_PATH, autocommit=False, timeout=1)
print("c1 in_transaction:", c1.in_transaction, "| c2 in_transaction:", c2.in_transaction)
```

`timeout=1` limits how long a statement waits if the database is busy to one second instead of the default five. If any later step seems to pause, that's a clue, not a hang.

### Cell 2: insert through c1, read through c1

> Prediction: does c1 see its own uncommitted Farm A?

```python
c1.execute("INSERT INTO clients (name) VALUES (?)", ("Farm A",))
print("c1 sees:", names(c1))
```

Two mechanics here:

- **The trailing comma in `("Farm A",)` is required.** Without it, `("Farm A")` is just a string, not a tuple. Python then treats each character as a separate parameter, and you get an "Incorrect number of bindings" error with 6 bindings. That's a Python language trap, tier 1.
- **`?` is the placeholder style for `sqlite3` and pyodbc; psycopg uses `%s`.** The principle is portable: values travel separately from the SQL text, never through string formatting, so quoting and SQL injection can't happen. The placeholder spelling isn't portable.

### Cell 3: read through c2

> Prediction: does c2 see Farm A? After this SELECT, what is `c2.in_transaction`?

```python
print("c2 sees:", names(c2))
print("c2 in_transaction:", c2.in_transaction)
```

### Cell 4: commit c1

> Prediction: does this commit succeed? Base it on what Cell 3 showed about c2's transaction state.

```python
try:
    c1.commit()
    print("c1 commit succeeded")
except sqlite3.Error as err:
    print("c1 commit failed:", type(err).__name__, "-", err)
    print("c1 in_transaction:", c1.in_transaction)
```

If it fails, don't fix anything yet. Note what failed, what the message says, and whether c1 still has its transaction (and therefore Farm A) pending.

### Cell 5: end c2's transaction, then retry

> Prediction: after c2 ends its transaction, does c1's commit go through? Does c2 then see Farm A?

```python
c2.rollback()          # c2 only read, so commit() would work equally well here
print("c2 in_transaction after rollback:", c2.in_transaction)

if c1.in_transaction:
    c1.commit()
    print("c1 commit retried and succeeded")

print("c2 sees now:", names(c2))
```

### Cell 6: cleanup

```python
c1.close()
c2.close()
```

## Questions to answer from what you observe

1. Can a connection see its own uncommitted writes? Can another connection see them?
2. What did c2's SELECT leave behind, and what effect did that have on c1's commit?
3. If Cell 4 failed: was c1's change lost, or still pending and retryable? That distinction is exactly "confirmed non-commit" from the Lab 04 contract.
4. What does this imply for a backend that reuses one long-lived connection and only ever reads through it?

## Optional comparison

Rerun Cells 1–5 with both connections in the **default** legacy mode (drop `autocommit=False`). Compare c2's `in_transaction` after its SELECT, and whether c1's commit behaves differently. That contrast answers "what happens to an already-started read transaction?" more sharply than either mode alone.

Further extension, tier 3: SQLite also has a WAL journal mode (`PRAGMA journal_mode=WAL`) that changes how readers and writers interact. PostgreSQL and Azure SQL behave somewhat more like WAL than like SQLite's default. Worth a look only after the default mode makes sense.

Bring back the result of Cell 4 especially. It decides how the Lab 05 service should manage connections.

# 3. Rollback and IDs

Your conclusions are right. One refinement in vocabulary will make the snapshot-isolation connection exact.

## What actually happened in B

c2 didn't lock the whole database. Its SELECT, inside an open transaction, took a **shared lock**: "I'm reading; nobody may change what I'm reading until I finish." c1 could still *write*, since its INSERT succeeded, but **committing** required an exclusive lock, and that has to wait until every shared lock is released.

So SQLite's default mode gave c2 a stable view of the data **by blocking writers**. Snapshot isolation (MVCC) gives readers the same guarantee a different way: it **keeps the old version** of the data for the reader, so the writer can commit without waiting. Two strategies for one problem: block, or keep versions. PostgreSQL uses versions. SQLite's WAL mode is version-like too. Azure SQL Database, as far as I know, enables a snapshot-style read-committed mode by default, but verify that when you get there. **(Lesson candidate: a reader's consistent view can be bought by blocking writers or by keeping old versions. The database's choice determines whether reads can stall writes.)**

## The practical rule for the Lab 05 service

In `autocommit=False` mode, **any read opens a transaction that stays open until you commit or roll back.** A connection that served a GET and was then left idle keeps blocking every writer's commit. That rules out "one long-lived connection, reused for reads." Each request should open its connection (or take one from a pool), do its work, explicitly end the transaction, even after a pure read, and release it.

Your Cell 4 result also matters for the contract. `database is locked` at commit left c1's change **pending, not committed**. If the backend rolls back at that point, the outcome is a confirmed non-commit caused by transient contention. That's precisely the profile of a safe-retry error. Experiment F will test it directly.

## Experiment C, cell by cell

This version tests three things: rollback, whether a rolled-back insert uses up an ID, and whether a deleted ID can be **reused**. You'll run it twice, once per primary-key declaration, by changing one variable.

### Cell 0: helpers

```python
DB_PATH = "experiment_c.db"

def fresh(pk_decl):
    if os.path.exists(DB_PATH):
        os.remove(DB_PATH)
    conn = sqlite3.connect(DB_PATH, autocommit=False)
    conn.execute(f"CREATE TABLE clients (id {pk_decl}, name TEXT NOT NULL)")
    conn.commit()
    return conn

def rows(conn):
    return conn.execute("SELECT id, name FROM clients ORDER BY id").fetchall()

def seq(conn):
    try:
        return conn.execute("SELECT name, seq FROM sqlite_sequence").fetchall()
    except sqlite3.OperationalError:
        return "no sqlite_sequence table"
```

**Why an f-string is acceptable here when Cell 2 of B insisted on `?`:** placeholders can only carry *values*, never SQL structure such as a column type or table name. Inserting `pk_decl` with an f-string is safe only because it's a constant you wrote yourself. It must never come from user input. That's a tier-2 distinction.

`sqlite_sequence` is an internal SQLite table that exists only when some table uses `AUTOINCREMENT`. It records the highest ID ever issued. It's SQLite-specific, so note it as such.

### Cell 1: start with one committed client

```python
PK_DECL = "INTEGER PRIMARY KEY"        # second run: "INTEGER PRIMARY KEY AUTOINCREMENT"

conn = fresh(PK_DECL)
conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm A",))
conn.commit()
print("rows:", rows(conn), "| sequence:", seq(conn))
```

### Cell 2: insert Farm B, then roll back

> Prediction: what ID does Farm B get? After rollback and reopen, is it gone?

```python
cur = conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm B",))
print("Farm B was assigned id:", cur.lastrowid)
print("before rollback:", rows(conn), "| sequence:", seq(conn))

conn.rollback()
print("after rollback:", rows(conn), "| sequence:", seq(conn))

conn.close()
conn = sqlite3.connect(DB_PATH, autocommit=False)
print("after reopen:", rows(conn))
```

`cur.lastrowid` is the ID SQLite just assigned. A portability note: `lastrowid` isn't reliably available across drivers. The portable-ish alternative is `INSERT ... RETURNING id`, supported by SQLite 3.35+ and PostgreSQL, while SQL Server uses `OUTPUT` instead. Check your `sqlite3.sqlite_version` if you want to try it.

### Cell 3: does the rolled-back ID get reused?

> Prediction: Farm C's ID. Is it Farm B's old ID, or the next one after it?

```python
conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm C",))
conn.commit()
print("rows:", rows(conn), "| sequence:", seq(conn))
```

### Cell 4: delete the highest ID, then insert

> Prediction: after deleting the client with the highest ID, does Farm D receive that same ID?

```python
max_id = conn.execute("SELECT MAX(id) FROM clients").fetchone()[0]
conn.execute("DELETE FROM clients WHERE id = ?", (max_id,))
conn.commit()
print("deleted id", max_id, "→ rows:", rows(conn), "| sequence:", seq(conn))

conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm D",))
conn.commit()
print("after inserting Farm D:", rows(conn), "| sequence:", seq(conn))
```

### Cell 5: close, then repeat

```python
conn.close()
```

Change `PK_DECL` to `"INTEGER PRIMARY KEY AUTOINCREMENT"` and rerun Cells 1–5. Compare the two runs side by side, especially the sequence column.

## Questions to answer from the two runs

1. **Rollback:** is Farm B gone in both runs? Did its attempt leave any trace (an ID consumed, a sequence change)?
2. **ID reuse:** in which declaration can a deleted client's ID be handed to a *new* client?
3. **The stale-tab case from Lab 04:** tab B still shows the deleted client and sends `PATCH /clients/<that id>`. Under each declaration, what would the request do: return 404 `CLIENT_NOT_FOUND`, or silently rename a different client? Which declaration does G-01 ("immutable numeric primary key") therefore require?
4. **Portability:** PostgreSQL's sequences are deliberately *not* rolled back, so a failed insert there does consume an ID. Given that databases disagree about gaps, what should the application assume about ID sequences? That's your original question: "why shouldn't application identity depend on uninterrupted ID sequences?"

Question 3 is the one that turns this from a curiosity into a schema decision, so bring back what you find there especially.

# 4. Rollback of multiple changes

You're asking the right question, and there's one distinction that makes it straightforward.

## The distinction: an ID nobody saw vs an ID somebody holds

Look again at your rollback run. Farm B's attempted ID was given to Farm C afterward, in both runs. Is that a problem? No. Farm B was **never committed**, so no other connection, API response, or browser tab ever saw that ID. No one can be holding a reference to it, so handing it to Farm C can't make anyone act on the wrong client.

The deletion case is different. Farm C was **committed**: it was returned by the API, listed in the UI, and maybe still on screen in a stale tab. Reassigning its ID to Farm D is what creates your failure scenario.

So the property the application actually needs is narrower than "IDs never repeat":

> **Once an ID has been committed, it is never assigned to a different record.**

Whether uncommitted IDs are reused (SQLite) or discarded (PostgreSQL) doesn't matter, because nobody can reference them either way. That's why the rollback difference doesn't need accounting for: it falls outside the property you depend on.

## What the application should assume about IDs

Only two things:

1. **Unique.**
2. **Never reassigned once committed.**

And deliberately nothing else. Specifically, don't assume:

- **Gapless:** PostgreSQL rollbacks leave gaps, and SQL Server's `IDENTITY` can jump by large amounts (I believe 1000) after a server restart.
- **Next = max + 1:** never compute an ID in the application. The database assigns it, and the application reads it back.
- **Ordered by commit time:** with concurrent sessions, session A can get ID 5, session B ID 6, and B commit first. ID order roughly follows creation, but not reliably.
- **Meaningful:** don't derive counts or anything else from ID values.

Treat the ID as an **opaque handle**: a value you store, send, and compare for equality, nothing more. The frontend already does this: `editClient` checks `data.id === clientId` and nothing else.

## How that becomes portable

The *requirement* (unique, never reassigned) is portable. The *declaration that provides it* is database-specific, and that's exactly what your "clearly mark `sqlite3`-specific parts" plan is for:

| Database | Declaration that guarantees no reuse | Notes |
|---|---|---|
| SQLite | `INTEGER PRIMARY KEY AUTOINCREMENT` | Plain `INTEGER PRIMARY KEY` reuses deleted max IDs, as you observed |
| PostgreSQL | Identity column or `serial` | Gaps are normal; no reuse unless someone resets the sequence |
| Azure SQL | `IDENTITY` | Gaps are normal; no reuse unless someone reseeds it |

The "unless someone resets it" caveat is operational: a manual reseed, or a table truncation in some databases, can bring old IDs back. That's worth one line in the schema notes so a future maintenance task doesn't break G-01.

This is a decision worth recording with a date, something like *"Lab 05 schema uses `AUTOINCREMENT`: G-01 requires committed IDs never be reassigned; plain `INTEGER PRIMARY KEY` reuses the highest deleted ID."*

**(Lesson candidate: define the identity contract by what references depend on: unique and never reassigned once committed. Gaps, ordering, and the fate of uncommitted IDs are implementation details that differ per database and should not be relied on.)**

One tier-3 note for later: if production uses 64-bit IDs, JavaScript's `Number` can only represent integers exactly up to 2^53. Real IDs won't reach that soon, but it's why some APIs send large IDs as strings.

## Experiment D, cell by cell

In `autocommit=False` mode, `in_transaction` will be `True` almost all the time, so it won't tell you much here. Use a **second connection as the observer** instead: it shows you what is committed at each step.

### Cell 0: setup with two committed clients

```python
DB_PATH = "experiment_d.db"
conn = fresh("INTEGER PRIMARY KEY AUTOINCREMENT")   # helper from experiment C
conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm A",))
conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm B",))
conn.commit()

observer = sqlite3.connect(DB_PATH, autocommit=False, timeout=1)

def look(label):
    print(f"{label:<28} writer sees: {rows(conn)}")
    print(f"{'':<28} observer sees: {rows(observer)}")
    observer.rollback()     # end the observer's read transaction so it never blocks the writer

look("start")
```

The `observer.rollback()` inside `look` applies what you learned in B: a read in this mode leaves a transaction open, so the observer ends it after every look. Without that, the observer would block the writer's commits later in this experiment.

### Cell 1: update, then delete, without committing

> Prediction: what does each connection see after each statement?

```python
conn.execute("UPDATE clients SET name = ? WHERE id = ?", ("Farm A (renamed)", 1))
look("after UPDATE")

conn.execute("DELETE FROM clients WHERE id = ?", (2,))
look("after DELETE")
```

### Cell 2: roll back

> Prediction: are both changes undone, or only the most recent one?

```python
conn.rollback()
look("after rollback")
```

### Cell 3: the boundary variant

This answers "what determines which statements belong to a transaction?" by moving the boundary.

> Prediction: with a commit between the two statements, what does the rollback undo?

```python
conn.execute("UPDATE clients SET name = ? WHERE id = ?", ("Farm A (renamed)", 1))
conn.commit()
look("after UPDATE + commit")

conn.execute("DELETE FROM clients WHERE id = ?", (2,))
conn.rollback()
look("after DELETE + rollback")
```

### Cell 4: cleanup

```python
conn.close()
observer.close()
```

## Questions to answer

1. Did the rollback undo both statements in Cell 2? What did the observer see throughout Cell 1?
2. In Cell 3, what decided which statement survived? Put it in one sentence. That sentence is the rule for where your service code must place `commit()`.
3. **Where this matters in Client CRUD:** each mutation looks like a single statement today, but spec §10 also requires an **audit record** for every successful mutation. Inserting a client and writing its audit row is a two-statement operation. What has to be true about those two statements for the "failed transactions leave no partial changes" criterion to hold? And where, in terms of the service function, does the transaction boundary therefore belong?

Question 3 is what connects this experiment to the actual service design, so bring your answer back with your observations.

# 5. Failed statement

Your observations are right, and so is the principle: group statements into transactions deliberately, so each failure leaves either all of a workflow or none of it. Two of the conclusions need sharpening, though, because they'll shape the service code.

## "All or nothing, so it's safe to retry": not quite

Atomicity guarantees the outcome is **one of two states**, all or nothing. It doesn't tell you **which** one happened. Suppose the connection drops while `COMMIT` is in flight, or the commit succeeds and the HTTP response is lost. The database is in exactly one of the two states, but your code doesn't know which. Retrying is safe only when you **know** the result was "nothing," for example because your code caught the error and rolled back itself.

**(Lesson candidate: atomicity removes partial states, not uncertainty. It makes outcomes binary; it doesn't make them known. Retry is safe only after a confirmed rollback.)**

A small terminology point: `commit` is less like a checkpoint and more like closing a door. It ends the transaction permanently, and `rollback` can only undo the transaction that is still open. Partial undo inside an open transaction does exist (`SAVEPOINT`), but that's tier 3 and you don't need it here.

## Explicit `BEGIN`/`COMMIT` in SQL is less portable than it looks

**The SQL spelling differs across databases.** SQLite uses `BEGIN` (plus variants like `BEGIN IMMEDIATE`), PostgreSQL accepts `BEGIN` or `START TRANSACTION`, and SQL Server uses `BEGIN TRANSACTION`. Writing it in SQL moves the dialect problem into your query text rather than removing it.

**It conflicts with how drivers manage transactions.** In `autocommit=False` mode, `sqlite3` already has a transaction open. Execute `BEGIN` on top of it and SQLite refuses with "cannot start a transaction within a transaction." To write your own `BEGIN`, the driver has to be in true autocommit mode, at which point you're managing what the driver would otherwise manage.

The portable layer is actually the **Python DB-API**: `conn.commit()` and `conn.rollback()` exist with the same meaning in `sqlite3`, psycopg, and pyodbc. What you really want to be portable is **where the transaction boundary sits** in your service code, and the DB-API makes that possible without any SQL keywords.

So the pattern I'd aim for: one small helper (a context manager, for example) that represents a unit of work. On entry it provides a connection; on success it commits; on any exception it rolls back and re-raises. Every service operation runs inside exactly one of these. If you later need SQLite's `BEGIN IMMEDIATE` (taking the write lock up front, which is relevant to experiment F), it lives inside that helper and is marked SQLite-specific. **(Lesson candidate: make the transaction boundary portable by placing it in one helper at the service-operation level. The boundary's location is the design; the keywords are an implementation detail.)**

Applied to the audit question: "insert client" and "write audit row" both happen inside one unit of work, with one commit at the end. Either both persist, or, after the rollback, neither does.

## Experiment E, cell by cell

This version includes the refinement from earlier: a **valid** insert first, *then* the duplicate, in the same transaction. That's the only way to tell "the failed statement was undone" apart from "the whole transaction was undone."

### Cell 0: setup with a named UNIQUE constraint

```python
DB_PATH = "experiment_e.db"
if os.path.exists(DB_PATH):
    os.remove(DB_PATH)

conn = sqlite3.connect(DB_PATH, autocommit=False, timeout=1)
conn.execute("""
    CREATE TABLE clients (
        id   INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        CONSTRAINT uq_clients_name UNIQUE (name)
    )
""")
conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm A",))
conn.commit()

observer = sqlite3.connect(DB_PATH, autocommit=False, timeout=1)

def look(label):
    print(f"{label:<32} writer: {rows(conn)}")
    print(f"{'':<32} observer: {rows(observer)}")
    observer.rollback()

look("start")
```

### Cell 1: valid insert, then duplicate

> Predictions: which exception class? Does the message mention `uq_clients_name`, or something else? After the error, is Farm B still visible to the writer? Is `in_transaction` still `True`?

```python
conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm B",))
look("after valid insert")

try:
    conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm A",))
except sqlite3.Error as err:
    print("type:", type(err).__name__)
    print("message:", err)
    print("sqlite_errorname:", getattr(err, "sqlite_errorname", None))
    print("sqlite_errorcode:", getattr(err, "sqlite_errorcode", None))

print("in_transaction:", conn.in_transaction)
look("after failed duplicate")
```

These printed fields are the raw input for your SQLite translation function. Copy the output into a markdown cell verbatim.

### Cell 2: can the connection still be used?

> Prediction: does a new statement run normally after the error?

```python
conn.execute("INSERT INTO clients (name) VALUES (?)", ("Farm C",))
look("after another valid insert")
```

### Cell 3: an open transaction after an error blocks others

This simulates a service that caught the error and then did nothing: no commit, no rollback.

> Prediction: can the observer write while the writer's transaction is still open?

```python
try:
    observer.execute("INSERT INTO clients (name) VALUES (?)", ("Farm Z",))
    observer.commit()
    print("observer write succeeded")
except sqlite3.Error as err:
    print("observer write failed:", type(err).__name__, "-", err)
    observer.rollback()
```

### Cell 4: commit, and see what persists

> Prediction: after `commit()`, which of Farm B and Farm C persist? Think about what this means if a service caught the duplicate error and something later committed this connection.

```python
conn.commit()
look("after commit")
```

### Cell 5: duplicate on UPDATE (AT-35)

> Prediction: does a colliding UPDATE fail the same way? Is the original row unchanged afterwards?

```python
target_id = conn.execute("SELECT id FROM clients WHERE name = ?", ("Farm C",)).fetchone()[0]
try:
    conn.execute("UPDATE clients SET name = ? WHERE id = ?", ("Farm A", target_id))
except sqlite3.Error as err:
    print("type:", type(err).__name__, "| message:", err, "| errorname:", getattr(err, "sqlite_errorname", None))
conn.rollback()
look("after failed UPDATE + rollback")
```

### Cell 6: cleanup

```python
conn.close()
observer.close()
```

## Questions to answer

1. **Statement or transaction:** after the duplicate failed, what was undone, the statement or the whole transaction? (Cells 1 and 4 together answer this.)
2. **Danger:** given that answer, what must a service function do whenever *any* exception occurs inside a unit of work? What would go wrong in Cells 3 and 4 if it didn't?
3. **Translation input:** which fields would your SQLite translation function use to recognize "unique violation," and which part tells you *which* constraint failed? Did the message name `uq_clients_name`, or the column?
4. **Portability:** PostgreSQL behaves differently after an error: the whole transaction enters an aborted state and refuses further statements until you roll back. Does your answer to question 2 work for both databases?

Questions 2 and 3 feed directly into the unit-of-work helper and the translation function, so bring those back with your observations.

# 6. Two writers

Your conclusions are right. One detail in the error fields is mixed up, and it matters for the translation function.

## Which field holds what

From your description, two different fields got combined. For a unique violation they should look like this (check against your printed output):

| Field | Value | Use |
|---|---|---|
| `err.sqlite_errorname` | `SQLITE_CONSTRAINT_UNIQUE` | The violation **kind**, as readable text |
| `err.sqlite_errorcode` | `2067` | The same kind as a number (SQLite's base constraint code 19, extended with the "unique" detail) |
| `str(err)` | `UNIQUE constraint failed: clients.name` | The only place that says **which column** |

So the SQLite translation function should get the kind from `sqlite_errorname` (or the code), which is stable and needs no parsing. It should only parse the message when it needs to know *which* unique constraint failed. Note also what the message names: the **column** (`clients.name`), not `uq_clients_name`. On SQLite, the constraint name you chose never appears in the error. That's fine for Client CRUD, since `clients` will have one unique column (the normalized key), but it's SQLite-specific and belongs in your notes.

## Two levels of atomicity

Your 5-statement example is exactly right, and it's worth naming what you saw:

- **Statement atomicity is the database's job.** The failed INSERT left no trace. Even a single statement affecting many rows is undone entirely if it fails partway.
- **Transaction atomicity is your job.** The database undid only the failed statement and left the earlier ones pending. Turning "statement 3 failed" into "nothing happened" requires your code to call `rollback()`.

**(Lesson candidate: the database makes each statement atomic; only your rollback makes the workflow atomic. A caught exception followed by a commit persists a partial transaction.)**

On "gather error information, then roll back": the order is fine, and the information isn't lost by rolling back, because it lives on the exception object, not the connection. One tier-2 detail for the unit-of-work helper: `rollback()` itself can raise, for example if the connection has died. If it does, the helper should not let that second error replace the original one, because the original is what explains what happened.

## Experiment F, cell by cell

You'll time each attempt, because "how long did it wait?" is part of what you're observing.

### Cell 0: setup

```python
import time

DB_PATH = "experiment_f.db"
if os.path.exists(DB_PATH):
    os.remove(DB_PATH)

setup = sqlite3.connect(DB_PATH, autocommit=False)
setup.execute("""
    CREATE TABLE clients (
        id   INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        CONSTRAINT uq_clients_name UNIQUE (name)
    )
""")
setup.execute("INSERT INTO clients (name) VALUES (?)", ("Farm A",))
setup.commit()
setup.close()

w1 = sqlite3.connect(DB_PATH, autocommit=False, timeout=0.5)
w2 = sqlite3.connect(DB_PATH, autocommit=False, timeout=0.5)

def attempt(conn, label, sql, params):
    start = time.perf_counter()
    try:
        conn.execute(sql, params)
        print(f"{label}: statement OK after {time.perf_counter() - start:.2f}s")
        return True
    except sqlite3.Error as err:
        print(f"{label}: {type(err).__name__} after {time.perf_counter() - start:.2f}s")
        print(f"  message: {err} | errorname: {getattr(err, 'sqlite_errorname', None)}")
        return False
```

### Cell 1: first writer writes, no commit

```python
attempt(w1, "w1 insert Farm B", "INSERT INTO clients (name) VALUES (?)", ("Farm B",))
```

### Cell 2: second writer attempts a write

> Predictions: does w2's INSERT fail immediately, after about 0.5 s, or succeed? Which exception class and errorname? Compare with experiment E.

```python
attempt(w2, "w2 insert Farm C", "INSERT INTO clients (name) VALUES (?)", ("Farm C",))
print("w2 in_transaction:", w2.in_transaction)
w2.rollback()
```

### Cell 3: release, then retry deliberately

> Prediction: once w1 commits, does the same w2 operation succeed?

```python
w1.commit()
if attempt(w2, "w2 retry Farm C", "INSERT INTO clients (name) VALUES (?)", ("Farm C",)):
    w2.commit()
print("rows:", rows(w2))
w2.rollback()     # end w2's read transaction
```

### Cell 4: contention hiding a business outcome

This variant matters most for the contract. w1 inserts a name *without committing*, and w2 tries to insert the **same** name.

> Predictions: does w2 get a unique violation or a lock error while w1 is uncommitted? After w1 commits, what does w2's retry get? And if w1 had rolled back instead, what would the retry get?

```python
attempt(w1, "w1 insert Farm D", "INSERT INTO clients (name) VALUES (?)", ("Farm D",))
attempt(w2, "w2 insert Farm D (while w1 open)", "INSERT INTO clients (name) VALUES (?)", ("Farm D",))
w2.rollback()

w1.commit()
attempt(w2, "w2 retry Farm D (after w1 commit)", "INSERT INTO clients (name) VALUES (?)", ("Farm D",))
w2.rollback()
```

To answer the last prediction directly, rerun the cell with `w1.rollback()` in place of `w1.commit()` and a new name such as `"Farm E"`.

### Cell 5 (optional): both read before writing

This is the case `BEGIN IMMEDIATE` exists for. Both connections read first, which takes shared locks, then both try to write.

> Prediction: does the second writer wait for its timeout, or fail at once?

```python
rows(w1); rows(w2)       # both now hold read transactions
attempt(w1, "w1 insert Farm F", "INSERT INTO clients (name) VALUES (?)", ("Farm F",))
attempt(w2, "w2 insert Farm G", "INSERT INTO clients (name) VALUES (?)", ("Farm G",))
try:
    w1.commit(); print("w1 commit OK")
except sqlite3.Error as err:
    print("w1 commit:", type(err).__name__, "-", err)
w1.rollback(); w2.rollback()
```

### Cell 6: cleanup

```python
w1.close()
w2.close()
```

## Questions to answer

1. **What "database is locked" means:** what was w2 waiting for, and what did the timeout control?
2. **Contention vs business rejection:** compare the exception class and errorname with experiment E's. In Cell 4, while w1 was uncommitted, did w2 learn whether "Farm D" was a duplicate? What does that say about what a lock error tells you?
3. **The contract:** suppose the backend catches the lock error, rolls back, and responds. Is that a confirmed non-commit? Is retrying safe, given that Cell 4's retry could come back as success *or* as a duplicate? Should the retry happen inside the backend (briefly, before responding) or be offered to the user? And what HTTP status and code would you register for it?
4. **Lost responses:** if w2's commit succeeded but the HTTP response never reached the browser, what does the Lab 04 contract say the frontend must conclude?

After F, you'll have observed every input the service needs: storage, visibility, transaction boundaries, failure behavior, and contention. Bring back your answers to questions 2 and 3 especially. They decide whether the retry allowlist gets its first entry.

# 6b. conclusion and next experiment proposal

Your answers to the key questions are right. Contention says "you can't try yet," not "no," and that's precisely why it qualifies for retry. A few details need sharpening, and one of your answers mixes up two parts of the contract.

## Refinements

**A lock blocks writers, not everyone.** In B, c2 could still *read* while c1 held an uncommitted write. "Database is locked" means another connection holds the lock *you* need, which for a write is the write lock. Reads usually proceed in SQLite's default mode; what they block is someone else's *commit*.

**The timeout isn't an infinite wait being cut short.** `timeout` sets how long `sqlite3` keeps retrying before it gives up and raises. With `timeout=0`, w2 would fail immediately. The default is 5 seconds; it never waits forever. So the timeout is actually a small **server-side retry loop**, and that matters below.

**Confirmed non-commit requires your rollback.** In Cell 2 the INSERT itself failed, so nothing ran. But in experiment B, the lock error happened at **`commit()`**, after the statements had already executed, and the transaction was left pending. In both cases the outcome becomes a confirmed non-commit only once the backend rolls back. That's the unit-of-work helper's job again: a lock error is just another exception that triggers rollback.

**Why retry is safe.** You put it well: because the first attempt didn't commit, a retry is a fresh attempt evaluated against the *current* state, so it can't apply anything twice. Its result can be success, duplicate, or busy again, and each of those is already a classified outcome. **(Lesson candidate: a lock error is not an answer. It means the database never evaluated your request, so after a confirmed rollback, retrying is a fresh attempt, not a repeat.)**

## Code vs message

What you proposed is the **message**. The **code** is the stable, machine-readable identifier the frontend's allowlist and recovery mapping key on:

- **Code:** something like `SERVICE_BUSY`. I'd avoid `DATABASE_BUSY`, since the contract shouldn't reveal what's behind the API; your Lab 04 principle was stable codes independent of database technology. Your call, though.
- **Message:** your sentence, though "in a few seconds" is more accurate than "a few minutes." Contention clears when the other transaction finishes.
- **HTTP status:** **503 Service Unavailable** is the conventional choice for a temporary condition. It can include a `Retry-After` header.

The status–code agreement rule earns its keep here. Proxies and load balancers also emit 503s, with no envelope. Under your contract, a bare 503 without `SERVICE_BUSY` stays unknown, and only 503 with the code becomes retryable. That's exactly the "5xx is unknown unless an explicit code confirms non-commit" rule from the Lab 04 doc.

This becomes the **first entry in the retry allowlist**, and the `retry` recovery path finally gets used. The Lab 04 recovery table already describes it: Retry resubmits the preserved payload, and the draft stays intact.

## One coordination detail: two timeouts in series

You chose user-initiated retry, which fits the spec. But `sqlite3`'s busy timeout already runs a short server-side retry *before* the user is involved. The two layers have to fit together. If the backend's busy timeout were longer than the frontend's long-request threshold (10 s in the spec), the user would get "Keep Waiting / Stop Waiting" for what is really contention. So keep the backend's busy timeout well below that threshold, a few seconds at most. **(Lesson candidate: retry and timeout layers stack. Each inner layer must give up well before the outer layer starts treating silence as uncertainty.)**

## The translation layer gains a third kind

The neutral output of the translation function now has three kinds: **unique**, **foreign key**, and **busy**. On SQLite, busy shows up as `sqlite_errorname == 'SQLITE_BUSY'` with an `OperationalError`, which is a different class from E's `IntegrityError`. The other databases have their own equivalents, such as lock-timeout and deadlock errors with specific codes. Those get mapped when you write their translation functions; the neutral kind stays the same.

Your lost-response answer is exactly the Lab 04 contract: unknown, refresh, don't retry.

## Before designing the service: one experiment is missing

Your six experiments never tested **foreign keys**, and AT-36 ("referenced Delete is blocked") depends entirely on them. SQLite has two traps here, and both are worth observing rather than taking on trust:

1. **FK enforcement is off by default and is set per connection** (`PRAGMA foreign_keys = ON`). Read it back with `PRAGMA foreign_keys` to confirm it took effect.
2. **SQLite's documentation says that pragma has no effect inside an open transaction.** In `autocommit=False` mode, the module may already have a transaction open by the time you run the pragma. So the test is: connect in `autocommit=False` mode, set the pragma, read it back, and see whether it's actually on. If it isn't, you've found something the service's connection setup must handle, and it's SQLite-specific.

A small experiment G would cover it:

- Create `clients` plus one dependent table (e.g. `client_notes` with `client_id REFERENCES clients(id) ON DELETE RESTRICT`).
- Insert a client and a note.
- Try to delete the client with the pragma **off**, then with it **on**.
- Record the exception class, `sqlite_errorname`, and message, the same way as in E.
- Confirm by reading back whether the pragma took effect in your connection mode.

After that, you'll have observed every input the service design needs: storage, visibility, transaction boundaries, failure behavior, contention, and referential integrity. Then the next bounded step is designing the service's four pieces before writing code:

1. **Schema:** `AUTOINCREMENT`, the normalized name key, the dependent table, all constraints named.
2. **Unit-of-work helper:** connection setup (including the pragma), commit, rollback, error preservation.
3. **SQLite translation function:** driver exception to neutral kind.
4. **Service operations:** neutral kind plus operation context to the existing domain outcomes, plus the new busy outcome.

The normalized-key decision from earlier (computed in Python vs. in the database) is still open, and it belongs to piece 1.

# 7. FK reference 

