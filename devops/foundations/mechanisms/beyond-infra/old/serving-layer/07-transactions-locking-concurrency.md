# Transactions, Locking, and Concurrency

> **Status:** first draft — living document.
> **Companion to:** the SQL Syntax Reference Sheet and the SQLite + FastAPI Reference Sheet.
> **Approach:** engine-independent concepts first, SQLite as the concrete implementation, then what changes on PostgreSQL and Azure SQL. The goal is that the SQLite knowledge transfers rather than having to be unlearned.

---

## Table of contents

| Part | Topic |
|---|---|
| 0 | The four questions |
| **A** | **Engine-independent theory** |
| 1 | Vocabulary: ACID, boundaries, autocommit |
| 2 | Anomalies and isolation levels |
| 3 | Concurrency control families |
| **B** | **SQLite: the concrete implementation** |
| 4 | The lock ladder (rollback journal mode) |
| 5 | WAL mode |
| 6 | `BEGIN` variants |
| 7 | Contention: `SQLITE_BUSY`, `SQLITE_BUSY_SNAPSHOT`, the busy handler |
| 8 | **Worked scenario: a reader and a writer, step by step** |
| 9 | COMMIT, durability, checkpointing |
| 10 | Savepoints |
| **C** | **Python: taking manual control** |
| 11 | Where automatic transactions come from |
| 12 | Turning everything off |
| 13 | Retry logic |
| 14 | FastAPI wiring |
| 15 | Workflows |
| **D** | **Transfer** |
| 16 | PostgreSQL |
| 17 | Azure SQL / SQL Server |
| 18 | Cross-engine translation table |
| 19 | Traps ledger |

---

## Part 0 — The four questions

Every transaction system, on every engine, answers the same four questions. If you can answer them for a new database, you can use it correctly on day one. The rest of this document is these four questions asked repeatedly.

1. **Where do transaction boundaries come from?** Did *you* write `BEGIN`, or did the engine, the driver, or the ORM write it for you? Almost every "my write disappeared" bug is here.
2. **What is acquired, and when?** A lock? A snapshot? At `BEGIN`, at first read, at first write, or at `COMMIT`? The gap between "transaction started" and "protection acquired" is where race conditions live.
3. **What happens on conflict?** Does the engine make you *wait*, does it *fail immediately*, or does it *abort you and demand a retry*? These are three genuinely different contracts and they require three different client behaviours.
4. **Who retries?** In every serious concurrency model, the answer is "the application". An engine that can abort your transaction is an engine whose client needs a retry loop.

Hold these four while reading. SQLite's answers are unusual in the details but conventional in the structure.

---

# Part A — Engine-independent theory

## Part 1 — Vocabulary

### ACID, precisely

- **Atomicity** — the transaction's effects are all applied or none are. The mechanism is always some form of *undo log* (a rollback journal, PostgreSQL's old row versions) or *redo log* (a write-ahead log). Something durable records enough to reconstruct one of the two valid states after a crash mid-write.
- **Consistency** — constraints and invariants hold at transaction boundaries. This is the least interesting letter: it's mostly a statement that the other three plus your constraints give you what you want. It's the application's job as much as the engine's.
- **Isolation** — the degree to which concurrent transactions can observe each other's incomplete work. This is the letter with a dial on it, and Part 2 is entirely about that dial.
- **Durability** — once `COMMIT` returns, the data survives a crash. The mechanism is an `fsync` (or equivalent) at some point before the commit is acknowledged. **Durability is tunable and frequently tuned down**, so "committed" means different things in different configurations.

### Transaction boundaries

A transaction has exactly three possible ends:

```
BEGIN ──────► COMMIT     (durable, visible to others)
       └────► ROLLBACK   (no effect)
       └────► crash      (resolved to one of the above at recovery)
```

The boundary question that matters in practice is not *how* to write `COMMIT` but *who* wrote `BEGIN`. There are four common sources, and in a typical Python stack up to three of them are active simultaneously:

| Source | Example |
|---|---|
| You, explicitly | `conn.execute("BEGIN IMMEDIATE")` |
| The engine, per-statement | SQLite's autocommit mode: every bare statement is its own transaction |
| The driver | Python `sqlite3` issuing an implicit `BEGIN` before `INSERT` |
| The framework/ORM | SQLAlchemy sessions, Django's `ATOMIC_REQUESTS` |

Layered implicit transaction control is the single most common source of confusion. Making it explicit at exactly one layer is the whole point of Part 12.

### Autocommit

"Autocommit mode" means: **no transaction is open, and each statement is implicitly wrapped in its own transaction.** A single `UPDATE` touching 10,000 rows is still atomic — it either all happens or none of it does. Autocommit doesn't mean "no transactions"; it means "one transaction per statement, chosen for you".

This is the default on SQLite (at the engine level), on SQL Server, and on MySQL. It is *not* the default in the Python DB-API, which specifies that connections start in a transactional mode — which is exactly why the Python `sqlite3` driver inserts `BEGIN` statements you didn't write.

### Read-only transactions

A transaction that only reads still needs a boundary if you want its reads to be *consistent with each other*. Two `SELECT`s in autocommit mode are two transactions and can see two different database states. Two `SELECT`s inside one transaction see one state (at `REPEATABLE READ` or stronger, or under any snapshot-based engine).

This matters more than it sounds. Any report, reconciliation, or multi-query API response that must internally agree — "these monthly numbers must sum to this annual total" — needs its reads inside one transaction, not merely on one connection.

### Invariants

1. A transaction is a boundary, not an operation. The interesting question is always where the boundary is and who put it there.
2. Autocommit is not the absence of transactions; it is the smallest possible transaction, chosen implicitly.
3. If two reads must agree with each other, they belong in one transaction — even though neither of them writes.

---

## Part 2 — Anomalies and isolation levels

Isolation levels are defined by which anomalies they permit. Learn the anomalies; the levels are then just a lookup table.

### The anomalies

**Dirty read** — you read a row another transaction has written but not committed. If it rolls back, you acted on data that never existed.

**Non-repeatable read** — you read a row, someone else commits an update to it, you read it again and get a different value. Your transaction is internally inconsistent.

```
T1: SELECT amount FROM t WHERE id=1;   -- 100
T2:                                     UPDATE t SET amount=200 WHERE id=1; COMMIT;
T1: SELECT amount FROM t WHERE id=1;   -- 200  ← same query, different answer
```

**Phantom read** — you run a query returning a *set* of rows, someone else commits an insert matching your predicate, you re-run and the set has grown. Non-repeatable read is about a row changing; phantom is about the *population* changing.

**Lost update** — two transactions read the same value, both compute a new value from it, both write. The second write silently overwrites the first's, and the first update is lost.

```
T1: read balance=100
T2: read balance=100
T1: write balance=100-30 = 70
T2: write balance=100-50 = 50   ← T1's deduction vanished
```

This is the anomaly your application code produces most often, because it is the shape of every read-modify-write.

**Write skew** — two transactions read an overlapping set, each checks an invariant that currently holds, each writes to a *different* row, and the combined result violates the invariant. Neither wrote what the other read, so no write-write conflict is detected.

```
Rule: at least one revision per dataset must remain published.
T1: sees revisions A(published), B(published) → unpublishes A
T2: sees revisions A(published), B(published) → unpublishes B
Both commit. Zero published revisions. The rule is broken and nothing conflicted.
```

Write skew is the anomaly that snapshot isolation *permits* and true serializability forbids. It's the reason `SERIALIZABLE` is not merely `REPEATABLE READ` with extra caution.

### The standard levels

| Level | Dirty read | Non-repeatable read | Phantom | Write skew |
|---|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible | possible |
| READ COMMITTED | no | possible | possible | possible |
| REPEATABLE READ | no | no | possible* | possible |
| SNAPSHOT | no | no | no | **possible** |
| SERIALIZABLE | no | no | no | no |

\* The SQL standard permits phantoms at `REPEATABLE READ`. PostgreSQL's `REPEATABLE READ` is implemented as snapshot isolation and prevents them anyway. This is the first of many places where the standard's names and an engine's actual behaviour diverge — **the level name tells you less than the engine's implementation does.**

### The critical point about lost updates

The standard levels say nothing about lost updates, because the standard assumes updates are expressed as `UPDATE t SET x = x - 30`, which engines handle atomically. But application code almost never does that. It does:

```python
row = conn.execute("SELECT amount FROM t WHERE id = ?", (1,)).fetchone()
new = row["amount"] - 30
conn.execute("UPDATE t SET amount = ? WHERE id = ?", (new, 1))
```

Read, compute in Python, write back. **No isolation level below `SERIALIZABLE`/`SNAPSHOT`-with-conflict-detection protects this pattern**, because the read and the write are separate statements with a window between them. Three fixes, in order of preference:

1. **Do it in SQL** — `UPDATE t SET amount = amount - 30 WHERE id = 1`. One statement, atomic, no window.
2. **Declare the write intent up front** — start a write transaction *before* the read, so nobody else can interleave. This is `BEGIN IMMEDIATE` in SQLite, `SELECT ... FOR UPDATE` in PostgreSQL, `UPDLOCK` in SQL Server. Same idea in three costumes.
3. **Optimistic concurrency** — include the old value or a version column in the `WHERE`, then check `rowcount`:
   ```sql
   UPDATE t SET amount = ?, version = version + 1 WHERE id = ? AND version = ?
   ```
   `rowcount == 0` means someone else got there first; re-read and retry.

Option 2 is the one this document keeps returning to, because it's the one whose SQLite spelling is unusual.

### Invariants

1. Isolation levels are defined by the anomalies they forbid. Memorize four anomalies, not five level names.
2. The level name is a label, not a specification. Two engines' `REPEATABLE READ` can differ substantially.
3. Read-modify-write in application code is unsafe at every level unless you declare the write intent before the read, or verify at write time.

---

## Part 3 — Concurrency control families

There are three underlying strategies. Every engine is one of these or a hybrid, and knowing which one you're on predicts its failure modes.

### Two-phase locking (pessimistic)

Acquire locks before touching data; hold them until commit ("growing phase" then "shrinking phase"). Readers take shared locks, writers take exclusive locks; they conflict.

- **Conflicts cause waiting.** A blocked transaction sits until the holder commits.
- **Deadlock is possible** — cycles of waiters. Requires either detection (find the cycle, kill a victim) or prevention (refuse to enter a wait that could cycle).
- Characteristic symptom: **blocking**. Queries that are normally fast take seconds because something else holds a lock.
- Used by: SQL Server in its default on-prem configuration, older MySQL/MyISAM, and — in a very coarse form — SQLite.

### Multi-version concurrency control / snapshot isolation (optimistic-ish)

Writers create new versions rather than overwriting; readers see a consistent snapshot as of some point in time.

- **Readers never block writers; writers never block readers.** This is the headline property.
- Writer-writer conflicts still exist and are resolved by locking a row, or by aborting one transaction.
- Characteristic symptoms: **version/garbage accumulation** (PostgreSQL bloat and vacuum, SQLite WAL growth) and **serialization failures** at higher levels.
- Used by: PostgreSQL, Oracle, SQL Server with RCSI/snapshot enabled (the Azure SQL default), and SQLite's WAL mode in a degenerate single-writer form.

### Optimistic concurrency control

Do the work without locks, then validate at commit; abort if someone conflicted.

- **Conflicts cause aborts, not waits.** Retry is mandatory and normal.
- Best when conflicts are rare; degrades badly under contention.
- Used by: PostgreSQL `SERIALIZABLE` (SSI adds validation on top of MVCC), and by application-level version-column patterns on any engine.

### The transferable consequence

The strategy determines what your client code must handle:

| Strategy | Client must handle |
|---|---|
| Locking | timeouts, deadlock errors → **retry**, and keep transactions short |
| MVCC/snapshot | stale-snapshot conflicts → **retry**, and keep transactions short |
| Optimistic | validation failures → **retry**, and keep transactions short |

All three converge on the same two obligations: **retry on conflict, and hold transactions briefly.** If you build those two habits, moving between engines is a matter of learning which exception class to catch.

### Where SQLite sits

SQLite is a hybrid that's easy to misclassify:

- In **rollback journal** mode it is coarse-grained two-phase locking over the *whole database file*.
- In **WAL** mode readers get real snapshot isolation, while writes remain serialized by a single database-wide write lock.
- In both modes there is **exactly one writer at a time**, so the effective isolation level is `SERIALIZABLE` and there is no dial to turn. SQLite has no `SET TRANSACTION ISOLATION LEVEL`.

That last point is why SQLite is a good place to learn this material: the isolation level is fixed at the strongest setting, so nothing here is about choosing a level. It's all about *boundaries and locks*, which is the part that actually transfers.

---

# Part B — SQLite: the concrete implementation

## Part 4 — The lock ladder (rollback journal mode)

This is the classic model. It applies when `journal_mode` is `DELETE`, `TRUNCATE`, `PERSIST`, or `MEMORY` — which includes the **default** for a freshly created database.

Five states, held on the database file:

```
UNLOCKED ──► SHARED ──► RESERVED ──► PENDING ──► EXCLUSIVE
```

| State | Who holds it | Concurrent SHARED allowed? | Meaning |
|---|---|---|---|
| **UNLOCKED** | — | — | no access; the starting state |
| **SHARED** | many at once | yes | "I am reading." No writing may occur anywhere while any SHARED is held. |
| **RESERVED** | **one** at a time | **yes** | "I intend to write." Existing and *new* readers continue. Changes buffer in the writer's page cache. |
| **PENDING** | one | **no new ones** | "I want to commit; I'm waiting for existing readers to finish." Blocks new readers so the writer can't starve. |
| **EXCLUSIVE** | one | no | "I am writing to the database file." Nothing else may touch it. |

The crucial and non-obvious property is **RESERVED coexists with SHARED**. A writer can start a write transaction, do all its work, and hold RESERVED for a long time while readers come and go freely. Contention only bites at the *end*, at commit time, when RESERVED must escalate through PENDING to EXCLUSIVE — and that escalation must wait for every existing reader to release its SHARED lock.

```
timeline ──────────────────────────────────────────────►
reader A  [SHARED..................]
reader B          [SHARED...]
writer    [RESERVED................][PENDING..][EXCL][commit]
                                     ▲
                                     waits here for A to finish;
                                     new readers blocked from here
```

The design rationale: allowing new readers during PENDING would let a stream of short reads starve the writer forever. Blocking new readers guarantees the writer eventually drains the existing ones and proceeds.

**The practical consequence for a serving layer:** in rollback journal mode, a long-lived reader delays every writer's commit. And in Python, a *cursor with unfetched rows is a long-lived reader* — the read transaction stays open until the statement is exhausted, reset, or the connection commits. A streaming CSV export can block your write path for its entire duration.

This is the primary argument for WAL.

---

## Part 5 — WAL mode

```sql
PRAGMA journal_mode = WAL;    -- persistent in the file; set once at setup
```

### The mechanism

Instead of copying original pages out to a journal and writing in place, WAL **appends** new page images to a separate `-wal` file. The main database file is not modified during a normal commit.

Three files exist: `forecast.db`, `forecast.db-wal`, `forecast.db-shm` (a shared-memory index into the WAL).

**Readers** determine an *end mark* — the last commit frame in the WAL as of the moment their read transaction begins — and read the database as "main file, overlaid with WAL frames up to my end mark". Frames appended after that point are invisible to them. That is snapshot isolation, implemented very directly: the snapshot literally is a number.

**Writers** append frames past the end of the WAL and, at commit, write a commit frame and advance the WAL's end mark. A single database-wide **WRITER lock** ensures one writer at a time.

### The properties that follow

- **Readers do not block the writer. The writer does not block readers.** Both proceed against different parts of the WAL.
- **Still exactly one writer at a time.** WAL improves read/write concurrency, not write/write concurrency.
- A reader sees a *stable snapshot* for the duration of its transaction, no matter how many commits land meanwhile.
- Commits are usually faster: sequential appends rather than random in-place writes plus journal copying.

### Checkpointing

The WAL can't grow forever. **Checkpointing** copies WAL frames back into the main database file and (when safe) resets the WAL to zero length.

```sql
PRAGMA wal_autocheckpoint = 1000;          -- default: auto-checkpoint at 1000 pages
PRAGMA wal_checkpoint(PASSIVE);            -- copy what's safely copyable, never blocks
PRAGMA wal_checkpoint(FULL);               -- wait for readers, copy everything
PRAGMA wal_checkpoint(TRUNCATE);           -- FULL, then reset the WAL file to zero bytes
```

A checkpoint can only copy frames that **no active reader still needs**. So:

> **A long-running read transaction pins the WAL and prevents it from being reset.**

This is WAL's characteristic failure mode: not blocking, but *unbounded `-wal` file growth*. If you find a multi-gigabyte `-wal` next to a small `.db`, the cause is a reader that never finished — a forgotten cursor, an idle connection with an open transaction, a crashed worker.

Note the symmetry with PostgreSQL: a long-running transaction there prevents vacuum from reclaiming old row versions, producing table bloat. Same underlying cause — **old versions cannot be discarded while someone might still need to read them** — with an engine-specific symptom. Recognizing that pattern is most of what transfers.

### Constraints and gotchas

- **WAL requires shared memory**, so all connections must be on the same machine. It does not work correctly over NFS or many network/container volume mounts.
- **WAL is not available for `:memory:` databases.**
- **A `mode=ro` connection to a WAL database still needs write access to the `-shm` file** and to the directory, because participating in the WAL requires updating the shared read marks. `[trap]` A truly read-only filesystem needs `?immutable=1` (which promises the file will not change, and disables locking entirely — wrong for a live database). If your intent is just "this connection must not write", the right tool is:
  ```sql
  PRAGMA query_only = ON;    -- connection-level; rejects writes, no file-permission requirements
  ```
- Durability under WAL depends on `synchronous` (Part 9).

### Invariants

1. In WAL, a reader's snapshot is fixed at the moment its read transaction starts, and nothing that commits afterwards is visible to it.
2. WAL removes reader/writer blocking. It does not create a second writer.
3. The cost of snapshot isolation is retained old versions. Whoever holds the oldest snapshot determines how much cannot be reclaimed.

---

## Part 6 — `BEGIN` variants

```sql
BEGIN [DEFERRED | IMMEDIATE | EXCLUSIVE] [TRANSACTION];
COMMIT;    -- or END;
ROLLBACK;
```

### What each one acquires, and when

| Variant | At `BEGIN` | On first read | On first write |
|---|---|---|---|
| **DEFERRED** (default) | **nothing** | read lock / snapshot | write lock — **may fail here** |
| **IMMEDIATE** | **write lock** | (already held) | (already held) |
| **EXCLUSIVE** | write lock; in rollback mode also blocks readers | — | — |

**`BEGIN DEFERRED`** — the default, and the source of most trouble. `BEGIN` alone does *nothing at all*: no locks, no snapshot, no error. The transaction becomes real at the first statement. If that statement is a `SELECT`, you acquire a read snapshot and only *later* attempt to acquire the write lock, which can fail.

**`BEGIN IMMEDIATE`** — takes the write lock right away. Fails immediately with `SQLITE_BUSY` if another write transaction is active, which is a *recoverable* condition your busy handler can wait out. Crucially, it also fixes the read snapshot at that moment, so everything you read afterwards is guaranteed current and protected.

**`BEGIN EXCLUSIVE`** — in **WAL mode, identical to IMMEDIATE**. In rollback journal mode, it additionally prevents other connections from *reading* for the duration. Rarely what you want; the legitimate uses are schema migrations and offline maintenance.

### The rule

> **If a transaction will write, start it with `BEGIN IMMEDIATE`.**

Not "if it starts with a write" — *if it will ever write*. A transaction that reads a balance and then updates it must announce its write intent before the read, or it is subject to the upgrade failure in Part 7 and the lost-update anomaly in Part 2.

Pure reads use `BEGIN DEFERRED` (or `BEGIN`), which is correct and cheap: they take a read snapshot at the first `SELECT` and never need to escalate.

### The transferable idea

`BEGIN IMMEDIATE` is SQLite's spelling of a universal pattern: **declare the strongest lock you will need at the start, rather than escalating in the middle.** The other spellings:

| Engine | Same idea |
|---|---|
| SQLite | `BEGIN IMMEDIATE` |
| PostgreSQL | `SELECT ... FOR UPDATE` on the rows you'll modify, or `LOCK TABLE` |
| SQL Server | `SELECT ... WITH (UPDLOCK)`, or the U (update) lock mode generally |
| Oracle | `SELECT ... FOR UPDATE` |

All four exist because shared→exclusive escalation mid-transaction is the classic deadlock generator, and every mature engine offers a way to skip it. SQL Server's dedicated "update lock" mode exists for exactly this reason: it's compatible with shared locks but not with other update locks, so only one transaction can be in the "planning to write this row" state at a time.

---

## Part 7 — Contention: `SQLITE_BUSY`, `SQLITE_BUSY_SNAPSHOT`, and the busy handler

### The busy handler

```sql
PRAGMA busy_timeout = 5000;        -- milliseconds
```
```python
sqlite3.connect(path, timeout=5.0)  # seconds — same underlying setting
```

When a lock can't be acquired, SQLite calls the busy handler, which sleeps and retries until the timeout expires, then returns `SQLITE_BUSY` (surfacing in Python as `OperationalError: database is locked`).

The default `busy_timeout` at the C level is **0** — fail immediately. Python's driver sets 5 seconds by default via the `timeout` parameter. Anything that opens a connection outside your factory may not have it set.

### The two failures the busy handler cannot help with

**1. The promotion deadlock (rollback journal mode).**

Two connections both hold SHARED and both want RESERVED. Neither can get it while the other holds SHARED. If both went to sleep in a busy handler, neither would ever wake to release anything — a genuine deadlock.

SQLite's design choice is **deadlock prevention, not detection**: rather than waiting, it returns `SQLITE_BUSY` immediately to one of them, in the hope that the client will roll back and release its read lock, letting the other proceed. The busy handler is deliberately *not invoked*.

This is the source of the surprising report "I set `busy_timeout` to 30 seconds and still got `database is locked` instantly." The timeout was never consulted, because waiting would have been the wrong answer.

**2. `SQLITE_BUSY_SNAPSHOT` (WAL mode).**

A deferred transaction reads (fixing its snapshot), another connection commits, and then the first transaction tries to write. Its snapshot is now stale — writing on top of it could corrupt the serial order. SQLite returns `SQLITE_BUSY_SNAPSHOT`.

Waiting cannot fix this either: the snapshot will never become fresh again. The only resolution is `ROLLBACK` and start over, taking a new snapshot.

### The unifying rule

> `SQLITE_BUSY` from **lock acquisition** is a *wait* condition — `busy_timeout` handles it.
> `SQLITE_BUSY` from **lock promotion** is an *abort* condition — only rollback-and-retry handles it.

`BEGIN IMMEDIATE` converts the second, unrecoverable-by-waiting case into the first, waitable one. That is the entire practical argument for it.

### Detecting which you got

```python
except sqlite3.OperationalError as exc:
    code = getattr(exc, "sqlite_errorname", "")
    # 'SQLITE_BUSY'          -> waitable; busy_timeout already tried
    # 'SQLITE_BUSY_SNAPSHOT' -> must rollback and retry from the top
    # 'SQLITE_LOCKED'        -> conflict within the same connection (shared cache)
```

`sqlite_errorname` requires Python 3.11+. Below that you're inspecting message strings.

### Invariants

1. `busy_timeout` handles waiting for a lock. It does nothing for a transaction that must be abandoned.
2. A transaction that reads before declaring write intent can fail in a way no amount of waiting resolves.
3. SQLite prevents deadlocks by refusing to wait. PostgreSQL and SQL Server detect deadlocks and kill a victim. Both approaches hand you the same obligation: a retry loop.

---

## Part 8 — Worked scenario: a reader and a writer, step by step

The setup you asked about:

```python
# conn1 — read-only
conn1 = sqlite3.connect("file:forecast.db?mode=ro", uri=True, timeout=5.0)

# conn2 — writer, all automatic transaction control disabled
conn2 = sqlite3.connect("forecast.db", timeout=5.0, isolation_level=None)
conn2.execute("PRAGMA busy_timeout = 5000")
conn2.execute("PRAGMA foreign_keys = ON")
```

### Case A — WAL mode (the recommended configuration)

```
t   conn1 (reader)                       conn2 (writer)                      state
──────────────────────────────────────────────────────────────────────────────────
1   connect                                                                  no locks held
2                                        connect                             no locks held
3   SELECT ... (executes)                                                    conn1 takes a READ
                                                                             MARK: snapshot =
                                                                             WAL end mark E0
4                                        BEGIN IMMEDIATE                     conn2 acquires the
                                                                             WRITER lock.
                                                                             SUCCEEDS — reader
                                                                             does not block it.
                                                                             conn2's snapshot
                                                                             also fixed at E0.
5   still fetching rows...                                                   unaffected
6                                        INSERT INTO ...                     frames appended to
                                                                             -wal past E0.
                                                                             Invisible to conn1.
7   SELECT ... again (same txn)                                              STILL sees E0 —
                                                                             conn2's insert is
                                                                             not visible
8                                        COMMIT                              commit frame written,
                                                                             WAL end mark -> E1,
                                                                             WRITER lock released
9   SELECT ... (same open txn)                                               STILL sees E0
10  end transaction / finish cursor                                          read mark released
11  SELECT ... (new txn)                                                     now sees E1 —
                                                                             insert visible
```

The things to take from this:

- **Step 4 succeeds.** `BEGIN IMMEDIATE` is not blocked by an active reader. This is WAL's central benefit.
- **Steps 7 and 9 are the surprise for most people.** `conn1` does not see the committed insert, even after `COMMIT` returned. It is holding a snapshot. This is correct behaviour, not staleness — but if `conn1` is a long-lived connection in a serving layer with an accidentally-open transaction, it will serve stale data indefinitely.
- **Step 10 is what releases the WAL.** Until `conn1` ends its read transaction, frames from E0 onward cannot be checkpointed away.
- If a `conn3` also ran `BEGIN IMMEDIATE` between steps 4 and 8, it would get `SQLITE_BUSY` and its busy handler would wait up to 5 seconds for conn2's `COMMIT`. That's the waitable, healthy case.

**The `mode=ro` wrinkle.** `conn1` as written needs write permission on `forecast.db-shm` and on the containing directory, because taking a read mark means writing into the shared-memory index. `mode=ro` restricts writes to the *database*, not to the WAL machinery. If the process genuinely cannot write there, the connection fails to open. Use a normal read-write connection with `PRAGMA query_only = ON` when the goal is "this code path must not write". `[trap]`

### Case B — rollback journal mode (the default for a new database)

```
t   conn1 (reader)                       conn2 (writer)                      state
──────────────────────────────────────────────────────────────────────────────────
1   connect                                                                  UNLOCKED
2                                        connect                             UNLOCKED
3   SELECT ... (executes)                                                    conn1: SHARED
4                                        BEGIN IMMEDIATE                     conn2: RESERVED
                                                                             SUCCEEDS — RESERVED
                                                                             and SHARED coexist
5   still fetching rows...                                                   both held
6                                        INSERT INTO ...                     buffered in conn2's
                                                                             page cache + journal;
                                                                             db file untouched
7                                        COMMIT  ───────────────┐            conn2: RESERVED ->
                                                                │            PENDING. New readers
                                                                │            now BLOCKED.
                                                                │            Waits for conn1's
                                                                │            SHARED to clear.
8   still fetching rows...              (blocked, busy handler  │            up to busy_timeout
                                         sleeping)              │
9   cursor exhausted / txn ends                                 │            conn1: SHARED
                                                                │            released
10                                       ◄──────────────────────┘            conn2: PENDING ->
                                                                             EXCLUSIVE, writes,
                                                                             commits, UNLOCKED
11  SELECT ... (new txn)                                                     sees the insert
```

The contrast with Case A is the whole reason WAL exists:

- **Step 7–9: the writer's `COMMIT` blocks on the reader.** Not the `BEGIN`, not the `INSERT` — the commit. If `conn1` holds SHARED for longer than `busy_timeout`, `conn2`'s commit fails with `database is locked` after having done all its work.
- **Step 7 also blocks new readers.** PENDING deliberately starves incoming reads so the writer can finish. A slow reader plus a waiting writer stalls the whole database.
- There are no snapshots here. Once `conn2` commits, any new read sees the new data immediately.

### Case C — the failure this all exists to prevent

Same WAL setup, but `conn2` uses the default deferred transaction and reads first:

```
t   conn2 (deferred, reads then writes)              conn3 (another writer)
────────────────────────────────────────────────────────────────────────────
1   BEGIN                    -- acquires nothing
2   SELECT amount FROM t     -- snapshot fixed at E0
3                                                    BEGIN IMMEDIATE; UPDATE t; COMMIT;
                                                     -- WAL end mark -> E1
4   UPDATE t SET amount = ?  -- needs the write lock, but conn2's snapshot is E0
                             -- and the database has moved to E1
                             -- ► SQLITE_BUSY_SNAPSHOT, IMMEDIATELY
                             -- busy_timeout is NOT consulted
5   ROLLBACK and start over from step 1
```

Two distinct bugs are visible here. The obvious one is the error at step 4. The subtler one is that if step 3 hadn't happened, `conn2` would have *succeeded* — and computed its new amount from a value that could have gone stale. The error is SQLite protecting you from a lost update. `BEGIN IMMEDIATE` at step 1 prevents both: `conn3` would have waited at its own `BEGIN IMMEDIATE`, and `conn2`'s read at step 2 would have been protected by a lock it already held.

### Invariants

1. In WAL, a writer's `BEGIN IMMEDIATE` never waits for readers; in rollback journal mode, a writer's `COMMIT` always might.
2. A reader's view is fixed when its read transaction starts, not when each statement runs. Reopen the transaction to see new data.
3. A transaction that reads before it declares write intent has a window in which the world can move underneath it. Closing that window is what `BEGIN IMMEDIATE` is for.

---

## Part 9 — COMMIT, durability, checkpointing

### What `COMMIT` actually guarantees

`COMMIT` returning successfully means the transaction is **atomic and visible**. Whether it is **durable against power loss** depends on `synchronous`:

```sql
PRAGMA synchronous = OFF;      -- 0: no fsync. A power cut can corrupt the database.
PRAGMA synchronous = NORMAL;   -- 1: the practical default under WAL
PRAGMA synchronous = FULL;     -- 2: fsync at every commit. The default in rollback mode.
PRAGMA synchronous = EXTRA;    -- 3: FULL plus a directory sync
```

| Setting | Survives process crash | Survives OS crash / power loss | Risk of corruption |
|---|---|---|---|
| `OFF` | yes | **no** | **yes** |
| `NORMAL` (WAL) | yes | recent commits may be lost | no |
| `FULL` | yes | yes | no |

`NORMAL` under WAL is the standard production choice: the WAL is fsynced at checkpoints rather than at every commit, so a power failure can lose the last few transactions but cannot corrupt the database. `OFF` is only ever acceptable for a rebuildable artifact — a batch-built database you can regenerate from source data — where losing it costs a re-run, not data.

Note that "durable" and "visible" separate here. Another connection sees your commit as soon as it lands, whether or not it has been fsynced. That's true of every engine with tunable durability, including PostgreSQL's `synchronous_commit = off`.

### Checkpointing and WAL size

```sql
PRAGMA wal_autocheckpoint = 1000;         -- pages; 0 disables automatic checkpointing
PRAGMA wal_checkpoint(TRUNCATE);          -- force a full checkpoint and reset the file
```

Automatic checkpointing runs after a commit that pushes the WAL past the threshold, on the committing connection's thread. That means one unlucky write occasionally pays the checkpoint cost. For latency-sensitive write paths, disable autocheckpoint and run `wal_checkpoint(PASSIVE)` from a background job instead.

Diagnosing WAL growth:

```python
busy, log_pages, checkpointed = conn.execute("PRAGMA wal_checkpoint(PASSIVE)").fetchone()
# log_pages       — frames currently in the WAL
# checkpointed    — frames successfully copied back
# large log_pages with checkpointed much smaller => a reader is pinning the WAL
```

### After a rebuild-and-swap

If you publish a new database by `os.replace()`, existing connections keep their old file handle and continue serving the old inode. `COMMIT` semantics have nothing to say about this — it's a filesystem-level version skew. Per-request connections avoid it entirely; a pooled design needs a generation check.

---

## Part 10 — Savepoints

Savepoints are SQLite's only nesting primitive. `BEGIN` inside a `BEGIN` is an error, not a nested transaction.

```sql
SAVEPOINT sp_name;
    ...
ROLLBACK TO sp_name;      -- undo to the savepoint; the savepoint REMAINS active
RELEASE sp_name;          -- discard the savepoint, keeping its work
```

Semantics worth being precise about:

- `SAVEPOINT` issued **outside** any transaction *starts* a transaction (a deferred one — so the same write-intent caveat applies; take `BEGIN IMMEDIATE` first if you will write).
- `ROLLBACK TO sp` undoes work but **does not remove the savepoint**. You must also `RELEASE` it, or roll back to it again later.
- `RELEASE` of the outermost savepoint commits the transaction if the savepoint started it.
- Savepoints nest by name and form a stack; releasing an outer one releases everything above it.

```python
def with_savepoint(conn, name: str):
    conn.execute(f"SAVEPOINT {name}")
    try:
        yield
    except Exception:
        conn.execute(f"ROLLBACK TO {name}")
        conn.execute(f"RELEASE {name}")
        raise
    else:
        conn.execute(f"RELEASE {name}")
```

(Savepoint names are identifiers, so they can't be bound as parameters — use an allow-list or a generated internal name, never user input.)

Typical use: a bulk import where one bad record should be skipped without discarding the whole batch.

**Transfer note:** savepoints are standard SQL and behave near-identically in PostgreSQL (`SAVEPOINT`/`ROLLBACK TO`/`RELEASE`) and SQL Server (`SAVE TRANSACTION` / `ROLLBACK TRANSACTION name` — note SQL Server has no `RELEASE`). They are how ORMs implement "nested transactions" everywhere; there is no such thing as a genuinely nested transaction in any of these engines.

---

# Part C — Python: taking manual control

## Part 11 — Where automatic transactions come from

Three layers can issue `BEGIN`/`COMMIT` on your behalf. To manage transactions yourself you need to know exactly what each one does.

### Layer 1 — the SQLite engine

SQLite is in **autocommit mode** whenever no transaction is open. Every statement executed in autocommit mode is implicitly wrapped in its own transaction. You cannot turn this off; you suppress it by having a transaction open.

`sqlite3_get_autocommit()` is exposed in Python as:

```python
conn.in_transaction    # True when a transaction is open (engine-level truth)
```

This is the ground truth and is worth asserting in tests and debugging.

### Layer 2 — the Python `sqlite3` driver (legacy mode)

With `isolation_level=""` (the default), the driver:

- issues `BEGIN` (deferred) before `INSERT`, `UPDATE`, `DELETE`, `REPLACE`,
- does **not** issue `BEGIN` before `SELECT`, DDL, or PRAGMAs,
- never commits by itself — only `conn.commit()` or `with conn:` does that,
- discards uncommitted work on `close()` with no warning.

The consequences, restated as failure modes:

```python
conn.execute("INSERT ...")     # driver silently issues BEGIN
conn.close()                   # work discarded, no error                     [trap]

conn.execute("INSERT ...")           # transaction now open
conn.execute("PRAGMA foreign_keys = ON")   # silent no-op inside a transaction [trap]

conn.execute("CREATE TABLE ...")     # no implicit BEGIN → autocommitted
conn.execute("INSERT ...")           # implicit BEGIN → needs a commit
# crash: table exists, rows don't
```

`isolation_level` accepts `""`, `"DEFERRED"`, `"IMMEDIATE"`, `"EXCLUSIVE"`, or `None`. The string values change *which* `BEGIN` the driver injects; `None` stops it injecting anything.

### Layer 3 — Python 3.12's `autocommit` attribute

```python
sqlite3.connect(path, autocommit=sqlite3.LEGACY_TRANSACTION_CONTROL)  # current default
sqlite3.connect(path, autocommit=False)   # PEP 249 style: a transaction is always open
sqlite3.connect(path, autocommit=True)    # driver never issues BEGIN
```

`autocommit=False` is cleaner than legacy mode — no guessing about which statements trigger an implicit `BEGIN` — but it opens **deferred** transactions, which is precisely the mode with the promotion hazard. For write paths you still want `BEGIN IMMEDIATE`, which means explicit control.

Check `sys.version_info` before depending on the attribute; the legacy constant remains the default for backward compatibility.

### Invariants

1. `conn.in_transaction` is the only reliable answer to "is a transaction open right now".
2. Closing a connection is not a commit, at any layer.
3. Connection-level PRAGMAs must run before any transaction opens — which in legacy mode means before your first DML statement.

---

## Part 12 — Turning everything off

The configuration you asked for: no automatic `BEGIN`, no automatic `COMMIT`, every boundary written by hand.

### The connection factory

```python
import sqlite3
from typing import Literal

def connect(
    db_path: str,
    *,
    mode: Literal["read", "write"] = "read",
    busy_timeout_ms: int = 5000,
) -> sqlite3.Connection:
    """The only place in the codebase that calls sqlite3.connect()."""
    conn = sqlite3.connect(
        db_path,
        isolation_level=None,      # ← the driver will never issue BEGIN or COMMIT
        check_same_thread=True,
        timeout=busy_timeout_ms / 1000,
    )
    conn.row_factory = sqlite3.Row

    # All of these are safe here precisely because no transaction is open:
    # with isolation_level=None there is no implicit BEGIN to have started one.
    conn.execute(f"PRAGMA busy_timeout = {busy_timeout_ms}")
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA synchronous = NORMAL")

    if mode == "read":
        conn.execute("PRAGMA query_only = ON")

    assert not conn.in_transaction, "connection should start in autocommit"
    return conn
```

`isolation_level=None` is the switch. From here on, SQLite is in autocommit mode until *you* write `BEGIN`, and every statement you run outside an explicit transaction is its own transaction.

Note `PRAGMA query_only = ON` rather than `mode=ro` — it gives read-only enforcement without requiring the connection to avoid touching the WAL index files (Part 5).

`PRAGMA journal_mode = WAL` is deliberately absent: it is persistent in the file, so it belongs in your schema setup or migration, not in a per-connection factory. Running it per connection is wasted work at best and a failure at worst if a transaction happens to be open.

### The transaction context managers

```python
from contextlib import contextmanager
from typing import Iterator
import sqlite3

@contextmanager
def read_txn(conn: sqlite3.Connection) -> Iterator[sqlite3.Connection]:
    """A consistent read snapshot across multiple statements."""
    conn.execute("BEGIN DEFERRED")
    try:
        yield conn
    finally:
        # A read transaction has nothing to commit; ROLLBACK is the cheap, correct end.
        conn.execute("ROLLBACK")

@contextmanager
def write_txn(conn: sqlite3.Connection) -> Iterator[sqlite3.Connection]:
    """A write transaction that declares its intent up front."""
    conn.execute("BEGIN IMMEDIATE")
    try:
        yield conn
    except BaseException:
        conn.execute("ROLLBACK")
        raise
    else:
        conn.execute("COMMIT")
```

Usage:

```python
conn = connect("forecast.db", mode="write")
with write_txn(conn):
    conn.execute("UPDATE revisions SET status='superseded' WHERE dataset_id=? AND status='published'", (ds,))
    conn.execute("UPDATE revisions SET status='published' WHERE revision_id=?", (rid,))
# committed here; both updates or neither
```

Deliberate choices in the above:

- **`BEGIN IMMEDIATE` for writes**, always — even when the first statement is a `SELECT`. That's the point.
- **`except BaseException`** rather than `except Exception`, so that `KeyboardInterrupt` and `SystemExit` also roll back rather than leaving a transaction open.
- **Explicit SQL `COMMIT`/`ROLLBACK`** rather than `conn.commit()`. Both work under `isolation_level=None`, but writing the SQL keeps the symmetry with `BEGIN` visible and makes the code independent of driver behaviour.
- **Read transactions end with `ROLLBACK`**, not `COMMIT`. Semantically identical for a read-only transaction, but `ROLLBACK` states the intent and is unambiguous if a stray write ever sneaks in.

### What to avoid once you're in manual mode

```python
conn.commit()                  # works, but hides the boundary — prefer explicit COMMIT
with conn:                     # driver-managed transaction; you turned this off, don't reintroduce it
conn.executescript(...)        # implicitly commits pending work first — never inside a transaction
```

`executescript` is for schema files run outside any transaction. If a migration needs atomicity, run the statements individually inside your own `write_txn`.

### Verifying it works

```python
conn = connect(":memory:", mode="write")
assert not conn.in_transaction

conn.execute("CREATE TABLE t (a INTEGER)")
assert not conn.in_transaction          # autocommit: DDL committed on its own

conn.execute("INSERT INTO t VALUES (1)")
assert not conn.in_transaction          # ← the proof: NO implicit BEGIN happened

with write_txn(conn):
    assert conn.in_transaction
    conn.execute("INSERT INTO t VALUES (2)")
assert not conn.in_transaction
```

Under the default `isolation_level=""`, the third assertion fails. That single assertion is a good permanent test — it detects anyone reintroducing driver-managed transactions.

### Invariants

1. `isolation_level=None` at connect time is the switch that makes every subsequent boundary yours.
2. In manual mode, any statement not inside your explicit `BEGIN` is still a transaction — the engine's, one per statement. There is no "no transaction" state.
3. One factory, one pair of context managers, no `with conn:` anywhere. Mixed transaction control is worse than either style alone.

---

## Part 13 — Retry logic

Retry is not an optimization; it is a required part of the contract on every engine that can refuse or abort a transaction.

```python
import random, time, sqlite3
from typing import Callable, TypeVar

T = TypeVar("T")

RETRYABLE = {"SQLITE_BUSY", "SQLITE_BUSY_SNAPSHOT", "SQLITE_LOCKED"}

def is_retryable(exc: sqlite3.OperationalError) -> bool:
    name = getattr(exc, "sqlite_errorname", "")
    if name:
        return name in RETRYABLE
    return "locked" in str(exc) or "busy" in str(exc)      # pre-3.11 fallback

def with_retry(fn: Callable[[], T], *, attempts: int = 5, base: float = 0.05) -> T:
    for attempt in range(attempts):
        try:
            return fn()
        except sqlite3.OperationalError as exc:
            if not is_retryable(exc) or attempt == attempts - 1:
                raise
            delay = base * (2 ** attempt) * (0.5 + random.random())   # backoff + jitter
            time.sleep(delay)
    raise AssertionError("unreachable")
```

Used as:

```python
def publish():
    with write_txn(conn):
        conn.execute("UPDATE ...")
        conn.execute("INSERT ...")

with_retry(publish)
```

### The rules that make retry correct

1. **The retried unit must be the whole transaction**, from `BEGIN` to `COMMIT`. Retrying a single failed statement inside a half-finished transaction is meaningless — the transaction is already doomed.
2. **The function must be idempotent in effect.** Because it re-reads inside the new transaction, a `read → compute → write` closure is naturally safe. A closure that mutates Python state outside the transaction is not.
3. **Jitter is not optional.** Without it, N contending workers retry in lockstep and collide again.
4. **Cap the attempts and let the last failure propagate.** Infinite retry converts a contention problem into a hung request.
5. **Don't retry non-retryable errors.** `IntegrityError` means your data is wrong; retrying it just fails more slowly. `no such table` is a deployment bug.

### Why this transfers unchanged

The structure is identical on other engines; only the predicate changes:

| Engine | Retry on |
|---|---|
| SQLite | `SQLITE_BUSY`, `SQLITE_BUSY_SNAPSHOT`, `SQLITE_LOCKED` |
| PostgreSQL | SQLSTATE `40001` (serialization failure), `40P01` (deadlock detected) |
| Azure SQL / SQL Server | error `1205` (deadlock victim), plus transient faults `40613`, `40197`, `49918`, `10928`, `10929` |

Azure SQL deserves emphasis: it is a managed service that will move your database between nodes for maintenance and failover, so connections drop as a matter of routine operation. **Retry there is mandatory even without any contention at all.** A SQL Server application ported to Azure SQL without retry logic will appear to work and then fail unpredictably in production.

---

## Part 14 — FastAPI wiring

Two dependencies, matching the two context managers.

```python
# db.py
from typing import Annotated, Iterator
from fastapi import Depends
import sqlite3

def get_read_conn() -> Iterator[sqlite3.Connection]:
    conn = connect(settings.db_path, mode="read")
    try:
        conn.execute("BEGIN DEFERRED")      # one snapshot for the whole request
        yield conn
        conn.execute("ROLLBACK")
    except BaseException:
        try:
            conn.execute("ROLLBACK")
        except sqlite3.Error:
            pass
        raise
    finally:
        conn.close()

def get_write_conn() -> Iterator[sqlite3.Connection]:
    conn = connect(settings.db_path, mode="write")
    try:
        conn.execute("BEGIN IMMEDIATE")
        yield conn
        conn.execute("COMMIT")
    except BaseException:
        conn.execute("ROLLBACK")
        raise
    finally:
        conn.close()

ReadDB  = Annotated[sqlite3.Connection, Depends(get_read_conn)]
WriteDB = Annotated[sqlite3.Connection, Depends(get_write_conn)]
```

```python
@router.get("/forecasts/{fid}")
def get_forecast(fid: int, db: ReadDB):          # plain def — sqlite3 blocks
    ...

@router.post("/revisions/{rid}/publish", status_code=204)
def publish(rid: int, db: WriteDB):
    db.execute("UPDATE revisions SET status='superseded' WHERE dataset_id=(SELECT dataset_id FROM revisions WHERE revision_id=?) AND status='published'", (rid,))
    db.execute("UPDATE revisions SET status='published' WHERE revision_id=?", (rid,))
```

### Design notes

**Transaction-per-request.** The `yield` dependency's cleanup runs *after* the response is produced, so a handler that returns normally commits and one that raises — including `HTTPException` — rolls back. This is the property that makes the pattern safe: you never have to remember to commit.

**A read transaction for read endpoints is not overkill.** An endpoint that issues three queries which must agree with each other (monthly detail, category totals, annual summary) will otherwise see three different database states if a publish lands mid-request. That's exactly the reconciliation failure that's hardest to reproduce.

**Where retry goes.** Not in the dependency — by the time the dependency's cleanup runs, the handler has already executed and its Python-side effects have happened. Retry belongs either in middleware that re-runs the whole request, or (simpler and more honest) inside the handler around an explicit closure:

```python
@router.post("/revisions/{rid}/publish", status_code=204)
def publish(rid: int):
    def txn():
        conn = connect(settings.db_path, mode="write")
        try:
            with write_txn(conn):
                conn.execute("UPDATE ...", (rid,))
        finally:
            conn.close()
    with_retry(txn)
```

For a low-write service, the DI form with no retry plus a 503 on `SQLITE_BUSY` is a perfectly reasonable choice — it pushes the retry to the client, which is where HTTP expects it.

**Threading.** `def` handlers run in a threadpool, and each request opens its own connection in its own thread, so `check_same_thread=True` is satisfied without effort. A shared connection across threadpool workers would require `check_same_thread=False` plus a lock, and would serialize everything.

**The streaming trap.** A `StreamingResponse` generator is consumed after the handler returns but *before* the dependency's cleanup, so the connection and its read transaction survive — correct. But the read transaction lives for the entire duration of the download, pinning the WAL. For a large export, either accept the WAL growth, or copy the rows out first, or run the export against a separate snapshot database. `[trap]`

---

## Part 15 — Workflows

### Read-modify-write (the canonical dangerous pattern)

```python
def adjust_amount(conn, stream_id: int, delta: int) -> int:
    with write_txn(conn):                                  # BEGIN IMMEDIATE first
        row = conn.execute(
            "SELECT amount_cents FROM crop_input_streams WHERE stream_id = ?",
            (stream_id,),
        ).fetchone()
        if row is None:
            raise StreamNotFound(stream_id)
        new = row["amount_cents"] + delta
        if new < 0:
            raise ValueError("would go negative")
        conn.execute(
            "UPDATE crop_input_streams SET amount_cents = ? WHERE stream_id = ?",
            (new, stream_id),
        )
        return new
```

The read is inside a transaction that already holds the write lock, so no other writer can interleave. The business rule (`new < 0`) is enforced against a value nobody else can change underneath it, and raising inside the block rolls back.

If the rule were expressible in SQL, the single-statement form would be better still — no window at all:

```sql
UPDATE crop_input_streams SET amount_cents = amount_cents + :delta
WHERE stream_id = :id AND amount_cents + :delta >= 0;
-- then check rowcount: 0 means not found OR rule violated
```

### Bulk load

```python
def load_streams(conn, rows: Iterable[tuple]) -> int:
    with write_txn(conn):
        cur = conn.executemany(
            """INSERT INTO crop_input_streams
                 (revision_id, crop_code, category, month_num, amount_cents)
               VALUES (?, ?, ?, ?, ?)""",
            rows,
        )
        return cur.rowcount
```

One transaction for the whole batch, not one per row. The difference is orders of magnitude, because each autocommit transaction otherwise pays a durability cost. For very large loads, chunk into transactions of ~10k–50k rows so a failure doesn't discard an hour of work and the WAL doesn't grow without bound.

### Multi-statement invariant (the write-skew case)

```python
def unpublish(conn, revision_id: int) -> None:
    with write_txn(conn):
        remaining = conn.execute(
            """SELECT COUNT(*) AS n FROM revisions
               WHERE dataset_id = (SELECT dataset_id FROM revisions WHERE revision_id = ?)
                 AND status = 'published' AND revision_id <> ?""",
            (revision_id, revision_id),
        ).fetchone()["n"]
        if remaining == 0:
            raise ValueError("cannot unpublish the last published revision")
        conn.execute("UPDATE revisions SET status='draft' WHERE revision_id = ?", (revision_id,))
```

On SQLite this is safe with no further thought, because the single write lock makes write transactions genuinely serial. **On PostgreSQL at the default `READ COMMITTED`, or at `REPEATABLE READ`, this exact code has a write-skew bug** — two concurrent calls for different revisions both see `remaining >= 1` and both commit. Fixing it there requires `SERIALIZABLE` plus a retry loop, a `SELECT ... FOR UPDATE` on the dataset row, or a partial unique index that makes the invalid state unrepresentable.

This is the most important portability lesson in the document: **SQLite's single-writer design silently protects code that other engines will not.** Anything written against SQLite and later moved to PostgreSQL needs its multi-statement invariants re-audited.

### Batch rebuild and atomic publish

```python
def rebuild_and_publish(src_csv_dir: str, target: str) -> None:
    tmp = target + ".building"
    conn = connect(tmp, mode="write")
    try:
        conn.execute("PRAGMA journal_mode = WAL")
        conn.executescript(open("schema.sql").read())       # outside any transaction
        with write_txn(conn):
            load_everything(conn, src_csv_dir)
        with read_txn(conn):
            assert_reconciliation(conn)                     # verify before publishing
        conn.execute("PRAGMA wal_checkpoint(TRUNCATE)")     # fold the WAL in
        conn.execute("PRAGMA optimize")
    finally:
        conn.close()
    os.replace(tmp, target)     # atomic on POSIX, same filesystem
```

The checkpoint before the swap matters: without it you'd be renaming a `.db` whose recent contents live in a `-wal` file that is *not* being renamed with it.

### Schema migration

```python
def migrate(conn) -> None:
    version = conn.execute("PRAGMA user_version").fetchone()[0]
    with write_txn(conn):                                 # DDL is transactional in SQLite
        if version < 2:
            conn.execute("ALTER TABLE crop_input_streams ADD COLUMN note TEXT")
        if version < 3:
            conn.execute("CREATE INDEX idx_streams_month ON crop_input_streams(month_num)")
        conn.execute(f"PRAGMA user_version = {TARGET_VERSION}")
    conn.execute("PRAGMA foreign_key_check")              # after, outside the txn
```

DDL inside a transaction works on SQLite and PostgreSQL, and on SQL Server. It does **not** on MySQL/MariaDB, where DDL causes an implicit commit — a failed multi-step migration there leaves you partially migrated.

`PRAGMA user_version` cannot be parameterized (it's a pragma, not a value), hence the f-string with an internal constant.

### Long report with a consistent snapshot

```python
with read_txn(conn):
    monthly  = fetch_monthly(conn)
    by_cat   = fetch_by_category(conn)
    annual   = fetch_annual(conn)
assert sum(monthly.values()) == annual         # guaranteed by the snapshot
```

Without the enclosing `read_txn`, each `fetch_*` is its own transaction and the assertion can fail for reasons that will never reproduce in testing.

---

# Part D — Transfer

## Part 16 — PostgreSQL

### Model

Full MVCC. Every write creates a new row version; readers see versions visible to their snapshot. **Readers never block writers, writers never block readers.** Many concurrent writers, unlike SQLite.

### Boundaries

```sql
BEGIN;                              -- or BEGIN TRANSACTION / START TRANSACTION
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   -- must be the first statement
COMMIT;  /  ROLLBACK;
SAVEPOINT sp;  ROLLBACK TO sp;  RELEASE sp;
```

Psycopg (v2 and v3) defaults to **transactional** mode — it opens a transaction implicitly before your first statement and holds it until you commit. Same shape as Python `sqlite3`'s legacy mode, same fix: `conn.autocommit = True` and write your own `BEGIN`.

### Isolation levels, as actually implemented

| Level | Behaviour |
|---|---|
| `READ COMMITTED` (default) | a **new snapshot per statement**; two `SELECT`s in one transaction can differ |
| `REPEATABLE READ` | one snapshot for the whole transaction; **no phantoms** (stronger than the standard requires); permits write skew; can raise `40001` |
| `SERIALIZABLE` | SSI — tracks read/write dependencies and aborts transactions that would produce a non-serializable outcome; prevents write skew; raises `40001` more often |

Note that `READ UNCOMMITTED` is accepted but behaves as `READ COMMITTED` — PostgreSQL has no dirty reads at all.

### Locks

Row-level, acquired explicitly:

```sql
SELECT ... FOR UPDATE;         -- exclusive row lock; blocks other FOR UPDATE and writes
SELECT ... FOR NO KEY UPDATE;  -- weaker; allows FK-only references
SELECT ... FOR SHARE;          -- shared row lock
SELECT ... FOR UPDATE NOWAIT;         -- fail instead of waiting
SELECT ... FOR UPDATE SKIP LOCKED;    -- skip locked rows — the queue-worker idiom
```

Table-level via `LOCK TABLE`, and advisory locks (`pg_advisory_lock`) for application-defined mutexes that don't correspond to a row.

`SELECT ... FOR UPDATE` is the direct analogue of `BEGIN IMMEDIATE`: it declares write intent before you compute, closing the lost-update window. `SKIP LOCKED` has no SQLite equivalent and is the standard way to build a job queue on Postgres.

### Deadlocks

Real deadlocks are possible (row-level locks in differing orders) and PostgreSQL **detects** them — after `deadlock_timeout` (default 1s) it looks for a cycle and aborts one transaction with `40P01`. This differs from SQLite's *prevention* approach, but produces the same client obligation: catch and retry.

### Operational differences that will surprise you

- **Idle in transaction is expensive.** An open transaction holds back the oldest visible snapshot, preventing vacuum from reclaiming dead tuples anywhere in the database. Symptom: table and index bloat, degrading performance. Set `idle_in_transaction_session_timeout`. This is the same phenomenon as SQLite's WAL growth, with a worse blast radius.
- **A failed statement poisons the transaction.** After any error, the transaction enters an aborted state and every subsequent statement returns `25P02 current transaction is aborted` until you `ROLLBACK` — or roll back to a savepoint. SQLite has no equivalent; a failed statement there leaves the transaction usable. This catches people constantly. `[trap]`
- `DDL is transactional`, like SQLite.
- `synchronous_commit = off` is the analogue of `PRAGMA synchronous = NORMAL`: visible immediately, durable slightly later.

### The one-line summary

SQLite gives you serializability for free by allowing one writer. PostgreSQL gives you many writers and makes serializability something you opt into and retry for.

---

## Part 17 — Azure SQL / SQL Server

### Model

Historically **pessimistic two-phase locking**. Modern SQL Server also supports row-versioning modes, and this is where Azure SQL differs from on-prem in a way that matters:

> **Azure SQL Database has `READ_COMMITTED_SNAPSHOT` (RCSI) enabled by default. On-premises SQL Server does not.**

So the same application code, at the same nominal isolation level `READ COMMITTED`, behaves differently: on Azure readers get a row-versioned snapshot and don't block; on-prem they take shared locks and do. Porting in either direction changes your concurrency characteristics without changing a line of SQL. `[trap]`

### Boundaries

```sql
BEGIN TRANSACTION;
    ...
COMMIT TRANSACTION;   -- or ROLLBACK TRANSACTION;

SAVE TRANSACTION sp;
ROLLBACK TRANSACTION sp;      -- no RELEASE equivalent

SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
SET XACT_ABORT ON;            -- abort the whole transaction on any error
SELECT @@TRANCOUNT;           -- nesting depth
```

Autocommit per statement is the default. `SET IMPLICIT_TRANSACTIONS ON` makes statements open transactions you must close — an easy way to create accidental long-running transactions.

**Nested transactions are pseudo-nested.** `BEGIN TRAN` inside `BEGIN TRAN` only increments `@@TRANCOUNT`; only the outermost `COMMIT` actually commits, but **any `ROLLBACK` rolls back everything** and resets `@@TRANCOUNT` to 0. Genuine partial rollback requires `SAVE TRANSACTION`. This asymmetry has no SQLite or PostgreSQL equivalent and is a frequent source of bugs in stored-procedure code.

`SET XACT_ABORT ON` is close to essential in application code — without it, some errors abort only the statement and leave the transaction open and half-applied.

### Lock modes

| Mode | Meaning |
|---|---|
| **S** shared | readers |
| **X** exclusive | writers |
| **U** update | "I'm reading with intent to write" — compatible with S, incompatible with other U |
| **IS / IX / SIX** intent | held at coarser granularity to make conflict detection cheap |
| **Sch-S / Sch-M** schema | schema stability vs. modification |

The **U lock exists specifically to prevent the shared→exclusive conversion deadlock** — the same problem SQLite solves with `BEGIN IMMEDIATE` and PostgreSQL with `FOR UPDATE`. Three engines, three spellings, one problem. If you internalize nothing else from this part, internalize that.

```sql
SELECT * FROM t WITH (UPDLOCK, HOLDLOCK) WHERE id = @id;   -- take the U lock now
```

### Lock escalation

Unique among the three: SQL Server escalates many fine-grained locks to a coarse one (roughly 5,000 locks on an object → a table lock). An `UPDATE` touching many rows can suddenly lock an entire table, blocking everything. Symptom: a query that scales fine and then abruptly causes system-wide blocking. Mitigations: batch large updates into chunks, or `ALTER TABLE ... SET (LOCK_ESCALATION = DISABLE)`.

### Deadlocks and transient faults

- Deadlock detection by a background monitor; a victim is chosen by rollback cost and killed with error **1205**. Retry required.
- `WITH (NOLOCK)` is `READ UNCOMMITTED` in disguise — it permits dirty reads, and also missing and duplicated rows during page splits. It is widely used and widely misused; it is not a performance setting, it is a correctness trade.
- **Azure-specific:** transient faults from failover and resource governance (`40613`, `40197`, `49918`, `10928`, `10929`) are routine. Retry with backoff is mandatory infrastructure, not an enhancement.

### `SNAPSHOT` isolation

```sql
ALTER DATABASE db SET ALLOW_SNAPSHOT_ISOLATION ON;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

Transaction-scoped snapshot, like PostgreSQL's `REPEATABLE READ`. Permits write skew. Conflicting writes raise error **3960** — retry. Distinct from RCSI, which is statement-scoped and applies to `READ COMMITTED`.

---

## Part 18 — Cross-engine translation table

| Concept | SQLite | PostgreSQL | Azure SQL / SQL Server |
|---|---|---|---|
| Concurrency model | 1 writer; WAL gives readers snapshots | MVCC, many writers | 2PL; RCSI on by default in Azure |
| Start transaction | `BEGIN [DEFERRED\|IMMEDIATE\|EXCLUSIVE]` | `BEGIN` | `BEGIN TRANSACTION` |
| Default level | serializable (fixed) | READ COMMITTED | READ COMMITTED (+RCSI on Azure) |
| Configurable level | **no** | yes | yes |
| Declare write intent early | `BEGIN IMMEDIATE` | `SELECT ... FOR UPDATE` | `WITH (UPDLOCK)` |
| Lock granularity | whole database | row | row → page → table (escalates) |
| Readers block writers | rollback mode: yes; WAL: no | no | on-prem yes; Azure RCSI no |
| Deadlock strategy | **prevention** (returns BUSY) | **detection** (`40P01`) | **detection** (error `1205`) |
| Wait-for-lock timeout | `PRAGMA busy_timeout` | `lock_timeout` | `SET LOCK_TIMEOUT` |
| Retryable errors | `SQLITE_BUSY`, `SQLITE_BUSY_SNAPSHOT` | `40001`, `40P01` | `1205`, `3960`, transient `4xxxx` |
| Nested transactions | savepoints only | savepoints only | pseudo-nested + `SAVE TRANSACTION` |
| Savepoint release | `RELEASE` | `RELEASE` | *(none)* |
| Error aborts transaction | no | **yes** (`25P02`) | depends; `SET XACT_ABORT ON` |
| DDL in a transaction | yes | yes | yes |
| Version/WAL cleanup | WAL checkpoint | `VACUUM` | ghost cleanup / version store in tempdb |
| Cost of a long transaction | WAL growth | table bloat, blocked vacuum | version store growth, blocking |
| Durability dial | `PRAGMA synchronous` | `synchronous_commit` | delayed durability |
| Queue idiom | none | `FOR UPDATE SKIP LOCKED` | `WITH (READPAST, UPDLOCK)` |

### The five things that transfer unchanged

1. **Declare write intent before you read, if you're going to write.** Different spelling, same necessity, every engine.
2. **Retry on conflict.** Different error codes, same loop.
3. **Keep transactions short.** Every engine punishes long transactions; only the symptom differs.
4. **Do read-modify-write in one SQL statement when you can.** Removes the window entirely, everywhere.
5. **Make invalid states unrepresentable with constraints.** A partial unique index enforcing "one published revision per dataset" is correct under every isolation level on every engine, with no locking reasoning required. Constraints are the concurrency control that doesn't need retry logic.

---

## Part 19 — Traps ledger

| # | Trap | Why | Fix |
|---|---|---|---|
| 1 | Write vanishes after script ends | no implicit commit on `close()` | explicit `COMMIT`, or a context manager |
| 2 | `PRAGMA foreign_keys = ON` silently ignored | a transaction was already open | `isolation_level=None`; run pragmas at connect time |
| 3 | `database is locked` instantly despite a long `busy_timeout` | promotion deadlock — busy handler deliberately skipped | `BEGIN IMMEDIATE` for write transactions |
| 4 | `SQLITE_BUSY_SNAPSHOT` on the `UPDATE` in a read-then-write txn | deferred txn's snapshot went stale | `BEGIN IMMEDIATE` before the read |
| 5 | Reader never sees a committed write | it's holding a snapshot from before the commit | end and restart the read transaction |
| 6 | `-wal` file grows to gigabytes | a long-lived reader pins frames from checkpointing | close the reader; check for unfetched cursors |
| 7 | Writer's `COMMIT` blocks for seconds | rollback journal mode; PENDING waiting on a reader's SHARED | switch to WAL |
| 8 | `mode=ro` connection fails to open a WAL database | read marks require writing `-shm` | `PRAGMA query_only = ON` instead |
| 9 | Streaming export holds a transaction for minutes | the cursor's read txn lives as long as the generator | accept it, or materialize first |
| 10 | Lost update in read-modify-write | window between `SELECT` and `UPDATE` | `BEGIN IMMEDIATE`, or do it in one statement |
| 11 | Retry loop makes things worse | no jitter, all workers collide again | exponential backoff **plus** jitter |
| 12 | Retrying an `IntegrityError` | it's a data bug, not contention | retry only lock/serialization errors |
| 13 | `BEGIN` inside `BEGIN` errors out | SQLite has no nested transactions | savepoints |
| 14 | `executescript` commits your open transaction | it issues an implicit `COMMIT` first | never call it inside a transaction |
| 15 | Bulk insert is glacially slow | one autocommit transaction per row | wrap the batch in one transaction |
| 16 | Rows lost after an atomic file swap | the `-wal` wasn't folded into the `.db` | `wal_checkpoint(TRUNCATE)` before `os.replace` |
| 17 | Multi-statement invariant breaks after porting to Postgres | SQLite's single writer was hiding write skew | `SERIALIZABLE` + retry, `FOR UPDATE`, or a constraint |
| 18 | Every Postgres statement fails after one error | transaction is in aborted state (`25P02`) | `ROLLBACK`, or roll back to a savepoint |
| 19 | Azure SQL app fails intermittently with no contention | managed-service failovers are routine | mandatory transient-fault retry |
| 20 | SQL Server `ROLLBACK` undoes more than expected | nested transactions are pseudo-nested | `SAVE TRANSACTION` |
| 21 | Two `SELECT`s in one request disagree | each ran in its own transaction | wrap the request's reads in one read transaction |

---

## Appendix A — Setup checklist

**At database creation (once, persistent):**
```sql
PRAGMA journal_mode = WAL;
PRAGMA user_version = <n>;
```

**In the connection factory (every connection, in this order):**
```python
sqlite3.connect(path, isolation_level=None, timeout=5.0)
conn.row_factory = sqlite3.Row
conn.execute("PRAGMA busy_timeout = 5000")
conn.execute("PRAGMA foreign_keys = ON")
conn.execute("PRAGMA synchronous = NORMAL")
conn.execute("PRAGMA query_only = ON")        # read connections only
assert not conn.in_transaction
```

**In application code:**
- reads that must agree → `BEGIN DEFERRED` … `ROLLBACK`
- anything that writes → `BEGIN IMMEDIATE` … `COMMIT` / `ROLLBACK`
- no `with conn:` anywhere
- no `executescript` inside a transaction
- retry `SQLITE_BUSY` / `SQLITE_BUSY_SNAPSHOT` with backoff and jitter
- one transaction per batch, not per row

**Assertions worth keeping permanently in tests:**
```python
assert not conn.in_transaction            # after connect
conn.execute("INSERT ...")
assert not conn.in_transaction            # proves no driver-injected BEGIN
assert conn.execute("PRAGMA foreign_keys").fetchone()[0] == 1
assert conn.execute("PRAGMA journal_mode").fetchone()[0] == "wal"
```

## Appendix B — To add as this evolves

- Measuring lock contention: `sqlite3_status`, `set_progress_handler`, timing instrumentation
- Concurrency tests: spawning real threads/processes to reproduce `SQLITE_BUSY` deterministically
- `BEGIN CONCURRENT` (the experimental SQLite branch) and what it would change
- Shared-cache mode, `SQLITE_LOCKED`, and `PRAGMA read_uncommitted`
- PostgreSQL SSI internals: predicate locks, rw-antidependencies, false positives
- Connection pooling and transaction scope (pgbouncer transaction mode and what it forbids)
- Distributed transactions, two-phase commit, and why to avoid them
- Idempotency keys as an alternative to retry-safety reasoning
- Outbox pattern for transactional messaging