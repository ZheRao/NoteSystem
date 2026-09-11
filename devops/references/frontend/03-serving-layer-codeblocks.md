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
    ##  row_id = the id column (primary key) of the 'table_name' table, e.g., 'client_id' is the 'row_id' for 'client' table
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

    ## trigger to ensure audit table is append-only
    conn.execute(
        """
        CREATE TRIGGER IF NOT EXISTS audit_log_no_update
        BEFORE UPDATE ON audit_log
        BEGIN SELECT RAISE(ABORT, 'audit_log is append-only'); END
        """
    )
    conn.execute(
        """
        CREATE TRIGGER IF NOT EXISTS audit_log_no_delete
        BEFORE DELETE ON audit_log
        BEGIN SELECT RAISE(ABORT, 'audit_log is append-only'); END
        """
    )

    conn.commit()
    conn.close()
    print("Client and Audit table created successfully\n")
```

**Key points**
- `TRIGGER` that enforce append-only rule to `audit_log` table
- initialize database file with `WAL` mode, only once not per connection
- didn't include index creation


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

**Key points**
- [Why need two layers of abstraction?](../../foundations/mechanisms/beyond-infra/serving-layer/05-appendix-notes.md)
- [Explicitly managing write transaction](./02-sqlite-fastapi.md)
- Seperating `get_write_db()` and `get_read_db()` with clearer FastAPI endpoint delaration with `conn=Depends(get_write_db)` as opposed to `conn=Depends(get_db(write=True))`
  - FastAPI dependencies are much clearner when you can hand `Depends()` a dependency function directly
  - parameterizing `get_db(write=False)` generally pushes you toward wrappers/factories anyway



# Sample CRUD API endpoints with audit logs

```py
from pydantic import BaseModel
import sqlite3
from fastapi import Depends, FastAPI, HTTPException
import uuid

from store import get_read_db, get_write_db

class ClientCreate(BaseModel):
    client_name: str
    modified_by: str

class ClientRead(BaseModel):
    client_id: int
    client_name: str
    last_modified: str
    modified_by: str

class ClientUpdate(BaseModel):
    new_client_name: str
    modified_by: str

class ClientDelete(BaseModel):
    modified_by: str

app = FastAPI(title="test", version="0.1.0")

# helper for maintaining audit trail
AUDIT_COLUMNS: dict[str, set[str]] = {
    "client": {"client_name"}
}

def write_audit(
    conn: sqlite3.Connection,
    table_name: str,
    row_id: int,
    operation: str,
    changed_by: str,
    before: dict | None = None,
    after: dict | None = None
) -> None:
    """
    Receives
    - row-level modification identity: table_name, row_id, operation, changed_by, represented by one create/update/delete transaction
    - column-level modification: all columns:value pairs in the before/after transaction, this function will only record actual column value changes across the transaction
    """
    change_id = str(uuid.uuid4())   # generate id for the change, may include multiple column changes
    # ensure 'before' and 'after' are dictionaries even if they are 'None' 
    before = before or {}
    after = after or {}
    tracked = AUDIT_COLUMNS[table_name]    # raise if table name doesn't exist in audit_columns config, need to understand how to proporgate the error message to frontend

    rows:list[tuple[str, str, str, str, str, str, str | None, str | None]] = []
    for col in sorted(tracked):
        old, new = before.get(col), after.get(col)
        # if the updating operation is not mutating column values, skip
        if (operation == "update") and (old == new):
            continue
        rows.append(
            (change_id, table_name, str(row_id), operation, changed_by, col,
             None if (old is None) else str(old),
             None if (new is None) else str(new),
             )
        )

    # actual assertion
    if rows:
        query = """
                INSERT INTO audit_log
                (change_id, table_name, row_id, operation, changed_by, col_changed, old_value, new_value)
                VALUES
                (?, ?, ?, ?, ?, ?, ?, ?)
                """
        conn.executemany(
            query,
            rows
        )

    


# insert client
@app.post("/client", status_code=201, response_model=ClientRead)
def client_create(
    payload: ClientCreate,
    conn: sqlite3.Connection = Depends(get_write_db)
) -> ClientRead:
    query = """
            INSERT INTO client (client_name, modified_by)
            VALUES (?, ?)
            RETURNING client_id, client_name, last_modified, modified_by
            """
    try:
        row = conn.execute(query, (payload.client_name, payload.modified_by)).fetchone()
    except sqlite3.IntegrityError:
        raise HTTPException(status_code=409, detail=f"Client '{payload.client_name}' already exists")

    # audit log
    write_audit(
        conn,
        table_name="client",
        row_id=row["client_id"],
        operation="create",
        changed_by=payload.modified_by,
        before=None,
        after=dict(row)
    )

    return ClientRead(**dict(row))

# read client list
@app.get("/client", response_model=list[ClientRead])
def client_read(
    conn: sqlite3.Connection = Depends(get_read_db)
) -> list[ClientRead]:
    query = """
            SELECT client_id, client_name, last_modified, modified_by 
            FROM client
            ORDER BY last_modified DESC
            """
    rows = conn.execute(query).fetchall()
    return [ClientRead(**dict(row)) for row in rows]

# delete client
@app.delete("/client/{client_id}", status_code=204)
def client_delete(
    client_id: int,
    payload: ClientDelete,
    conn: sqlite3.Connection = Depends(get_write_db)
) -> None:
    query = """
            DELETE FROM client WHERE client_id = ?
            RETURNING client_id, client_name, last_modified, modified_by
            """
    row = conn.execute(query, (client_id,)).fetchone()
    if row is None:
        raise HTTPException(status_code=404, detail=f"Client {client_id} not found\n")

    # audit log
    write_audit(
        conn,
        table_name="client",
        row_id=row["client_id"],
        operation="delete",
        changed_by=payload.modified_by,
        before=dict(row),
        after=None
    )

# update client name
@app.patch("/client/{client_id}", response_model=ClientRead)
def client_update(
    client_id: int,
    payload: ClientUpdate,
    conn: sqlite3.Connection = Depends(get_write_db)
) -> ClientRead:
    query_read = """
                SELECT client_id, client_name, last_modified, modified_by FROM client
                WHERE client_id = ?
                """
    row_read = conn.execute(query_read, (client_id,)).fetchone()
    if row_read is None:
        raise HTTPException(status_code=404, detail=f"Client {client_id} not found\n")
    query_update = """
            UPDATE  client
            SET     client_name = ?,
                    modified_by = ?,
                    last_modified = datetime('now')
            WHERE   client_id = ?
            RETURNING client_id, client_name, last_modified, modified_by
            """
    try:
        row_updated = conn.execute(
            query_update,
            (payload.new_client_name, payload.modified_by, client_id)
        ).fetchone()
    except sqlite3.IntegrityError:
        raise HTTPException(
            status_code=409,
            detail=f"Client '{payload.new_client_name}' already exists\n"
        )
    if row_updated is None:
        raise HTTPException(status_code=404, detail=f"Client {client_id} not found\n")

    # audit log
    write_audit(
        conn,
        table_name="client",
        row_id=row_updated["client_id"],
        operation="update",
        changed_by=payload.modified_by,
        before=dict(row_read),
        after=dict(row_updated)
    )

    return ClientRead(**dict(row_updated))
```

**Key points**
- CRUD operations
  - write = `POST`
    - return inserted row as if it's a read operation
    - no path parameters
    - body parameter `payload` to capture values to be written
    - `409` for already-written (duplicated writes)
  - read = `GET`
    - return all clients, multiple rows
  - delete = `DELETE`
    - return nothing
    - capture `id` as path parameter
    - capture `modified_by` as body parameter
    - `404` data aimed to delete not found
  - update = `PATCH`
    - have to read target data first, store as old, for audit log entry
    - capture `id` in path parameter
    - capture new information in body parameter `payload`
    - `404` target modification row not found
    - `409` new data already exists (if the target column requires `UNIQUE`) 
- one helper function insersion for `audit_log` for all operations
  - keep `AUDIT_COLUMNS` as target column to be tracked
  - multiple column changes in one transaction = multiple insersion entries to `audit_log`
  - function uses `executemany` for potential batch processing










