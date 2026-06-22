# The Serving Layer & Frontend Communication — A Beginner's Field Guide

**Stack covered:** FastAPI · SQLite (via SQLAlchemy) · REST · GraphQL · talking to all of it from a frontend

> This is a living document. It's written to be read top-to-bottom the first time, then used as a lookup reference afterward. Sections are deliberately self-contained so you can add your own notes, snippets, and "gotchas I hit" as you go.

---

## Table of Contents

1. [The big picture: what a "serving layer" even is](#1-the-big-picture)
2. [The request/response lifecycle](#2-the-requestresponse-lifecycle)
3. [FastAPI fundamentals](#3-fastapi-fundamentals)
4. [Pydantic: the data-shape layer](#4-pydantic)
5. [SQLite + SQLAlchemy: the storage layer](#5-sqlite--sqlalchemy)
6. [Putting it together: a complete REST CRUD API](#6-complete-rest-crud-api)
7. [REST done properly (conventions that matter)](#7-rest-conventions)
8. [GraphQL: a different shape of API](#8-graphql)
9. [The frontend side: actually calling your API](#9-the-frontend-side)
10. [REST vs GraphQL: when to use which](#10-rest-vs-graphql)
11. [Cross-cutting concerns you'll hit soon](#11-cross-cutting-concerns)
12. [Where to go next](#12-where-to-go-next)
13. [Glossary](#13-glossary)

---

## 1. The Big Picture
<a name="1-the-big-picture"></a>

Before any code, build the mental model. Almost every web app is three layers talking to each other:

```
  ┌─────────────┐        HTTP         ┌──────────────┐      SQL       ┌──────────────┐
  │   FRONTEND  │  ───────────────▶   │ SERVING LAYER│  ──────────▶   │   DATABASE   │
  │ (browser /  │   request           │  (FastAPI)   │   query        │   (SQLite)   │
  │  mobile app)│  ◀───────────────   │              │  ◀──────────   │              │
  └─────────────┘        JSON         └──────────────┘    rows        └──────────────┘
     "the view"          response       "the brain"                    "the memory"
```

- **Frontend** — what the user sees and clicks. It holds *no source of truth*; it asks the serving layer for data and tells it about changes.
- **Serving layer (a.k.a. backend / API server)** — the program you'll write in FastAPI. It receives requests, decides what's allowed, talks to the database, applies your business rules, and sends back a response. This is the part this guide is mostly about.
- **Database** — durable storage. SQLite is a file on disk that speaks SQL. The serving layer is the *only* thing that talks to it; the frontend never touches the database directly.

**The single most important idea for a beginner:** the frontend and the serving layer are *separate programs*, often in separate languages, that only communicate by sending text (usually JSON) over HTTP. They share no memory and no variables. Every interaction is a message. Internalize this and 80% of the confusion disappears.

---

## 2. The Request/Response Lifecycle
<a name="2-the-requestresponse-lifecycle"></a>

Everything in HTTP is a **request** followed by a **response**. Understanding the anatomy of each makes APIs feel concrete instead of magic.

### An HTTP request has four parts

```
POST /users HTTP/1.1                ← (1) METHOD + PATH + protocol
Host: api.example.com               ← (2) HEADERS (metadata)
Content-Type: application/json
Authorization: Bearer abc123

{ "name": "Ada", "email": "a@x.io" } ← (4) BODY (the payload, optional)
```

1. **Method (verb):** what kind of action. `GET` (read), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (remove). These are conventions, not enforced rules — but breaking them confuses everyone, so follow them.
2. **Path (URL):** *what* you're acting on, e.g. `/users` or `/users/42`.
3. **Headers:** metadata — what format the body is in, who you are, what response formats you accept.
4. **Body:** the actual data, used by `POST`/`PUT`/`PATCH`. `GET` and `DELETE` usually have none.

There are also **query parameters** — the `?key=value` part of a URL, e.g. `/users?limit=10&active=true`. Use them for filtering, sorting, and pagination on reads.

### An HTTP response has three parts

```
HTTP/1.1 201 Created                ← (1) STATUS CODE + reason
Content-Type: application/json      ← (2) HEADERS

{ "id": 42, "name": "Ada" }         ← (3) BODY
```

**Status codes** are a compact language. Memorize the families:

| Range | Meaning | You'll use most |
|-------|---------|-----------------|
| `2xx` | Success | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | Redirect | (rarely hand-written in APIs) |
| `4xx` | *Client's* fault | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `422 Unprocessable Entity` |
| `5xx` | *Server's* fault | `500 Internal Server Error` |

The 4xx-vs-5xx distinction matters: `4xx` says "you sent me something wrong," `5xx` says "I broke." FastAPI returns `422` automatically when incoming data fails validation — you'll see this constantly.

### The full round trip in our stack

1. User clicks a button → frontend runs `fetch()` → an HTTP request leaves the browser.
2. FastAPI matches the request's method+path to one of your functions ("path operations").
3. FastAPI validates the incoming data against the shape you declared (Pydantic).
4. Your function runs — typically querying SQLite via SQLAlchemy.
5. You return a Python object; FastAPI serializes it to JSON and attaches a status code.
6. The response travels back; the frontend's `fetch()` promise resolves; the UI updates.

Keep this list visible the first few times you read code — every example below is just one of these steps.

---

## 3. FastAPI Fundamentals
<a name="3-fastapi-fundamentals"></a>

FastAPI is a Python framework for building the serving layer. Its selling points: you describe your data with normal Python type hints, and it gives you **automatic validation, automatic JSON serialization, and automatic interactive documentation** for free.

### 3.1 Setup

```bash
# A virtual environment keeps this project's packages isolated.
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install "fastapi[standard]"  # includes uvicorn, the server that runs your app
```

> `fastapi[standard]` pulls in `uvicorn` (the ASGI server that actually listens on a port) plus sensible extras. You write the *app*; uvicorn *runs* it.

### 3.2 Your first endpoint

Create `main.py`:

```python
from fastapi import FastAPI

app = FastAPI()                      # the application object

@app.get("/")                        # method = GET, path = "/"
def read_root():
    return {"message": "Hello, serving layer!"}
```

Run it:

```bash
fastapi dev main.py        # development mode: auto-reloads on file save
# (equivalently: uvicorn main:app --reload)
```

Now visit:
- `http://127.0.0.1:8000/` → your JSON message.
- `http://127.0.0.1:8000/docs` → **interactive API docs** (Swagger UI), auto-generated. You can fire real requests from this page. This is your best friend while learning.
- `http://127.0.0.1:8000/redoc` → an alternative docs view.

Notice you returned a Python `dict` and got JSON back. FastAPI did the conversion. That's the recurring theme: *you work in Python objects; FastAPI handles the wire format.*

### 3.3 Path parameters — data baked into the URL

```python
@app.get("/users/{user_id}")
def get_user(user_id: int):          # the int hint is doing real work
    return {"user_id": user_id, "type": str(type(user_id))}
```

- `GET /users/42` → `{"user_id": 42, ...}` and `user_id` is an actual `int`.
- `GET /users/abc` → automatic `422` error, because `"abc"` isn't an `int`.

That type hint isn't decoration. FastAPI reads it to (a) convert the string from the URL into the right type and (b) reject bad input before your code runs.

### 3.4 Query parameters — the `?key=value` part

Any function argument that *isn't* in the path becomes a query parameter:

```python
@app.get("/users")
def list_users(limit: int = 10, active: bool = True, search: str | None = None):
    # GET /users?limit=5&active=false&search=ada
    return {"limit": limit, "active": active, "search": search}
```

- Defaults make a parameter optional (`limit` defaults to 10).
- `str | None = None` means "optional, may be omitted entirely."
- Booleans accept `true/false/1/0/yes/no` and get converted for you.

**Rule of thumb:** path params identify *which resource* (`/users/42`); query params *modify a read* (filter/sort/paginate).

### 3.5 Request bodies — sending structured data in

For `POST`/`PUT`/`PATCH` you send a JSON body. You declare its shape with a Pydantic model (next section), and FastAPI validates it:

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str
    age: int | None = None

@app.post("/users", status_code=201)
def create_user(user: UserCreate):   # FastAPI sees a Pydantic type → reads the BODY
    # `user` is now a validated object, not a raw dict
    return {"created": user.name, "email": user.email}
```

How FastAPI decides where each argument comes from:
- type is a Pydantic model → it's the **request body**
- name matches a `{placeholder}` in the path → **path parameter**
- otherwise → **query parameter**

### 3.6 Returning data and controlling the status code

```python
from fastapi import HTTPException

fake_db = {1: {"id": 1, "name": "Ada"}}

@app.get("/users/{user_id}")
def get_user(user_id: int):
    if user_id not in fake_db:
        raise HTTPException(status_code=404, detail="User not found")
    return fake_db[user_id]          # 200 OK by default
```

- Return a dict / list / Pydantic model → it becomes the JSON body.
- Default success code is `200`; override with `status_code=...` in the decorator (e.g. `201` for creation).
- To signal an error, `raise HTTPException(...)`. FastAPI turns it into a proper error response like `{"detail": "User not found"}` with the right status. **Don't** return error dicts manually — raising keeps your code clean and the status correct.

### 3.7 Dependency injection — shared, reusable setup

"Dependencies" are functions FastAPI runs *before* your endpoint, feeding their result in. They're how you share things like a database session, the current user, or pagination params without copy-pasting.

```python
from fastapi import Depends
from typing import Annotated

def pagination(skip: int = 0, limit: int = 20):
    return {"skip": skip, "limit": limit}

@app.get("/items")
def list_items(page: Annotated[dict, Depends(pagination)]):
    return page          # ?skip=40&limit=20 flows in automatically
```

This looks minor now, but it's *the* mechanism you'll use to hand a database connection to every endpoint (Section 5). Learn the pattern here where it's simple.

### 3.8 async vs regular functions

You'll see both `def` and `async def` endpoints. Short version for now:

- Use plain `def` when your code does normal blocking work (most ORM/SQLite calls). FastAPI runs it in a thread pool so it won't block other requests.
- Use `async def` when you're calling libraries designed for it (they use `await`), like an async HTTP client or an async database driver.
- **When unsure, use plain `def`.** Mixing blocking calls inside `async def` is a classic beginner mistake that silently hurts performance. We'll use `def` throughout this guide for clarity.

---

## 4. Pydantic — the Data-Shape Layer
<a name="4-pydantic"></a>

Pydantic is the library FastAPI uses to describe and validate data. A Pydantic model is a Python class where each attribute has a type. Pydantic enforces those types at runtime and produces clear errors when data doesn't fit.

> **Versions:** modern FastAPI uses **Pydantic v2**. If you find old tutorials using `class Config:` and `.dict()`, those are v1. The v2 equivalents are `model_config = ConfigDict(...)` and `.model_dump()`. This guide uses v2.

### 4.1 Validation and constraints

```python
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    email: EmailStr                          # validates real email format
    age: int | None = Field(default=None, ge=0, le=130)  # 0 ≤ age ≤ 130
```

If a client sends `{"name": "", "email": "not-an-email"}`, FastAPI returns a `422` with a precise list of what failed and where. You wrote zero validation code. (`EmailStr` needs `pip install "pydantic[email]"`.)

### 4.2 The crucial pattern: separate models for input and output

A frequent beginner trap is using one model everywhere. In practice the data coming *in* differs from what goes *out*:

- On **create**, the client shouldn't send an `id` (the database assigns it) and definitely shouldn't send a hashed password back and forth.
- On **read**, you want to return the `id` and maybe a `created_at`, but never the password hash.

So you make a small family of models:

```python
from pydantic import BaseModel, EmailStr, ConfigDict
from datetime import datetime

class UserBase(BaseModel):
    name: str
    email: EmailStr

class UserCreate(UserBase):          # what the client SENDS to create
    password: str                    # plaintext in, you hash it server-side

class UserRead(UserBase):            # what you SEND BACK
    id: int
    created_at: datetime
    # note: no password field — it never leaves the server
    model_config = ConfigDict(from_attributes=True)  # allow building from ORM objects
```

`from_attributes=True` lets FastAPI build a `UserRead` directly from a SQLAlchemy row object (reading `.id`, `.name`, etc.), which you'll rely on in the next section.

### 4.3 `response_model` — enforcing the output shape

Tell the endpoint what it's allowed to return, and FastAPI will filter the output to exactly those fields:

```python
@app.post("/users", response_model=UserRead, status_code=201)
def create_user(user: UserCreate):
    saved = save_to_db(user)         # returns an object that HAS a password hash
    return saved                     # FastAPI strips it to UserRead's fields only
```

This is a safety net: even if your database object carries sensitive fields, `response_model=UserRead` guarantees they're never serialized into the response. Get in the habit of always setting `response_model`.

---

## 5. SQLite + SQLAlchemy — the Storage Layer
<a name="5-sqlite--sqlalchemy"></a>

### 5.1 Why SQLite is perfect for learning

SQLite is a full SQL database that lives in a **single file** (or even in memory). There's no separate server process to install, configure, or keep running — your app just opens the file. That makes it ideal for learning and for small apps. The same code you write against SQLite mostly transfers to PostgreSQL or MySQL later, because they all speak SQL.

Its main limitation: it handles concurrent *writes* poorly (it locks the whole file briefly). Fine for learning and modest traffic; you'd graduate to PostgreSQL when you need many simultaneous writers.

### 5.2 Two ways to talk to it

**(a) The raw `sqlite3` module** — built into Python, good to see once so the ORM feels less magical:

```python
import sqlite3
conn = sqlite3.connect("app.db")
conn.execute("CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, text TEXT)")
conn.execute("INSERT INTO notes (text) VALUES (?)", ("hello",))  # ? prevents SQL injection
conn.commit()
rows = conn.execute("SELECT id, text FROM notes").fetchall()
conn.close()
```

Note the `?` placeholder — **never** build SQL by string-concatenating user input (that's the classic SQL-injection hole). Always parameterize.

**(b) SQLAlchemy (an ORM)** — the practical choice. An **ORM** (Object-Relational Mapper) lets you work with Python classes and objects instead of writing SQL strings by hand. You define a `User` class; SQLAlchemy translates `session.add(user)` into the right `INSERT`. We'll use SQLAlchemy for the rest of the guide.

```bash
pip install sqlalchemy
```

> **Versions:** this guide uses **SQLAlchemy 2.0** style (`Mapped`, `mapped_column`, `DeclarativeBase`). Older tutorials use `Column(...)` and `declarative_base()` — that still works but the 2.0 style has better type hints and is what new code should use.

### 5.3 The three pieces of any SQLAlchemy setup

Create `database.py`:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

# 1) ENGINE — manages the actual connection to the database file.
SQLALCHEMY_DATABASE_URL = "sqlite:///./app.db"   # relative file in project dir
engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    # Needed ONLY for SQLite + FastAPI: SQLite by default forbids using a
    # connection across threads, but FastAPI may use several. This relaxes that.
    connect_args={"check_same_thread": False},
)

# 2) SESSION FACTORY — a Session is your "workspace" for a unit of work
#    (a batch of reads/writes you commit together).
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)

# 3) BASE — the parent class your table models inherit from.
class Base(DeclarativeBase):
    pass
```

Mental model:
- **Engine** = the pipe to the database (created once, lives for the app's lifetime).
- **Session** = a short-lived scratchpad for one request's worth of work. Open it, do your reads/writes, commit, close.
- **Base** = the registry that knows about all your tables.

### 5.4 Defining a table (a "model")

Create `models.py`:

```python
from datetime import datetime, timezone
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import Mapped, mapped_column
from database import Base

class User(Base):
    __tablename__ = "users"

    id:         Mapped[int]      = mapped_column(primary_key=True)
    name:       Mapped[str]      = mapped_column(String(50))
    email:      Mapped[str]      = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str]   = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
```

> **Don't confuse these with Pydantic models.** SQLAlchemy models (`models.py`) describe *database tables and rows*. Pydantic models (Section 4) describe *the JSON shape on the wire*. They look similar and often share field names, but they're different jobs: one is "how it's stored," the other is "how it's communicated." Keeping them separate is normal and healthy.

### 5.5 The database-session dependency (this is the key FastAPI ↔ DB glue)

You want each request to get its own Session, used and then closed cleanly even if something errors. A FastAPI dependency with `yield` does exactly that:

```python
# in database.py
from collections.abc import Generator
from sqlalchemy.orm import Session

def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()          # open a fresh session for this request
    try:
        yield db                 # hand it to the endpoint
    finally:
        db.close()               # always close, success or failure
```

Endpoints then ask for it via `Depends(get_db)`. The code before `yield` runs on the way in; the code after `yield` runs on the way out. This guarantees no leaked connections — a real problem if you manage sessions by hand.

### 5.6 Creating the tables

For learning, the simplest approach is to let SQLAlchemy create any missing tables at startup, using FastAPI's **lifespan** hook:

```python
# in main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from database import Base, engine
import models  # noqa: F401 — importing registers the models on Base

@asynccontextmanager
async def lifespan(app: FastAPI):
    Base.metadata.create_all(bind=engine)   # runs once at startup
    yield                                    # app runs here
    # (shutdown cleanup would go after the yield)

app = FastAPI(lifespan=lifespan)
```

> `create_all` only creates tables that don't exist — it does **not** alter existing ones. The moment you start changing columns on a live database, you'll want a *migration tool* called **Alembic**. That's a "later" topic, but now you know the name to search for.

---

## 6. Putting It Together: a Complete REST CRUD API
<a name="6-complete-rest-crud-api"></a>

Now we assemble everything into a working "users" API supporting all five CRUD operations. **CRUD** = Create, Read, Update, Delete — the four basic things you do to stored data. This is the shape of probably 80% of real endpoints you'll ever write.

### 6.1 Recommended file layout

```
myapp/
├── database.py     # engine, SessionLocal, Base, get_db   (Section 5.3 & 5.5)
├── models.py       # SQLAlchemy tables                     (Section 5.4)
├── schemas.py      # Pydantic request/response models      (Section 4)
└── main.py         # the FastAPI app + endpoints
```

Separating "schemas" (Pydantic, the wire) from "models" (SQLAlchemy, the database) is a convention worth adopting early — it keeps the two responsibilities from blurring.

### 6.2 `schemas.py`

```python
from datetime import datetime
from pydantic import BaseModel, EmailStr, ConfigDict, Field

class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    email: EmailStr
    password: str = Field(min_length=8)

class UserUpdate(BaseModel):                 # all optional → partial updates
    name: str | None = Field(default=None, min_length=1, max_length=50)
    email: EmailStr | None = None

class UserRead(BaseModel):
    id: int
    name: str
    email: EmailStr
    created_at: datetime
    model_config = ConfigDict(from_attributes=True)
```

### 6.3 `main.py` — the endpoints

```python
from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session
from sqlalchemy.exc import IntegrityError
from contextlib import asynccontextmanager
from typing import Annotated
import hashlib

from database import Base, engine, get_db
import models
import schemas

@asynccontextmanager
async def lifespan(app: FastAPI):
    Base.metadata.create_all(bind=engine)
    yield

app = FastAPI(lifespan=lifespan, title="Users API")

# A tiny helper so the example is self-contained.
# NOTE: sha256 is NOT secure for real passwords — see Section 11. Use it only to learn.
def hash_password(raw: str) -> str:
    return hashlib.sha256(raw.encode()).hexdigest()

DB = Annotated[Session, Depends(get_db)]     # shorthand so every signature is short


# ---- CREATE ---------------------------------------------------------------
@app.post("/users", response_model=schemas.UserRead, status_code=status.HTTP_201_CREATED)
def create_user(payload: schemas.UserCreate, db: DB):
    user = models.User(
        name=payload.name,
        email=payload.email,
        password_hash=hash_password(payload.password),
    )
    db.add(user)                  # stage the insert
    try:
        db.commit()               # execute it
    except IntegrityError:        # e.g. email already exists (unique constraint)
        db.rollback()
        raise HTTPException(status.HTTP_409_CONFLICT, "Email already registered")
    db.refresh(user)              # reload so user.id and created_at are populated
    return user                   # response_model strips it down to UserRead


# ---- READ (list, with pagination) -----------------------------------------
@app.get("/users", response_model=list[schemas.UserRead])
def list_users(db: DB, skip: int = 0, limit: int = 20):
    return db.query(models.User).offset(skip).limit(limit).all()


# ---- READ (single) --------------------------------------------------------
@app.get("/users/{user_id}", response_model=schemas.UserRead)
def get_user(user_id: int, db: DB):
    user = db.get(models.User, user_id)        # fetch by primary key
    if user is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "User not found")
    return user


# ---- UPDATE (partial) -----------------------------------------------------
@app.patch("/users/{user_id}", response_model=schemas.UserRead)
def update_user(user_id: int, payload: schemas.UserUpdate, db: DB):
    user = db.get(models.User, user_id)
    if user is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "User not found")
    # exclude_unset → only the fields the client actually sent
    for field, value in payload.model_dump(exclude_unset=True).items():
        setattr(user, field, value)
    db.commit()
    db.refresh(user)
    return user


# ---- DELETE ---------------------------------------------------------------
@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(user_id: int, db: DB):
    user = db.get(models.User, user_id)
    if user is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "User not found")
    db.delete(user)
    db.commit()
    # 204 → return nothing
```

### 6.4 Read it as a table

| Action | Method | Path | Body? | Success code |
|--------|--------|------|-------|--------------|
| Create | `POST` | `/users` | yes | `201 Created` |
| List | `GET` | `/users?skip=0&limit=20` | no | `200 OK` |
| Read one | `GET` | `/users/{id}` | no | `200 OK` |
| Update | `PATCH` | `/users/{id}` | yes (partial) | `200 OK` |
| Delete | `DELETE` | `/users/{id}` | no | `204 No Content` |

### 6.5 Try it without a frontend

Run `fastapi dev main.py`, open `http://127.0.0.1:8000/docs`, and exercise each endpoint from the Swagger page. Or from a terminal with `curl`:

```bash
# create
curl -X POST localhost:8000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Ada","email":"ada@x.io","password":"hunter2hunter2"}'

# list
curl localhost:8000/users

# get one
curl localhost:8000/users/1

# partial update
curl -X PATCH localhost:8000/users/1 \
  -H "Content-Type: application/json" -d '{"name":"Ada L."}'

# delete
curl -X DELETE localhost:8000/users/1
```

Being able to test the backend *in isolation* like this — before any frontend exists — is a skill in itself. The API is a contract; prove it works on its own first.

---

## 7. REST Done Properly
<a name="7-rest-conventions"></a>

You just built a **REST** API. REST isn't a technology or a library — it's a *style* of designing HTTP APIs around **resources** (nouns) acted on by **methods** (verbs). Following its conventions makes your API predictable to other developers (and to your future self).

### 7.1 The core conventions

**1. URLs name resources (nouns), not actions.**

```
GOOD   /users            /users/42          /users/42/orders
BAD    /getUsers         /createUser        /deleteUserById?id=42
```

The verb is the HTTP method, not part of the URL. `GET /users` reads; `POST /users` creates. You don't put "get" or "create" in the path.

**2. Use plural nouns for collections.** `/users`, `/orders`, `/products`. A single item is that collection plus an id: `/users/42`.

**3. Nest to show relationships.** "The orders belonging to user 42" → `GET /users/42/orders`. Don't nest more than ~2 levels deep; it gets unwieldy.

**4. Match the method to the intent:**

| Method | Intent | Idempotent? | Safe (no change)? |
|--------|--------|-------------|-------------------|
| `GET` | read | yes | yes |
| `POST` | create | no | no |
| `PUT` | replace entirely | yes | no |
| `PATCH` | update partially | no* | no |
| `DELETE` | remove | yes | no |

*"**Idempotent**" means doing it twice has the same effect as doing it once. `DELETE /users/42` twice → still just deleted (second returns 404, but the world state is identical). `POST /users` twice → two users created. This matters because clients/proxies may safely retry idempotent requests after a network hiccup, but must be careful retrying `POST`.

**5. Return the right status code** (revisit the table in Section 2). Created → `201`. Deleted with no body → `204`. Not found → `404`. Bad input → `422`/`400`. Conflict (e.g. duplicate email) → `409`.

**6. Filtering, sorting, and pagination live in query params:**

```
GET /users?active=true&sort=-created_at&skip=40&limit=20
```

(`-created_at` is a common convention for "descending by created_at".)

### 7.2 Shape your errors consistently

Pick one error shape and use it everywhere, so the frontend can handle errors generically. FastAPI's default is:

```json
{ "detail": "User not found" }
```

That's fine to start. As the app grows you might standardize on something richer (an error `code`, a human `message`, maybe a `fields` map for validation). The important thing is *consistency*.

### 7.3 Versioning (know it exists)

Once other people depend on your API, you can't freely change it without breaking them. The common defense is a version prefix: `/api/v1/users`. You don't need this on day one, but design as if a `v2` will exist someday — it keeps you from painting yourself into a corner.

### 7.4 A quick maturity checklist

- [ ] URLs are nouns; verbs are HTTP methods
- [ ] Collections are plural
- [ ] Correct status codes (not `200` for everything)
- [ ] Consistent error shape
- [ ] Validation on all input (Pydantic gives this free)
- [ ] Pagination on list endpoints (never return unbounded lists)
- [ ] `response_model` set on every endpoint (no accidental data leaks)

---

## 8. GraphQL — a Different Shape of API
<a name="8-graphql"></a>

REST gives each resource its own URL. **GraphQL** takes a different approach: there's a **single endpoint** (usually `/graphql`), and the *client* sends a **query** describing exactly which fields it wants. The server returns precisely that shape — no more, no less.

### 8.1 The problems GraphQL is trying to solve

With REST you sometimes hit:

- **Over-fetching:** `GET /users/42` returns 20 fields but your screen needs only `name` and `avatar`. You downloaded 18 fields for nothing.
- **Under-fetching / waterfalls:** to render a profile you call `/users/42`, then `/users/42/orders`, then `/orders/{id}/items` — three round trips, each waiting on the last.

GraphQL lets the client ask for the user *and* their orders *and* each order's items in **one request**, getting back only the named fields.

### 8.2 What a query looks like

The client sends this (it's a string, posted to `/graphql`):

```graphql
query {
  user(id: 42) {
    name
    email
    orders {
      id
      total
    }
  }
}
```

And gets back exactly:

```json
{
  "data": {
    "user": {
      "name": "Ada",
      "email": "ada@x.io",
      "orders": [
        { "id": 1, "total": 19.99 },
        { "id": 2, "total": 5.00 }
      ]
    }
  }
}
```

Notice the response *mirrors the query*. The client controls the shape.

### 8.3 The three building blocks

- **Schema** — a strongly-typed description of every type and field available. This is the contract.
- **Query** — a read operation (like `GET`).
- **Mutation** — a write operation (create/update/delete; like `POST`/`PUT`/`DELETE`). All writes go through mutations, not by changing the HTTP method.

There are also *subscriptions* (real-time updates over a persistent connection) — a more advanced topic to file away for later.

### 8.4 GraphQL with FastAPI (using Strawberry)

`Strawberry` is a modern, type-hint-based GraphQL library that pairs naturally with FastAPI.

```bash
pip install strawberry-graphql
```

```python
import strawberry
from strawberry.fastapi import GraphQLRouter
from fastapi import FastAPI

# 1) Define your types with normal Python type hints + a decorator.
@strawberry.type
class User:
    id: int
    name: str
    email: str

# 2) Define the read entry points (the "Query" type).
@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: int) -> User:
        # In real code you'd load from the DB here.
        return User(id=id, name="Ada", email="ada@x.io")

# 3) Define writes (the "Mutation" type).
@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_user(self, name: str, email: str) -> User:
        # ...save to DB, then return the created object...
        return User(id=99, name=name, email=email)

# 4) Wire it into FastAPI at a single route.
schema = strawberry.Schema(query=Query, mutation=Mutation)
graphql_app = GraphQLRouter(schema)

app = FastAPI()
app.include_router(graphql_app, prefix="/graphql")
```

Run the app and open `http://127.0.0.1:8000/graphql` for an interactive in-browser query explorer (similar in spirit to FastAPI's `/docs`).

You can run REST and GraphQL side by side in the same FastAPI app — they're not mutually exclusive. A common pattern is REST for simple resource access plus GraphQL for complex, client-driven reads.

### 8.5 The catch (so you go in with eyes open)

GraphQL's flexibility shifts complexity onto the server:

- **Caching is harder.** REST leans on the URL + HTTP caching; one GraphQL URL with varying bodies doesn't cache as simply.
- **The N+1 query problem** is easy to trigger: asking for 50 users and their orders can naively fire 51 database queries. The fix is a batching tool called a **DataLoader** — note the term for later.
- **You can't just eyeball status codes.** GraphQL often returns `200` even on logical errors, putting problems in an `"errors"` array in the body. Your frontend must check that array, not just the HTTP status.

---

## 9. The Frontend Side — Actually Calling Your API
<a name="9-the-frontend-side"></a>

Now the other half: code running in the browser that talks to your serving layer. The universal tool is the built-in **`fetch()`** function. Everything here is plain JavaScript — it works the same whether you later use React, Vue, Svelte, or no framework at all.

### 9.1 The mental model again

The frontend and backend are *separate programs*. The frontend can't call your Python functions. It can only send HTTP requests and read the responses. `fetch()` is how it sends them.

`fetch()` returns a **Promise** — a placeholder for a value that arrives *later* (the network takes time). You handle it with `await` inside an `async` function. If you've not met Promises yet, the one-line version: `await` pauses until the network replies, without freezing the whole page.

### 9.2 The four REST calls from the browser

```javascript
const BASE = "http://127.0.0.1:8000";

// READ a list -----------------------------------------------------
async function listUsers() {
  const res = await fetch(`${BASE}/users?limit=20`);   // GET is the default
  if (!res.ok) {                                        // res.ok is true for 2xx
    throw new Error(`Request failed: ${res.status}`);
  }
  const users = await res.json();                       // parse the JSON body
  return users;
}

// READ one --------------------------------------------------------
async function getUser(id) {
  const res = await fetch(`${BASE}/users/${id}`);
  if (res.status === 404) return null;                  // handle the expected case
  if (!res.ok) throw new Error(`Failed: ${res.status}`);
  return res.json();
}

// CREATE ----------------------------------------------------------
async function createUser(data) {
  const res = await fetch(`${BASE}/users`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },    // tell the server it's JSON
    body: JSON.stringify(data),                          // object → JSON string
  });
  if (res.status === 409) throw new Error("Email already in use");
  if (!res.ok) throw new Error(`Failed: ${res.status}`);
  return res.json();
}

// UPDATE (partial) ------------------------------------------------
async function updateUser(id, changes) {
  const res = await fetch(`${BASE}/users/${id}`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(changes),
  });
  if (!res.ok) throw new Error(`Failed: ${res.status}`);
  return res.json();
}

// DELETE ----------------------------------------------------------
async function deleteUser(id) {
  const res = await fetch(`${BASE}/users/${id}`, { method: "DELETE" });
  if (!res.ok) throw new Error(`Failed: ${res.status}`);
  // 204 → no body to parse
}
```

Three things to lock in:
1. **`fetch` does NOT throw on `404`/`500`.** It only rejects on *network* failure (server unreachable). You must check `res.ok` / `res.status` yourself. This surprises everyone once.
2. **Always set `Content-Type: application/json`** and `JSON.stringify` the body when sending data, or FastAPI won't recognize it as JSON and you'll get a `422`.
3. **`res.json()` is itself async** — it returns a Promise, hence the second `await`.

### 9.3 Putting it on a page

```html
<button id="load">Load users</button>
<ul id="list"></ul>

<script>
  document.getElementById("load").addEventListener("click", async () => {
    const ul = document.getElementById("list");
    ul.innerHTML = "Loading…";                 // 1. show loading state
    try {
      const users = await listUsers();         // 2. fetch
      ul.innerHTML = users
        .map(u => `<li>${u.name} — ${u.email}</li>`)
        .join("");                             // 3. render
    } catch (err) {
      ul.innerHTML = `Error: ${err.message}`;  // 4. handle failure
    }
  });
</script>
```

That loading → success → error trio is the rhythm of *every* data-fetching UI. Frameworks like React dress it up, but underneath it's exactly this.

### 9.4 Calling a GraphQL endpoint

GraphQL from the browser is just a `POST` to one URL, with the query string in the body:

```javascript
async function fetchUserWithOrders(id) {
  const query = `
    query GetUser($id: Int!) {
      user(id: $id) { name email orders { id total } }
    }`;
  const res = await fetch(`${BASE}/graphql`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query, variables: { id } }),  // query + variables
  });
  const payload = await res.json();
  if (payload.errors) {                          // GraphQL puts logic errors HERE,
    throw new Error(payload.errors[0].message);  // often even when status is 200
  }
  return payload.data.user;                      // real data lives under `data`
}
```

Note how it differs from REST on the client: always `POST`, always the same URL, the *query* selects fields, and you check `payload.errors` rather than relying on the HTTP status.

### 9.5 CORS — the wall you WILL hit

The first time your frontend (say `http://localhost:5173`) calls your API (`http://localhost:8000`), the browser will likely block it with a **CORS** error in the console. This is a browser security feature: by default, a page from one **origin** (scheme + host + port) may not call a *different* origin.

Your API has to explicitly opt in. In FastAPI:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  # your frontend's exact origin
    allow_credentials=True,
    allow_methods=["*"],                       # GET, POST, PATCH, DELETE, ...
    allow_headers=["*"],
)
```

Key points:
- CORS is enforced by the **browser**, not your server — which is why `curl` and the `/docs` page work fine while your web page gets blocked. (Same-origin tools don't trigger it.)
- For development you can use `allow_origins=["*"]`, but **never ship `*` to production** if you also use credentials — list your real domains.
- A CORS error is *not* a bug in your fetch code; it's a missing server-side permission. Fix it on the FastAPI side.

---

## 10. REST vs GraphQL: When to Use Which
<a name="10-rest-vs-graphql"></a>

Neither is "better" — they trade different things. As a beginner, **default to REST**; reach for GraphQL when you feel the specific pains it solves.

| | REST | GraphQL |
|---|------|---------|
| Endpoints | many (one per resource) | one (`/graphql`) |
| Who picks the response shape | the server | the client |
| Over/under-fetching | common | largely solved |
| HTTP caching | easy (per-URL) | harder |
| Status codes | meaningful (`404`, `201`…) | usually `200`; errors in body |
| Learning curve | gentle | steeper (schema, resolvers, N+1) |
| Great for | CRUD, public APIs, file up/download | complex nested reads, many client types, mobile |
| Tooling baggage | minimal | schema + resolver layer to maintain |

**Choose REST when:** you're building straightforward CRUD, you want HTTP caching, you value simplicity, or you're learning (you are — start here).

**Choose GraphQL when:** different screens/clients need different slices of the same data, you're fighting request waterfalls, or mobile clients need to minimize payload size over slow networks.

**You can run both.** A perfectly normal architecture is REST for the bread-and-butter resources plus a GraphQL endpoint for a few data-hungry screens. They coexist in one FastAPI app.

---

## 11. Cross-Cutting Concerns You'll Hit Soon
<a name="11-cross-cutting-concerns"></a>

These don't belong to one section — they touch the whole app. You don't need them on day one, but you'll meet each one fast, so here's a map.

### 11.1 Real password hashing
The `sha256` helper in Section 6 was a teaching shortcut and is **not safe for real passwords** (it's fast, which helps attackers). Use a purpose-built, deliberately-slow algorithm via `passlib` with `bcrypt` or `argon2`:
```bash
pip install "passlib[bcrypt]"
```
```python
from passlib.context import CryptContext
pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")
hashed = pwd.hash("plaintext")
ok = pwd.verify("plaintext", hashed)   # True / False
```

### 11.2 Authentication & authorization
- **Authentication** = "who are you?" (logging in). **Authorization** = "are you allowed to do this?" (permissions).
- The common token scheme is **JWT** (JSON Web Token): the client logs in once, gets a signed token, and sends it on each request via an `Authorization: Bearer <token>` header.
- FastAPI has first-class support via `fastapi.security` (`OAuth2PasswordBearer`). You implement it as a *dependency* (Section 3.7) that reads the header, validates the token, and returns the current user — then any endpoint can require it with `current_user: User = Depends(get_current_user)`.

### 11.3 Configuration & secrets
Never hard-code database URLs, secret keys, or passwords in source. Keep them in environment variables, loaded with `pydantic-settings`:
```bash
pip install pydantic-settings
```
This reads from a `.env` file (which you add to `.gitignore` so it never reaches version control).

### 11.4 Schema migrations (Alembic)
`Base.metadata.create_all` only creates *missing* tables; it won't add a column to an existing one. Once real data exists, use **Alembic** to version and apply schema changes safely. Search "FastAPI SQLAlchemy Alembic" when you get there.

### 11.5 Testing your API
FastAPI ships with a `TestClient` (built on `httpx`) that calls your app in-process — no running server needed:
```python
from fastapi.testclient import TestClient
from main import app
client = TestClient(app)

def test_create_user():
    r = client.post("/users", json={"name": "Ada", "email": "a@x.io", "password": "longenough"})
    assert r.status_code == 201
    assert r.json()["email"] == "a@x.io"
```
Pair it with `pytest`. Tests are how you change code later without fear.

### 11.6 Logging & errors
Use Python's `logging` module rather than `print`. For production you'll also want a global exception handler so unexpected errors return a clean `500` JSON body instead of a stack trace.

### 11.7 Deployment, briefly
In development you run `fastapi dev`. In production you run `uvicorn`/`gunicorn` behind a real web server or platform, typically inside a Docker container. SQLite can serve small production apps; for anything with real concurrency you'll move to PostgreSQL — and because you used SQLAlchemy, that migration is mostly a connection-string change plus testing, not a rewrite.

---

## 12. Where to Go Next
<a name="12-where-to-go-next"></a>

A suggested path, building on this guide in roughly this order:

1. **Type the CRUD app from Section 6 yourself**, by hand, and test every endpoint from `/docs`. Don't copy-paste — typing it is where it sticks.
2. **Add a second resource** (e.g. `posts`) and a **relationship** (a user *has many* posts). Learn SQLAlchemy `relationship()` and foreign keys.
3. **Build a tiny frontend page** (plain HTML + the `fetch` functions from Section 9) that lists and creates users. Hit the CORS wall on purpose and fix it.
4. **Add authentication** (Section 11.2): a login endpoint that returns a JWT, and a protected endpoint that requires it.
5. **Write tests** (Section 11.5) for your endpoints.
6. **Introduce Alembic** (Section 11.4) and make your first migration.
7. **Try GraphQL** (Section 8) for one read that's annoying in REST, to feel the difference firsthand.
8. **Pick a frontend framework** (React is the common choice) and rebuild your page — now your `fetch` knowledge transfers directly.

### Reference docs worth bookmarking
- FastAPI official docs — exceptionally good, tutorial-first. Read its "Tutorial - User Guide" top to bottom; it overlaps with and deepens this document.
- SQLAlchemy 2.0 "ORM Quick Start" and the Unified Tutorial.
- Pydantic v2 docs (validation, `Field`, settings).
- Strawberry GraphQL docs (for the GraphQL path).
- MDN's pages on `fetch`, Promises, and CORS (for the frontend side).

> Tip: when you search, append the year or "2.0" / "v2" to avoid landing on outdated tutorials — this ecosystem changed its idioms recently (Pydantic v1→v2, SQLAlchemy 1.x→2.0), and stale guides cause a lot of beginner confusion.

---

## 13. Glossary
<a name="13-glossary"></a>

| Term | Plain-English meaning |
|------|----------------------|
| **Serving layer / backend / API server** | the program (your FastAPI app) that sits between the frontend and the database |
| **Endpoint / route / path operation** | one method+path combo your server responds to, e.g. `GET /users` |
| **HTTP method / verb** | the kind of action: GET, POST, PUT, PATCH, DELETE |
| **Path parameter** | a value embedded in the URL path: the `42` in `/users/42` |
| **Query parameter** | a value after `?` in the URL: `?limit=10` |
| **Request body** | the JSON payload sent with POST/PUT/PATCH |
| **Status code** | the 3-digit result code on a response (200, 404, 500…) |
| **JSON** | the text format objects are sent as over the wire |
| **CRUD** | Create, Read, Update, Delete — the four basic data operations |
| **REST** | an API style organized around resource URLs + HTTP verbs |
| **GraphQL** | an API style with one endpoint where the client queries exact fields |
| **Resolver** | a server function that produces the value for a GraphQL field |
| **Mutation** | a GraphQL write operation |
| **ORM** | Object-Relational Mapper; lets you use Python objects instead of raw SQL (SQLAlchemy) |
| **Engine** | SQLAlchemy's connection manager to the database |
| **Session** | a short-lived workspace for one unit of database work |
| **Model (SQLAlchemy)** | a Python class mapped to a database table |
| **Schema (Pydantic)** | a Python class describing the JSON shape on the wire |
| **Migration** | a versioned, applied change to your database structure (Alembic) |
| **Dependency injection** | FastAPI running setup functions for you and passing results in (`Depends`) |
| **Middleware** | code that wraps every request/response (e.g. CORS) |
| **CORS** | browser rule controlling which origins may call your API |
| **Origin** | scheme + host + port, e.g. `http://localhost:8000` |
| **Idempotent** | an operation that has the same effect whether done once or many times |
| **Promise** | a JavaScript placeholder for a value arriving later (network results) |
| **`fetch()`** | the browser function for making HTTP requests |
| **JWT** | a signed token proving who a logged-in user is |
| **ASGI** | the async server interface FastAPI/uvicorn speak |
| **uvicorn** | the server program that actually runs your FastAPI app |

---

*End of guide. This is yours now — annotate it, correct it, and grow it as you learn. The fastest way to make it stick is to build the Section 6 app and break things on purpose.*
