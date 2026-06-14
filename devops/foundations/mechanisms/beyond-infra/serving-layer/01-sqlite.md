# Concept & Where It Fits

This is a really well-chosen first milestone, and the file's comments hint at why without quite spelling it out. Let me give you the big-picture role first, then go deep on `sqlite3` itself.

Let me sketch where this sits before explaining it.Big picture first. The file you've got is the first brick of what's called the **serving layer** — and understanding that term is the key to seeing why this is milestone zero rather than an afterthought.

Your data system (the pipeline that produced `crop_input_streams.csv`) does *batch* work: it crunches numbers and spits out a file. A file is correct but inert — it just sits on disk. Your frontend, on the other hand, needs to ask narrow questions fast and repeatedly: "just the source breakdown for March, for client c0001." You can't do that against a raw CSV without loading the whole thing into memory and filtering every single time. The serving layer is the stage that sits *between* your pipeline's output and your application, turning that inert file into something you can interrogate quickly. That's exactly what `build_db.py` does.

Here's where it lives in the ecosystem:

![alt text](images/image1.png)

Notice the three teal boxes on the right. Those aren't decoration — they're the three functions in your file (`level0_monthly`, `level1_sources`, `level2_lineage`), and each one *is* a frontend interaction in embryo. Level 0 is the chart you see on load; level 1 is what fires when you click a month; level 2 is what fires when you click a source and finally see the individual streams plus the revision that produced them (the "traceability payoff" your comments mention).

That's what makes this the right first step, and why it comes before any API or UI:

The serving layer is a *contract*. By writing these three queries first, you've defined every question the frontend is allowed to ask — before committing to FastAPI, before a single React component. When you build the API next, each endpoint is a thin wrapper around one of these functions. When you build the UI, each click calls one endpoint. You've sequenced the work so the riskiest question ("does my data actually support the drill-downs the product needs?") gets answered cheaply, up front.

And SQLite specifically is chosen to remove distractions. No server to run, no Postgres to install, no credentials, no network. It's one file. So you can focus 100% on getting the query *shape* right. If you later outgrow it, the SQL ports to Postgres almost unchanged — so this milestone is low-commitment as well as low-overhead.

That's the "why." Now the "what" — `sqlite3` from zero.

## What SQLite actually is

SQLite is a relational database **engine**, but with one unusual property: it's *embedded*. "Embedded" means it runs inside your own program rather than as a separate server you connect to over a network. Postgres and MySQL are the opposite — a server process runs somewhere, holds the data, and your code talks to it through a socket with a host, port, username, and password. SQLite has none of that. The entire database — every table, index, and row — lives in a single ordinary file on disk. In your case that file is `forecast.sqlite`. Copy that one file and you've copied the whole database. Delete it and it's gone. That simplicity is why SQLite is the most widely deployed database on Earth (it's inside your phone, your browser, most apps you use).

## What the `sqlite3` module is

The engine itself is written in C. `sqlite3` is the Python module in the standard library that wraps that C engine so you can drive it with Python code. Because it ships with Python, there's nothing to install — that's exactly why your file's comments say the only `pip install` you need is pandas. You just `import sqlite3` and you have a full database at your fingertips.

## The two core objects: Connection and Cursor

Almost everything in `sqlite3` flows through two objects.

A **Connection** is your handle to one database file. You get it with `sqlite3.connect(DB_PATH)`, which opens the file — and *creates* it if it doesn't exist yet. That's the line `conn = sqlite3.connect(DB_PATH)` in your `build()` function. The connection represents the open database; you do all your work through it, and you call `conn.close()` when you're finished to release the file.

A **Cursor** is the object that actually executes a SQL statement and holds the resulting rows. Normally you'd write `cur = conn.cursor()`, then `cur.execute(sql)`, then pull results with `cur.fetchall()` (all rows), `cur.fetchone()` (one row), or by iterating over it. You don't *see* explicit cursors in your file, and that confuses most beginners, so here's the resolution: `conn.execute(...)` is a shortcut. It quietly creates a cursor, runs your statement on it, and returns it. Your `conn.execute("CREATE INDEX ...")` is doing exactly that. So cursors are present — they're just created implicitly for you.

## Parameterized queries — the `?` and `params=[...]`

This is the single most important practical concept in the file, and it appears in all three query functions. Look at `level1_sources`:

```
WHERE orchestration_key = ? AND month = ?    ...    params=[orch, month]
```

The `?` marks are **placeholders**. You never paste the actual values into the SQL string yourself — you hand them separately as `params`, and SQLite slots them in. There are two reasons this matters, and they're both worth internalizing as a reflex:

First, safety. If you instead built the string with an f-string like `f"... month = '{month}'"`, and `month` ever came from a user, a malicious value could break out of the quotes and rewrite your query — that's SQL injection. With placeholders, the value is *always* treated as data, never as part of the SQL command, so injection is impossible.

Second, correctness. SQLite handles the fiddly type and quoting rules for you: strings get quoted, numbers don't, and Python's `None` becomes SQL `NULL`. Doing that by hand is error-prone.

One subtlety you'll notice: the *table name* in your file **is** dropped in with an f-string — `FROM {TABLE}`. Isn't that the thing I just warned against? It's acceptable here only because `TABLE` is a hardcoded constant *you* control, never user input. And it's actually necessary: the `?` placeholder mechanism only works for *values*, not for *identifiers* like table or column names. So the rule of thumb is: `?` for values, always; inject an identifier with a string only when you fully control it.

## Transactions and `commit()`

SQLite groups changes into **transactions**. When you modify the database — inserting rows, creating an index — those changes are provisional until you call `conn.commit()`. Until then, if the program crashed, they'd vanish. That's why you see `conn.commit()` right after the `CREATE INDEX` line: it makes that change durable. Reads (any `SELECT`) don't change anything, so they never need a commit. You might wonder why there's no commit immediately after `df.to_sql(...)` even though that writes a lot of data — it's because pandas commits its own writes internally. The explicit `conn.commit()` later is there to persist the index you created with raw SQL.

## How pandas plugs in

Two pandas methods in your file lean on the sqlite3 connection to do their work, which is why the division of labor can be hard to see at first.

`df.to_sql(TABLE, conn, if_exists="replace", index=False)` takes your DataFrame and, using the connection, creates the `streams` table and inserts every row. `if_exists="replace"` means "if the table is already there, drop it and rebuild it" — that's what makes re-running the script *idempotent* (you get the same clean result every time, no duplicate rows piling up). `index=False` tells pandas not to write the DataFrame's row-number index as an extra column.

`pd.read_sql_query(sql, conn, params=[...])` is the read direction: pandas runs your `SELECT` through the connection, fetches all the rows, and hands them back as a DataFrame. The `params` argument is what feeds those `?` placeholders. So you get SQL's filtering and aggregation power, but your results land back in the pandas API you already know.

So the three-way split is: **SQLite** stores and queries the data; **sqlite3** is the Python driver that opens the file and runs statements (`connect`, `execute`, `commit`, `close`); and **pandas** is a friendly layer on top that moves bulk data in (`to_sql`) and pulls query results out (`read_sql_query`) as DataFrames.

## The whole lifecycle of your file, in order

Putting it together, `build_db.py` runs this sequence: `connect` opens (or creates) `forecast.sqlite` → `to_sql` loads the CSV into the `streams` table → `execute(CREATE INDEX)` plus `commit` builds and persists an index → `read_sql_query` runs each of the three drill-down `SELECT`s and returns DataFrames → `close` releases the file.

One last concept worth knowing as a beginner, since it's in there: the **index**. `CREATE INDEX ... ON streams (orchestration_key, month, source)` builds a precomputed lookup structure so the database can jump straight to matching rows instead of scanning all 997 of them. Since every one of your queries filters on those exact columns, indexing them is the natural choice. As the comment honestly notes, at 997 rows it makes no measurable difference — but it's the correct habit to build now, because on real data volumes it's the difference between instant and sluggish.

If you want, I can take the next logical step with you — sketching how each of these three functions becomes a FastAPI endpoint, since that's the milestone the comments are pointing toward.

# Deeper: Serving vs. ETL, Complete Flow

This is a great pair of questions, and they're actually the same confusion viewed from two angles. Both dissolve once you see one distinction clearly: *where the data lives and when the computation happens.* Let me build that up carefully, starting from your ETL intuition since that's solid ground.

## Why sqlite3 feels shapeless compared to ETL

In pandas ETL, the data is a *value you hold in your process*. `df = pd.read_csv(...)` pulls the whole thing into RAM, and from then on `df` **is** the data — you can inspect `.shape`, `.head()`, transform it in place. Input and output are both DataFrames sitting in your memory. You can see them.

With `sqlite3`, the data is **not in your process at all**. It's in the file (`forecast.sqlite`), out there on disk. What you hold in Python is never the data itself — it's *handles* and, only at the very end, small *result sets*. That's why you can't picture it: you're used to holding the whole dataset, and now you're holding a doorknob to a room the data lives in.

The one-sentence shift: **in ETL, computation comes to the data (you load everything in and operate on it in Python); in SQLite, the query goes to the data (you send a text description of what you want, the engine computes it inside the file, and only the result rows travel back to you).**

Here's that picture literally:

![alt text](images/image2.png)

## Every sqlite3 call, with its exact input and output shape

Now let's make those arrows concrete. Here is every object you touch, what goes in, and what comes out — no hand-waving.

`sqlite3.connect("forecast.sqlite")` — **in:** a path string. **out:** a `Connection` object. This is *not* data; it's a handle, like the object you get from Python's `open()`. It represents "the door to this database is open." It holds zero rows. Its whole job is to be the thing you run statements through and the thing that owns the open transaction.

`conn.cursor()` — **in:** nothing. **out:** a `Cursor` object. Also not data. Think of it as a read-head or a result-stream: it will run one statement and let you walk through whatever rows that statement produces. (Your demo file never wrote `conn.cursor()` explicitly because `conn.execute(...)` makes one implicitly and hands it back — that's the only reason cursors seemed absent.)

`cursor.execute(sql, params)` — **in:** a SQL string, plus optionally a tuple/list of values for the `?` placeholders. **what happens:** the text is handed to the engine; the engine does all the reading, filtering, and aggregating *inside itself*, in C, against the file. None of that work happens in Python. **out:** it returns the cursor again (for convenience) — crucially, it does **not** return the rows. After `execute`, the rows merely *exist and are ready to be pulled*; they haven't crossed into Python yet.

Then you actually pull the rows, and *this* is where a shape finally appears:

```python
cur = conn.execute("SELECT month, net FROM streams WHERE orchestration_key = ?", (orch,))
cur.fetchone()    # -> one row, as a tuple:  ('2026-01', 152340.5)   (or None when exhausted)
cur.fetchall()    # -> a list of tuples:     [('2026-01', 152340.5), ('2026-02', -8800.0), ...]
cur.fetchmany(50) # -> a list of up to 50 tuples
for row in cur:   # -> yields one tuple at a time (streaming, low memory)
    ...
```

So the fundamental shape of any query result is: **a list of rows, where each row is a tuple, and the tuple's positions line up one-to-one with the columns in your SELECT clause, in order.** If the SELECT says `SELECT month, net`, every row is `(month_value, net_value)`, so `row[0]` is the month and `row[1]` is the net. There is genuinely nothing more to it than that — text describing columns goes in, a list of position-matched tuples comes out.

The values inside those tuples are native Python objects: SQLite's INTEGER becomes `int`, REAL becomes `float`, TEXT becomes `str`, NULL becomes `None`, BLOB becomes `bytes`. So a fetched row is just ordinary Python data you can use anywhere.

### Making the shape less brittle (and JSON-ready)

Positional tuples are fragile — reorder your SELECT and every `row[0]` silently means something different. So in real serving code you almost always set one line:

```python
conn.row_factory = sqlite3.Row
```

After that, fetched rows are `sqlite3.Row` objects you can index *by name*: `row["month"]`, `row["net"]` (and still `row[0]` if you want). Better still, `dict(row)` turns a row into a real dictionary: `{"month": "2026-01", "net": 152340.5}`. Hold onto that — it's the exact shape that becomes JSON in your second question.

How does Python even know the names "month" and "net"? After executing a SELECT, the cursor exposes `cursor.description` — a tuple of column metadata, one entry per output column, with the column name as the first item of each. That's the only place the names live, and it's precisely what pandas reads to label its DataFrame columns.

### Writes have no return shape — they have a side effect

```python
cur = conn.execute("INSERT INTO streams (...) VALUES (?, ?)", (a, b))
cur.lastrowid   # the autogenerated primary key of the row just inserted
cur.rowcount    # how many rows were affected
conn.executemany("INSERT INTO streams (...) VALUES (?, ?)", list_of_tuples)  # bulk insert
```

A write doesn't hand you rows back; its "output" is that the file now contains new data (after `commit()`). `CREATE TABLE` / `CREATE INDEX` go even further — they return nothing and change only the schema.

### The pandas bridge — why your demo hid all of this

Your two pandas calls are just the cursor dance, wrapped:

`pd.read_sql_query(sql, conn, params=...)` literally does: `execute` the SQL → `fetchall` the tuples → read `cursor.description` for the names → assemble a DataFrame with those columns. That's why it *feels* like ETL: pandas runs the whole tuple-stream for you and hands back the familiar DataFrame, so the cursor never surfaces.

`df.to_sql(table, conn, ...)` is the reverse: it issues a `CREATE TABLE` (if needed) and an `executemany` of INSERTs to push every DataFrame row down into the file.

So the division of labor in one line: **SQLite stores and computes; `sqlite3` is the thin Python driver (connect / execute / commit / close) that ships text in and pulls tuples out; pandas is a convenience layer that moves a whole DataFrame in (`to_sql`) and pulls a whole result set out as a DataFrame (`read_sql_query`).** The reason it felt shapeless is that the real "shape" was never a Python value you held — it was a query string going out and a list of tuples coming back.

One last contrast worth naming, because it's the other thing ETL doesn't have: a DataFrame is *eager and complete* — you have all of it at once. A cursor is *lazy* — it's a stream you pull from, and once exhausted it's empty. And the connection is *stateful and long-lived* — it owns an open transaction, which is exactly why `commit()` and `close()` exist here but have no equivalent for a plain DataFrame.

## Your runtime loop is correct — I muddled two timeframes

Now your second question, and I owe you a clarification, because you described the loop **exactly right** and I made it sound like I was contradicting it. I wasn't. I collapsed two different *times* into one sentence. Let me separate them.

Your loop — click → frontend issues an API call → serving layer parses it → serving layer queries the DB → gets rows → serializes to JSON → frontend renders — is the **run-time** loop. It happens live, on every single user interaction, forever, once the thing is deployed. Nothing about it is wrong.

When I said "pre-define the queries," I did **not** mean the queries run ahead of time, and I did **not** mean results are precomputed or cached. I meant something that happens at a completely different time: **build time** — i.e., while you, the developer, are writing the system, before any API or frontend exists. At build time you *author and lock down the set of questions the serving layer will be willing to answer, and the shape of each answer.* That's a design decision, captured in code. `build_db.py` is a build-time artifact: it's you deciding "the serving layer will support exactly these three queries, returning exactly these column shapes." Those same three functions are what your run-time loop will call later.

This maps perfectly onto your ETL experience. You *define* an ETL job's transforms and schema once, at design time; the job *executes* on each scheduled batch. Defining ≠ running. Here it's identical, except the trigger is a user's click instead of a cron schedule: you define the query functions once at build time, and they execute on each click at run time.

Here's the run-time loop with the data shape on every hop — notice it's the same tuples-and-text story from your first question, now stretched across the network:

![alt text](images/image3.png)

## The loop hop by hop, with the shape at each boundary

Walk one real interaction — a user clicking "March" — and watch the shape change at each handoff:

The user clicks the March bar in the chart. The frontend turns that gesture into an **HTTP request**, something like `GET /sources?orch=c0001_2026_3-pre-harvest&month=2026-03`. Shape: a URL with query parameters — just strings.

The serving layer (FastAPI, when you build it) receives that request. Its job is to validate the parameters, decide which query this endpoint maps to (here, `level1_sources`), and call that function with `orch` and `month`. Shape going inward: a SQL string plus a parameter tuple `(orch, month)` — those are your `?` placeholders from the first question.

SQLite runs the parameterized query and returns **rows as tuples** (or `sqlite3.Row` objects), e.g. `[("fertilizer", -3200.0), ("seed", -1150.0), ("subsidy", 4000.0)]`.

The serving layer serializes those rows into **JSON** — `[{"source": "fertilizer", "amount": -3200.0}, ...]`. This is the exact spot where `dict(row)` from the first question earns its keep: rows-of-tuples become a JSON array of objects.

That JSON travels back over HTTP. The frontend deserializes it and **renders**: draws the bars, formats the currency, colors the negative outflows red, and wires up a click handler so that clicking "fertilizer" fires the *next* request (level 2). Then it waits for the next click, and the loop runs again.

## The roles, drawn sharply

What makes this clean is that each layer speaks exactly two "languages" — one to each neighbor — and translates between them, and knows nothing about the layer two steps away:

SQLite knows only SQL-in and rows-out. It has never heard of HTTP, JSON, or a pixel. Give it text, it gives you tuples.

The serving layer is the translator and the gatekeeper. Outward it speaks HTTP/JSON; inward it speaks SQL/rows. It turns requests into query calls, turns rows into JSON, validates inputs, enforces auth, and shapes errors. It owns no rendering and the database owns no web concepts — the serving layer is the membrane between "the web" and "the data."

The frontend speaks clicks/pixels to the human and HTTP/JSON to the serving layer. It turns JSON into something a person sees and turns the person's next gesture into the next request. It has never heard of SQL.

## Why the queries are fixed at build time — and what that has to do with the `?`

Here's the synthesis that ties both of your questions together, and it's the single most important idea.

The frontend is **not** allowed to send arbitrary SQL across the network — that would be a catastrophic security hole (anyone could read or drop anything). So the *structure* of every query is decided by you, in advance, at build time, and frozen into named functions/endpoints. The frontend can only pick from that fixed menu. What the frontend *does* get to supply at run time is the **values** — which orchestration, which month, which source.

That is exactly what the `?` placeholders are for. The query *shape* (`WHERE orchestration_key = ? AND month = ?`) is authored once at build time; the *values* that fill the `?` arrive fresh on every click at run time. So the `?` is literally the seam between the two timeframes you were trying to reconcile: build-time structure on the left of the `=`, run-time data on the right. "Pre-defining the queries" and "the runtime loop fills them in" are not in tension — they're the two ends of the same placeholder.

And to kill one likely ETL-flavored misreading: "pre-define" here means *fix the contract* (the set of allowed queries and the schema of each response). It does **not** mean *precompute or materialize the results* — that's a separate, optional optimization (caching, materialized views) you'd add later only if performance demanded it. Nothing in this design computes answers ahead of the user asking.

## Where `build_db.py` actually sits in all this

The demo file is doing two jobs that will later split apart:

The CSV→SQLite load (`connect`, `to_sql`, the index, `commit`) is a **batch / build step**. It runs once whenever your pipeline produces a fresh CSV — not on the request path. It's the seam between your ETL world and the serving world: ETL produces the file, this step loads it into the queryable store.

The three query functions are a **stand-in for the serving layer's logic**. Right now they print to a terminal so you can verify the shapes with `python build_db.py` — no browser, no network needed. When you add FastAPI, each function moves behind an HTTP route almost unchanged; you wrap it in an endpoint, set `conn.row_factory = sqlite3.Row`, and add the one `dict(row)`→JSON step. The SQL itself barely moves.

So the progression from here is small and concrete: keep the load step as a batch job, lift the three query functions behind three FastAPI endpoints, return JSON instead of printing, then build a frontend whose three interactions call those three endpoints. If you'd like, I can show you one of those functions converted into a FastAPI endpoint end to end — request in, JSON out — so you can see the tuple-to-JSON hop in real code.