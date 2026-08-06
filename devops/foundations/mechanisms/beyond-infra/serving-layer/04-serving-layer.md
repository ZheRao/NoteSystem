# Prerequisites for the Serving Layer

*The concepts the serving-layer walkthrough assumed. Built from the ground up, anchored to ETL and pandas where the analogy holds.*

---

## How to read this

Seven parts, in dependency order. Part 3 (concurrency) is the keystone — items 4 and 5 on your list are both consequences of it, and so is the whole connection-lifecycle question. If you read only one part slowly, read that one.

| Part | Answers |
|---|---|
| 1 | The request/response model, and where 422 comes from *(your item 1)* |
| 2 | URIs, `uri=True`, and why `#` in a path breaks things *(items 2, 3)* |
| 3 | Concurrency from zero: threads, the GIL, the event loop *(item 4)* |
| 4 | ASGI, uvicorn, FastAPI, dependency injection *(items 1, 5)* |
| 5 | What a "connection" is, and the four lifecycle strategies |
| 6 | SQLite's concurrency model, locking, WAL |
| 7 | What changes when you add writes |

A framing that will help throughout: **you already know this material in batch form.** An ETL job is a program that runs start to finish and owns the machine. A web server is the same work, sliced into thousands of tiny independent jobs that arrive unpredictably and overlap. Almost every complication below comes from that one difference — *overlap*.

---

## Part 1 — The request/response model

### 1.1 What a web server actually does

Strip away the frameworks and a web server is a loop:

```
bind to a TCP port
loop forever:
    accept a connection
    read bytes until you have a complete HTTP request
    figure out what was asked for
    produce a response
    write the response bytes back
    close (or keep alive for reuse)
```

An HTTP request is just formatted text over a TCP socket:

```
GET /forecast/c0001_2026_3-pre-harvest/month/4 HTTP/1.1
Host: api.growlytics.example
Accept: application/json
```

And the response:

```
HTTP/1.1 200 OK
Content-Type: application/json

[{"month_num": 4, "net": 12045.22, ...}]
```

That's the whole protocol at this level. Everything FastAPI does is helping you turn the first block into the second.

### 1.2 The request path

**The request path** (or "hot path") means: code that runs while a user is waiting. This is the phrase behind invariant 1 in `build_serving_store.py` — *"a user click must never reach this file."*

The distinction matters because of the cost asymmetry:

| | Batch (your ETL, the build) | Request path (the API) |
|---|---|---|
| Runs | Scheduled, once | Thousands of times, unpredictably |
| Someone waiting? | No | Yes, watching a spinner |
| Acceptable duration | Minutes | Milliseconds |
| Overlapping copies | One | Many, simultaneously |
| Failure blast radius | One job | One user, or all of them |

A 40-second pandas transform is unremarkable in a batch job and a catastrophe in a route handler. Your architecture is built to keep expensive work in the left column, and that's why the store is precomputed rather than derived per request.

### 1.3 Statelessness

Each HTTP request is independent. The server keeps nothing in memory about you between requests — no cursor position, no open transaction, no "current forecast." Everything needed must arrive in the request itself, or be reconstructed from durable storage.

This is why `Depends(store.get_db)` opens a connection per request rather than reusing "your" connection: **there is no "your" connection.** Request 2 may be handled by a different thread, a different process, or a different machine behind a load balancer.

Compare with your `queries.py` smoke test, which opens one connection and runs all four levels against it. That's a batch program — it has a beginning and an end and owns its resources throughout. A web server has no equivalent scope. Statelessness is the constraint that generates most of Part 5.

### 1.4 Anatomy of a URL

```
https://api.growlytics.example/forecast/c0001_2026_3-pre-harvest/month/4?category=Revenue
└─┬──┘  └──────────┬─────────┘└─────────────────┬──────────────────────┘ └───────┬────┘
scheme        authority                       path                            query
```

Two ways to pass data, and FastAPI treats them differently:

**Path parameters** are part of the path, declared in the route with braces:

```python
@app.get("/forecast/{orchestration_key}/month/{month_num}")
def month(orchestration_key: str, month_num: int, ...):
```

FastAPI matches the pattern and binds each segment to the same-named function argument. Path params are **required** — a different path is a different route, or a 404.

**Query parameters** come after `?` as `key=value&key=value`. Any function argument that *isn't* a path param and isn't a `Depends` is read from the query string:

```python
def sources(orchestration_key: str, month_num: int, category: str, db = Depends(...)):
##           └── path param ──┘  └─ path ─┘  └─ query param ─┘  └─ dependency ─┘
```

`category` becomes required because it has no default. Give it `category: str | None = None` and it becomes optional.

The conventional rule: path params **identify a resource**, query params **filter or modify** it. Your API follows this — the orchestration and month identify what you're looking at; `category` narrows it.

### 1.5 Status codes

Five families. The first digit carries the meaning:

| | Meaning | Who's at fault | Yours |
|---|---|---|---|
| 2xx | Success | — | 200 |
| 3xx | Redirect | — | — |
| 4xx | **Client error** — the request was wrong | Caller | 404, 422 |
| 5xx | **Server error** — the request was fine, you broke | You | 500, 503 |

The 4xx/5xx boundary is the important one, because it's a diagnostic signal. 4xx means "don't page me, the caller sent nonsense." 5xx means "page me." Getting this wrong is how you end up either ignoring real alarms or chasing phantom ones.

The ones you'll use:

- **404 Not Found** — the resource doesn't exist. Note it's also the *correct* answer for "exists but you're not allowed to know it exists," which is why §11 of the walkthrough recommends 404 over 403 for another client's key: 403 confirms existence.
- **422 Unprocessable Entity** — the request was well-formed HTTP but the *contents* failed validation. See below.
- **500 Internal Server Error** — an unhandled exception. Always a bug.
- **503 Service Unavailable** — a dependency is down. Not a bug; a condition. This is what a missing store *should* return.

### 1.6 Where 422 comes from (your item 1)

Here is the mechanism behind *"FastAPI coerces path/query params into the annotated types and returns 422 on failure."*

HTTP is text. Every path segment and query value arrives as a **string**. `month/4` gives you `"4"`, not `4`.

Now look at the annotation:

```python
def month(orchestration_key: str, month_num: int, ...):
```

In ordinary Python, that annotation is inert — decoration for your editor. Nothing checks it, nothing converts anything:

```python
def f(x: int): return x + 1
f("4")          ## TypeError at runtime; the annotation did nothing to help
```

FastAPI makes annotations *operational*. At import time it inspects every route's signature and builds a Pydantic validation model from it. Then, per request, for each parameter it:

1. pulls the raw string out of the path or query,
2. attempts to coerce it to the annotated type,
3. on success, passes the converted value,
4. on failure, aborts the route and returns **422** with a body naming the offending field.

So:

```
GET /forecast/X/month/4      → month_num = 4       (int)   route runs
GET /forecast/X/month/abc    → coercion fails      → 422, route never runs
GET /forecast/X/month/4.7    → not an int          → 422
```

```json
{"detail": [{"type": "int_parsing", "loc": ["path", "month_num"],
             "msg": "Input should be a valid integer, unable to parse string as an integer"}]}
```

Three consequences worth holding onto:

- **Your route body never sees a bad type.** By the time it runs, `month_num` is an `int`. That's why there's no `try: int(month_num)` anywhere in `api.py` — the framework already did it, and did it uniformly.
- **The annotation is the validation.** Change `month_num: int` to `month_num: str` and the coercion stops happening. The type hint isn't documentation here; it's executable.
- **Coercion is lax by default.** `"4"` → `4`, and `2026.0` → `2026`. It won't accept `"4.7"` as an int. This laxness is why §9.1 of the walkthrough notes that a pandas float upcast (`2026.0`) sails through the response model unnoticed.

You can tighten it beyond types, which is §10.3's suggestion:

```python
from typing import Literal
from fastapi import Path

def sources(month_num: int = Path(ge=1, le=12),
            category: Literal["Revenue", "Input Cost", "Operating Cost"] = ...):
```

Now out-of-range months and misspelled categories are 422s instead of empty results — and both show up in the auto-generated OpenAPI docs at `/docs`, which is a free artifact of the same machinery.

---

## Part 2 — URIs and the `file:` scheme

### 2.1 URI grammar (your items 2 and 3)

A URI has a fixed grammar, and **the delimiters are structural, not decorative**:

```
scheme:path?query#fragment
```

| Character | Role | Consequence |
|---|---|---|
| `:` | ends the scheme | — |
| `?` | **starts the query** | everything after is parameters, not path |
| `#` | **starts the fragment** | everything after is discarded by most parsers |
| `%` | starts a percent-escape | `%41` means `A`; a bare `%` is malformed |
| space | not legal in a URI | must be `%20` |

### 2.2 Why SQLite has a URI mode

Normally you pass SQLite a plain filename:

```python
sqlite3.connect("/data/store.db")
```

That gives you no way to express *options*. SQLite's answer was to accept a URI instead, using the query string for flags:

```python
sqlite3.connect("file:/data/store.db?mode=ro", uri=True)
##                └┬─┘└──────┬──────┘└───┬───┘
##              scheme     path        flags
```

`uri=True` is the switch that says *"parse this as a URI, not as a literal filename."* Without it, SQLite would try to open a file whose name is literally `file:/data/store.db?mode=ro`.

Useful flags:

| Flag | Effect |
|---|---|
| `mode=ro` | read-only; writes raise, file is not created if missing |
| `mode=rw` | read-write, but don't create |
| `mode=rwc` | read-write-create (the default behavior) |
| `mode=memory` | in-memory database |
| `immutable=1` | promise the file never changes; skips locking |
| `cache=shared` | share a page cache between connections in one process |

This is the same idea as a JDBC or SQLAlchemy connection string, and the same idea as `?sslmode=require` on a Postgres URL: cram configuration into the location string so a single string fully specifies "which database, opened how."

### 2.3 What breaks

Now the bug:

```python
uri = f"file:{config.STORE_PATH}?mode=ro"
```

`STORE_PATH` is interpolated raw. If it contains a structural character, the URI reparses in ways you didn't intend:

| Path | Resulting URI | What SQLite sees |
|---|---|---|
| `/data/store.db` | `file:/data/store.db?mode=ro` | ✅ correct |
| `/my data/store.db` | `file:/my data/store.db?mode=ro` | ⚠️ space is illegal; behavior varies |
| `/data/v2#final/store.db` | `file:/data/v2#final/store.db?mode=ro` | ❌ **path truncated to `/data/v2`**, and `mode=ro` silently lost |
| `/data/50%/store.db` | `file:/data/50%/store.db?mode=ro` | ❌ `%/s` is not a valid escape |
| `/data/a?b/store.db` | `file:/data/a?b/store.db?mode=ro` | ❌ path ends at `a`, flags become garbage |

The `#` case is the dangerous one, and note the *second* failure in that row: not only is the path wrong, **`mode=ro` is discarded too**, because it ended up inside the fragment. So the read-only guarantee — invariant 3, the thing this file exists to enforce — silently evaporates. You'd get a confusing "unable to open database file," or worse, a writable connection.

The fix is percent-encoding — replacing structural characters with `%` + hex so the parser reads them as data:

```python
from urllib.parse import quote
uri = f"file:{quote(str(config.STORE_PATH))}?mode=ro"
```

`quote` leaves `/` alone (it's a legitimate path separator) and escapes the rest. This is the same class of fix as SQL parameter binding or HTML escaping: **whenever data is interpolated into a language with structural characters, escape it.** URIs, SQL, HTML, shell — same lesson, four syntaxes.

### 2.4 Why Azure specifically

Local paths are tidy because you chose them. Deployment paths are handed to you:

- Windows-based App Service plans use backslashes and drive letters — `C:\home\site\wwwroot\data\store.db`. Backslash isn't a URI path separator, and `C:` looks like a scheme.
- Azure DevOps and GitHub Actions produce agent workspace paths containing spaces and sometimes `#` from branch or build names.
- Container mounts and blob-backed shares can carry percent-encoded segments already, so a naive re-encode double-escapes.

Two lines of hardening now versus a deployment failure that manifests as "unable to open database file" with no hint about why. Cheap insurance.

---

## Part 3 — Concurrency from zero

This is the keystone. Take it slowly.

### 3.1 The problem is waiting

Your ETL job spends most of its wall-clock time **waiting** — for disk reads, for a network response, for a database. During that wait the CPU is idle. In a batch job, who cares.

A web server can't afford it. Ten users request forecasts simultaneously. If you handle them strictly one at a time, user 10 waits for the other nine. Almost all of that wait is your server sitting idle on a disk read it could have overlapped.

**Concurrency is the art of using the waiting time.** Everything below is a different strategy for doing that, with different tradeoffs.

### 3.2 I/O-bound vs CPU-bound

The single most useful classification in this whole area:

| | I/O-bound | CPU-bound |
|---|---|---|
| Time goes to | Waiting for something external | Executing instructions |
| Examples | Network calls, disk reads, DB queries | Sorting, aggregation, parsing, math |
| Can you overlap? | **Yes** — waits overlap freely | No — the CPU is genuinely busy |
| Right tool | async, or threads | More processes, or more machines |

This determines which concurrency model helps. Ten concurrent network calls can share one core happily, because nine of them are always idle. Ten concurrent `SUM()` aggregations cannot — they each need the CPU.

**Where does your API sit?** Mostly CPU-and-disk, not network-wait. A SQLite query is a local file read plus in-process B-tree walking and arithmetic — no socket, no remote server. This is exactly why `api.py`'s docstring says *"routes are sync (SQLite reads are fast and CPU-cheap)"*. It's the correct call, and §3.7 explains why the alternative would be worse.

### 3.3 Process, thread, coroutine

Three units of concurrent execution, from heaviest to lightest:

**Process** — an independent program with its own memory space. Two processes cannot see each other's variables. Isolation is total: one crashing doesn't touch the other. Expensive to create (milliseconds), expensive to communicate between (serialize through a pipe or socket).

*ETL analogy: two separate `python etl.py` invocations.*

**Thread** — a separate line of execution **inside one process, sharing all memory**. Cheap to create (microseconds), free to communicate (just read the same variable). And that sharing is exactly the danger — see §3.9.

*ETL analogy: `ThreadPoolExecutor` over a list of files. Same process, same globals, several files in flight.*

**Coroutine** — a function that can voluntarily pause and hand control back, then resume where it left off. Not a thread: coroutines within one thread never run *simultaneously*, they take turns. Absurdly cheap — thousands per thread. But cooperative: a coroutine that never pauses starves every other coroutine.

*Closest pandas analogy: a generator. `def gen(): yield 1; yield 2` pauses at each `yield` and resumes on the next `next()`. A coroutine is that, with the scheduler deciding when to resume you.*

```
Process   ██████████████  own memory, own everything
  Thread  ─────────────   shares memory, OS decides who runs when
  Thread  ─────────────
    Coroutine ····        shares thread, YOU decide when to yield
    Coroutine ····
```

### 3.4 The GIL, and why threads still help

CPython has a **Global Interpreter Lock**: only one thread may execute Python bytecode at a time. So pure-Python threads give you zero parallelism on CPU work — four threads summing numbers on four cores run about as fast as one.

Which raises the obvious question: why does FastAPI's threadpool help at all?

**Because the GIL is released during blocking operations.** When a thread makes a call that waits — a socket read, a disk read, or a C extension that explicitly drops the lock — it releases the GIL so another thread can run Python.

And here's the part that matters for you specifically: **Python's `sqlite3` module releases the GIL around the underlying C calls.** So while thread A is inside SQLite executing a query — walking B-trees, reading pages, aggregating — thread B is free to run Python. Your SQLite queries genuinely execute in parallel across threads. The threadpool isn't a workaround here; it's the right mechanism.

(CPython 3.13+ ships an optional free-threaded build without the GIL. Not the default, and not something to plan around yet.)

### 3.5 Blocking, non-blocking, and the event loop

**Blocking** means: this call does not return until the work is done, and the thread can do nothing else meanwhile.

```python
data = conn.execute("SELECT ...").fetchall()   ## blocking: thread is stuck here
```

The **event loop** is the alternative model. One thread, one queue of pending work, one loop:

```
loop forever:
    is any awaited operation ready?  → resume that coroutine
    did a coroutine hit an await?    → park it, note what it's waiting for
    nothing ready?                   → sleep until something is
```

One thread interleaves thousands of coroutines, because each one voluntarily steps aside whenever it would otherwise wait. This is what `uvicorn` runs at its core.

**Analogy.** Blocking is a bank teller who serves one customer to completion, including the twenty minutes they spend on the phone with their spouse. The event loop is a teller who says "make your call, step aside, wave when you're ready" and serves eleven other people meanwhile. Same teller, same total work, far less idle time.

### 3.6 What `await` actually does

```python
async def handler():
    rows = await fetch_from_network()    ## ← pause here, let others run
    return process(rows)
```

`await` means: *"this will take a while; park me, run someone else, resume me when it's ready."* The event loop keeps a set of parked coroutines and resumes whichever becomes ready.

The critical, non-obvious point: **`await` only yields control if the thing you're awaiting is designed to.** `await` on an async database driver yields. A plain blocking call inside an `async def` does not — it just blocks, and it blocks *the entire event loop*.

### 3.7 The cardinal sin

```python
@app.get("/forecast/{key}/cashflow")
async def cashflow(key: str):                 ## ← async
    rows = conn.execute("SELECT ...").fetchall()   ## ← blocking. Disaster.
```

That route is `async`, so FastAPI runs it **directly on the event loop thread**. The `sqlite3` call blocks. The event loop cannot turn. For those 200 milliseconds, *every* request in the process is frozen — not just this one. Ten concurrent users see 2 seconds of latency instead of 200ms, and the effect compounds under load.

This is the most common serious mistake in FastAPI code, and it is the reason `api.py` uses `def` rather than `async def`. Getting this right matters more than almost any other performance decision in the file:

| Route style | Where it runs | Blocking call inside is… |
|---|---|---|
| `async def` | Event loop thread | **Catastrophic** — freezes all requests |
| `def` | Threadpool worker | **Fine** — freezes only that worker |

**The rule:** `async def` only if everything inside it is awaitable. Blocking libraries — `sqlite3`, `requests`, `pandas`, ordinary file I/O — go in a plain `def` and let the framework put them in a thread. Since `sqlite3` has no async interface and releases the GIL anyway, `def` is strictly correct here.

### 3.8 Threadpools, and the number 40

When FastAPI sees a plain `def` route, it dispatches it via `anyio.to_thread.run_sync()`, which hands the work to a pool of reusable worker threads. The event loop `await`s that handoff, so the loop stays free while the worker blocks.

Why a *pool* rather than a thread per request: thread creation costs microseconds and each thread reserves stack memory, so ten thousand concurrent requests would mean ten thousand threads and an out-of-memory crash. A pool bounds it — N workers, extra work queues up.

**AnyIO's default limiter is 40 threads.** That's a real capacity number for your API: 41 simultaneous in-flight requests means one is queued waiting for a worker. Worth knowing before you go looking for the bottleneck elsewhere.

### 3.9 Race conditions

Threads share memory. Two threads touching the same mutable thing without coordination produce a **race condition**: correctness depending on timing.

```python
counter = 0
def increment():
    global counter
    counter = counter + 1      ## read, add, write — three steps, not one
```

Two threads, interleaved badly:

```
Thread A: read counter (0)
Thread B: read counter (0)
Thread A: write 1
Thread B: write 1          ← two increments, one result. Lost update.
```

The general shape is **read-modify-write**: any sequence where you read state, decide based on it, and write back. If another thread writes between your read and your write, your decision was based on stale data. Remember this shape — it comes back in Part 7 as the central hazard of adding writes, and it's the same bug at a different scale.

**Why your API is immune today.** Two structural reasons, both worth being conscious of because both are load-bearing:

1. **Nothing is shared.** Each request gets its own connection, its own cursor, its own result lists. No mutable state crosses request boundaries. Nothing to race on.
2. **Nothing is written.** The store is read-only. Concurrent readers of unchanging data cannot conflict — there's no interleaving that produces a wrong answer.

Both properties are consequences of the architecture, not accidents. Point 1 is why §3.1 of the walkthrough insists that caching a connection at module scope would be a real bug rather than a harmless optimization. Point 2 is what evaporates in Part 7.

### 3.10 Why SQLite cares which thread you're on (your item 4)

Now item 4 assembles from the pieces.

A `sqlite3.Connection` object holds **mutable internal state**: the open file handle, the page cache, prepared statements, current transaction status. If two threads used one connection simultaneously, they'd corrupt that state — classic race condition, and one that manifests as garbled results or crashes rather than clean errors.

So Python's `sqlite3` ships a guard: **it records the thread that created the connection and raises if a different thread uses it.**

```python
sqlite3.ProgrammingError: SQLite objects created in a thread can only be used in that same thread.
```

That's a *conservative* guard. It prevents genuine concurrent misuse, but it also rejects perfectly safe patterns — including yours.

Because look at what FastAPI does with a sync generator dependency. Three separate pieces of work, each dispatched to the threadpool:

```
1. run get_db() up to the yield          → threadpool worker  (may be thread #7)
2. run the route body                    → threadpool worker  (may be thread #12)
3. run get_db()'s teardown after yield   → threadpool worker  (may be thread #3)
```

Nothing guarantees those three land on the same worker. The pool hands out whichever thread is free. So the connection is *created* on one thread, *used* on another, and *closed* on a third.

That's safe — **only one thread touches it at any moment**, and they're strictly sequential, so there's no concurrent access and no race. But the guard can't tell "sequential use across threads" from "simultaneous use across threads." It only compares thread IDs. So it fires.

```python
check_same_thread=False
```

turns the guard off, and you take responsibility for the invariant it was approximating. Here that responsibility is discharged by the design: one connection per request, never stored anywhere, never shared. "Intermittent under load" in the walkthrough is because thread assignment is a scheduling accident — with light traffic the pool may hand you the same worker every time and everything looks fine; under load, assignments scatter and the errors appear.

**The invariant to hold onto:** `check_same_thread=False` is safe if and only if a connection is used by exactly one request at a time. Break that — cache one globally, share one across requests — and you've removed the guard *and* violated the thing it was guarding.

---

## Part 4 — ASGI, uvicorn, FastAPI

### 4.1 The layers

Four distinct things, often conflated:

```
your routes                 ← api.py
  FastAPI                   ← routing, validation, dependency injection
    Starlette               ← ASGI toolkit: requests, responses, middleware
      ASGI                  ← the interface spec (a calling convention)
        uvicorn             ← the server: owns the socket and the event loop
```

**ASGI** (Asynchronous Server Gateway Interface) is just an agreed function signature between servers and frameworks:

```python
async def app(scope, receive, send): ...
```

`scope` describes the request, `receive` awaits incoming data, `send` emits the response. That's the entire contract. It's what lets you swap uvicorn for hypercorn without touching your code — the same role WSGI played for Flask and Django, extended to support async and websockets.

**uvicorn** owns the TCP socket and runs the event loop. **FastAPI** adds the parts you actually write against: routing, validation from type hints, dependency injection, OpenAPI generation.

### 4.2 How FastAPI dispatches

Per route, at import time, FastAPI inspects the function and decides:

```
async def  →  await it directly on the event loop
def        →  await anyio.to_thread.run_sync(it)
```

That's the whole mechanism behind §3.7. One keyword changes where your code executes, and the consequences of getting it wrong are severe and non-local.

### 4.3 Dependency injection (your item 5)

Start with the problem. Without DI, every route manages its own connection:

```python
@app.get("/orchestrations")
def orchestrations():
    conn = store._open_readonly()
    try:
        return queries.list_orchestrations(conn)
    finally:
        conn.close()
```

Five routes, five copies of that boilerplate, five chances to forget the `finally`. And in tests you can't substitute a fixture connection without editing the route.

Dependency injection inverts it: the route *declares what it needs* and the framework provides it.

```python
@app.get("/orchestrations")
def orchestrations(db: sqlite3.Connection = Depends(store.get_db)):
    return queries.list_orchestrations(db)
```

**`Depends(f)` means: before running this route, call `f()`, and pass the result as this argument.** That's the entire concept. A dependency is any callable — a function, a class with `__call__`, another dependency-taking function.

Three clarifications that matter:

**The type annotation is not how FastAPI finds the dependency.** `db: sqlite3.Connection` is for you and your editor. FastAPI keys off the `Depends(...)` default value. Write `db = Depends(store.get_db)` with no annotation and it works identically. This is the opposite of how path params work — there, the annotation *is* the mechanism (§1.6) — and conflating the two is a common source of confusion.

**Note it's `Depends(store.get_db)`, not `Depends(store.get_db())`.** You pass the function itself; FastAPI calls it. Passing the result would call it once at import time and reuse one connection forever — precisely the module-level-sharing bug from §3.10.

**Dependencies compose.** A dependency can itself take dependencies, and FastAPI resolves the whole graph, deduplicating shared nodes within a request. This is what makes the auth suggestion in the walkthrough a small change rather than a refactor:

```python
def current_client(token: str = Depends(oauth2_scheme)) -> str: ...

def scoped_key(orchestration_key: str,
               client: str = Depends(current_client)) -> str: ...

@app.get("/forecast/{orchestration_key}/cashflow")
def cashflow(key: str = Depends(scoped_key), db = Depends(store.get_db)): ...
```

Authorization becomes a declared requirement rather than a line you must remember in every handler. Forgetting a dependency is a visible absence in a signature; forgetting a line inside a body is invisible.

### 4.4 Generator dependencies

When a dependency is a **generator function**, FastAPI treats code before `yield` as setup and after as teardown:

```python
def get_db() -> Iterator[sqlite3.Connection]:
    with connect() as conn:      ## setup
        yield conn               ## ← the route runs here
                                 ## teardown when the with-block exits
```

Mechanically, FastAPI wraps the generator in a context manager and registers it on an `AsyncExitStack` scoped to the request. `AsyncExitStack` is a stack of cleanup actions: when the request ends — success, exception, or client disconnect — the stack unwinds and every registered teardown runs, in reverse registration order.

This is the same pattern as `contextlib.contextmanager`, which you already use in `store.connect()`. FastAPI is applying it at request scope instead of block scope.

Why generators rather than a separate `on_finish` callback: the `try/finally` inside the generator makes cleanup **structurally guaranteed** rather than dependent on the framework remembering. If the route raises, the exception propagates back *into* the generator at the `yield` point, the `finally` runs, the connection closes. You cannot leak a connection without deleting the `finally`.

Teardown runs **after the response is sent**, which is why §2 of the walkthrough notes the connection stays open through JSON serialization.

---

## Part 5 — What a connection is, and lifecycle strategies

### 5.1 In-process library vs client/server

The most important distinction for reasoning about SQLite, and the source of most confusion carried over from other databases.

**Postgres, MySQL, SQL Server** are **client/server**. A separate long-running process owns the data. "Connecting" means opening a TCP socket, negotiating TLS, authenticating, and having the server allocate a session (Postgres forks a backend process). Cost: **milliseconds to tens of milliseconds.** Your queries are messages sent over a wire.

**SQLite is a library.** There is no server. It compiles into your process, and `conn.execute(...)` is a *function call* into C code that reads your file directly. "Connecting" means opening a file handle and reading the schema. Cost: **microseconds to low milliseconds.**

```
Postgres:  your process ──TCP──> postgres process ──> disk
SQLite:    your process ────────────────────────────> disk
```

Nearly every "best practice" you'll read about connection pooling comes from the client/server world, where connections are expensive and limited. Applying it reflexively to SQLite is cargo-culting. The right question is always *what does a connection actually cost here*, and the answer differs by three orders of magnitude.

### 5.2 What a SQLite connection holds

- an open OS file handle
- the parsed schema
- a **page cache** of recently-read B-tree pages (`PRAGMA cache_size`, ~2MB default)
- prepared statement cache
- transaction state
- lock state

Two of those drive the whole lifecycle discussion. The **page cache** is per-connection and dies on close — that's the cold-cache cost in §5 of the walkthrough. The **lock and transaction state** is mutable, which is why sharing a connection across threads is unsafe (§3.10).

### 5.3 The four strategies

**A — One connection per request** *(what you do)*

```python
def get_db():
    with connect() as conn:
        yield conn
```

✅ No sharing, so no races. Bounded lifetime, so no leaks. Trivially correct.
❌ Cold page cache each time. One `open()` + schema read per request.

**B — One connection for the whole app**

```python
conn = sqlite3.connect(..., check_same_thread=False)   ## module level
```

✅ Warm cache, zero per-request setup.
❌ **Shared mutable state across threads.** Threadpool workers would use it simultaneously; interleaved cursors and transaction state corrupt each other. Needs a lock around every use, which serializes all queries and destroys the concurrency you were buying. This is the bug §3.10 warns about.

**C — A connection pool**

Keep N open, check one out per request, return it after.

✅ Warm caches, no per-request open, bounded count.
❌ Real machinery: checkout/return discipline, leak detection, health checks, thread affinity. Solves a problem SQLite mostly doesn't have.

**D — One connection per thread** (thread-local)

```python
_local = threading.local()
def get_conn():
    if not hasattr(_local, "conn"):
        _local.conn = sqlite3.connect(...)
    return _local.conn
```

✅ Warm cache, no locking (each thread has its own), no pool bookkeeping.
❌ Lifetime is the thread's, so connections effectively never close; a stale one survives a store swap. And it interacts badly with the fact that FastAPI may run your request's phases on different threads.

### 5.4 The decision table

| | SQLite | Postgres |
|---|---|---|
| Connection cost | µs–low ms | ms–tens of ms |
| Hard connection limit | No | Yes (`max_connections`) |
| Cross-request state to preserve | Page cache only | Session, prepared statements, temp tables |
| **Right default** | **Per request** | **Pool** |

**Per-request is right for you** because the only thing you lose is the page cache, and `PRAGMA mmap_size` recovers most of that by moving caching to the OS page cache — which is shared across connections and survives close. You get warm-cache behavior without giving up the isolation that makes correctness obvious.

That's the shape of a good engineering answer here: identify the single real cost, address it directly, keep the simple model.

---

## Part 6 — SQLite concurrency mechanics

Read-only today, but this is the ground Part 7 stands on.

### 6.1 Locking is file locking

With no server to arbitrate, SQLite coordinates concurrent access using **OS file locks** on the database file. Every process and connection that wants access takes a lock; the OS ensures they're mutually consistent.

This has a hard consequence: **SQLite's concurrency correctness depends on the filesystem implementing locking correctly.** Local disks do. Many network filesystems do not, or do so unreliably. Hold that thought for §6.6.

### 6.2 Rollback journal mode: the writer excludes readers

The default (`PRAGMA journal_mode=DELETE`), and what your build produces.

To write, SQLite copies the original pages into a `-journal` sidecar file, then modifies the database in place. If the process dies mid-write, the next connection sees the journal and rolls back — that's the atomicity mechanism.

The locking progression:

```
readers hold           SHARED
a writer needs         RESERVED → PENDING → EXCLUSIVE
EXCLUSIVE requires     no SHARED locks held by anyone
```

So **a writer must wait for all readers to finish, and while it holds EXCLUSIVE, no reader can start.** Readers and writers are mutually exclusive.

For your read-only serving layer this costs nothing — readers share freely. It becomes the central constraint the moment there's a writer.

### 6.3 WAL mode: readers and the writer coexist

`PRAGMA journal_mode=WAL` inverts the mechanism. Instead of modifying the database and journaling the old pages, the writer **appends new pages to a `-wal` file** and leaves the main database untouched. Readers consult the WAL to see recent changes; periodically a "checkpoint" folds the WAL back into the main file.

The result:

| | Journal | WAL |
|---|---|---|
| Readers block the writer | **Yes** | No |
| Writer blocks readers | **Yes** | No |
| Concurrent writers | No | No — still exactly one |
| Reader consistency | — | Snapshot as of when the read began |
| Extra files | `-journal` (transient) | `-wal`, `-shm` (persistent) |

**WAL is the correct choice for any read-write SQLite workload.** The rule to memorize: *many readers, one writer, and they don't block each other.*

Two costs. It needs the `-shm` shared-memory file, so all participants must be on the same host — no network filesystems. And a read-only connection (`mode=ro`) needs the sidecars present and readable, which is the gotcha §5 of the walkthrough flags; `immutable=1` is the usual escape.

### 6.4 `SQLITE_BUSY` and `busy_timeout`

When a lock can't be acquired, SQLite doesn't queue — it **fails immediately**:

```
sqlite3.OperationalError: database is locked
```

Which surprises everyone once. The fix is to tell SQLite to retry:

```python
conn.execute("PRAGMA busy_timeout = 5000")   ## retry for up to 5s
```

Now it polls for five seconds before giving up. **Any read-write SQLite deployment should set this**, and its absence is the single most common cause of "SQLite doesn't work under load" complaints. It's not a load problem; it's a missing retry policy.

### 6.5 Transactions and Python's autocommit quirk

A **transaction** is a group of statements that either all take effect or none do — atomicity. In SQLite, `BEGIN` … `COMMIT`, and `ROLLBACK` to discard.

Python's `sqlite3` has a historically confusing default. With `isolation_level=""` (the default), the module **implicitly opens a transaction before DML** (`INSERT`/`UPDATE`/`DELETE`) and waits for you to `commit()`. DDL and `SELECT` behave differently across versions. That's why `build_serving_store.py` needs its explicit `conn.commit()`.

Clearer options:

```python
sqlite3.connect(path, isolation_level=None)   ## autocommit; you write BEGIN yourself
sqlite3.connect(path, autocommit=False)       ## Python 3.12+, explicit and predictable
```

For anything transactional, prefer managing `BEGIN`/`COMMIT` explicitly. Implicit transaction boundaries you didn't choose are a bad foundation for financial writes.

### 6.6 The Azure warning

Worth stating plainly because it's a real and common production failure:

**Do not put a read-write SQLite database on a network filesystem.** Azure Files, SMB/CIFS shares, NFS, and most container volume mounts either don't implement the locking primitives SQLite needs or implement them unreliably. The failure mode isn't a clean error — it's intermittent `database is locked` and, in the bad cases, **silent corruption**. WAL is strictly unusable there, since `-shm` requires real shared memory.

Your options on Azure:

| Approach | Verdict |
|---|---|
| SQLite on a local/ephemeral disk, read-only, rebuilt and redeployed | ✅ Fits your current architecture exactly |
| SQLite on Azure Files, read-write | ❌ Corruption risk |
| SQLite read-only on a mounted share | ⚠️ Works, but slow; prefer copying to local disk at startup |
| Postgres (Azure Database for PostgreSQL) for writes | ✅ The answer once you have real writes |

Your build/serve split happens to be the pattern that survives this constraint intact: an immutable artifact deployed alongside the app, never written at runtime. Keep that property as long as you can.

---

## Part 7 — When you add writes

You asked how the architecture should change. The short version: **don't convert the read store into a read-write store.** Add a separate write path and keep the projection.

Here's the reasoning.

### 7.1 What writes actually break

Your current design rests on two properties, and both are load-bearing for everything above:

1. **The store is derived.** It can be deleted and rebuilt from the data system. That's why `if_exists="replace"` is safe and why there are no migrations.
2. **The store is immutable while serving.** No concurrent modification, no locking contention, no race conditions, no transactions to reason about.

A naive "just add `INSERT` to the API" destroys both. The store stops being disposable — it now contains data that exists nowhere else, so a rebuild would delete user work. And it stops being immutable, so every concept in Part 6 becomes live.

### 7.2 ACID

The four guarantees a transactional database offers:

- **Atomicity** — all statements in a transaction take effect, or none. No half-applied edits.
- **Consistency** — declared constraints hold at commit. (This is where the FK constraints and `UNIQUE` indexes you're adding start earning their keep at runtime, not just at build time.)
- **Isolation** — concurrent transactions don't see each other's uncommitted work.
- **Durability** — once committed, it survives a crash.

SQLite provides all four, and its isolation is `SERIALIZABLE` — the strongest level — because there's only ever one writer. Ironically, SQLite's biggest limitation is also what makes reasoning about it easy.

### 7.3 The anomaly that will actually bite you

Of the classic isolation anomalies, one matters disproportionately for a forecast-editing UI: **lost update**. It's the read-modify-write race from §3.9, at the data layer.

```
10:00  User A loads revision 3 for editing
10:01  User B loads revision 3 for editing
10:05  User A changes acres 500 → 520, saves
10:06  User B changes yield 45 → 47, saves
       ← B's save was computed from a snapshot that predates A's change.
         Depending on how you write, A's change is silently gone.
```

Both transactions were individually atomic. Neither database nor framework did anything wrong. The data is still corrupt — and note that *nothing errors*, so you find out when a client asks why their acreage reverted.

### 7.4 Optimistic concurrency

The standard fix, and it fits your model well. Carry a version on each editable row and make the write conditional on it:

```sql
UPDATE forecast_input
   SET value = ?, version = version + 1
 WHERE input_output_key = ? AND version = ?;      -- version the client read
```

Then **check the affected row count**. Zero means someone else wrote first — the row no longer has the version you read. Return `409 Conflict` and let the UI reload and show the user what changed.

"Optimistic" because it assumes conflicts are rare and detects them, rather than preventing them with locks. That's the right assumption for a forecast editor where two users rarely touch the same input.

Checking rowcount is the whole mechanism, and forgetting it is the whole bug. An `UPDATE` that matches nothing succeeds silently.

### 7.5 Idempotency

HTTP requests get retried — by users double-clicking, by proxies, by mobile networks. Without protection, one logical action becomes two rows.

The fix: the client generates a unique key per *intent* and sends it; the server enforces uniqueness on it.

```
POST /forecast/c0001_2026/inputs
Idempotency-Key: 7f3a9c21-...
```

A duplicate key returns the original result instead of applying the change twice. This matters more for money than almost anywhere else — a duplicated cost entry is a wrong forecast.

### 7.6 The recommendation: writes create revisions

Here's the part specific to your system, and I think it's the important insight.

**You already have an immutable, revision-based data model.** Inputs and outputs carry `revision_num`; resolution picks the highest revision at or below a bound; revisions are meant to be stable references so a client can reopen a three-month-old forecast and see what they saw then.

So the natural shape for user edits is not `UPDATE`. It's **append a new revision**:

```
User edits acres for client 1
  → INSERT a new input row: revision_num = max(existing) + 1
  → the old revision stays, untouched, forever
  → resolution logic picks it up automatically on the next build
```

What this buys you, essentially for free:

| | `UPDATE` in place | Append a revision |
|---|---|---|
| Audit trail | Build one separately | It *is* the data |
| Lost update | Real hazard | Structurally impossible — no shared row to overwrite |
| "What did we show in March?" | Unanswerable | A query |
| Revision immutability (contract §6) | Violated | Preserved |
| Concurrent writers | Contend on rows | Contend only on revision-number allocation |
| Rollback a bad edit | Restore from backup | Point at the prior revision |

The concurrency problem shrinks to *"two writers must not claim the same revision number,"* which a `UNIQUE (entity_key, revision_num)` constraint solves outright: the loser gets an `IntegrityError`, retries with the next number, done. No optimistic-concurrency version columns, no lost updates, because **nothing is ever overwritten.**

This is close to event sourcing, and to how ledgers work: you don't erase an entry, you post a correcting one. For a financial product that's not just convenient, it's closer to what auditability requires. Your engine already works this way — user edits should follow the same rule rather than introducing a second, mutable model alongside it.

### 7.7 Keep the projection

With writes as appends, the architecture extends rather than changes:

```
┌─────────────────┐   writes (append revisions)
│  Write store    │ ◀───────────────────────  POST /inputs
│  (system of     │
│   record)       │
└────────┬────────┘
         │  batch: engine + build_serving_store.py
         ▼
┌─────────────────┐   reads
│  Serving store  │ ──────────────────────▶   GET /forecast/...
│  (projection,   │
│   read-only)    │
└─────────────────┘
```

This is **CQRS** — Command Query Responsibility Segregation — the pattern of separating the write model from the read model. You've already built the read side, and the reason it's fast is precisely that it's a denormalized projection optimized for the drill-down.

Everything you've built survives: `mode=ro`, connection-per-request, the grain rule, the indexes, the disposable rebuildable store. The write path is new code beside it, not a rewrite of it.

The cost is **staleness**: an edit isn't visible until the next build. Your options, in ascending complexity — rebuild on a schedule; rebuild triggered by a write; or serve pending edits from the write store and overlay them at read time. For a forecasting product where a client edits assumptions and then wants to see the new forecast, a triggered rebuild is usually the honest answer, with an explicit "recalculating" state in the UI. Don't reach for the overlay until someone actually complains.

### 7.8 When to leave SQLite

SQLite is genuinely production-grade — the "toy database" reputation is wrong. It handles substantial read concurrency and moderate write throughput fine. The real thresholds are architectural, not about scale:

| Signal | Verdict |
|---|---|
| Read-only serving, single host | ✅ SQLite, as now |
| Occasional writes, one host, WAL + `busy_timeout` | ✅ SQLite is fine |
| **More than one app instance writing** | ❌ Move to Postgres |
| **Database must live on a network share** | ❌ Move to Postgres (§6.6) |
| Sustained concurrent write load | ❌ Move to Postgres |
| Need row-level locking, `SELECT FOR UPDATE` | ❌ Postgres |
| Need real `NUMERIC`/`DECIMAL` for money | ⚠️ Strong Postgres argument |

That last row is worth weighing given the float-money issue. Postgres has a true arbitrary-precision `NUMERIC` type; SQLite has only `INTEGER`/`REAL`/`TEXT`, so exact decimal money means integer cents or text by convention. If you're going to touch the money representation anyway, doing it at the same time as a Postgres migration saves doing it twice.

Note the top row of that table: the thing that most often forces the move is **horizontal scaling** — the moment you run two instances behind a load balancer, they cannot share a SQLite file safely. Your read-only store sidesteps this entirely, because each instance can have its own copy of an immutable artifact. Another reason the projection is worth keeping.

### 7.9 Migrations

The one operational discipline that arrives with a write store. Once data exists only there, you can no longer drop and recreate — you must *evolve* the schema while preserving contents.

The minimum viable version:

- A `schema_version` table holding a single integer.
- Numbered, ordered, forward-only migration scripts: `001_create_inputs.sql`, `002_add_version_column.sql`.
- On startup: read the current version, apply every later migration in order, in a transaction, updating the version as you go.

Alembic (with SQLAlchemy) is the standard Python tool. For a schema this size, fifty lines of your own is also a legitimate choice — the discipline matters more than the tooling.

Your *read* store needs none of this, and that's worth appreciating as a real architectural benefit: `if_exists="replace"` works precisely because the projection is disposable. Adding a write store means adopting migrations for it while keeping the read side migration-free.

---

## Cheat sheet

**HTTP**
- Request path = code that runs while a user waits. Batch work belongs off it.
- Stateless: nothing persists between requests. Hence per-request resources.
- Path params identify, query params filter.
- 4xx = caller's fault, 5xx = yours. 422 = validation failed, 503 = dependency down.
- Type annotations on route params are executable: FastAPI coerces, and returns 422 on failure.

**URIs**
- `?` starts the query, `#` starts the fragment, `%` starts an escape. Structural, not decorative.
- `uri=True` makes SQLite parse the filename as a URI so flags fit in the query string.
- Interpolating unescaped data into a URI is the same bug class as SQL injection. Use `quote()`.
- A `#` in the path truncates it **and** discards `mode=ro`.

**Concurrency**
- I/O-bound work overlaps; CPU-bound work doesn't.
- Process = own memory. Thread = shared memory, OS schedules. Coroutine = shared thread, yields voluntarily.
- The GIL allows one thread of Python bytecode at a time, but is released during I/O — and `sqlite3` releases it, so queries do run in parallel.
- `await` yields control only if what you await is awaitable. A blocking call in `async def` freezes every request in the process.
- **`def` for blocking work, `async def` only when everything inside is awaitable.**
- AnyIO's threadpool defaults to 40 workers. That's your concurrency ceiling for sync routes.
- Race condition = correctness depending on timing. The shape is always read-modify-write.
- `check_same_thread=False` is safe iff one connection is used by one request at a time.

**Connections**
- SQLite is an in-process library; Postgres is client/server. Connection costs differ ~1000×.
- Per-request is right for SQLite; pooling is right for Postgres. Don't cargo-cult across.
- Per-request's only real cost is the cold page cache; `PRAGMA mmap_size` recovers it.
- Never share one connection across requests. It's mutable state.

**SQLite concurrency**
- Coordination is via file locks, so it depends on the filesystem being honest.
- Journal mode: readers and writer are mutually exclusive. WAL: many readers + one writer, no blocking.
- WAL for anything read-write. Always set `busy_timeout`, or you'll get "database is locked" under trivial load.
- Never read-write SQLite on a network filesystem.

**Writes**
- Writes break both current invariants: the store stops being disposable and stops being immutable.
- Lost update is the anomaly to design against. Optimistic concurrency = version column + conditional `UPDATE` + **check the rowcount**.
- Better for your model: **appends, not updates.** New revision per edit. Audit trail free, lost updates impossible, immutability preserved.
- Keep the read projection (CQRS). The cost is staleness; usually pay it with a triggered rebuild.
- Leave SQLite when you need multiple writing instances, a network filesystem, or exact decimal money.
- A write store needs migrations. Your read store doesn't, and that's a feature.

# The GrowLytics Serving Layer — Complete Walkthrough

*`store.py`, `queries.py`, `schema.py`, `api.py` — line by line, plus the mechanics they depend on: FastAPI's dependency lifecycle, connection-per-request and the page cache, what `mode=ro` actually guarantees, and where the current indexes fail to serve the current queries.*

---

## Table of contents

- [Prerequisites for the Serving Layer](#prerequisites-for-the-serving-layer)
  - [How to read this](#how-to-read-this)
  - [Part 1 — The request/response model](#part-1--the-requestresponse-model)
    - [1.1 What a web server actually does](#11-what-a-web-server-actually-does)
    - [1.2 The request path](#12-the-request-path)
    - [1.3 Statelessness](#13-statelessness)
    - [1.4 Anatomy of a URL](#14-anatomy-of-a-url)
    - [1.5 Status codes](#15-status-codes)
    - [1.6 Where 422 comes from (your item 1)](#16-where-422-comes-from-your-item-1)
  - [Part 2 — URIs and the `file:` scheme](#part-2--uris-and-the-file-scheme)
    - [2.1 URI grammar (your items 2 and 3)](#21-uri-grammar-your-items-2-and-3)
    - [2.2 Why SQLite has a URI mode](#22-why-sqlite-has-a-uri-mode)
    - [2.3 What breaks](#23-what-breaks)
    - [2.4 Why Azure specifically](#24-why-azure-specifically)
  - [Part 3 — Concurrency from zero](#part-3--concurrency-from-zero)
    - [3.1 The problem is waiting](#31-the-problem-is-waiting)
    - [3.2 I/O-bound vs CPU-bound](#32-io-bound-vs-cpu-bound)
    - [3.3 Process, thread, coroutine](#33-process-thread-coroutine)
    - [3.4 The GIL, and why threads still help](#34-the-gil-and-why-threads-still-help)
    - [3.5 Blocking, non-blocking, and the event loop](#35-blocking-non-blocking-and-the-event-loop)
    - [3.6 What `await` actually does](#36-what-await-actually-does)
    - [3.7 The cardinal sin](#37-the-cardinal-sin)
    - [3.8 Threadpools, and the number 40](#38-threadpools-and-the-number-40)
    - [3.9 Race conditions](#39-race-conditions)
    - [3.10 Why SQLite cares which thread you're on (your item 4)](#310-why-sqlite-cares-which-thread-youre-on-your-item-4)
  - [Part 4 — ASGI, uvicorn, FastAPI](#part-4--asgi-uvicorn-fastapi)
    - [4.1 The layers](#41-the-layers)
    - [4.2 How FastAPI dispatches](#42-how-fastapi-dispatches)
    - [4.3 Dependency injection (your item 5)](#43-dependency-injection-your-item-5)
    - [4.4 Generator dependencies](#44-generator-dependencies)
  - [Part 5 — What a connection is, and lifecycle strategies](#part-5--what-a-connection-is-and-lifecycle-strategies)
    - [5.1 In-process library vs client/server](#51-in-process-library-vs-clientserver)
    - [5.2 What a SQLite connection holds](#52-what-a-sqlite-connection-holds)
    - [5.3 The four strategies](#53-the-four-strategies)
    - [5.4 The decision table](#54-the-decision-table)
  - [Part 6 — SQLite concurrency mechanics](#part-6--sqlite-concurrency-mechanics)
    - [6.1 Locking is file locking](#61-locking-is-file-locking)
    - [6.2 Rollback journal mode: the writer excludes readers](#62-rollback-journal-mode-the-writer-excludes-readers)
    - [6.3 WAL mode: readers and the writer coexist](#63-wal-mode-readers-and-the-writer-coexist)
    - [6.4 `SQLITE_BUSY` and `busy_timeout`](#64-sqlite_busy-and-busy_timeout)
    - [6.5 Transactions and Python's autocommit quirk](#65-transactions-and-pythons-autocommit-quirk)
    - [6.6 The Azure warning](#66-the-azure-warning)
  - [Part 7 — When you add writes](#part-7--when-you-add-writes)
    - [7.1 What writes actually break](#71-what-writes-actually-break)
    - [7.2 ACID](#72-acid)
    - [7.3 The anomaly that will actually bite you](#73-the-anomaly-that-will-actually-bite-you)
    - [7.4 Optimistic concurrency](#74-optimistic-concurrency)
    - [7.5 Idempotency](#75-idempotency)
    - [7.6 The recommendation: writes create revisions](#76-the-recommendation-writes-create-revisions)
    - [7.7 Keep the projection](#77-keep-the-projection)
    - [7.8 When to leave SQLite](#78-when-to-leave-sqlite)
    - [7.9 Migrations](#79-migrations)
  - [Cheat sheet](#cheat-sheet)
- [The GrowLytics Serving Layer — Complete Walkthrough](#the-growlytics-serving-layer--complete-walkthrough)
  - [Table of contents](#table-of-contents)
  - [1. The 30-second model](#1-the-30-second-model)
  - [2. Anatomy of one request](#2-anatomy-of-one-request)
  - [3. `store.py` — the read-only door](#3-storepy--the-read-only-door)
    - [3.1 `_open_readonly`](#31-_open_readonly)
    - [3.2 `connect()` and `get_db()`](#32-connect-and-get_db)
  - [4. Deep dive: FastAPI's dependency lifecycle](#4-deep-dive-fastapis-dependency-lifecycle)
  - [5. Deep dive: connection-per-request, and what it actually costs](#5-deep-dive-connection-per-request-and-what-it-actually-costs)
  - [6. `queries.py` — the read surface](#6-queriespy--the-read-surface)
    - [6.1 `_rows`](#61-_rows)
    - [6.2 `list_orchestrations` — two bugs](#62-list_orchestrations--two-bugs)
    - [6.3 `net_cash_flow`](#63-net_cash_flow)
    - [6.4 `month_breakdown`](#64-month_breakdown)
    - [6.5 `category_sources`](#65-category_sources)
    - [6.6 `source_lineage`](#66-source_lineage)
    - [6.7 The `__main__` smoke test](#67-the-__main__-smoke-test)
  - [7. Deep dive: the index/query mismatch](#7-deep-dive-the-indexquery-mismatch)
  - [8. Deep dive: the rounding divergence](#8-deep-dive-the-rounding-divergence)
  - [9. `schema.py` — the contract](#9-schemapy--the-contract)
    - [9.1 What `response_model` actually does](#91-what-response_model-actually-does)
    - [9.2 Field-by-field notes](#92-field-by-field-notes)
    - [9.3 Validators as the last line of defense](#93-validators-as-the-last-line-of-defense)
  - [10. `api.py` — the HTTP surface](#10-apipy--the-http-surface)
    - [10.1 The shape is right](#101-the-shape-is-right)
    - [10.2 Inconsistent empty-result semantics](#102-inconsistent-empty-result-semantics)
    - [10.3 Unvalidated `category`](#103-unvalidated-category)
    - [10.4 `/health` cannot fail](#104-health-cannot-fail)
    - [10.5 Storage errors surface as 500s](#105-storage-errors-surface-as-500s)
    - [10.6 Module path drift](#106-module-path-drift)
  - [11. Security posture](#11-security-posture)
  - [12. Findings, ranked](#12-findings-ranked)
  - [13. How to test each layer](#13-how-to-test-each-layer)
  - [14. Cheat sheet](#14-cheat-sheet)
- [Code Appendix](#code-appendix)
  - [`store.py`](#storepy)
  - [`schema.py`](#schemapy)
  - [`queries.py`](#queriespy)
  - [`api.py`](#apipy)

---

## 1. The 30-second model

Four files, four responsibilities, and the discipline is that none of them reaches into another's job:

| File | Owns | Knows nothing about |
|---|---|---|
| `store.py` | Opening and closing connections safely | SQL, HTTP, business meaning |
| `queries.py` | SQL, grain separation, lineage traversal | HTTP, Pydantic, config |
| `schema.py` | The wire contract | SQL, connections |
| `api.py` | Routing, status codes, CORS | SQL, connection lifetime |

The property that makes this design good, and the one to protect above all others: **`queries.py` is pure.** Every function takes a connection and returns plain Python. No global state, no config import (outside `__main__`), no framework. That means you can test the entire business logic of the product with an in-memory SQLite database and zero HTTP — see §13. The moment a query function starts importing `config` or raising `HTTPException`, that property is gone and testing gets 10× harder.

The other load-bearing idea is the **grain separation rule**, documented at the top of `queries.py`: money is read only from `cash_flow_streams`, never through a `input_output_linkage` join, because that join is many-to-many on `output_orchestration_key` and would fan out every stream row by its input count. Level 3 joins linkage precisely because it *lists* rather than *sums*, so the fan-out is the desired output. This is correctly implemented and correctly documented. It is the single most important invariant in the file.

---

## 2. Anatomy of one request

Trace `GET /forecast/c0001_2026_3-pre-harvest/month/4`:

```
HTTP request arrives (uvicorn, async event loop)
  │
  ├─ CORS middleware: origin in config.CORS_ORIGINS? → else no CORS headers
  │
  ├─ FastAPI matches route, parses path params
  │    orchestration_key: str = "c0001_2026_3-pre-harvest"
  │    month_num: int = 4                    ← coerced + validated here
  │
  ├─ Resolve dependencies: Depends(store.get_db)
  │    ├─ get_db() runs in the threadpool → enters connect()
  │    ├─ _open_readonly()
  │    │    ├─ STORE_PATH.exists()           ← a stat() per request
  │    │    └─ sqlite3.connect("file:...?mode=ro", uri=True)
  │    └─ yields conn ────────────────────────┐
  │                                           │
  ├─ Route body runs in threadpool            │
  │    └─ queries.month_breakdown(conn, ...)  │
  │         ├─ conn.execute(SELECT ... GROUP BY category)
  │         ├─ _rows(cur) → list[dict]
  │         └─ Python-side inflow/outflow/net
  │                                           │
  ├─ Route returns dict                       │
  │    └─ if not categories → HTTPException(404)
  │                                           │
  ├─ response_model=MonthBreakdown            │
  │    ├─ validate every field                │
  │    ├─ DROP undeclared keys (silently)     │
  │    └─ missing required key → 500          │
  │                                           │
  ├─ JSON serialized, response sent           │
  │                                           │
  └─ Dependency teardown ─────────────────────┘
       └─ finally: conn.close()
```

Two things worth fixing in your mental model of that trace:

- **Validation happens twice, in opposite directions.** Inbound, FastAPI coerces path/query params into the annotated types and returns 422 on failure. Outbound, Pydantic validates the response and returns 500 on failure. Inbound errors are the client's fault; outbound errors are yours.
- **Teardown runs after the response is sent.** The connection is still open while JSON is being serialized, and closes after. So connection lifetime is slightly longer than query time.

---

## 3. `store.py` — the read-only door

### 3.1 `_open_readonly`

```python
if not config.STORE_PATH.exists():
    raise FileNotFoundError(...)
conn = sqlite3.connect(f"file:{config.STORE_PATH}?mode=ro", uri=True, check_same_thread=False)
conn.row_factory = sqlite3.Row
```

**What `mode=ro` buys you.** The URI form (`uri=True`) lets you pass SQLite flags in the filename. `mode=ro` opens with `SQLITE_OPEN_READONLY`, so any `INSERT`/`UPDATE`/`DELETE`/`CREATE` raises `sqlite3.OperationalError: attempt to write a readonly database`. This is invariant 3 enforced by the OS-level file handle rather than by convention — much stronger than "we just don't write." Note it does **not** create the file if absent (unlike the default mode), which is why the explicit `.exists()` check gives a better error than SQLite's opaque "unable to open database file."

**The URI string is not escaped.** This is a latent portability bug:

```python
f"file:{config.STORE_PATH}?mode=ro"
```

A path containing a space, `?`, `#`, or `%` produces a malformed URI. `#` is the worst case — SQLite treats it as a fragment delimiter and silently truncates the path. On Windows, backslashes need conversion too. This matters specifically because you're heading for Azure, where deployment paths are not always as tidy as local ones. The fix:

```python
from urllib.parse import quote
uri = f"file:{quote(str(config.STORE_PATH))}?mode=ro"
```

`Path.as_uri()` also works, and handles the Windows drive-letter form, but it emits `file:///C:/...` which SQLite accepts.

**`check_same_thread=False`** deserves real explanation, because it's a safety guard being deliberately switched off. By default Python's `sqlite3` raises if a connection object is used from a thread other than the one that created it — a guard against races on a shared connection. You need it off here because FastAPI runs sync dependencies and sync routes in an `anyio` worker threadpool, and **there is no guarantee that the generator's setup, the route body, and the generator's teardown all land on the same worker thread.** With the guard on, you'd get intermittent `ProgrammingError`s under load.

Turning it off is safe *given the surrounding design*: each connection is created for exactly one request, handed to exactly one route, and closed. It is never stored in a module-level variable, never shared, never touched concurrently. That "given" is the whole justification — the day someone caches a connection at module scope to save the open cost, this flag stops being a harmless accommodation and becomes a data race.

**`row_factory = sqlite3.Row`** makes rows behave like mappings (`row["month_num"]`) as well as tuples. The docstring correctly notes `queries.py` doesn't depend on it — `_rows` builds dicts from `cursor.description`. Since `Row` supports `.keys()`, `_rows` could be simplified to `[dict(r) for r in cur.fetchall()]`, but only if `row_factory` is guaranteed set. The current version works with or without it, which is arguably the better coupling: `queries.py` stays independent of how the connection was configured.

**A `stat()` per request.** The `.exists()` check is one syscall — negligible in absolute terms, but it's also a time-of-check-to-time-of-use window and, more importantly, it's checking a *startup* condition on the *request* path. A missing store means the app should never have booted. See §10.4.

### 3.2 `connect()` and `get_db()`

```python
@contextmanager
def connect() -> Iterator[sqlite3.Connection]:
    conn = _open_readonly()
    try:
        yield conn
    finally:
        conn.close()

def get_db() -> Iterator[sqlite3.Connection]:
    with connect() as conn:
        yield conn
```

This pairing is the nicest thing in the file. `connect()` is a standard context manager for scripts and tests; `get_db()` is the FastAPI dependency. `get_db` delegates rather than duplicating, so **both entry points share one lifecycle** — one place to add a PRAGMA, one place to change how connections open, and no chance of the script path and the request path drifting apart.

The `finally` is what makes it airtight: an exception anywhere in the route propagates back into the generator at the `yield`, and the connection still closes. Without `try/finally`, a 500 would leak a file handle per request.

Two small notes:

- The docstring example says `Depends(store.get_conn)` — that function doesn't exist; it's `get_db`. Doc drift, worth fixing before it confuses you in three months.
- `get_db` has no `try/except`. If `_open_readonly` raises (missing store, corrupt file, locked), FastAPI turns it into a bare 500 with a traceback in the logs. Mapping storage failures to 503 is a better signal to the frontend — 500 says "bug," 503 says "dependency down."

---

## 4. Deep dive: FastAPI's dependency lifecycle

You've used `Depends` correctly, so this section is about knowing *why* it works, which is what lets you extend it.

**A dependency is just a callable.** `Depends(store.get_db)` tells FastAPI: before running the route, call `get_db()`; whatever it produces, pass as the `db` argument. The type annotation `db: sqlite3.Connection` is for you and your editor — FastAPI keys off `Depends`, not the type.

**Generator dependencies get setup/teardown semantics.** Because `get_db` is a generator function, FastAPI treats the code before `yield` as setup and after as teardown. Mechanically it wraps the generator in a context manager and registers it on an `AsyncExitStack` scoped to the request. When the request finishes — success, exception, or client disconnect — the stack unwinds and every teardown runs, in reverse order.

**Where it runs.** Sync generator dependencies and sync route functions are both dispatched to the threadpool, so a slow query occupies a worker thread rather than blocking the event loop. This is the design choice `api.py`'s docstring describes, and for SQLite it's the right one: reads are CPU-and-disk work, not awaitable I/O, so there's nothing for `async` to overlap. An `async def` route doing a blocking `sqlite3` call would be strictly worse — it would stall the entire event loop.

**Caching within a request.** By default FastAPI caches a dependency's result per request, so if five dependencies all depended on `get_db`, you'd get one connection, not five. (`Depends(f, use_cache=False)` opts out.) Not currently exercised, but it's the mechanism that makes dependency composition cheap.

**Overriding for tests.** `app.dependency_overrides[store.get_db] = ...` swaps the dependency without touching route code. This is the single most useful consequence of injecting the connection rather than opening it inside each route — see §13.

**What to add next.** The natural extensions all slot in as dependencies rather than route edits: an auth dependency that resolves a caller to a `client_id`; a dependency that validates `orchestration_key` belongs to that client; a request-ID dependency for logging. That's the payoff of the current shape.

---

## 5. Deep dive: connection-per-request, and what it actually costs

Opening SQLite is cheap — no network handshake, no auth, just a file open and a schema read. So connection-per-request is the right default. But there is one real cost, and it's invisible until you look for it.

**Each connection has its own page cache.** SQLite caches recently-read B-tree pages per connection (`PRAGMA cache_size`, default ~2MB). Close the connection and the cache dies with it. So **every request starts with a cold cache** and re-reads pages from the filesystem. The OS filesystem cache absorbs much of this, which is why it doesn't show up at low traffic — but it means you're paying syscall and page-decode cost on every request that a long-lived connection would pay once.

Three ways to address it, in increasing order of commitment:

**1. Memory-mapped I/O.** The cheapest real win:

```python
conn.execute("PRAGMA mmap_size = 268435456")   # 256MB
```

SQLite maps the database file into the process address space and reads pages directly from the OS page cache — which **is** shared across connections and survives connection close. Read-only mmap is safe, and this largely neutralizes the cold-cache problem without changing your connection model at all. Add it in `_open_readonly` and it applies to both entry points.

**2. `immutable=1`.** If the store genuinely does not change while serving:

```python
f"file:{path}?immutable=1"
```

SQLite then skips all locking and change-counter checks. Measurably faster, and it also sidesteps the WAL gotcha below. But it's a *promise*, and if a build overwrites the file while a connection is open you get undefined behavior — potentially wrong data rather than an error. Only take this with the atomic-swap deployment described below.

**3. A connection pool.** Keep N connections alive and hand them out. Real complexity (lifecycle, thread affinity, health checks) for a benefit you should measure first. Don't do this until `mmap_size` has proven insufficient.

**The WAL gotcha.** Your build uses SQLite's default journal mode (delete), so this doesn't bite today — but if you ever switch the store to WAL for build speed, `mode=ro` will fail to open it unless the `-shm` and `-wal` sidecar files are present and readable. `immutable=1` is the usual escape. Worth knowing before it surprises you.

**Atomic swap on rebuild.** Related, and important for the same reasons `_preflight` matters in the loader: if the build writes in place while the API is serving, readers can observe a torn store. The clean pattern is to build to `store.db.new`, then `os.replace(new, live)` — atomic on POSIX, so any connection opened before the swap keeps reading the old inode until it closes, and any opened after gets the new one. No downtime, no torn reads, and it makes `immutable=1` genuinely safe.

---

## 6. `queries.py` — the read surface

### 6.1 `_rows`

```python
def _rows(cur: sqlite3.Cursor) -> list[dict]:
    cols = [c[0] for c in cur.description]
    return [dict(zip(cols, r)) for r in cur.fetchall()]
```

`cursor.description` is a 7-tuple per column; only `[0]` (name) is populated by `sqlite3`. The names come from the `AS` aliases in the SQL, which is why every query aliases explicitly — those aliases *are* the API field names. Renaming an alias is a breaking wire change even though nothing in Python moved.

`fetchall()` materializes the whole result. Fine at your cardinality (one orchestration's months, one month's categories). If a result ever gets large, `fetchmany` in a loop or a generator would bound memory — but don't restructure for that until it's real.

### 6.2 `list_orchestrations` — two bugs

```sql
SELECT DISTINCT
    s.orchestration_key, s.client_id, c.client_name,
    s.forecast_year, s.orchestration_revision_name AS revision_name
FROM cash_flow_streams AS s
LEFT JOIN clients AS c ON c.client_id = s.client_id
ORDER BY s.client_id, s.forecast_year, s.orchestration_revision_name
```

**Bug 1: this full-scans the fact table to produce a handful of rows.** `SELECT DISTINCT` over every row in `cash_flow_streams` — hundreds of thousands or millions — plus a temp B-tree to dedup, plus a join, on **every page load of the app**. No index helps: the four selected columns aren't a leftmost prefix of anything, so there's nothing to seek into and nothing covering.

The `versions` table exists in the store for exactly this purpose and isn't being used. The semantics you actually want — *"revisions that have servable data"* — are expressible without touching the fact table's rows:

```sql
SELECT v.orchestration_key, v.client_id, c.client_name,
       v.forecast_year, v.orchestration_revision_name AS revision_name,
       v.orchestration_revision_num AS revision_num
FROM versions AS v
LEFT JOIN clients AS c ON c.client_id = v.client_id
WHERE EXISTS (SELECT 1 FROM cash_flow_streams s
              WHERE s.orchestration_key = v.orchestration_key)
ORDER BY v.client_id, v.forecast_year, v.orchestration_revision_num
```

The `EXISTS` is an indexed point-probe per version row that short-circuits on the first hit — it never scans. Adjust the column names to whatever `versions` actually carries; the shape is the point.

**Bug 2: `ORDER BY` on a revision *name* is a string sort.** Given keys like `c0001_2026_3-pre-harvest`, your revision names are of the form `1-pre-seeding`, `3-pre-harvest`. Lexicographically, `"10-post-harvest" < "2-pre-seeding"` — so the tenth revision sorts second. This is the backend twin of the hardcoded `REV_ORDER` array in `ForecastExplorer.jsx`: the ordering doesn't exist as data, so it has to be faked somewhere, and both places get it wrong differently.

Fix both at once by exporting `orchestration_revision_num` and sorting on it, then adding it to the `Orchestration` model so the frontend can sort too and delete its literal array.

**A third, subtler point.** `DISTINCT` combined with `LEFT JOIN clients` means that if `clients` ever holds two rows for one `client_id`, the join fans out and `DISTINCT` silently collapses it again. The picker looks fine while the same duplicate corrupts money elsewhere. `DISTINCT` here is load-bearing in a way you didn't intend — another argument for the `UNIQUE` index on `clients.client_id`.

### 6.3 `net_cash_flow`

```sql
SELECT month_num,
       MAX(month) AS month,
       ROUND(SUM(value), 2) AS net,
       ROUND(SUM(CASE WHEN value > 0 THEN value ELSE 0 END), 2) AS inflow,
       ROUND(SUM(CASE WHEN value < 0 THEN value ELSE 0 END), 2) AS outflow
FROM cash_flow_streams
WHERE orchestration_key = ?
GROUP BY month_num
ORDER BY month_num
```

The conditional-aggregate pattern (`SUM(CASE WHEN ...)`) computes three aggregates in one pass — much better than three queries or three subqueries. Correct and idiomatic.

**`MAX(month)` is a "pick any" aggregate.** `month` isn't in the `GROUP BY`, so you need *some* aggregate to make it legal, and `MAX` is the conventional choice. It encodes an unstated assumption: **exactly one `month` label per `month_num` per orchestration.** If a bad export ever produced two labels for month 4, this silently picks the alphabetically-later one and the chart mislabels an axis tick with no error anywhere. That's a one-line contract assertion:

```sql
SELECT orchestration_key, month_num, COUNT(DISTINCT month) c
FROM cash_flow_streams GROUP BY 1,2 HAVING c > 1;   -- must be empty
```

**Rounding**: `ROUND(SUM(value), 2)` rounds once, after summing — correct. Compare with `month_breakdown`, which does not (§8).

**Index**: `WHERE orchestration_key = ? GROUP BY month_num`. The composite index is `(orchestration_key, month, source)` — note `month`, not `month_num`. So the index gives the seek on `orchestration_key` and then nothing: `GROUP BY month_num` needs a temp B-tree sort. See §7.

### 6.4 `month_breakdown`

```python
categories = _rows(cur)
# return categories                                    ← dead code, delete
inflow = sum(c["amount"] for c in categories if c["amount"] > 0)
outflow = sum(c["amount"] for c in categories if c["amount"] < 0)
return {..., "net": round(inflow + outflow, 2), "categories": categories}
```

Computing the totals in Python from the already-fetched rows avoids a second query — good instinct. But it sums **already-rounded** category amounts, which is where the divergence in §8 comes from.

`ORDER BY amount DESC` sorts inflows first, outflows last. Reasonable, but note the consequence: **category order changes month to month**, since it's value-ordered, not category-ordered. If `ForecastExplorer.jsx` assigns colors or legend positions by array index rather than by category name, the same category changes color as you click through months. Order by category name (or an exported `category_sort_order`) if you want visual stability, or make sure the frontend maps category → color explicitly.

### 6.5 `category_sources`

```sql
SELECT output_orchestration_key, output_name, source, crop_sys AS crop,
       revision_name AS chosen_revision,
       ROUND(total_value, 2) AS annualized,
       perc AS allocation_pct,
       ROUND(value, 2) AS allocated
FROM cash_flow_streams
WHERE orchestration_key = ? AND month_num = ? AND category = ?
ORDER BY ABS(value) DESC
```

**Two revision columns, and the distinction matters.** The fact table carries both `orchestration_revision_name` (used by the picker — which revision of the *whole forecast* this is) and `revision_name` (used here as `chosen_revision` — which revision of *this specific output* was resolved into that forecast). The second is the visible payoff of your resolution rule: within one orchestration revision, different sources can sit at different output revisions. Worth naming clearly in the docstring, because the two column names are similar enough to conflate and the meanings are quite different.

**The docstring states an invariant the code depends on:** *"The month filter guarantees one row per key, so `total_value` is read directly (no risk of summing it 12x)."* That's correct given the declared grain — `(orchestration_key, output_orchestration_key, month, source)` — but nothing enforces it. If a grain violation ever produced two rows for the same key within one month, the UI shows the same output twice with the same annualized figure, and the annualized column becomes double-counted the moment anyone sums it. This is precisely the grain-uniqueness assertion from the data contract; this query is why it's the highest-priority check on that list.

`ORDER BY ABS(value) DESC` can't use an index (expression on a column), so it sorts. Fine — the row count within one month+category is small.

### 6.6 `source_lineage`

**`LIMIT 1` on the header masks duplicates:**

```sql
SELECT ... FROM output WHERE output_orchestration_key = ? LIMIT 1
```

With no `ORDER BY`, `LIMIT 1` returns whichever row the scan reaches first — arbitrary and not stable across rebuilds. If `output` ever has two rows for a key, Level 3 shows one of them and you never learn the other exists. `LIMIT 1` is doing duplicate-suppression disguised as a performance hint. The honest version is a `UNIQUE` index on `output.output_orchestration_key` (making duplicates impossible) and no `LIMIT` — or fetch both and raise if you get two.

**Two different "annualized" numbers.** Level 2 reports `annualized` from `cash_flow_streams.total_value`. Level 3 reports `annualized` from `output.value`. The user drills from one to the other expecting the number to persist. Nothing in the code guarantees they agree — that's a cross-table reconciliation the data contract needs to assert:

```sql
SELECT s.output_orchestration_key
FROM (SELECT DISTINCT output_orchestration_key, total_value FROM cash_flow_streams) s
JOIN output o USING (output_orchestration_key)
WHERE ABS(s.total_value - o.value) > 0.005;   -- must be empty
```

**Your amortization case lands exactly here.** Amortization keys exist in `input` and `input_output_linkage` but not in `output` and not in `cash_flow_streams`. So `source_lineage("c0001_2026_amortization_1-pre-seeding")` returns `{"output": None, "inputs": [...]}`. The `api.py` guard is `if result["output"] is None and not result["inputs"]` — an **`and`**, so inputs-present-output-missing returns **200 with `output: null`**. `Lineage.output` is nullable, so it serializes cleanly, and the frontend receives a payload it must guard.

That's not reachable by clicking, since the UI only reaches Level 3 from a Level 2 row and Level 2 comes from `cash_flow_streams`. But it *is* reachable by URL, and your keys are guessable (§11). Decide deliberately whether that's a 404 or a documented partial-lineage response — right now it's neither, it's an accident of the boolean operator.

### 6.7 The `__main__` smoke test

Genuinely useful — it proves the whole drill path with no FastAPI and no network, which is the payoff of pure query functions. Three problems:

```python
conn = sqlite3.connect(config.STORE_PATH)          # ← writable!
```

This bypasses `store.py` entirely and opens a **read-write** connection, violating invariant 3 in the one file that documents it most carefully. Use the context manager that exists for exactly this:

```python
from growlytics_platform.serving_system.utils import store
with store.connect() as conn:
    ...
```

Also: `c0001_2026_3-pre-harvest` is hardcoded, so the smoke test breaks on any dataset that lacks it — pull the first key from `list_orchestrations` instead. And `category` is reassigned from a dict to a string mid-block, which makes the code hard to follow; use distinct names. The commented-out `list_clients` block is dead — delete it.

---

## 7. Deep dive: the index/query mismatch

This is the most actionable finding in the review, and it connects directly to the leftmost-prefix rule from the loader walkthrough.

`build_serving_store.py` indexes `cash_flow_streams` on:

```python
("orchestration_key", "month", "source")   # "the level 0 -> 1 drill path"
("output_orchestration_key",)
("client_id",)
("orchestration_key",)
```

But the three money queries filter and group on:

| Query | Predicate | Group/order |
|---|---|---|
| `net_cash_flow` | `orchestration_key` | `GROUP BY month_num` |
| `month_breakdown` | `orchestration_key`, `month_num` | `GROUP BY category` |
| `category_sources` | `orchestration_key`, `month_num`, `category` | `ORDER BY ABS(value)` |

**Neither `month_num` nor `category` is in any index.** The composite index's second and third columns (`month`, `source`) are never used as predicates by any query in this file. So the composite index does no more work than the plain `("orchestration_key",)` index sitting next to it: SQLite seeks on the first column and then filters every candidate row for `month_num` and `category` individually, and sorts for every `GROUP BY`.

The index that matches the actual access pattern:

```sql
CREATE INDEX idx_cash_flow_streams__drill
  ON cash_flow_streams (orchestration_key, month_num, category);
```

All three queries become leftmost prefixes of it — 1, 2, and 3 columns respectively — and `GROUP BY month_num` and `GROUP BY category` both get their grouping order for free from the index, eliminating the temp B-tree sorts.

Verify rather than trust:

```sql
EXPLAIN QUERY PLAN
SELECT category, ROUND(SUM(value),2) FROM cash_flow_streams
WHERE orchestration_key = 'c0001_2026_3-pre-harvest' AND month_num = 4
GROUP BY category;
```

Before: `SEARCH ... USING INDEX idx_..._orchestration_key` plus `USE TEMP B-TREE FOR GROUP BY`. After: `SEARCH ... USING INDEX idx_cash_flow_streams__drill (orchestration_key=? AND month_num=?)` and no temp B-tree.

Two follow-ons. If `month` and `source` really aren't filtered anywhere, the current composite index is pure build-time and disk cost — replace it rather than adding alongside. And if Level 0 becomes a bottleneck, appending `value` makes it covering:

```sql
CREATE INDEX idx_cash_flow_streams__drill_cov
  ON cash_flow_streams (orchestration_key, month_num, category, value);
```

Measure before adding the covering column; it duplicates the money column on disk.

---

## 8. Deep dive: the rounding divergence

**Two endpoints report different values for the same month's net.**

`net_cash_flow` (the chart) rounds once, at the end:

```sql
ROUND(SUM(value), 2) AS net
```

`month_breakdown` (the drill panel) rounds each category first, then sums the rounded values in Python:

```python
categories = [... ROUND(SUM(value), 2) AS amount ...]
inflow  = sum(c["amount"] for c in categories if c["amount"] > 0)
outflow = sum(c["amount"] for c in categories if c["amount"] < 0)
net = round(inflow + outflow, 2)
```

`round(Σ xᵢ) ≠ Σ round(xᵢ)`. Two categories whose raw sums are `1000.125` and `2000.125` round to `1000.13` and `2000.13`, totalling `2000.26`; the raw total `3000.25` rounds to `3000.25`. A one-cent divergence, and with N categories the bound is N × 0.005.

The user-visible symptom: the chart tooltip says one number, the panel that opens on click says another. Both are "correct." In a financial product aimed at farm clients, that's the kind of discrepancy that costs you trust disproportionately to its magnitude — a client who finds a penny they can't explain starts checking everything.

**The narrow fix** — let SQL compute the totals from raw values in the same pass:

```sql
SELECT category,
       ROUND(SUM(value), 2) AS amount,
       ROUND(SUM(SUM(value)) OVER (), 2) AS net_all      -- window over groups
FROM cash_flow_streams
WHERE orchestration_key = ? AND month_num = ?
GROUP BY category
```

Or simply run the same conditional-aggregate expression `net_cash_flow` uses, as a second small query, so both endpoints derive `net` by identical arithmetic. Slightly less elegant, but it makes agreement structural rather than coincidental.

**The real fix** — stop representing money as float. Every `value` is a `REAL` in SQLite, a `float` in Python, and a JSON number on the wire. `ROUND` papers over binary-representation error rather than eliminating it, and `Decimal` in Python doesn't help if the value arrived as a float. The durable answer is integer cents (or `Decimal` end to end), rounding exactly once at the presentation boundary. That's a real refactor touching the engine, the store, the models, and the frontend — but it's the difference between reconciliation checks you can set to zero tolerance and reconciliation checks you have to fuzz.

**Meanwhile, assert the agreement** so a regression can't slip in — a Pydantic validator (§9.3) catches it at the boundary on every response.

---

## 9. `schema.py` — the contract

### 9.1 What `response_model` actually does

This is the mechanism to internalize, because its failure modes are asymmetric:

| Situation | Behavior |
|---|---|
| Query returns a column the model doesn't declare | **Silently dropped** from the response |
| Query omits a column the model declares as required | `ResponseValidationError` → **HTTP 500** |
| Query returns `"2026"` for an `int` field | Coerced to `2026` (Pydantic lax mode) |
| Query returns `2026.0` for an `int` field | Coerced to `2026` (integral float is accepted) |
| Query returns `2026.5` for an `int` field | 500 |
| Query returns `None` for a non-`Optional` field | 500 |

So: **an added column is invisible, a removed column is loud.** If you rename `crop_sys` in the data system and update the SQL alias but not the model, the field vanishes from the API and the frontend renders blanks — no error anywhere in Python. That's the failure mode to watch, and the reason it's worth keeping the models genuinely in lockstep with the aliases rather than treating them as loose documentation.

The `2026.0 → 2026` row is worth noting alongside the dtype-pinning recommendation from the data contract: pandas upcasting `forecast_year` to float would pass silently here, so this layer won't catch it for you.

### 9.2 Field-by-field notes

**`Orchestration.client_name: str | None = None`** — permissively nullable, which is right today given `LEFT JOIN clients`. Once the FK edge `cash_flow_streams.client_id -> clients.client_id` is enforced upstream, this can tighten to `str` and the `LEFT JOIN` can become an inner `JOIN`. A nullable field the data guarantees non-null is a guard the frontend has to write for no reason.

**`Orchestration` has no `revision_num`** — so even if the backend sorted correctly, the frontend couldn't re-sort after a client-side filter. Adding it is what lets `REV_ORDER` be deleted.

**`LineageInput.last_modified: str`** — a timestamp typed as string passes through unvalidated, so a malformed one reaches the browser intact. Typing it `datetime` makes Pydantic parse and normalize to ISO-8601, turning a display bug into a 500 at the boundary. Worth it for a field the UI formats.

**`SourceRow.allocation_pct: float`** — unbounded. This is exactly the allocation-domain rule from the contract doc, and it's enforceable in one line:

```python
from pydantic import Field
allocation_pct: float = Field(ge=0, le=1)
```

Confirm your convention first — if `perc` is stored as `0–100` rather than `0–1`, use `le=100`. Either way, an out-of-range allocation becomes a 500 instead of a nonsense percentage in the UI.

**`Lineage.output: LineageOutput | None = None`** — the nullable discussed in §6.6. Legitimate given the amortization design, but it means the frontend *must* guard, and that guard needs to exist deliberately rather than by luck.

### 9.3 Validators as the last line of defense

Pydantic v2 lets you assert cross-field invariants at the serialization boundary — the last point before data leaves your control:

```python
from pydantic import model_validator

class MonthBreakdown(BaseModel):
    ...
    @model_validator(mode="after")
    def totals_are_consistent(self):
        if abs(self.inflow + self.outflow - self.net) > 0.011:
            raise ValueError(
                f"net {self.net} != inflow {self.inflow} + outflow {self.outflow}"
            )
        if abs(sum(c.amount for c in self.categories) - self.net) > 0.011:
            raise ValueError("categories do not sum to net")
        return self
```

Cheap, runs on every response, and it converts the §8 class of bug from *"a client notices a penny"* into *"a test fails."* Tune the tolerance to your category count; set it to zero once money is integer cents.

---

## 10. `api.py` — the HTTP surface

### 10.1 The shape is right

Each route resolves the connection, calls one query function, and returns it. No SQL, no connection management, no business logic. That's what "thin" should mean, and it's why the file is short enough to review at a glance.

**Every query uses `?` placeholders.** Worth stating explicitly: user input reaches SQL only as bound *values*, never interpolated into SQL text, so there's no injection surface here. Contrast with `build_serving_store.py`, where table and column *identifiers* had to be interpolated because placeholders can't carry identifiers — and that was safe only because those identifiers come from a literal dict in the source. Values: always bind. Identifiers: interpolate, and validate the source.

### 10.2 Inconsistent empty-result semantics

Three of four data routes 404 on empty; `sources` returns `[]`:

| Route | Empty behavior |
|---|---|
| `/forecast/{k}/cashflow` | 404 |
| `/forecast/{k}/month/{n}` | 404 |
| `/forecast/{k}/month/{n}/sources` | **`[]`** |
| `/forecast/lineage/{k}` | 404 only if output **and** inputs are both empty |

Pick a convention and apply it uniformly, because the frontend has to branch differently for each. The defensible REST reading: a **collection** endpoint returns `200 []` (the collection exists and is empty), a **single-resource** endpoint returns 404 (the resource doesn't exist). By that rule `cashflow` and `sources` are collections → `200 []`, `month` and `lineage` are resources → 404. What matters more than which rule you pick is that all four follow it.

### 10.3 Unvalidated `category`

```python
category: str,     # required query param, any string accepted
```

FastAPI enforces presence but not value, so a typo returns `200 []` — indistinguishable from a real empty category. Since categories are a closed set, make it one:

```python
from typing import Literal
category: Literal["Revenue", "Input Cost", "Operating Cost", ...]
```

Now a typo is a 422 with the valid values listed, and the enum shows up in the OpenAPI docs. Same for `month_num` bounds — `month_num: int = Path(ge=1, le=12)` if the horizon is a calendar year.

This is the API-layer half of the enum-domain check in the data contract: the contract guarantees the data contains only known categories; this guarantees the API only accepts known categories. Both halves are needed.

### 10.4 `/health` cannot fail

```python
@app.get("/health")
def health() -> dict:
    return {"status": "ok"}
```

This returns 200 whether or not the store exists, is readable, or contains data. A load balancer or Azure health probe will happily route traffic to an instance that 500s on every real request. Make it exercise the actual dependency:

```python
@app.get("/health")
def health(db: sqlite3.Connection = Depends(store.get_db)) -> dict:
    n = db.execute("SELECT COUNT(*) FROM cash_flow_streams").fetchone()[0]
    if n == 0:
        raise HTTPException(503, "store is empty")
    return {"status": "ok", "rows": n}
```

**And fail fast at startup.** A missing store is a boot-time condition, not a per-request one. With a lifespan handler the process refuses to start rather than serving 500s:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    with store.connect() as conn:
        conn.execute("SELECT 1 FROM cash_flow_streams LIMIT 1")
    yield

app = FastAPI(..., lifespan=lifespan)
```

Same all-or-nothing instinct as `_preflight()` in the loader, applied to the serving process: verify the world is sane before accepting work.

### 10.5 Storage errors surface as 500s

Nothing catches `sqlite3.OperationalError`. A store swapped mid-read, a permissions change, a corrupt page — all become a bare 500 with a traceback. A small exception handler distinguishes "our bug" from "dependency unavailable":

```python
@app.exception_handler(sqlite3.OperationalError)
async def sqlite_unavailable(request, exc):
    return JSONResponse({"detail": "store unavailable"}, status_code=503)
```

### 10.6 Module path drift

The docstrings say `serving_system.serve.store` / `serve.schema`, while the imports say `utils.store` / `utils.schema`. `queries.py` and `store.py` have the same mismatch in their own headers. Harmless today, actively misleading in six months. Pick the real layout and fix all four headers.

---

## 11. Security posture

Worth being blunt: **the API is currently unauthenticated and unscoped.** For a local R&D proof-of-concept that's a reasonable place to be. Before it reaches Azure, these are the items that matter.

**CORS is not authorization.** `allow_origins=config.CORS_ORIGINS` is enforced by *browsers*, on *cross-origin* requests, and it restricts which web pages may read responses. It does nothing against `curl`, a script, or any non-browser client. Whatever the deployed API can serve, anyone who can reach it can read. `allow_methods=["GET"]` is a good belt-and-braces touch given the read-only store, but it's the same kind of control — advisory to browsers, not a security boundary.

**Every endpoint is cross-client readable, and the keys are guessable.** `/forecast/lineage/{output_orchestration_key}` is documented as *"global on the output key"* — no client filter at all. And your keys are structured and predictable: `c0001_2026_amortization_1-pre-seeding` differs from client 2's by one digit. So a caller holding one client's key can enumerate another's by incrementing it. The same applies to `/forecast/{orchestration_key}/...`, since `orchestration_key` embeds the client ID, and `/orchestrations` simply lists every client in the system.

This is the client-scoping item I flagged as BLOCK in the data contract, showing up at the API layer. The fix is a dependency, which is exactly what the current shape makes easy:

```python
def current_client(token: str = Depends(oauth2_scheme)) -> str:
    ...  # resolve caller -> client_id

def scoped_key(orchestration_key: str, client: str = Depends(current_client)) -> str:
    if not orchestration_key.startswith(client):     # or look it up properly
        raise HTTPException(404)                     # 404, not 403 — don't confirm existence
    return orchestration_key
```

Two details worth getting right: return **404 rather than 403** for a key belonging to another client, since 403 confirms the key exists and hands an attacker an enumeration oracle. And **scope by lookup, not by string prefix** — deriving authorization from key structure means a key-format change silently becomes an authorization change.

For lineage specifically, resolve the output key back to its client through `cash_flow_streams`/`output` and compare, rather than trusting the key's shape.

**Smaller items.** Error messages echo the requested key back (`f"No forecast for {orchestration_key!r}"`) — fine for JSON, but it does confirm-or-deny existence to an unauthenticated caller, which is the same enumeration concern. There's no rate limiting, so `/orchestrations`' full table scan (§6.2) is a cheap way to load the server. And there's no request logging, so you'd have no record of any of this having happened.

---

## 12. Findings, ranked

| # | Finding | Where | Severity |
|---|---|---|---|
| 1 | No authentication or client scoping; keys are enumerable | `api.py`, all routes | **Blocker before deploy** |
| 2 | Chart and drill panel disagree on `net` (round-then-sum) | `queries.month_breakdown` | **High** — user-visible |
| 3 | Indexes don't match queries; `month_num`/`category` unindexed | build spec vs `queries.py` | **High** — perf |
| 4 | Picker full-scans the fact table on every page load | `list_orchestrations` | **High** — perf |
| 5 | `ORDER BY` revision *name* sorts `10-` before `2-` | `list_orchestrations` | **High** — wrong output |
| 6 | `LIMIT 1` silently hides duplicate `output` rows | `source_lineage` | Medium |
| 7 | Level 2 and Level 3 `annualized` come from different tables | `category_sources` / `source_lineage` | Medium |
| 8 | `/health` passes with no store | `api.py` | Medium |
| 9 | No startup check; missing store = per-request 500s | `api.py` | Medium |
| 10 | Inconsistent empty-result semantics across four routes | `api.py` | Medium |
| 11 | Smoke test opens a **writable** connection | `queries.__main__` | Medium — breaks invariant 3 |
| 12 | URI string unescaped (spaces, `#`, Windows paths) | `store._open_readonly` | Medium — bites on deploy |
| 13 | `category` accepts any string; typo returns `200 []` | `api.sources` | Low |
| 14 | `sqlite3.OperationalError` → bare 500 | `api.py` | Low |
| 15 | Cold page cache per request (no `mmap_size`) | `store.py` | Low — until traffic |
| 16 | `MAX(month)` assumes one label per `month_num` | `net_cash_flow` | Low — assert upstream |
| 17 | Category order varies month to month | `month_breakdown` | Low — UI stability |
| 18 | Module paths in docstrings don't match imports | all four | Low |
| 19 | Dead code: `# return categories`, `list_clients` block | `queries.py` | Cosmetic |
| 20 | Docstring references nonexistent `store.get_conn` | `store.py` | Cosmetic |

Suggested order of attack: **5, 2, 3, 4** first — all small diffs, all either wrong output or a large constant-factor win. Then **11, 8, 9, 12** as a hygiene pass. Then **1** as its own deliberate piece of work before anything leaves localhost. Then the rest.

---

## 13. How to test each layer

The purity of `queries.py` is what makes this cheap. Don't lose it.

**Query functions — no HTTP, no files:**

```python
import sqlite3, pytest
from growlytics_platform.serving_system.serve import queries

@pytest.fixture
def conn():
    c = sqlite3.connect(":memory:")
    c.executescript(open("tests/fixtures/schema.sql").read())
    c.executescript(open("tests/fixtures/seed.sql").read())
    return c

def test_net_equals_inflow_plus_outflow(conn):
    for row in queries.net_cash_flow(conn, "c0001_2026_3-pre-harvest"):
        assert abs(row["inflow"] + row["outflow"] - row["net"]) < 0.011

def test_chart_and_panel_agree(conn):        # regression test for §8
    key = "c0001_2026_3-pre-harvest"
    chart = {r["month_num"]: r["net"] for r in queries.net_cash_flow(conn, key)}
    for m, net in chart.items():
        assert abs(queries.month_breakdown(conn, key, m)["net"] - net) < 0.011
```

An in-memory database makes these run in milliseconds, so they can run on every save.

**Routes — swap the dependency, keep the app:**

```python
from fastapi.testclient import TestClient
from growlytics_platform.serving_system.serve.api import app
from growlytics_platform.serving_system.utils import store

def test_cashflow_404s_on_unknown_key(conn):
    app.dependency_overrides[store.get_db] = lambda: conn
    try:
        assert TestClient(app).get("/forecast/nope/cashflow").status_code == 404
    finally:
        app.dependency_overrides.clear()
```

Always clear the overrides in a `finally` — `dependency_overrides` is module-level state on `app`, and a leaked override silently contaminates every later test.

**The response models test themselves.** Every `TestClient` call that returns 200 has already passed Pydantic validation, so route tests double as contract tests. Add the §9.3 validators and they double as arithmetic tests too.

**Fixtures should include the pathological rows**, not just the happy path: a duplicate `output` key (does #6 surface?), an amortization key with linkage but no output (what does lineage return?), a month whose categories don't cleanly sum (does #2 fire?). Checks that only ever see clean data aren't checks.

---

## 14. Cheat sheet

**Layering**

- `store` = connection lifecycle only. `queries` = SQL + grain, pure. `schema` = wire contract. `api` = routing + status codes.
- `queries.py` takes a connection and returns plain Python. Keep it that way; it's why testing is cheap.
- Money reads only from `cash_flow_streams`. Linkage is joined only by Level 3, which lists rather than sums.

**FastAPI**

- `Depends(f)` = call `f`, inject result. Generator dependencies get setup/teardown; teardown runs after the response is sent, even on exception.
- Sync routes and sync dependencies run in a threadpool — hence `check_same_thread=False`, and hence never share a connection.
- Dependencies are cached per request by default.
- `app.dependency_overrides[dep] = fake` is the test seam. Clear it afterward.
- Inbound validation failure → 422 (client's fault). Outbound → 500 (yours).
- `response_model` **drops** undeclared fields silently and **500s** on missing ones. Renames that add are invisible; renames that remove are loud.

**SQLite on the request path**

- `mode=ro` makes writes impossible at the file-handle level. It won't create the file. Escape the URI.
- Page cache is per connection, so connection-per-request means a cold cache each time. `PRAGMA mmap_size` shares the OS page cache and largely fixes it.
- `immutable=1` is faster but promises the file won't change — only with atomic-swap deploys.
- WAL + `mode=ro` needs the `-shm`/`-wal` sidecars. Default journal mode avoids it.
- Rebuild to a temp path, then `os.replace()`. Atomic, no torn reads.

**SQL correctness**

- Bind values with `?`. Interpolate identifiers only from trusted literals.
- `ROUND(SUM(x))` ≠ `SUM(ROUND(x))`. Round once, as late as possible, by the same arithmetic in every endpoint.
- Bare columns under `GROUP BY` need an aggregate; `MAX(col)` means "pick any" and encodes an unasserted assumption.
- `LIMIT 1` without `ORDER BY` returns an arbitrary row — it hides duplicates rather than resolving them.
- Index the columns you actually filter and group on. `EXPLAIN QUERY PLAN`, then believe it.
- Sort on numbers, not on names that contain numbers.

**Before deploying**

- Authentication, then per-client scoping as a dependency. 404 not 403. Scope by lookup, not by key prefix.
- CORS is a browser convention, not an authorization boundary.
- `/health` must be able to fail. Startup must be able to refuse.
- Rate limiting, request logging.


# Code Appendix

## `store.py`

```py
"""
src.growlytics_platform.serving_system.serve.store — the read-only door to the store.

The build-time loader is the single writer (Invariant 3). Everything on the
request path goes through here, and here we open SQLite in read-only URI mode so
the API *cannot* mutate the store even by accident. One short-lived connection
is handed to each request via the FastAPI dependency and closed when the request
ends.

"""

from __future__ import annotations

import sqlite3
from collections.abc import Iterator
from contextlib import contextmanager

from growlytics_platform.serving_system.utils import config


def _open_readonly() -> sqlite3.Connection:
    """Open the store read-only. Raises if the store file does not exist yet
    (i.e. the loader has not been run)."""
    if not config.STORE_PATH.exists():
        raise FileNotFoundError(
            f"Store not found at {config.STORE_PATH}. "
            f"Run `python build/build_serving_store.py` first."
        )
    # The URI form lets us request read-only mode explicitly.
    conn = sqlite3.connect(f"file:{config.STORE_PATH}?mode=ro", uri=True, check_same_thread=False)
    # Rows come back as mapping-friendly objects if used directly; queries.py
    # does not rely on this, but it is good hygiene.
    conn.row_factory = sqlite3.Row
    return conn


@contextmanager
def connect() -> Iterator[sqlite3.Connection]:
    """For scripts and the queries.py smoke test:

        with store.connect() as conn:
            rows = queries.level0_monthly(conn, orch)

    Opens one read-only connection and guarantees it is closed.
    """
    conn = _open_readonly()
    try:
        yield conn
    finally:
        conn.close()


def get_db() -> Iterator[sqlite3.Connection]:
    """FastAPI dependency. FastAPI calls this once per request, injects the
    yielded connection into the route, then runs the cleanup after the response
    is sent:

        @app.get(...)
        def route(conn: sqlite3.Connection = Depends(store.get_conn)):
            ...

    It simply delegates to connect(), so both entry points share one lifecycle.
    """
    with connect() as conn:
        yield conn

```

## `schema.py`

```py
"""
src.growlytics_platform.serving_system.serve.schema — API response shapes.

One model (or list of one) per query function. These are the contract the
frontend codes against; keeping them thin and explicit means a change in query
output is caught here at serialization time rather than in the browser.

"""

from __future__ import annotations

from pydantic import BaseModel, Field

from datetime import date

# --- picker -------------------------------------------------------------- #
class Orchestration(BaseModel):
    orchestration_key: str
    client_id: str
    client_name: str | None = None
    forecast_year: int
    revision_name: str


# --- Level 0 / 1 : the monthly chart ------------------------------------- #
class MonthPoint(BaseModel):
    month_num: int = Field(ge=1, le=12)
    month: str
    net: float
    inflow: float
    outflow: float


# --- Level 1 : one month's category decomposition ------------------------ #
class CategoryAmount(BaseModel):
    category: str
    amount: float


class MonthBreakdown(BaseModel):
    orchestration_key: str
    month_num: int = Field(ge=1, le=12)
    inflow: float
    outflow: float
    net: float
    categories: list[CategoryAmount]


# --- Level 2 : sources within a category+month --------------------------- #
class SourceRow(BaseModel):
    output_orchestration_key: str
    output_name: str
    source: str
    crop: str | None = None
    chosen_revision: str
    annualized: float
    allocation_pct: float = Field(ge=0, le=1)
    allocated: float


# --- Level 3 : lineage to versioned inputs ------------------------------- #
class LineageInput(BaseModel):
    input_output_key: str
    source: str
    crop: str | None = None
    input_type: str
    value: float
    unit: str | None = None
    revision_name: str
    revision_num: int
    last_modified: date
    memo: str | None = None


class LineageOutput(BaseModel):
    output_orchestration_key: str
    output_name: str
    source: str
    crop: str | None = None
    category: str
    annualized: float


class Lineage(BaseModel):
    output: LineageOutput | None = None
    inputs: list[LineageInput]


```

## `queries.py`

```py
"""
src.growlytics_platform.serving_system.serve.queries — the read query surface.

WHY THIS SHAPE
    The four levels the client drills through map to five pure functions. Each
    takes an open read-only connection and returns plain Python (lists/dicts of
    scalars) so the API layer can wrap them in Pydantic without any DB concepts
    leaking upward.

THE ONE RULE THAT KEEPS THE NUMBERS HONEST (grain separation)
    cash_flow_streams <-> input_output_linkage is many-to-many on
    output_orchestration_key. Joining them and then SUM()-ing money would fan
    out every stream row by its number of input links and silently inflate
    totals. So:
      - Levels 0/1/2 (money) read ONLY cash_flow_streams. No linkage join.
      - Level 3 (lineage) is the ONLY place linkage is joined, and it LISTS
        input rows rather than summing money, so the fan-out is exactly what we
        want there.

RESOLUTION IS ALREADY MATERIALIZED
    cash_flow_streams is the resolved forecast state: filtering by
    orchestration_key already selects the correct output revision per source
    ("highest output revision <= orchestration revision"). We traverse; we never
    recompute the resolution here.
"""

from __future__ import annotations

import sqlite3


def _rows(cur: sqlite3.Cursor) -> list[dict]:
    """Turn a cursor into a list of column-keyed dicts."""
    cols = [c[0] for c in cur.description]
    return [dict(zip(cols, r)) for r in cur.fetchall()]


# --------------------------------------------------------------------------- #
# Picker — the orchestration revisions that actually have servable data.
# --------------------------------------------------------------------------- #
def list_orchestrations(conn: sqlite3.Connection) -> list[dict]:
    """Every orchestration revision that has resolved stream data, with its
    client label. Feeds the top-level selector."""
    cur = conn.execute(
        """
        SELECT DISTINCT
            s.orchestration_key                 AS orchestration_key,
            s.client_id                         AS client_id,
            c.client_name                       AS client_name,
            s.forecast_year                     AS forecast_year,
            s.orchestration_revision_name       AS revision_name
        FROM cash_flow_streams AS s
        LEFT JOIN clients AS c ON c.client_id = s.client_id
        ORDER BY s.client_id, s.forecast_year, s.orchestration_revision_name
        """
    )
    return _rows(cur)


# --------------------------------------------------------------------------- #
# Level 0/1 — net cash flow per month, with inflow/outflow carried on each point.
# --------------------------------------------------------------------------- #
def net_cash_flow(conn: sqlite3.Connection, orchestration_key: str) -> list[dict]:
    """One row per month for the whole forecast: net, plus the inflow/outflow
    split that Level 1 reveals on click. Pure streams grain."""
    cur = conn.execute(
        """
        SELECT
            month_num                                          AS month_num,
            MAX(month)                                         AS month,
            ROUND(SUM(value), 2)                               AS net,
            ROUND(SUM(CASE WHEN value > 0 THEN value ELSE 0 END), 2) AS inflow,
            ROUND(SUM(CASE WHEN value < 0 THEN value ELSE 0 END), 2) AS outflow
        FROM cash_flow_streams
        WHERE orchestration_key = ?
        GROUP BY month_num
        ORDER BY month_num
        """,
        (orchestration_key,),
    )
    return _rows(cur)


# --------------------------------------------------------------------------- #
# Level 1 detail — for one month: inflow/outflow totals + category decomposition.
# --------------------------------------------------------------------------- #
def month_breakdown(
    conn: sqlite3.Connection, orchestration_key: str, month_num: int
) -> dict:
    """The category rollup for a single month (Revenue, Input Cost, ...), plus
    the inflow/outflow/net totals so the caller doesn't re-derive them."""
    cur = conn.execute(
        """
        SELECT
            category                 AS category,
            ROUND(SUM(value), 2)     AS amount
        FROM cash_flow_streams
        WHERE orchestration_key = ? AND month_num = ?
        GROUP BY category
        ORDER BY amount DESC
        """,
        (orchestration_key, month_num),
    )
    categories = _rows(cur)
    # return categories
    # !!!!!! inflow, outflow, net is summing rounded totals, resulting in inaccuracy, but not being used at the moment
    inflow = sum(c["amount"] for c in categories if c["amount"] > 0)
    outflow = sum(c["amount"] for c in categories if c["amount"] < 0)
    return {
        "orchestration_key": orchestration_key,
        "month_num": month_num,
        "inflow": round(inflow, 2),
        "outflow": round(outflow, 2),
        "net": round(inflow + outflow, 2),
        "categories": categories,
    }


# --------------------------------------------------------------------------- #
# Level 2 — inside a category+month, each source's annualized / % / allocated.
#   Grain = output_orchestration_key (source x crop x resolved revision), which
#   is the atomic annualized unit and the exact key Level 3 traces from.
# --------------------------------------------------------------------------- #
def category_sources(
    conn: sqlite3.Connection,
    orchestration_key: str,
    month_num: int,
    category: str,
) -> list[dict]:
    """One row per resolved annualized component in this category for this
    month: its annual value, this month's allocation %, and the resulting
    allocated cash. The month filter guarantees one row per key, so total_value
    is read directly (no risk of summing it 12x)."""
    cur = conn.execute(
        """
        SELECT
            output_orchestration_key   AS output_orchestration_key,
            output_name                AS output_name,
            source                     AS source,
            crop_sys                   AS crop,
            revision_name              AS chosen_revision,
            ROUND(total_value, 2)      AS annualized,
            perc                       AS allocation_pct,
            ROUND(value, 2)            AS allocated
        FROM cash_flow_streams
        WHERE orchestration_key = ? AND month_num = ? AND category = ?
        ORDER BY ABS(value) DESC
        """,
        (orchestration_key, month_num, category),
    )
    return _rows(cur)


# --------------------------------------------------------------------------- #
# Level 3 — lineage: the exact versioned inputs behind one annualized output.
#   This is the ONLY function that joins linkage. It lists inputs, so the
#   many-to-many fan-out is the intended behaviour, not a bug.
# --------------------------------------------------------------------------- #
def source_lineage(
    conn: sqlite3.Connection, output_orchestration_key: str
) -> dict:
    """The resolved output header plus every versioned input that composed it.
    The inputs may sit at different revisions (e.g. acres@rev1, yield@rev2) —
    that mix is the 'highest input revision <= output revision' rule made
    visible, and is the core traceability story."""
    header_cur = conn.execute(
        """
        SELECT
            output_orchestration_key   AS output_orchestration_key,
            output_name                AS output_name,
            source                     AS source,
            crop_sys                   AS crop,
            category                   AS category,
            ROUND(value, 2)            AS annualized
        FROM output
        WHERE output_orchestration_key = ?
        LIMIT 1
        """,
        (output_orchestration_key,),
    )
    header = _rows(header_cur)

    inputs_cur = conn.execute(
        """
        SELECT
            cp.input_output_key   AS input_output_key,
            cp.input_source       AS source,
            cp.crop               AS crop,
            cp.input_type         AS input_type,
            cp.value              AS value,
            cp.unit               AS unit,
            cp.revision_name      AS revision_name,
            cp.revision_num       AS revision_num,
            cp.last_modified      AS last_modified,
            cp.memo               AS memo
        FROM input_output_linkage AS l
        JOIN input AS cp ON cp.input_output_key = l.input_output_key
        WHERE l.output_orchestration_key = ?
        ORDER BY cp.input_type, cp.revision_num
        """,
        (output_orchestration_key,),
    )
    return {
        "output": header[0] if header else None,
        "inputs": _rows(inputs_cur),
    }

# --------------------------------------------------------------------------- #
# Dev smoke test: prove the whole drill path with no FastAPI and no network.
#   python queries.py
# This block is the only place that opens a connection or imports config; the
# functions above stay pure.
# --------------------------------------------------------------------------- #
if __name__ == "__main__":
    from growlytics_platform.serving_system.utils import config

    conn = sqlite3.connect(config.STORE_PATH)
    try:
        # print("clients:")
        # for c in list_clients(conn):
        #     print("  ", c)

        print("\norchestrations:")
        for v in list_orchestrations(conn):
            print("  ", v)

        monthly = net_cash_flow(conn, "c0001_2026_3-pre-harvest")
        print("\nLEVEL 0 — monthly cash flow:")
        for r in monthly:
            print("  ", r)

        if monthly:
            month_num = monthly[0]["month_num"]
            category = month_breakdown(conn, "c0001_2026_3-pre-harvest", month_num)
            print("\nFull Category list:")
            print(category)
            print(f"\nLEVEL 1 — sources for month {month_num}:")
            for r in category["categories"]:
                print("  ", r)

            if category:
                category = category["categories"][0]["category"]
                category1 = category_sources(conn, "c0001_2026_3-pre-harvest", month_num, category)
                print(f"\nLEVEL 2 — source lineage behind month {month_num} / {category}:")
                for r in category1:
                    print("  ", r)
                
                if category1:
                    output1 = category1[0]
                    input1 = source_lineage(conn, output1["output_orchestration_key"])
                    print(f"\nLEVEL 3 — input lineage behind output_orchestration_key {output1['output_orchestration_key']} / source {output1['source']}:")
                    for r in input1["inputs"]:
                        print("  ", r)

    finally:
        conn.close()
```

## `api.py`

```py
"""
src.growlytics_platform.serving_system.serve.api — the read-only HTTP surface.

Thin by design: each route resolves the request-scoped connection, calls one
query function, and returns it. All grain/lineage logic lives in queries.py;
all I/O safety lives in store.py. Routes are sync (SQLite reads are fast and
CPU-cheap) and FastAPI runs them in its threadpool, so a slow query never blocks
the event loop.

RUN (dev):
    uvicorn api:app --reload --port 8000
    # in-tree:  uvicorn growlytics_platform.serving_system.serve.api:app --reload

IN YOUR TREE: change the three local imports to their package paths, e.g.
    from growlytics_platform.serving_system.utils import config
    from growlytics_platform.serving_system.serve import queries, store, schema
"""

from __future__ import annotations

import sqlite3

from fastapi import Depends, FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware

from growlytics_platform.serving_system.utils import config, store
from growlytics_platform.serving_system.serve import queries
from growlytics_platform.serving_system.utils.schema import (
    Lineage,
    MonthBreakdown,
    MonthPoint,
    Orchestration,
    SourceRow,
)

app = FastAPI(title="GrowLytics Serving API", version="0.1.0")

# Only the configured frontend origins may call this API (see config.py).
app.add_middleware(
    CORSMiddleware,
    allow_origins=config.CORS_ORIGINS,
    allow_methods=["GET"],
    allow_headers=["*"],
)


@app.get("/health")
def health() -> dict:
    return {"status": "ok"}


# --- picker -------------------------------------------------------------- #
@app.get("/orchestrations", response_model=list[Orchestration])
def orchestrations(db: sqlite3.Connection = Depends(store.get_db)):
    return queries.list_orchestrations(db)


# --- Level 0 / 1 : monthly net cash flow --------------------------------- #
@app.get("/forecast/{orchestration_key}/cashflow", response_model=list[MonthPoint])
def cashflow(orchestration_key: str, db: sqlite3.Connection = Depends(store.get_db)):
    rows = queries.net_cash_flow(db, orchestration_key)
    if not rows:
        raise HTTPException(404, f"No forecast for {orchestration_key!r}")
    return rows


# --- Level 1 : one month's category decomposition ------------------------ #
@app.get("/forecast/{orchestration_key}/month/{month_num}", response_model=MonthBreakdown)
def month(
    orchestration_key: str,
    month_num: int,
    db: sqlite3.Connection = Depends(store.get_db),
):
    result = queries.month_breakdown(db, orchestration_key, month_num)
    if not result["categories"]:
        raise HTTPException(404, f"No data for month {month_num} of {orchestration_key!r}")
    return result


# --- Level 2 : sources within a category (category is a query param) ------ #
@app.get("/forecast/{orchestration_key}/month/{month_num}/sources", response_model=list[SourceRow])
def sources(
    orchestration_key: str,
    month_num: int,
    category: str,
    db: sqlite3.Connection = Depends(store.get_db),
):
    return queries.category_sources(db, orchestration_key, month_num, category)


# --- Level 3 : lineage (global on the output key) ------------------------- #
@app.get("/forecast/lineage/{output_orchestration_key}", response_model=Lineage)
def lineage(
    output_orchestration_key: str,
    db: sqlite3.Connection = Depends(store.get_db),
):
    result = queries.source_lineage(db, output_orchestration_key)
    if result["output"] is None and not result["inputs"]:
        raise HTTPException(404, f"No lineage for {output_orchestration_key!r}")
    return result

```