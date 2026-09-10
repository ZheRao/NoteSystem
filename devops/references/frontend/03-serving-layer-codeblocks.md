# Create Database in `db_create.py`

```python
"""
initialize database
"""


import sqlite3

CONFIG_DB_PATH = 'test.db'

def initialize_database():
    conn = sqlite3.connect(CONFIG_DB_PATH)
    conn.execute("PRAGMA journal_mode = WAL")

    # create client table
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS clients(
            client_id INTEGER PRIMARY KEY,
            client_name TEXT NOT NULL UNIQUE,
            last_modified TEXT NOT NULL DEFAULT (datetime('now')),
            modified_by TEXT NOT NULL
        )
        """
    )

    # create audit table
    ##  change_id = write transaction id, one write might impact several cols, create separate entries to the audit table per col change
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS audit_log(
            audit_id        INTEGER PRIMARY KEY,
            table_name      TEXT NOT NULL,
            row_id          TEXT NOT NULL,
            change_id       TEXT NOT NULL,
            operation       TEXT NOT CHECK (operation IN ('create', 'update', 'delete')),
            col_changed     TEXT NOT NULL,
            old_value       TEXT,
            new_value       TEXT,
            changed_by      TEXT NOT NULL,
            changed_at      TEXT NOT NULL DEFAULT (datetime('now'))
        )
        """
    )

    conn.commit()
    conn.close()
    print("Client and Audit table created successfully\n")
```


# Build Connection-Creation Mechanism in `store.py`

```python
"""
Create SQLite connection
"""

import sqlite3
from contextlib import contextmanager
from typing import Iterator

CONFIG_DB_PATH = '../test.db'


def init_connect() -> sqlite3.Connection:
    conn = sqlite3.connect(
        CONFIG_DB_PATH,
        isolation_level=None,
    )
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA busy_timeout = 5000")
    return conn


@contextmanager
def connect_write() -> Iterator[sqlite3.Connection]:
    conn = init_connect()
    try:
        conn.execute("BEGIN IMMEDIATE")
        yield conn
        conn.execute("COMMIT")
    except Exception:
        conn.execute("ROLLBACK")
        raise
    finally:
        conn.close()

@contextmanager
def connect_read() -> Iterator[sqlite3.Connection]:
    conn = init_connect()
    try:
        yield conn
    finally:
        conn.close()

def get_write_db() -> Iterator[sqlite3.Connection]:
    with connect_write() as conn:
        yield conn

def get_read_db() -> Iterator[sqlite3.Connection]:
    with connect_read() as conn:
        yield conn
```

Key points
- [Why need two layers of abstraction?](../../foundations/mechanisms/beyond-infra/serving-layer/05-appendix-notes.md)
- [Explicitly managing write transaction](./02-sqlite-fastapi.md)
- Seperating `get_write_db()` and `get_read_db()` with clearer FastAPI endpoint delaration with `conn=Depends(get_write_db)` as opposed to `conn=Depends(get_db(write=True))`
  - FastAPI dependencies are much clearner when you can hand `Depends()` a dependency function directly
  - parameterizing `get_db(write=False)` generally pushes you toward wrappers/factories anyway













