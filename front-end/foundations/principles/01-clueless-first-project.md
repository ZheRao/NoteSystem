
# Concrete Compounding Learning

Yes. Here’s the version that compounds instead of becoming a throwaway tutorial.

The stack I’d use for your first vertical slice is:
- **PostgreSQL** as the queryable data engine,
- **FastAPI** as the backend API layer,
- **psycopg 3** to talk to PostgreSQL from Python,
- and the **official Postgres Docker image** so setup stays simple and reproducible. 
FastAPI’s official docs position it as a Python API framework built around standard type hints, 
PostgreSQL’s current docs provide the SQL/database side, psycopg 3 has an official pool package, 
and Docker documents using the official Postgres image for local development.

The goal is not “learn SQL.” The goal is to build the smallest real loop:
```
user action -> HTTP request -> backend parses params -> SQL query -> database result -> JSON response
```
That is the missing system category.

## What you are building

A tiny analytics service for Growlytics-style filtering.

Example user action:
> “Show total approved funding for Saskatchewan grants in 2025”

Your first slice will support simpler filters like:
- province
- status
- min_amount

The frontend can be fake at first. A browser URL or Swagger UI is enough. FastAPI includes automatic interactive API docs for endpoints, 
so you can test requests without building a UI first.

## Why this version compounds

Because it teaches five layers at once, in the right order:

1. **Data modeling**  
What table shape supports querying?
2. **API boundary design**  
What filters should the endpoint accept?
3. **Query translation**  
How do query params map to SQL predicates?
4. **Database execution**  
What happens inside a real SQL engine?
5. **System packaging**  
How do you run the whole thing reproducibly?

That is much closer to the real Growlytics bottleneck than isolated SQL practice.

## Repo structure

Use this:
```
growlytics-query-slice/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── db.py
│   ├── queries.py
│   └── models.py
├── sql/
│   ├── 001_schema.sql
│   └── 002_seed.sql
├── .env
├── docker-compose.yml
├── requirements.txt
└── README.md
```

**Why this structure matters**
- `sql/` keeps schema and seed data explicit
- `db.py` isolates connection logic
- `queries.py` isolates SQL generation
- `main.py` isolates HTTP handling

That mirrors your longer-term architecture instinct: separate transport, storage, and logic.

## Step 1: run Postgres locally

`docker-compose.yml`
```
services:
  postgres:
    image: postgres:18
    container_name: growlytics_postgres
    environment:
      POSTGRES_DB: growlytics
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: app_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```
PostgreSQL’s official docs currently list version 18 as current, and Docker’s official Postgres image is the standard local-container path 
for getting a database running quickly.

Run:
```bash
docker compose up -d
```

## Step 2: define a tiny schema

`sql/001_schema.sql`
```SQL
CREATE TABLE grants (
    grant_id SERIAL PRIMARY KEY,
    grant_name TEXT NOT NULL,
    province TEXT,
    applicant_type TEXT,
    status TEXT NOT NULL,
    amount NUMERIC(12, 2),
    open_date DATE,
    close_date DATE
);
```
`sql/002_seed.sql`
```SQL
INSERT INTO grants (
    grant_name, province, applicant_type, status, amount, open_date, close_date
) VALUES
('Agri-Tech Innovation Fund', 'SK', 'business', 'open', 50000.00, '2025-01-01', '2025-06-30'),
('Rural Expansion Support', 'AB', 'business', 'closed', 120000.00, '2025-02-01', '2025-04-30'),
('Women Founder Accelerator Grant', 'SK', 'individual', 'open', 30000.00, '2025-03-15', '2025-07-15'),
('Clean Farm Equipment Upgrade', 'MB', 'business', 'open', 85000.00, '2025-01-20', '2025-08-01'),
('Youth Entrepreneurship Stream', 'SK', 'individual', 'closed', NULL, '2025-01-10', '2025-03-31');
```
Load them:
```bash
docker exec -i growlytics_postgres psql -U app_user -d growlytics < sql/001_schema.sql
docker exec -i growlytics_postgres psql -U app_user -d growlytics < sql/002_seed.sql
```

## Step 3: install backend dependencies

`requirements.txt`
```
fastapi
uvicorn[standard]
psycopg[binary]
psycopg_pool
pydantic
python-dotenv
```
FastAPI’s official docs cover the app framework and interactive docs, and psycopg’s official docs include psycopg_pool for 
maintaining a controlled set of PostgreSQL connections.

Install:
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Windows PowerShell:
```PowerShell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

## Step 4: configure the connection

`.env`
```
DATABASE_URL=postgresql://app_user:app_password@localhost:5432/growlytics
```

`app/db.py`
```py
import os
from contextlib import contextmanager

from dotenv import load_dotenv
from psycopg_pool import ConnectionPool

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")
if not DATABASE_URL:
    raise ValueError("DATABASE_URL is not set")

pool = ConnectionPool(conninfo=DATABASE_URL, open=False)

def open_pool() -> None:
    pool.open()

def close_pool() -> None:
    pool.close()

@contextmanager
def get_conn():
    with pool.connection() as conn:
        yield conn
```
This is already a compounding choice: even though a single connection would work, the pool introduces the real concept that APIs serve 
many requests while constraining database connections. That is exactly the kind of system thinking you want. 
psycopg documents the pool package for creating and maintaining a limited amount of PostgreSQL connections.

## Step 5: define the response model

`app/models.py`
```py
from pydantic import BaseModel

class GrantSummary(BaseModel):
    grant_id: int
    grant_name: str
    province: str | None
    applicant_type: str | None
    status: str
    amount: float | None

class FundingAggregate(BaseModel):
    row_count: int
    total_amount: float | None
```

## Step 6: isolate SQL generation

`app/queries.py`
```py
from typing import Any

def build_grant_filter_query(
    province: str | None = None,
    status: str | None = None,
    min_amount: float | None = None,
):
    conditions: list[str] = []
    params: list[Any] = []

    if province is not None:
        conditions.append("province = %s")
        params.append(province)

    if status is not None:
        conditions.append("status = %s")
        params.append(status)

    if min_amount is not None:
        conditions.append("amount >= %s")
        params.append(min_amount)

    where_clause = ""
    if conditions:
        where_clause = "WHERE " + " AND ".join(conditions)

    sql = f"""
        SELECT
            grant_id,
            grant_name,
            province,
            applicant_type,
            status,
            amount
        FROM grants
        {where_clause}
        ORDER BY grant_id
    """
    return sql, params


def build_aggregate_query(
    province: str | None = None,
    status: str | None = None,
):
    conditions: list[str] = []
    params: list[Any] = []

    if province is not None:
        conditions.append("province = %s")
        params.append(province)

    if status is not None:
        conditions.append("status = %s")
        params.append(status)

    where_clause = ""
    if conditions:
        where_clause = "WHERE " + " AND ".join(conditions)

    sql = f"""
        SELECT
            COUNT(*) AS row_count,
            SUM(amount) AS total_amount
        FROM grants
        {where_clause}
    """
    return sql, params
```

### Why this file matters

This is the exact missing bridge in your mental model.

Not “user clicked a filter.”  
But:
- request param arrives
- predicate list is built
- SQL is assembled
- params are bound
- database executes

That is the backend translation layer.

Also notice: I used parameter binding (`%s` + params) instead of string interpolation for values. That is the habit you want from day one.

## Step 7: expose the API

`app/main.py`
```py
from contextlib import asynccontextmanager

from fastapi import FastAPI, Query

from app.db import close_pool, get_conn, open_pool
from app.models import FundingAggregate, GrantSummary
from app.queries import build_aggregate_query, build_grant_filter_query

@asynccontextmanager
async def lifespan(app: FastAPI):
    open_pool()
    yield
    close_pool()

app = FastAPI(title="Growlytics Query Slice", lifespan=lifespan)

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/grants", response_model=list[GrantSummary])
def list_grants(
    province: str | None = Query(default=None),
    status: str | None = Query(default=None),
    min_amount: float | None = Query(default=None),
):
    sql, params = build_grant_filter_query(
        province=province,
        status=status,
        min_amount=min_amount,
    )

    with get_conn() as conn:
        with conn.cursor() as cur:
            cur.execute(sql, params)
            rows = cur.fetchall()

    return [
        GrantSummary(
            grant_id=row[0],
            grant_name=row[1],
            province=row[2],
            applicant_type=row[3],
            status=row[4],
            amount=float(row[5]) if row[5] is not None else None,
        )
        for row in rows
    ]

@app.get("/funding/summary", response_model=FundingAggregate)
def funding_summary(
    province: str | None = Query(default=None),
    status: str | None = Query(default=None),
):
    sql, params = build_aggregate_query(province=province, status=status)

    with get_conn() as conn:
        with conn.cursor() as cur:
            cur.execute(sql, params)
            row = cur.fetchone()

    return FundingAggregate(
        row_count=row[0],
        total_amount=float(row[1]) if row[1] is not None else None,
    )
```

FastAPI’s current docs cover query parameters, path operations, and app structure, and the docs also describe lifespan/event patterns for startup and shutdown.

Run it:
```bash
uvicorn app.main:app --reload
```

Open:
```
http://127.0.0.1:8000/docs
```

## What to test

These are your fake “frontend actions.”

### 1. No filters
```
GET /grants
```

### 2. Filter by province
```
GET /grants?province=SK
```

### 3. Filter by province and status
```
GET /grants?province=SK&status=open
```

### 4. Aggregate summary
```
GET /funding/summary?province=SK&status=open
```

### That is the full loop:
- user intent
- HTTP request
- API parse
- SQL generation
- DB execution
- JSON response

## The most important learning pass

After you make it work, do **not** move on immediately.

Pause and inspect each layer.

### Pass 1: trace the request

Take:
```
GET /grants?province=SK&status=open
```
Then explicitly write down:
- frontend/user action
- URL query params
- Python variables
- generated SQL
- query result rows
- returned JSON

That trace is the real learning artifact.

### Add one debugging line

Temporarily add this inside each endpoint:
```py
print("SQL:", sql)
print("PARAMS:", params)
```
Now when you hit the endpoint, you will literally see the translation layer you were missing.

That one tiny addition will make the architecture click much faster than reading theory.

## Your first compounding extension

Once the base works, add one of these. Only one.

### Option A: null-aware filtering

Add:
```
GET /grants?status=open&has_amount=true
```
Then translate `has_amount=true` into:
```SQL
amount IS NOT NULL
```
This directly trains the SQL null behavior you were asking about.

### Option B: grouping

Add:
```
GET /funding/by-province
```
With SQL:
```SQL
SELECT province, COUNT(*), SUM(amount)
FROM grants
GROUP BY province
ORDER BY province;
```
This teaches the analytical side.

### Option C: text search

Add:
```
GET /grants/search?q=innovation
```
With SQL:
```SQL
WHERE grant_name ILIKE %s
```
This teaches search predicates.

Pick **A first**. It has the highest learning density for your current gap.

## How this maps to Growlytics scoping

This micro-project will let you estimate real product work much more honestly.

Because then you can separate:

**Already familiar**
- source ingestion
- normalization
- table design
- metrics logic

**Newly concrete**
- API endpoint design
- query generation
- null semantics in SQL
- interactive filtering
- latency/debugging concerns

That means Growlytics stops being a foggy “frontend/backend product” and becomes a set of scoped layers.

## The trap to avoid

Do not add these yet:
- auth
- React frontend
- ORMs
- deployment
- caching
- pagination
- background jobs

Those are later layers.

Right now the point is to make the translation boundary visible.

## Your compounding roadmap after this

After this slice works:

**Slice 1**  
Single table, simple filters, one aggregate.

**Slice 2**  
Join two tables:
- `grants`
- `applicants`

Then you’ll feel where joins and nulls actually matter.

**Slice 3**  
Materialized analytics table or pre-aggregated table.  
Then you’ll understand when to compute on read vs compute in ETL.

**Slice 4**  
Tiny frontend with dropdown filters.  
Only then add UI.  
That order matches your trajectory much better than starting from a flashy frontend.

## Blunt calibration

Your instinct here is correct:  
Growlytics is probably not something to walk away from if you keep it tightly scoped and force it to teach the missing system boundary instead of becoming vague chaos.

The wrong version is:
- “build the whole client product”

The right version is:
- “use the project to master one missing architectural seam at a time”

That is high future value.

**Lean in — core growth**.