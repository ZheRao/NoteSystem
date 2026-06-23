# Intro

Yes. Here’s the mental model you need first.

FastAPI is a Python program that **waits for HTTP requests** and returns data, usually JSON.

```text
Browser / React
      ↓ request
FastAPI endpoint
      ↓ calls Python logic
CSV / SQLite / forecast engine
      ↓
FastAPI returns JSON
      ↓
React displays it
```

## 1. What `uvicorn main:app --reload` means

Suppose you have:

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def home():
    return {"message": "GrowLytics API is running"}
```

When you run:

```bash
uvicorn main:app --reload
```

It means:

```text
main     → file name: main.py
app      → variable inside main.py: app = FastAPI()
--reload → restart automatically when code changes
```

Uvicorn is the actual web server program. FastAPI defines the API; Uvicorn hosts it and listens for requests. FastAPI’s docs describe Uvicorn as an ASGI server used to run FastAPI apps.

After running it, this exists:

```text
http://localhost:8000
```

Your computer is now listening on port `8000`.

## 2. What decorators are

This:

```python
@app.get("/warnings")
def get_warnings():
    return {"warning_count": 3}
```

means:

```text
When someone sends GET request to /warnings,
run get_warnings().
```

The decorator is the line:

```python
@app.get("/warnings")
```

FastAPI’s official tutorial calls this a “path operation decorator”: it connects a URL path and HTTP method to the function below it.

Without decorator:

```python
def get_warnings():
    return {"warning_count": 3}
```

This is just a normal Python function.

With decorator:

```python
@app.get("/warnings")
def get_warnings():
    return {"warning_count": 3}
```

Now it becomes an API endpoint.


## 4. The magic page

FastAPI automatically gives you API docs here:

```text
http://localhost:8000/docs
```

That page lets you test endpoints without React. This is huge for you because you can build the serving layer first.

## 5. Path parameters

Use these when part of the URL identifies something.

```python
@app.get("/clients/{client_id}/forecast")
def get_client_forecast(client_id: str):
    return {"client_id": client_id}
```

URL:

```text
http://localhost:8000/clients/c0001/forecast
```

Returns:

```json
{"client_id":"c0001"}
```

FastAPI receives `c0001` and passes it into:

```python
client_id
```

## 6. Query parameters

Use these for filters/options.

```python
@app.get("/warnings")
def get_warnings(client_id: str, year: int, threshold: float = 100000):
    return {
        "client_id": client_id,
        "year": year,
        "threshold": threshold,
    }
```

URL:

```text
http://localhost:8000/warnings?client_id=c0001&year=2026&threshold=150000
```

FastAPI automatically parses:

```python
client_id = "c0001"
year = 2026
threshold = 150000
```

This is perfect for GrowLytics.

```text
/warnings?client_id=c0001&year=2026&revision=post-seeding
```

## 7. How React talks to FastAPI

React will eventually do:

```javascript
const response = await fetch("http://localhost:8000/warnings/upcoming-expenses");
const data = await response.json();
```

That is just a browser sending an HTTP request to your backend.

But because React runs at:

```text
http://localhost:5173
```

and FastAPI runs at:

```text
http://localhost:8000
```

you need CORS.

Add this to FastAPI:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

CORS basically says:

```text
I allow the frontend at localhost:5173 to call this backend at localhost:8000.
```

## 8. The GrowLytics serving layer pattern

Do not put everything in `main.py`.

Use this shape:

```text
backend/
  main.py
  services/
    warnings.py
    forecast.py
    lineage.py
  repositories/
    csv_repository.py
```

Example:

```python
# services/warnings.py

import pandas as pd


def generate_upcoming_expense_warnings(df: pd.DataFrame, threshold: float):
    warnings = (
        df[df["amount"] >= threshold]
        .sort_values("amount", ascending=False)
    )

    return warnings.to_dict(orient="records")
```

Then:

```python
# main.py

from pathlib import Path

import pandas as pd
from fastapi import FastAPI

from services.warnings import generate_upcoming_expense_warnings

app = FastAPI()
DATA_ROOT = Path(__file__).resolve().parents[1] / "data"


@app.get("/warnings/upcoming-expenses")
def get_upcoming_expenses(threshold: float = 100000):
    df = pd.read_csv(DATA_ROOT / "crop_input_streams.csv")
    return generate_upcoming_expense_warnings(df, threshold)
```

This keeps FastAPI thin.

FastAPI should mostly do:

```text
Receive request
Validate parameters
Call service logic
Return response
```

Your real intelligence stays in services.

## 9. Your first 5 endpoints

Build these before touching React seriously:

```text
GET /
GET /forecast/summary
GET /warnings/upcoming-expenses
GET /lineage/{output_key}
GET /revisions/compare?base=rev1&target=rev2
```

That gives you the skeleton of the platform.


# Linking FastAPI with SQLite

Yes — `api.py` is exactly where the FastAPI app lives and where the HTTP endpoints your frontend will call are defined. And `store.py` is the dedicated connection door you wanted, so nothing else in the codebase ever opens or closes the store. Let me give you the whole picture, then the three files, all tested against the store you already have.

Here's the complete request path through the real files you'll have after this step:

![alt text](images/image5.png)

Let me build the files. First a one-line addition to `config.py` (the API needs to know which browser origins may call it):

Now let me install FastAPI and prove the whole stack works end to end with a test client hitting the real endpoints.

It all works end to end. Every endpoint returns the right JSON, `schema.py` coerced the integer sums into the declared `float` fields, the missing-`orchestration` case returns 404, and — note this — FastAPI returned a 422 with a precise error when the required `month` param was missing, *for free*, because the parameter is declared. That validation is one of the things you get by keeping the HTTP concerns in `api.py`. Let me hand you the files.Here's how the three files fit together and the mental model for each.

`store.py` is the door, and the two things to take from it are *read-only* and *one place*. The serving connection is opened in SQLite's `mode=ro`, which makes Invariant 3 physical rather than aspirational: if a query ever contained a stray write, it would error instead of corrupting your projection. The loader (`build_serving_store.py`) is the only thing that opens a read-write connection, so writer and reader literally use different doors. The compartmentalization you wanted means this file is also the *single* place that changes when you move to the cloud — `_open_readonly` becomes "get a connection from the Postgres pool," and nothing in `queries.py` or `api.py` is touched. It exposes the same lifecycle two ways: `connect()` (a context manager for scripts and your `queries.py` smoke test) and `get_conn()` (the same thing shaped as a FastAPI dependency). Both open one connection and guarantee it closes.

`schema.py` is the contract. Each model's fields match exactly what the corresponding `queries.py` function returns. FastAPI uses these to validate the response before it ships, serialize it to JSON, and — the nice payoff — auto-generate interactive docs. Your frontend's TypeScript types will mirror these models, so a shape change is felt deliberately on both sides instead of silently breaking the UI.

`api.py` is the only file that speaks HTTP, and every route is the same four steps: FastAPI validates the params, `Depends(store.get_conn)` injects a read-only connection, the route calls the matching `queries.py` function, and `response_model` shapes the JSON. That `Depends` is the dependency-injection pattern — you declare "this route needs a connection," and FastAPI calls `get_conn`, hands you the yielded connection, and runs the cleanup after the response is sent. Three things worth internalizing: the routes are sync `def` (not `async`) because `sqlite3` blocks and sync routes run in a threadpool so a slow query can't stall the server; the URL design mirrors your drill path (`/monthly` → `/sources?month=` → `/lineage?month=&source=`); and I left a visible authorization seam in the comments where the `client_id` check belongs — there's no auth yet, so that's the one real gap to close before this is exposed to real clients.

To run the whole thing locally:

```bash
python build/build_serving_store.py        # build time: load the store (once per revision)
uvicorn api:app --reload                    # run time: start the serving layer
```

Then open `http://localhost:8000/docs` — FastAPI builds a live, clickable documentation page from `schema.py` where you can try every endpoint in the browser. That page *is* your query surface made visible, and it's the thing you'll hand a frontend developer (or yourself) to see exactly what the UI can call.

From the frontend, each interaction is one `fetch` to one of these URLs — pick a client from `/clients`, a revision from `/orchestrations`, draw the chart from `/monthly`, and the two drill clicks hit `/sources` and `/lineage`. That's the entire contract the React app depends on.

You now have a complete, tested local serving layer: loader → store → queries → connection door → API. The natural next steps are `tests/test_queries.py` to lock the query shapes (the blueprint's smoke test, formalized), closing the authorization seam, or starting the frontend `apiClient.ts` against these endpoints. Want me to take the tests next, since they protect everything you've built so far?

# Flow of Establishing Connection per Request

Excellent questions — these three confusions are the exact things that make FastAPI feel like magic until they click. Let me build it from the foundation up, because once you see *who actually calls your route function*, all three answers fall out of that one fact.

## The one idea that unlocks FastAPI: you never call your route functions

In a normal script, you write a function and then you call it. In FastAPI you write the function and **the framework calls it for you**, once per matching HTTP request. You hand FastAPI a *specification* — your function's signature — and FastAPI reads that signature to figure out how to build the arguments and then invokes the function.

The pieces:

`app = FastAPI()` creates the application object. `@app.get("/clients")` is a decorator that registers your function in a routing table: "when an HTTP GET arrives at `/clients`, call `get_clients`." Uvicorn is the server process (`uvicorn api:app`) that actually owns the network socket — it receives the raw HTTP bytes, parses them, and calls into FastAPI, which matches the route and runs your function. Your function never touches sockets or HTTP text.

The part that feels magical: FastAPI **introspects your signature** — the parameter names, type annotations, and defaults — and from that alone decides where each argument's value comes from and validates it. That's why `month: str = Query(...)` with no value sent back a 422 automatically: you *declared* month as a required string, so FastAPI enforced it before your code ran. Here's the whole mapping a newcomer needs:

| How you declare the parameter | Where FastAPI gets its value |
|---|---|
| name matches a `{placeholder}` in the path | from the URL path |
| a plain type, not in the path (e.g. `month: str`) | from the query string (`?month=...`) |
| a Pydantic model | from the JSON request body |
| `= Depends(something)` | by resolving that dependency (see below) |

And `response_model=` works in the other direction: after your function returns, FastAPI validates the return value against that model, drops anything extra, coerces types (that's why your integer sums came back as `120000.0`), serializes to JSON, and uses the same models to build the live docs at `/docs`.

So your route function is half ordinary Python, half a form you fill out telling FastAPI what you need.

## Why `conn` is a parameter with `= Depends(...)` instead of created in the body

Now your first question answers itself. Since **FastAPI calls your function**, the only way to make it *give* you something is to declare it in the signature. `Depends(store.get_conn)` in the parameter's default slot is you telling FastAPI: "before you call me, resolve `store.get_conn` and pass the result in as `conn`."

A crucial clarification about that "default value": it is **not a real default** that Python would use if you called the function normally. It's *metadata* FastAPI reads off the signature. If you ever called `get_clients()` yourself in plain Python, `conn` would literally be a `Depends` object — useless. But you never call it; FastAPI does, and it replaces that marker with the actual resolved connection. (The function's parameter default is just a convenient place Python lets you attach that marker. Modern FastAPI also lets you write it more honestly as `conn: Annotated[sqlite3.Connection, Depends(store.get_conn)]`, which keeps the type and the marker separate and drops the fake-default oddity — same behaviour, clearer intent.)

Could you instead write `with store.connect() as conn:` inside the body? Yes — it would work and be correct. As with the inline-SQL question earlier, the body approach isn't *wrong*; `Depends` buys you four things the body can't:

It removes repetition — every route needs a connection, and `Depends` references the one acquire/release routine instead of wrapping every handler body in a `with`. It makes the dependency **overridable for tests** — `app.dependency_overrides[store.get_conn] = lambda: my_test_conn` swaps the real store for a test database without editing a single route, which a hardcoded body call can't give you. It **composes** — dependencies can depend on other dependencies (an `auth` check that itself needs the connection, a `current_user`, etc.), and FastAPI resolves that whole graph per request. And it lets the framework own the **lifecycle timing** correctly, which is the bridge to your third question.

One small caution while we're here: you pass `Depends(store.get_conn)` — the function itself, no parentheses. `Depends(store.get_conn())` would call it too early and be wrong.

## The `yield` — same keyword, completely different job

Here's the reframing that dissolves the confusion: `yield` has **two unrelated uses** that happen to share one keyword.

The use you know is **producing a sequence** — yield many times, the caller loops, each `yield` hands out the next value:

```python
def rows(cursor):
    for r in cursor:
        yield r          # runs repeatedly, emits a stream
```

The use here is **suspending a function so code can run around a section** — yield *once*, which splits the function into "before" (setup) and "after" (teardown):

```python
def managed_file():
    f = open("data.txt")   # SETUP — before the yield
    try:
        yield f            # PAUSE — hand f out, then wait
    finally:
        f.close()          # TEARDOWN — after the caller is done
```

This second pattern isn't about streaming at all. The single `yield` is a **pause point**: the function runs up to it, lends out the value, and *waits*; when whoever borrowed the value is finished, the function resumes and runs the cleanup after the `yield`. Both uses rely on the same underlying ability of a generator — to suspend and resume — but streaming exploits "resume to produce the next item," while resource management exploits "resume to clean up."

Why `yield` and not `return`? Because `return` *ends* the function immediately. If `connect()` did `return conn`, there would be no later moment at which to close it — you'd have to either close it before returning (handing back a dead connection) or never close it (a leak). `yield` is the only construct that lets one function run code at **two separate times** — once before lending the resource, once after it's no longer needed — which is exactly where `conn.close()` must live.

That is precisely what `@contextmanager` formalizes. Your `connect()`:

```python
@contextmanager
def connect():
    conn = _open_readonly()   # setup
    try:
        yield conn            # pause: `with connect() as conn:` borrows this
    finally:
        conn.close()          # teardown: runs when the with-block ends, even on error
```

And FastAPI's "yield dependencies" use the *same* protocol. `get_conn` is a generator; FastAPI drives it like a context manager spread across the request: it runs up to `yield conn` (opening the connection), injects the yielded connection into your route, lets the route run, and then — **after the response has been sent** — resumes the generator past the `yield`, running the cleanup that closes the connection. If your route raised, FastAPI throws the exception back into the generator at the `yield`, so the `finally` still closes it.

Here's the full per-request timeline, which is the thing that's hard to see from the code alone:Notice the shape of it: the connection opens *before* the pause, the route does all its work *during* the pause, and the connection closes *after* the pause — and "after" is genuinely after the response has gone out. The `yield` is the hinge between "open" and "close," with the entire request living in the gap.

Your two files are just this pattern nested twice. `connect()` owns the *what* — open read-only, close in `finally`. `get_conn()` owns the *when for FastAPI* — it wraps `connect()` and re-yields, so FastAPI can drive the open/close across the request lifecycle:

```python
def get_conn():
    with connect() as conn:   # connect() opens here
        yield conn            # FastAPI borrows conn for the whole request
    #                         # connect()'s finally closes it when FastAPI resumes
```

So the single sentence to keep: **`yield` here doesn't stream anything — it marks the line where the function pauses to lend out a resource, and the code after it is the cleanup that runs once the borrower is done.** `Depends` is what tells FastAPI to be that borrower, and declaring `conn` as a parameter is the only way to ask the framework — which calls your function — to hand it in.

If it would help cement it, the fastest experiment is to drop a `print("opening")` before the `yield` in `connect()` and `print("closing")` after it, then hit one endpoint: you'll see "opening" before the response and "closing" after it, every request. When you're ready, the next build step is still `tests/test_queries.py`, which will lean on exactly this — overriding `get_conn` to point at a throwaway test store.


# Design Pattern `query.py` & `schema.py` 

This is the missing key, and you've put your finger on exactly the right thing: **you can't design the query until you know how its result gets consumed.** So let me flip the direction and teach you the frontend patterns first — once you see how a chart eats data and how a click becomes a request, the query shapes design themselves. Your two examples have clean answers, and I'll resolve both as we go.

## Reframe: the unit of an API call is a *view*, not a data point

Here's the answer to your first question straight away: **one line chart is one API call that returns all 12 points** — never 12 calls. The reason is brutal and physical: every API call is a network round trip, and round trips cost real time (tens to hundreds of milliseconds each). Twelve calls to draw one chart would be twelve sequential waits. One call returns an array of twelve objects, and the chart draws them all.

So the mental unit you design around is a **view** — a chunk of the screen that fills together. The line chart is one view. Its API call returns *the whole dataset that view needs*, shaped as an array where **each element is one visual mark**.

## How a chart actually consumes data

A charting library (Recharts, Chart.js, Plotly — they all work this way) takes an **array of objects** and maps each object to one mark on screen. Concretely, in a React library it looks like:

```jsx
<LineChart data={monthlyArray}>          {/* the whole array from ONE fetch */}
  <Line dataKey="net" />                 {/* draw a line using each object's `net` */}
  <XAxis dataKey="month" />              {/* label x-axis from each object's `month` */}
</LineChart>
```

`monthlyArray` is exactly your `MonthlyPoint[]`. Each element is one vertex of the line. The library loops the array for you. **That's why `level0_monthly` returns all months in one shot** — the array *is* the chart's input. You designed it right; you just hadn't seen the consumer yet.

This gives you a direct dictionary from "what visual" to "what array shape," which is the thing you were missing:

| Visual | Array shape it wants | Your model |
|---|---|---|
| Line / area chart | `[{x, y1, y2, …}]` — one object per point | `MonthlyPoint[]` (x=`month`, y=`net`…) |
| Bar chart | `[{category, value}]` — one object per bar | `SourceAmount[]` |
| Stacked / grouped bars | `[{category, seriesA, seriesB}]` | (a new model) |
| Table | `[{col1, col2, …}]` — one object per row | `LineageRow[]` |
| Single KPI number | one object `{value}` (not an array) | (a small model) |
| Dropdown / picker | `[{id, label}]` | `Client[]`, `Orchestration[]` |

Design rule that falls out: **the API returns data at the exact granularity the visual consumes, pre-aggregated.** The chart wants per-month inflow/outflow/net already summed — so you `GROUP BY` on the server and return the small shaped result. You never ship 997 raw stream rows and sum them in JavaScript; that's slow, duplicates your business logic in the browser, and ships data the user never sees.

## How interaction works: the frontend holds state, the server stays stateless

Now your second question — clicking a month. The pattern across essentially all interactive UIs:

The **frontend remembers the selections** (which orchestration, which month, which source) in component state. The **server remembers nothing** — it just answers parameterized questions (this is invariant 10 from earlier, made concrete). When the user does something that needs new data, the frontend fires a fetch with the current selections as parameters, and re-renders when the answer arrives.

In React terms the drill-down is a chain of `state → fetch → render`:

```jsx
const [month, setMonth] = useState(null);

// when `month` changes, fetch that month's breakdown (one call)
const { data: sources } = useQuery(["sources", orch, month], () =>
  api.getSources(orch, month), { enabled: month !== null }
);

<LineChart data={monthly}>
  <Line dataKey="net" onClick={(p) => setMonth(p.month)} />  {/* click sets state */}
</LineChart>
{sources && <BarChart data={sources} />}                      {/* render when it arrives */}
```

Clicking a point calls `setMonth`, the changed state triggers the `/sources` fetch, and the bar chart renders the returned array. Each user action that needs new data = one state change = one fetch = one of your query functions.

**But here is the nuance that's tangling your example** — and it's an important design decision in its own right. You said clicking a month should show that month's *inflow, outflow, netflow*. Look closely: **that data is already in the point the user clicked.** A `MonthlyPoint` already carries `inflow`, `outflow`, and `net`. So showing those three numbers needs **no API call at all** — the frontend already has them in hand from the chart's data; it just displays the clicked object. A call is only needed for *genuinely new* data — the breakdown **by source**, which the monthly payload doesn't contain. That's what `/sources` is for.

So your interaction actually splits into two different mechanisms:

```
click a month → show its inflow/outflow/net   → NO call (already in MonthlyPoint)
click a month → show the SOURCE breakdown      → ONE call to /sources?month=…
```

This is a general, reusable design lever: **include-in-parent vs fetch-on-demand.** If a detail is small and always wanted, fold it into the parent payload so the drill is instant and call-free. If it's larger or only sometimes wanted, leave it out and fetch it lazily when the user drills. You're constantly making this trade, and naming it is half the battle.

Here's the whole interaction mapped to calls:Here's that interaction as a compact map (view on the left, its single call on the right, clicks drilling downward):

```text
Line chart          ── filled by 1 call ──►  GET /…/monthly   → MonthlyPoint[]  (whole series)
   │  click a month  →  month=2026-03
   ▼
Source breakdown    ── filled by 1 call ──►  GET /…/sources   → SourceAmount[]  (all sources)
   │  click a source →  source=fertilizer
   ▼
Lineage table       ── filled by 1 call ──►  GET /…/lineage   → LineageRow[]    (all rows)

one view = one call returning the full array · the frontend holds the selections
```

## The common patterns that shape your API design

These are the recurring frontend behaviors, each with the API-design consequence it forces:

**One call per view (granularity).** Your number of endpoints roughly equals your number of distinct views/levels, and each returns a complete, ready-to-render collection. This is the antidote to the "12 calls" instinct.

**Fetch lazily, on interaction — not everything upfront (chatty vs chunky).** Two failure modes sit at the extremes. *Too chatty*: many tiny calls (the 12-calls mistake, or fetching a list and then one call per item — the classic "N+1"). Latency stacks and the UI feels slow. *Too chunky*: one giant call that returns all months, all sources for every month, and all lineage upfront — you pay to fetch lineage for months the user never clicks, and the first paint is slow. The sweet spot, which your drill-down naturally is: fetch each level's data only when the user drills into it. The chart loads immediately; the breakdown loads when a month is clicked; lineage loads when a source is clicked.

**Every fetch has three states the UI must handle: loading, data, empty/error.** This lightly shapes your API. An *empty but valid* result should be a `200` with `[]` so the UI can cleanly show "no data for this month," while a *nonexistent resource* is a `404` (which is exactly why `level0_monthly` returns 404 for an unknown orchestration but `/sources` happily returns `[]` for a month with no streams). Designing that distinction deliberately saves the frontend from guessing.

**Caching keyed by parameters.** Data-fetching libraries (TanStack Query / React Query, SWR are the common ones) cache each response under a key built from the endpoint + its parameters. So if the user clicks March, then February, then March again, March is served from cache — no second call. This is a hidden reason your fixed-query-plus-runtime-values design matters: stable parameterized endpoints *are* good cache keys. Arbitrary, ad-hoc queries wouldn't cache cleanly.

**Aggregate on the server.** The browser should receive the smallest shaped result the visual needs, with the math already done. Summing, grouping, and computing `allocated_value` happen in SQL, not JS — one source of truth, less data on the wire.

**Independent data loads in parallel; dependent data is a deliberate waterfall.** The client picker and the revision picker don't depend on each other, so fetch them at the same time. But level 1 genuinely can't load until the user picks a month — that's a *user-driven* waterfall, which is fine and expected. The pattern to avoid is an *accidental* waterfall where you serialize calls that could have run together.

**Paginate only when a list is unbounded.** Your data is small and bounded, so returning the whole array is correct. If a lineage view could ever return thousands of rows, you'd add `limit`/`offset` parameters and the UI would page or virtualize. Design rule: return the whole thing when it's bounded and small; paginate when it isn't.

## Now the design loop runs in the right direction

This is the method you were missing. You design **from the visual inward**, not from the table outward:

First, pick the **visual** ("a monthly line chart of net cash flow"). Second, look up the **array shape that visual eats** (line chart → one object per point with x and y fields). Third, that shape *is* your **Pydantic model** (`MonthlyPoint`: `month`, `month_num`, `net`, …). Fourth, write the **query** to produce exactly that array at the right grain. Fifth, decide **upfront-vs-on-demand** for any detail (is it already in this payload, or a separate lazy call?). Sixth, wire the **thin route**.

Run that loop on your two examples and they fully resolve:

The monthly line chart wants one object per month → `MonthlyPoint[]` → one `/monthly` call returning all months → the chart renders the array. Not twelve calls.

Clicking a month to see its inflow/outflow/net needs nothing new — those fields rode along in the `MonthlyPoint` you clicked, so it's a pure frontend display, no call. Clicking a month to see the *source breakdown* is genuinely new data → `SourceAmount[]` → one `/sources?month=…` call, rendered as bars. The selection (`month`) lives in frontend state and becomes the query parameter.

That's the whole thing: **knowing the visual tells you the array; the array is the model; the model tells you the query; the interaction tells you when to call.** Design in that order and the serving layer stops being guesswork.

If you'd like to make this concrete, a good next move is a tiny working frontend — one page with the line chart wired to `/monthly`, a click handler that fetches `/sources`, and the loading/empty states handled — so you can watch one-call-per-view and the state-driven drill happen against the API you already built. Want me to put that together?


# Code Appendix

## `store.py`

```py
"""
store.py — the one door to the serving store.

COMPARTMENTALIZATION
    Nobody else opens or closes a connection. Routes, scripts, and the smoke
    test ask store.py for a connection and hand it to queries.py. Because every
    connection is created here, this is the SINGLE file you reimplement when the
    backend moves from a local SQLite file to a managed cloud database
    (Invariant 9). queries.py and api.py do not change.

READ-ONLY BY DESIGN
    The serving path must never write (Invariant 3). The connection handed out
    here is opened in SQLite's read-only mode, so a stray INSERT/UPDATE inside a
    query fails loudly instead of corrupting the projection. The ONLY writer is
    build/build_serving_store.py, which opens its own separate read-write
    connection. Reader and writer never share a door.

ONE CONNECTION PER UNIT OF WORK
    Each request (or script block) gets its own connection and closes it when
    done (Invariant 10). Connections are never held as shared global state, which
    is what keeps the serving layer stateless and safe to run as many instances.

TWO ENTRY POINTS, SAME LIFECYCLE
    connect()   — a context manager for scripts and tests.
    get_conn()  — the same thing packaged as a FastAPI dependency.
"""

from __future__ import annotations

import sqlite3
from collections.abc import Iterator
from contextlib import contextmanager

import config


def _open_readonly() -> sqlite3.Connection:
    """Open the store read-only. Raises if the store file does not exist yet
    (i.e. the loader has not been run)."""
    if not config.STORE_PATH.exists():
        raise FileNotFoundError(
            f"Store not found at {config.STORE_PATH}. "
            f"Run `python build/build_serving_store.py` first."
        )
    # The URI form lets us request read-only mode explicitly.
    conn = sqlite3.connect(f"file:{config.STORE_PATH}?mode=ro", uri=True)
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


def get_conn() -> Iterator[sqlite3.Connection]:
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
schema.py — the response contracts (Invariant 6).

One source of truth for the SHAPE of every response. Each model's fields match,
exactly, the keys a queries.py function returns. FastAPI uses these to:
  * validate that a response really has the promised shape before it ships,
  * serialize rows to JSON,
  * auto-generate the interactive API docs at /docs.

The frontend's TypeScript types (types.ts) will mirror these models. Changing a
shape here is a deliberate, versioned contract change felt on both sides.
"""

from __future__ import annotations

from pydantic import BaseModel


class Client(BaseModel):
    client_id: str
    client_name: str


class Orchestration(BaseModel):
    orchestration_key: str
    revision_name: str
    client_id: str
    client_name: str


class MonthlyPoint(BaseModel):
    month: str
    month_num: int
    inflow: float
    outflow: float
    net: float


class SourceAmount(BaseModel):
    source: str
    amount: float


class LineageRow(BaseModel):
    output_key: str
    output_name: str
    input_output_key: str
    crop_sys: str
    revision_name: str
    stream_value: float
    perc: float
    allocated_value: float

```

## `api.py`

```py
"""
api.py — the FastAPI serving layer. The ONLY file that speaks HTTP.

WHERE IT SITS
    This is the run-time entry point. You start it with:
        uvicorn api:app --reload
    It is a long-running process that wakes on each HTTP request, and it only
    ever READS the store (Invariant 3). It contains no SQL and no rendering.

WHAT EACH ROUTE DOES (always the same four steps)
    1. FastAPI parses + validates the path/query parameters (types, required-ness).
    2. store.get_conn (injected via Depends) hands the route a read-only conn.
    3. The route calls the matching queries.py function with that conn + params.
    4. The route returns the rows; `response_model` (schema.py) validates and
       serializes them to JSON.
    The route is deliberately thin — all real work lives in queries.py.

SYNC `def`, NOT `async def`
    sqlite3 is blocking. A plain `def` route runs in FastAPI's threadpool, so a
    slow query never blocks the event loop. Using `async def` with a blocking
    sqlite call would stall the whole server — so these stay sync.

URL DESIGN MIRRORS THE DRILL PATH
    /clients                                  -> the client picker
    /orchestrations?client_id=...             -> the revision picker
    /forecast/{orchestration_key}/monthly     -> LEVEL 0 (the chart)
    /forecast/{orchestration_key}/sources     -> LEVEL 1 (click a month)
    /forecast/{orchestration_key}/lineage     -> LEVEL 2 (click a source)
"""

from __future__ import annotations

import sqlite3

from fastapi import Depends, FastAPI, HTTPException, Query
from fastapi.middleware.cors import CORSMiddleware

import config
import queries
import schema
import store

app = FastAPI(title="GrowLytics Serving Layer", version="0.1.0")

# A browser on the frontend's origin must be granted permission to call this
# API; without this the browser blocks the requests. Origins come from config.
app.add_middleware(
    CORSMiddleware,
    allow_origins=config.CORS_ORIGINS,
    allow_methods=["GET"],
    allow_headers=["*"],
)


# --------------------------------------------------------------------------- #
# Control panels
# --------------------------------------------------------------------------- #
@app.get("/clients", response_model=list[schema.Client])
def get_clients(conn: sqlite3.Connection = Depends(store.get_conn)):
    return queries.list_clients(conn)


@app.get("/orchestrations", response_model=list[schema.Orchestration])
def get_orchestrations(
    client_id: str | None = Query(default=None, description="optional: scope to one client"),
    conn: sqlite3.Connection = Depends(store.get_conn),
):
    return queries.list_orchestrations(conn, client_id)


# --------------------------------------------------------------------------- #
# The drill-down
# --------------------------------------------------------------------------- #
@app.get("/forecast/{orchestration_key}/monthly", response_model=list[schema.MonthlyPoint])
def get_monthly(
    orchestration_key: str,
    conn: sqlite3.Connection = Depends(store.get_conn),
):
    # NOTE (authorization seam): there is no auth yet, so any caller can read any
    # orchestration. This is where you would check that the authenticated user is
    # allowed to see this orchestration_key's client before querying — the fix
    # for the "swap c0001 -> c0002" risk in the OWASP notes.
    rows = queries.level0_monthly(conn, orchestration_key)
    if not rows:
        raise HTTPException(status_code=404, detail=f"No data for {orchestration_key!r}")
    return rows


@app.get("/forecast/{orchestration_key}/sources", response_model=list[schema.SourceAmount])
def get_sources(
    orchestration_key: str,
    month: str = Query(..., description="e.g. 2026-03"),
    conn: sqlite3.Connection = Depends(store.get_conn),
):
    return queries.level1_sources(conn, orchestration_key, month)


@app.get("/forecast/{orchestration_key}/lineage", response_model=list[schema.LineageRow])
def get_lineage(
    orchestration_key: str,
    month: str = Query(..., description="e.g. 2026-03"),
    source: str = Query(..., description="e.g. fertilizer"),
    conn: sqlite3.Connection = Depends(store.get_conn),
):
    return queries.level2_lineage(conn, orchestration_key, month, source)


# --------------------------------------------------------------------------- #
# Liveness probe (handy for deploys / load balancers)
# --------------------------------------------------------------------------- #
@app.get("/health")
def health():
    return {"status": "ok"}

```