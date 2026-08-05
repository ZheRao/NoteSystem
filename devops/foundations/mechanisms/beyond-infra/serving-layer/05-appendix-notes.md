# Event Loop, Threads, Workers, FastAPI Execution Model

You are **very close**. Your model is mostly right, with one important correction:

> When function 1’s awaited result becomes ready, the event loop does **not interrupt function 7 immediately**.

The event loop is cooperative, not preemptive. Function 7 must first:

* reach its own `await`,
* return,
* or raise an exception.

Only then does the event loop regain control and choose which ready coroutine to resume next.

So the timeline is:

```text
Run function 1 → reaches await → park it
Run function 2 → reaches await → park it
Run function 3 → reaches await → park it
Run function 7 → function 1's result becomes ready
                 but function 7 keeps running
Function 7 reaches await
Event loop regains control
Event loop may resume function 1
```

The event loop never forcibly pauses ordinary Python code midway through a line or computation.

Now let’s build the five layers.

---

## 1. Process versus thread

### A process is a running program

When you launch:

```bash
python api.py
```

the operating system creates a **process**.

A process owns things such as:

* its memory,
* Python objects,
* imported modules,
* open files,
* environment variables,
* one or more threads.

Conceptually:

```text
Python process
├── memory
├── variables
├── imported code
└── threads
```

Separate processes normally do not share ordinary Python variables.

```text
Process A                     Process B

x = 10                        x = 20

separate memory               separate memory
```

Changing `x` in process A does not change `x` in process B.

### A thread is one path of execution inside a process

A process can contain multiple threads:

```text
Python process
├── Thread 1
├── Thread 2
├── Thread 3
└── Thread 4
```

These threads share the process’s memory.

```python
cache = {}
```

Every thread in that process can potentially access the same `cache`.

That makes communication easy, but it also creates race-condition risks.

### A thread is not inherently a worker

This answers your main question:

> **Thread** describes what the thing physically is.
> **Worker** describes the job it has been assigned.

A worker can be:

* a worker thread,
* a worker process,
* a coroutine worker,
* even a separate machine.

For example:

```text
Thread #1: event-loop thread
Thread #2: threadpool worker
Thread #3: threadpool worker
Thread #4: logging thread
```

All four are threads.

But they have different responsibilities.

The difference between an **event-loop thread** and a **threadpool worker** is not that they are different species. They are both operating-system threads.

The difference is their assigned role.

---

## 2. What the GIL locks

In ordinary CPython, the Global Interpreter Lock generally permits only one thread at a time to execute Python bytecode.

Suppose your process has four threads:

```text
Thread A
Thread B
Thread C
Thread D
```

For pure Python execution, it is approximately:

```text
Thread A executes Python
Thread B waits for GIL
Thread C waits for GIL
Thread D waits for GIL
```

Then the operating system and Python may switch:

```text
Thread B executes Python
Thread A waits
Thread C waits
Thread D waits
```

This is **thread switching**, but not true parallel execution of Python bytecode.

However, native code can release the GIL. For example, parts of:

* SQLite,
* NumPy,
* some pandas operations,
* network calls,
* file operations

may release it.

Then this can occur:

```text
CPU core 1: SQLite native C code in thread A
CPU core 2: Python bytecode in thread B
```

The uploaded documentation makes this distinction when explaining that `sqlite3` releases the GIL around underlying SQLite calls. 

But the GIL and the event loop are separate concepts.

The GIL answers:

> Which thread may execute Python bytecode right now?

The event loop answers:

> Which coroutine should this particular thread run next?

---

## 3. What a coroutine really is

A coroutine is not a thread.

It is better understood as:

> A function whose execution can be paused at designated points and later resumed from the same place.

Consider:

```python
async def task():
    print("A")
    await something()
    print("B")
```

Its execution state includes:

```text
Current position: after print("A")
Local variables: whatever currently exists
Waiting for: something()
```

When it reaches `await`, Python preserves that state.

Conceptually:

```text
Coroutine task
├── paused location
├── local variables
└── awaited operation
```

Later, it resumes from the same point:

```python
print("B")
```

A normal function does not offer this ability to the event loop.

```python
def task():
    step_one()
    step_two()
```

Once called, it executes until:

* it returns,
* raises,
* or its thread is externally switched by the operating system.

But the function itself does not cooperatively say:

> “Park me here and resume me later.”

### `async def` creates coroutine behavior

When you define:

```python
async def task():
    ...
```

calling it does not immediately execute the body like an ordinary function call.

It creates a coroutine object:

```python
coroutine = task()
```

That coroutine must then be driven by an event loop, usually by:

```python
await task()
```

or by creating a task for it.

### `await` is the voluntary handoff point

At a genuine asynchronous wait:

```python
result = await operation()
```

the coroutine can effectively say:

```text
My result is not ready.
Save my position.
Run another ready coroutine.
Resume me later.
```

The source describes coroutines as sharing one thread and taking turns rather than running simultaneously. 

---

## 4. How the event loop works

The event loop is a scheduler running inside one thread.

Imagine it has two collections:

```text
Ready tasks:
- Task C
- Task F

Waiting tasks:
- Task A waiting for socket
- Task B waiting for timer
- Task D waiting for database driver
```

Its simplified behavior is:

```text
1. Pick a ready coroutine.
2. Run it.
3. The coroutine reaches await or finishes.
4. Regain control.
5. Pick another ready coroutine.
6. Repeat.
```

The important phrase is:

> **Run it until it yields control.**

The event loop does not normally execute a coroutine for a fixed ten-millisecond time slice and then forcibly interrupt it.

That is what operating-system thread scheduling can do. Coroutine scheduling is cooperative.

### Your model, corrected

You described:

> The event loop goes into each function and executes until it hits `await`, then parks it and moves to the next.

Yes.

Then:

> When the first result comes back, it parks the current function and finishes the first.

Almost. The corrected version is:

> When the first result becomes ready, the event loop marks function 1 as ready. The current function continues until it reaches an `await` or returns. Then the event loop may choose function 1.

For example:

```python
async def task_seven():
    perform_python_work_for_ten_seconds()
    await network_call()
```

If task 1 becomes ready during those ten seconds, task 1 still cannot resume.

The event-loop thread is occupied inside:

```python
perform_python_work_for_ten_seconds()
```

Only when task 7 reaches:

```python
await network_call()
```

does the event loop regain control.

### What happens with `async def` containing no `await`?

```python
async def bad_task():
    do_work_for_ten_seconds()
    return "done"
```

When the event loop starts `bad_task`, it runs continuously for ten seconds.

```text
Event loop starts bad_task
↓
bad_task never yields
↓
event loop remains trapped
↓
other coroutines cannot run
↓
bad_task returns after ten seconds
↓
event loop finally regains control
```

You said it “behaves exactly like a worker.”

There is a subtle difference.

It behaves like an ordinary synchronous function **executing on the event-loop thread**. But it does not become a threadpool worker.

That distinction matters because:

```text
Long function on worker thread:
blocks one replaceable worker

Long function on event-loop thread:
blocks the central scheduler
```

The function may perform identical work, but **where it runs** changes the impact.

---

## 5. FastAPI’s execution model

FastAPI receives requests through an asynchronous server such as Uvicorn.

Inside one server process, the simplified structure is:

```text
Python server process
│
├── Event-loop thread
│   ├── receives requests
│   ├── manages sockets
│   ├── runs async routes
│   └── coordinates completions
│
└── Threadpool
    ├── Worker thread 1
    ├── Worker thread 2
    ├── Worker thread 3
    └── ...
```

Again:

* the event-loop thread is a thread,
* each threadpool worker is also a thread,
* “worker” is the role.

### When FastAPI sees `async def`

```python
@app.get("/forecast")
async def forecast():
    ...
```

FastAPI runs it directly on the event-loop thread.

Why?

Because FastAPI trusts that the route will cooperate:

```python
@app.get("/weather")
async def weather():
    result = await async_http_client.get(url)
    return result
```

Timeline:

```text
Event loop enters weather()
weather starts network request
weather reaches await
weather is parked
event loop handles another request
network response arrives
weather becomes ready
event loop eventually resumes weather
```

No extra thread is needed because the route spends most of its time parked.

### When FastAPI sees plain `def`

```python
@app.get("/forecast")
def forecast():
    rows = sqlite_query()
    return rows
```

FastAPI assumes:

> This function cannot cooperate with my event loop.

So it performs a threadpool handoff:

```text
Event loop receives request
↓
Submit forecast() to worker thread
↓
Event loop awaits worker completion
↓
Event loop remains available
```

The worker then runs:

```text
Worker thread #8
↓
forecast()
↓
sqlite_query()
↓
blocked until complete
```

Only worker #8 is occupied.

The central event-loop thread remains free.

This is why the source distinguishes:

* `async def`: event-loop thread
* `def`: threadpool worker 

---

## Why not have many event-loop threads?

You asked whether it is because orchestration across threads is difficult.

That is part of it, but the deeper answer is:

> A single event loop is intentionally designed to handle huge amounts of waiting work efficiently without needing many threads.

Suppose 10,000 network connections are mostly waiting.

With thread-per-request:

```text
10,000 connections
→ potentially 10,000 threads
```

That is expensive:

* each thread needs stack memory,
* the OS must schedule them,
* context switching becomes expensive,
* shared state requires locking.

With one event loop:

```text
10,000 connections
→ 10,000 lightweight coroutine states
→ mostly one thread
```

Because almost all those connections are waiting, one thread is enough to resume whichever small number are currently ready.

### Multiple event-loop threads are possible, but awkward

A coroutine and its asynchronous resources are usually associated with a particular event loop.

For example:

```text
Event loop A owns:
- task A1
- task A2
- socket A
- timer A

Event loop B owns:
- task B1
- task B2
- socket B
- timer B
```

Moving tasks or asynchronous objects freely between loops is difficult because each loop maintains its own:

* ready queue,
* timers,
* socket registrations,
* task state,
* callbacks.

Then you face questions such as:

```text
Which loop owns this socket?
Which loop should resume this coroutine?
How does loop A safely notify loop B?
Which thread may modify each loop's queue?
```

Locks and cross-thread communication would reduce the simplicity and efficiency of the event-loop model.

So yes, cross-thread orchestration is one reason.

But there is another major reason:

> Multiple event-loop threads often do not help much with Python CPU work because of the GIL.

If you created four event loops in four threads:

```text
Thread 1: event loop A
Thread 2: event loop B
Thread 3: event loop C
Thread 4: event loop D
```

and all four tried to execute Python bytecode, the GIL would still generally allow only one at a time.

You would gain complexity without receiving four-core Python parallelism.

---

## How servers actually scale beyond one event loop

They often use **multiple processes**, not multiple event loops inside one process.

For example:

```text
Server worker process 1
└── one event loop

Server worker process 2
└── one event loop

Server worker process 3
└── one event loop

Server worker process 4
└── one event loop
```

Each process has:

* separate memory,
* its own Python interpreter,
* its own GIL,
* its own event loop,
* its own threadpool.

Now four CPU cores can genuinely run four Python processes in parallel.

Conceptually:

```text
Incoming requests
        ↓
Load balancer / server socket
        ↓
┌────────────┬────────────┬────────────┬────────────┐
│ Process 1  │ Process 2  │ Process 3  │ Process 4  │
│ Event loop │ Event loop │ Event loop │ Event loop │
└────────────┴────────────┴────────────┴────────────┘
```

That is a common production architecture.

Within each process:

* one event loop handles async waiting efficiently,
* a threadpool isolates blocking functions,
* multiple processes provide CPU parallelism and isolation.

---

## A complete mental picture

Here is the entire structure:

```text
Machine
│
├── FastAPI process 1
│   │
│   ├── Event-loop thread
│   │   ├── async request A
│   │   ├── async request B
│   │   └── async request C
│   │
│   └── Threadpool
│       ├── worker thread 1 running sync route
│       ├── worker thread 2 running sync route
│       └── worker thread 3 idle
│
└── FastAPI process 2
    │
    ├── Event-loop thread
    │
    └── Threadpool
```

And the terminology:

| Term              | Meaning                                            |
| ----------------- | -------------------------------------------------- |
| Process           | Independent running program with its own memory    |
| Thread            | An OS-managed execution path inside a process      |
| Worker            | A role assigned to something that executes jobs    |
| Worker thread     | A thread assigned to execute submitted jobs        |
| Threadpool        | A reusable collection of worker threads            |
| Coroutine         | A function execution that can pause and resume     |
| Event loop        | Scheduler that runs ready coroutines on one thread |
| Event-loop thread | The thread currently running that scheduler        |

---

## One request through the whole system

Suppose two requests arrive.

### Request A: genuine asynchronous operation

```python
@app.get("/external-data")
async def external_data():
    result = await async_client.get(url)
    return result
```

Execution:

```text
Event-loop thread enters external_data()
↓
Starts network operation
↓
Reaches await
↓
Parks coroutine A
↓
Event loop remains available
```

### Request B: SQLite operation

```python
@app.get("/forecast")
def forecast():
    return sqlite_query()
```

Execution:

```text
Event-loop thread receives request B
↓
Submits forecast() to threadpool
↓
Worker thread #4 runs forecast()
↓
Worker thread #4 blocks in SQLite
↓
Event loop remains available
```

Now they coexist:

```text
Event-loop thread:
handling new requests and ready coroutines

Worker thread #4:
blocked/running inside SQLite

Coroutine A:
parked waiting for network
```

Later:

```text
Network result for A becomes ready
SQLite result for B becomes ready
```

The event loop processes their completions when it next has control.

---

## Your current understanding, rewritten precisely

Your current mental model can become:

> The event loop runs an `async` function directly on its own thread. It executes the coroutine until that coroutine reaches an `await` whose operation is not ready, or until it returns. At that point, the coroutine yields control and the event loop can run another ready coroutine. When an awaited result becomes available, the original coroutine is marked ready, but it cannot resume until the currently running coroutine yields control.

And:

> An `async def` function that performs long work without reaching `await` monopolizes the event-loop thread. A plain `def` FastAPI route is instead dispatched to a worker thread, so blocking that function occupies only one worker rather than the central event loop.

And finally:

> A worker is not a different execution primitive from a thread. “Thread” describes the mechanism; “worker” describes its responsibility. FastAPI typically uses one thread as the event-loop coordinator and other threads as workers for blocking synchronous code.

That is now a fundamentally sound model.

# PRAGMA, `check_same_thread`, and Contextmanager Decorator

Yes. These are three separate mechanisms, and the documentation is again explaining them at a level that assumes you already know several Python and SQLite conventions.

## 1. What is `PRAGMA`?

`PRAGMA` is SQLite’s way of issuing **configuration and diagnostic commands**.

It looks like SQL:

```python
conn.execute("PRAGMA busy_timeout = 5000")
```

but it is not querying or modifying your business tables like:

```sql
SELECT ...
INSERT ...
UPDATE ...
```

Instead, it tells SQLite something about how the database connection or database file should behave.

Think of the distinction as:

```text
SELECT / INSERT / UPDATE
→ operate on your data

PRAGMA
→ inspect or configure SQLite itself
```

### Example: `busy_timeout`

```python
conn.execute("PRAGMA busy_timeout = 5000")
```

This means:

> When this connection encounters a locked database, wait and retry for up to 5,000 milliseconds before failing.

Without a timeout, imagine:

```text
Connection A is writing
Connection B attempts to write
Connection B immediately receives:
database is locked
```

With:

```sql
PRAGMA busy_timeout = 5000
```

it becomes:

```text
Connection A is writing
Connection B finds database locked
Connection B waits
Connection A finishes after 200 ms
Connection B retries and proceeds
```

It does not guarantee success. If the lock still exists after five seconds, the operation can still fail.

Also, this setting generally applies to the **connection on which you execute it**:

```python
conn_a.execute("PRAGMA busy_timeout = 5000")
```

That does not necessarily configure every future connection automatically. This is why connection setup should be centralized.

For example:

```python
@contextmanager
def connect():
    conn = sqlite3.connect(...)
    conn.execute("PRAGMA busy_timeout = 5000")
    try:
        yield conn
    finally:
        conn.close()
```

Now every part of your application that uses `connect()` gets the same configuration.

That is what the sentence means by:

> “One place to add a PRAGMA, one place to change how connections open.”

Without centralization, you might accidentally have:

```python
## Script path
conn = sqlite3.connect(DB_PATH)
conn.execute("PRAGMA busy_timeout = 5000")
```

but:

```python
## API path
conn = sqlite3.connect(DB_PATH)
## forgot busy_timeout
```

Now scripts and API requests behave differently.

### Example: `journal_mode=WAL`

```python
conn.execute("PRAGMA journal_mode=WAL")
```

This changes SQLite’s journaling mechanism.

A journal is part of how SQLite protects your database while writes happen.

In simplified terms, SQLite must ensure:

> If a write crashes halfway through, the database must not be left half-old and half-new.

#### Traditional rollback journal

Before replacing pages in the main database file, SQLite records the original pages in a journal:

```text
Main database:
page 1, page 2, page 3

Writer wants to change page 2

1. Copy old page 2 into journal file
2. Modify page 2 in main database
3. Commit
4. Delete or clear journal
```

If something fails, SQLite can restore the old page from the journal.

#### WAL: write-ahead log

With WAL, SQLite initially leaves the main database file alone:

```text
Main database:
old pages remain here

WAL file:
new versions are appended here
```

A reader may need to consider both:

```text
Main database + latest committed WAL entries
```

Later, a **checkpoint** transfers the newer pages from the WAL into the main database file.

Simplified:

```text
Before checkpoint:

database.db
└── older page versions

database.db-wal
└── newer committed page versions

After checkpoint:

database.db
└── newer page versions incorporated

database.db-wal
└── cleared or reduced
```

One major benefit is that readers can often continue reading while one writer appends to the WAL. It improves read/write concurrency, although SQLite still normally allows only one writer at a time.

So:

```sql
PRAGMA journal_mode=WAL
```

means:

> Configure this SQLite database to use write-ahead logging.

Some pragmas are connection-specific; others affect the persistent database file. You do not need to memorize which category each pragma belongs to yet. The immediate model is:

> `PRAGMA` is SQLite’s configuration/control interface.

---

## 2. Understanding `check_same_thread=False`

Let us ignore FastAPI initially.

### A SQLite connection is a Python object

```python
conn = sqlite3.connect("forecast.db")
```

The object holds changing internal state:

```text
conn
├── database file handle
├── transaction state
├── cached statements
├── current cursor-related state
└── SQLite internal structures
```

Suppose thread 7 creates it:

```text
Thread 7:
conn = sqlite3.connect(...)
```

By default, Python records:

```text
This connection belongs to thread 7.
```

Then, if thread 12 tries:

```text
Thread 12:
conn.execute(...)
```

Python raises:

```text
SQLite objects created in a thread can only be used
in that same thread.
```

The purpose is to prevent this dangerous situation:

```text
Thread 7                      Thread 12

conn.execute(query A)         conn.execute(query B)
        ↓                              ↓
both modify the same connection's internal state
at the same time
```

Because threads share memory, both threads can potentially access the same object. The default guard prohibits cross-thread usage entirely.

The documentation describes this connection as mutable shared state and explains that Python records the creating thread. 

### But “different thread” does not necessarily mean “simultaneously”

There are two different situations.

#### Unsafe: simultaneous use

```text
10:00:00.000  Thread 7 uses conn
10:00:00.000  Thread 12 also uses conn
```

Both touch it at once.

That can create a race.

#### Potentially safe: sequential use

```text
10:00:00.000  Thread 7 creates conn
10:00:00.010  Thread 7 stops touching it

10:00:00.020  Thread 12 uses conn
10:00:00.030  Thread 12 stops touching it

10:00:00.040  Thread 3 closes conn
```

The thread changes, but only one thread touches the connection at any moment.

The `sqlite3` safety check is intentionally simple. It does not track whether two threads overlap in time. It merely checks:

```python
current_thread_id == connection_creation_thread_id
```

If not, it raises.

Therefore it rejects both:

```text
simultaneous cross-thread use  → genuinely dangerous
sequential cross-thread use    → may be safe
```

### Why might FastAPI switch threads?

Your dependency looks like:

```python
def get_db():
    with connect() as conn:
        yield conn
```

The dependency has two halves:

```python
def get_db():
    with connect() as conn:
        ## first half: setup
        yield conn
        ## second half: teardown
```

And your route is another sync function:

```python
def route(conn = Depends(get_db)):
    return query(conn)
```

Because these are synchronous functions, FastAPI runs them through its worker-thread pool.

The sequence may be:

```text
Step 1: Worker thread 7
        begin get_db()
        create connection
        reach yield

Step 2: Worker thread 12
        run route using that connection

Step 3: Worker thread 3
        resume get_db()
        exit with-block
        close connection
```

The threadpool does not promise:

> All pieces belonging to one request will always use worker 7.

Each time work is submitted, an available worker can receive it. The source illustrates this exact setup/use/teardown sequence. 

So with the default setting:

```python
sqlite3.connect(
    path,
    check_same_thread=True,  ## default
)
```

this could happen:

```text
Thread 7 creates conn       → allowed
Thread 12 uses conn         → ProgrammingError
```

### What does `check_same_thread=False` do?

```python
conn = sqlite3.connect(
    path,
    check_same_thread=False,
)
```

It tells the Python wrapper:

> Do not reject this connection solely because the current thread differs from the creating thread.

It does **not** make the connection magically safe.

It does not add locks.

It does not prevent simultaneous usage.

It merely disables the thread-identity check.

Therefore, your architecture must maintain the real safety rule:

> Only one thread may use this particular connection at one moment.

### Why your design can satisfy that rule

For request A:

```text
Request A gets connection A
```

For request B:

```text
Request B gets connection B
```

They do not share:

```text
Request A → connection A
Request B → connection B
Request C → connection C
```

Within request A, its connection may move sequentially:

```text
thread 7 creates it
then thread 12 uses it
then thread 3 closes it
```

But never:

```text
thread 7 and thread 12 use connection A simultaneously
```

So you have:

```text
Different threads?       Possibly yes.
Concurrent shared use?   No.
```

That is why turning off the conservative guard is acceptable in this design. The source states the controlling invariant as one connection used by exactly one request at a time. 

### Why a global connection would break the design

This is dangerous:

```python
CONN = sqlite3.connect(
    DB_PATH,
    check_same_thread=False,
)
```

Then every request shares it:

```text
Request A → global CONN
Request B → global CONN
Request C → global CONN
```

Now:

```text
Worker 7:  CONN.execute(query A)
Worker 12: CONN.execute(query B)
```

may happen simultaneously.

You have disabled the protection and then created the condition it was intended to protect against.

The safe pattern is:

```python
def get_db():
    with connect() as conn:
        yield conn
```

because each request opens and receives its own connection.

A compact way to remember it:

```text
check_same_thread=False
does not mean:
“SQLite connections are thread-safe.”

It means:
“Do not enforce same-thread identity;
my architecture will prevent concurrent sharing.”
```

---

## 3. `contextmanager`, `yield`, and guaranteed teardown

There are actually **two related generator patterns** here:

1. Your own `connect()` function may use `@contextmanager`.
2. FastAPI interprets a dependency containing `yield` as a setup/teardown dependency.

They look similar because they are based on the same idea.

## First: what is a normal context manager?

You already use context managers whenever you write:

```python
with open("data.txt") as file:
    text = file.read()
```

The important behavior is:

```text
Enter the with-block
Acquire resource

Run block

Leave the with-block
Release resource
```

Even if the block raises:

```python
with open("data.txt") as file:
    raise ValueError("something failed")
```

Python still closes the file.

Conceptually:

```python
resource = acquire()

try:
    use(resource)
finally:
    release(resource)
```

The `finally` runs whether the block:

* succeeds,
* returns,
* raises an exception.

That is the fundamental purpose of a context manager:

> Pair resource acquisition with guaranteed cleanup.

---

## What does `@contextmanager` do?

Suppose you write:

```python
from contextlib import contextmanager

@contextmanager
def connect():
    conn = sqlite3.connect("forecast.db")
    try:
        yield conn
    finally:
        conn.close()
```

Without `@contextmanager`, this would merely be a generator function because it contains `yield`.

The decorator transforms that generator into an object compatible with `with`:

```python
with connect() as conn:
    rows = conn.execute(...).fetchall()
```

The meaning is:

```text
Everything before yield
→ __enter__ / setup

The value yielded
→ value after “as”

Everything after yield
→ __exit__ / cleanup
```

So:

```python
@contextmanager
def connect():
    conn = sqlite3.connect(...)  ## setup
    try:
        yield conn               ## hand conn to with-block
    finally:
        conn.close()             ## teardown
```

maps to:

```python
with connect() as conn:
    route_or_script_uses(conn)
```

like this:

```text
1. Call connect()
2. Run until yield
3. Produce conn
4. Execute body of with-block
5. Resume connect() after yield
6. Run finally
7. Close conn
```

### The `yield` divides setup from teardown

This is the easiest mental model:

```python
@contextmanager
def resource():
    print("SETUP")

    yield "the resource"

    print("TEARDOWN")
```

Used as:

```python
with resource() as value:
    print("BODY:", value)
```

Output:

```text
SETUP
BODY: the resource
TEARDOWN
```

If the body fails:

```python
with resource() as value:
    print("BODY")
    raise RuntimeError("failure")
```

the conceptual flow is still:

```text
SETUP
BODY
TEARDOWN
exception continues outward
```

In real code, you normally place teardown in `finally` to structurally guarantee it:

```python
@contextmanager
def resource():
    print("SETUP")
    try:
        yield "resource"
    finally:
        print("TEARDOWN")
```

---

## Your two nested mechanisms

Your code is:

```python
def get_db() -> Iterator[sqlite3.Connection]:
    with connect() as conn:
        yield conn
```

Suppose `connect()` is approximately:

```python
@contextmanager
def connect():
    conn = sqlite3.connect(...)
    try:
        yield conn
    finally:
        conn.close()
```

There are now two layers.

### Inner layer: `connect()`

```python
with connect() as conn:
```

This guarantees that the physical SQLite connection eventually closes.

### Outer layer: `get_db()`

```python
def get_db():
    with connect() as conn:
        yield conn
```

FastAPI understands a dependency with `yield` as:

```text
Before yield:
set up dependency

At yield:
give value to route

After route:
resume dependency and tear it down
```

So the complete sequence is:

```text
FastAPI begins get_db()

    get_db enters connect()

        connect creates SQLite connection
        connect yields conn to get_db()

    get_db yields conn to FastAPI

FastAPI gives conn to route

Route executes

FastAPI resumes get_db()

    get_db exits its with connect() block

        FastAPI/Python resumes connect()
        connect's finally calls conn.close()

get_db finishes
```

Visually:

```text
connect():
    create conn
    ┌───────────────────────────────────────┐
    │ get_db():                             │
    │   yield conn                          │
    │   ┌───────────────────────────────┐   │
    │   │ route executes using conn     │   │
    │   └───────────────────────────────┘   │
    └───────────────────────────────────────┘
    close conn
```

---

## What does FastAPI do with `get_db()`?

You may have:

```python
@app.get("/forecast")
def forecast(conn: Connection = Depends(get_db)):
    return conn.execute(...).fetchall()
```

FastAPI sees that `get_db()` contains a `yield`.

It conceptually does:

```python
dependency_generator = get_db()

conn = next(dependency_generator)
```

Calling `next()` runs the dependency until:

```python
yield conn
```

Then FastAPI invokes the route:

```python
result = forecast(conn)
```

Afterward, FastAPI resumes the generator so cleanup can happen:

```python
next(dependency_generator)
```

That continues after:

```python
yield conn
```

which exits the `with` block, causing the connection cleanup.

This is simplified, but it captures the mechanism.

---

## What happens if the route raises?

Suppose:

```python
def forecast(conn = Depends(get_db)):
    raise ValueError("query failed")
```

FastAPI still must resume/close the dependency.

Conceptually:

```text
get_db starts
connection opens
get_db yields connection

route starts
route raises exception

FastAPI unwinds dependency
get_db exits with-block
connection closes

exception continues through FastAPI's error handling
```

That is the main guarantee.

The cleanup is tied structurally to the lifetime of the generator:

```python
def get_db():
    with connect() as conn:
        yield conn
```

The connection remains alive while the generator is paused at `yield`.

When the generator is resumed or closed, the `with` block exits.

---

## What is `AsyncExitStack`?

You do not need to understand its implementation to use `get_db`, but here is the simple model.

An exit stack is just a managed list of cleanup actions.

Imagine FastAPI creates:

```text
Cleanup stack for request A
```

As dependencies acquire resources, cleanup actions are registered:

```text
Cleanup stack:
1. close database connection
2. close HTTP session
3. remove temporary file
```

When the request ends, it executes them in reverse order:

```text
3. remove temporary file
2. close HTTP session
1. close database connection
```

Why reverse order?

Because resources are often nested:

```text
Open A
    Open B
        Open C
        Close C
    Close B
Close A
```

This matches ordinary nested `with` blocks.

`AsyncExitStack` is a version that can manage both ordinary and asynchronous cleanup behavior.

For your mental model:

> FastAPI keeps a request-specific cleanup checklist, and generator dependencies add their teardown work to that checklist.

You do not need to manually instantiate `AsyncExitStack` in this code.

---

## Why not use an `on_finish` callback?

Imagine manual code:

```python
def get_db():
    conn = connect()
    register_on_finish(conn.close)
    return conn
```

Now correctness depends on several moving parts:

* Did the callback get registered?
* Is it called after success?
* Is it called after an exception?
* Is it called if another dependency fails?
* Does cleanup happen in the correct order?

With generator/context-manager structure:

```python
def get_db():
    with connect() as conn:
        yield conn
```

the acquisition and cleanup are written together:

```text
open here
yield here
close automatically when scope exits
```

You cannot easily modify the opening logic and forget where cleanup lives because they are structurally paired.

---

## One nuance about `with sqlite3.connect(...)`

There is an important Python-specific subtlety.

This:

```python
with sqlite3.connect("forecast.db") as conn:
    ...
```

uses the SQLite connection’s built-in context-manager behavior primarily for **transaction management**:

* commit if the block succeeds,
* rollback if the block raises.

It does not necessarily mean the connection itself is closed on block exit.

Therefore, a custom function like this is useful:

```python
@contextmanager
def connect():
    conn = sqlite3.connect(
        DB_PATH,
        check_same_thread=False,
    )
    try:
        conn.execute("PRAGMA busy_timeout = 5000")
        yield conn
    finally:
        conn.close()
```

Now `conn.close()` is explicit and guaranteed.

Then:

```python
def get_db():
    with connect() as conn:
        yield conn
```

uses that guaranteed-close context manager for the lifetime of one request.

---

## All three concepts assembled

A realistic structure might be:

```python
from collections.abc import Iterator
from contextlib import contextmanager
import sqlite3


@contextmanager
def connect() -> Iterator[sqlite3.Connection]:
    conn = sqlite3.connect(
        "forecast.db",
        check_same_thread=False,
    )

    try:
        conn.execute("PRAGMA busy_timeout = 5000")
        yield conn
    finally:
        conn.close()


def get_db() -> Iterator[sqlite3.Connection]:
    with connect() as conn:
        yield conn
```

The responsibilities are:

```text
PRAGMA busy_timeout
→ configures SQLite behavior for this connection

check_same_thread=False
→ permits the one-request connection to move sequentially
  between FastAPI worker threads

@contextmanager connect()
→ pairs connection creation with guaranteed close

get_db() yield dependency
→ makes that connection live for exactly one request

FastAPI cleanup machinery
→ resumes/ends get_db when the request finishes
```

And the lifecycle is:

```text
Request begins
↓
FastAPI starts get_db()
↓
connect() opens connection
↓
PRAGMA configures connection
↓
get_db yields connection
↓
route uses connection
↓
route succeeds or raises
↓
FastAPI unwinds get_db()
↓
connect() finally runs
↓
connection closes
```

The shortest mental models are:

```text
PRAGMA
= configure SQLite

check_same_thread=False
= permit sequential cross-thread use;
  not permission for concurrent sharing

@contextmanager
= code before yield is setup,
  code after yield/finally is cleanup

FastAPI yield dependency
= keep resource alive for one request,
  then guarantee teardown
```


