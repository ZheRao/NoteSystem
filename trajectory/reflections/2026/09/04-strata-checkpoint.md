# Project Strata — September 5, 2026

## The original innocent idea

Project Strata began with an almost comically simple premise:

> We already have sophisticated cash-flow forecasting models implemented in Excel. Let's turn them into a proper software platform where clients can see clean, interactive forecasts on a website instead of working directly inside spreadsheets.

The original 12–16 month roadmap therefore looked fairly reasonable.

There were basically two big pieces:

1. **Forecast engine**

   * Take forecast inputs.
   * Compute monthly cash-flow forecasts.
   * Preserve lineage so every final number can be traced back to the individual input components that produced it.

2. **Serving layer + frontend**

   * Expose computed forecast information through APIs.
   * Display the results in a clean web application rather than Power BI.

At the time, this sounded difficult but conceptually bounded.

It was not bounded.

---

# Surprise #1: The forecast is not one state

About three weeks into understanding the project—before serious implementation had even started—I discovered a hidden requirement.

Forecast inputs change over time.

If the inputs change, the forecast changes.

And if the forecast changes, users need to understand:

> **Why did this forecast change from the previous forecast?**

That sounds like a product feature.

Architecturally, it completely changes the problem.

The original forecast engine was essentially one-dimensional:

```text
inputs
  ↓
computation
  ↓
outputs
  ↓
lineage back to inputs
```

But that isn't enough if the platform must preserve the evolution of the forecast.

Now the model becomes:

```text
input state 1 → output state 1 → lineage
input state 2 → output state 2 → lineage
input state 3 → output state 3 → lineage
...
```

The system must understand not merely:

> **Which inputs generated this output?**

but:

> **Which version of each input generated this version of the output?**

Suddenly the forecast has another dimension: **revision/state**.

That led to the revision architecture, orchestration rules, source-specific revisions, fixed forecast phases, and the requirement that the engine select the appropriate version of every source for every requested forecast state.

What was originally:

> “Compute the forecast.”

became:

> **“Compute a temporally/versioned forecast while preserving provenance across evolving input states.”**

Somehow, we made that work.

---

# Surprise #2: The spreadsheet was hiding an application architecture

Eventually the engine reached the point where it actually worked.

Great.

Now all I had to do was display the outputs.

Unfortunately, I knew essentially nothing about modern web application architecture.

My programming worldview was still primarily batch-oriented:

```text
input
 ↓
load into Python
 ↓
transform
 ↓
output
```

Pandas ETL made intuitive sense.

A long-lived server waiting for arbitrary requests did not.

Databases did not behave like DataFrames.

HTTP endpoints did not behave like ETL functions.

React state did not behave like a pipeline.

So I initially used Claude heavily to scaffold the FastAPI serving layer and React frontend while I concentrated on understanding the important orchestration and interfaces.

Then came a surprisingly difficult conceptual transition:

```text
ETL worldview

data → program → transformed data
```

became:

```text
application worldview

browser
   ↕
HTTP
   ↕
FastAPI
   ↕
database / forecast outputs
```

The server exists continuously.

Requests arrive independently.

State has ownership and lifetime.

The frontend has its own state.

The database contains persistent state outside the Python process.

The API contract connects these worlds.

And debugging became completely different.

A page fails to render.

Why?

Maybe React is requesting the wrong endpoint.

Maybe the endpoint returned the wrong shape.

Maybe the Pydantic contract is wrong.

Maybe the API logic transformed something incorrectly.

Maybe the underlying forecast output violates an assumed contract.

Maybe something crashed in the terminal twelve stack frames below where the visible failure appeared.

Tracing failures vertically across:

```text
React
 ↓
HTTP
 ↓
FastAPI
 ↓
contracts
 ↓
serving logic
 ↓
forecast output
```

was completely foreign to me.

It felt enormously heavy.

Somehow, that also eventually started making sense.

---

# Four months in: wait, did we actually build the original platform?

Roughly four months into the project:

* the forecast engine was complete enough to generate the required forecasts;
* revision-aware orchestration worked;
* lineage existed;
* monthly inflow/outflow/net values were available;
* category breakdowns could be exposed;
* a FastAPI serving layer existed;
* a React frontend existed;
* the frontend could call APIs and display forecast information;
* users could drill into months and see different stories about forecast results;
* I understood enough of the serving architecture that adding another read-only page no longer felt mysterious.

In other words:

**the original high-level platform vision basically existed as a working prototype.**

Surely this was the moment where things became easier.

Surely.

---

# Surprise #3: “We don't want to configure forecasts in Excel”

Oh.

Right.

Users also need to **change the inputs**.

Stakeholders understandably do not want the future product to depend on manipulating the original Excel workbook.

Therefore Strata now needs an **input configuration system**.

Which means the frontend is no longer read-only.

We now need:

```text
React form
   ↓
POST / PATCH / DELETE
   ↓
validation
   ↓
database mutation
   ↓
persistent state change
   ↓
forecast recomputation
   ↓
updated outputs
```

This changes the risk profile enormously.

With the read-only frontend, AI-generated code was acceptable as scaffolding because I could understand the architecture and verify the outputs.

For writes, that level of understanding is no longer enough.

If a GET endpoint is wrong, somebody sees the wrong value.

If a write path is wrong, the application can **corrupt the state that generates future forecasts**.

I therefore cannot merely control the orchestration.

I need to understand the underlying mechanisms well enough to reason about correctness myself.

So the innocent starting exercise became:

> Let's create a tiny CRUD application that lets users insert, edit and delete records in the `clients` table.

Very reasonable.

Very small.

Right?

---

# Surprise #4: The engine itself now needs surgery

Meanwhile, making the product interactive exposed architectural assumptions in the forecast engine that were acceptable for batch computation but inappropriate for a cloud application.

The engine:

* performs too many intermediate filesystem writes;
* historically recomputed far more than the specific client/year slice that changed;
* was designed around local/batch execution assumptions;
* contains a global linkage-tracking design that may scale badly and become painful as the system becomes incremental;
* still needs stronger database-native input/output boundaries.

The target architecture is increasingly moving toward something like:

```text
client/year configuration changes
        ↓
identify affected slice
        ↓
load required inputs
        ↓
compute primarily in memory
        ↓
persist only required outputs
        ↓
serve resulting state
```

So merely adding an input UI also forces reconsideration of the engine that the input UI feeds.

Naturally.

---

# Surprise #5: Apparently software needs engineering infrastructure too

At roughly the same time, another category of missing systems became obvious.

The prototype has been developed extremely quickly, but a production software system eventually requires things such as:

* formal automated testing;
* contract tests;
* packaging;
* reproducible environments;
* Docker/containerization;
* CI;
* deployment processes;
* logging;
* observability;
* database migrations;
* cloud infrastructure;
* error handling;
* production configuration;
* eventually authentication/authorization and security boundaries.

None of these creates an exciting stakeholder demo.

They nevertheless determine whether the platform becomes reliable software instead of an increasingly impressive prototype held together by one developer's memory.

But the product still needs visible features.

So those architectural debts continue quietly accumulating in the background.

Wonderful.

---

# Surprise #6: CRUD is not “just POST instead of GET”

Back to our supposedly tiny client-management application.

The first mental model was approximately:

```text
GET /clients
```

already works.

Therefore:

```text
POST /clients
DELETE /clients/{id}
PATCH /clients/{id}
```

should just be some additional endpoints.

And then the questions began.

What happens if two users edit the same client?

What if one request reads a value, performs logic in Python, and writes it back after another request has already changed it?

What if two writes arrive simultaneously?

What if a multi-step operation partially succeeds?

What happens if the process fails between statement #1 and statement #2?

What exactly does `COMMIT` guarantee?

What gets locked?

When?

What does `BEGIN IMMEDIATE` actually mean?

What anomalies are possible under concurrency?

Dirty reads?

Non-repeatable reads?

Phantoms?

Lost updates?

Write skew?

What isolation level do we actually need?

What does SQLite guarantee?

Which guarantees come from transactions generally, and which come specifically from SQLite's concurrency-control implementation?

How will those assumptions change when the system eventually moves to PostgreSQL or Azure SQL?

And suddenly:

> **“Build a button that edits a client's name.”**

has opened the door to:

**transaction theory and concurrency control.**

---

# Current side quest: apparently we are reading database theory now

So on Saturday night, September 5, 2026, the immediate project requirement is:

> Build CRUD operations for the `clients` table.

Naturally, this has resulted in studying:

* transactions;
* ACID;
* atomicity;
* transaction boundaries;
* locking;
* isolation;
* dirty reads;
* non-repeatable reads;
* phantom reads;
* lost updates;
* write skew;
* serializability;
* snapshot isolation;
* MVCC;
* two-phase locking;
* optimistic concurrency control;
* retry-on-conflict;
* short transaction lifetimes.

And I have just realized that Chapter 8 of *Designing Data-Intensive Applications, Second Edition* is basically an entire chapter devoted to the exact hole Project Strata has just dropped me into.

So despite currently being on Chapter 3, I am probably making an emergency expedition to Chapter 8.

Because apparently this is my life now.

---

# Things that quietly changed along the way

The technical changes are only part of the story.

My mental models have also changed dramatically.

### DataFrame → database

Originally, I thought about data primarily as something my Python process **holds**.

Now persistent state increasingly feels like something the application **accesses through explicit interfaces and lifecycles**.

### Batch → serving

Originally:

```text
program starts
→ data comes in
→ computation happens
→ output comes out
→ program ends
```

Now:

```text
server stays alive
→ requests arrive
→ request-specific resources are acquired
→ work happens
→ resources are released
→ response leaves
→ repeat
```

### Syntax → concepts

Months ago I needed command-by-command examples of `sqlite3` because every function felt independent.

Now connection lifecycle, transaction lifecycle, state ownership and persistence form a coherent model.

I may forget whether the exact pandas function is:

```python
df.to_sql(...)
```

or something else.

That no longer matters very much.

I know the operation I need to perform and can recover the syntax.

### AI-generated implementation → AI-assisted understanding

At the beginning of the serving-layer work, AI could generate large amounts of code while I understood primarily the orchestration.

Now that mutations and correctness matter, I am deliberately using AI differently:

> generate tutorials;
> generate scenarios;
> explain mechanisms;
> challenge my mental models;
> let me predict outcomes;
> help me understand the system well enough to make the engineering decisions myself.

AI is increasingly becoming a knowledge and feedback amplifier rather than simply an implementation shortcut.

### Low-breadth/high-depth → selective depth across breadth

My natural operating mode is to understand a small number of things extremely deeply.

Strata does not permit that.

It keeps demanding:

```text
Python
Pandas
financial modeling
data lineage
revision systems
SQLite
SQL
FastAPI
HTTP
React
JavaScript
database design
transactions
concurrency
testing
Docker
CI/CD
Azure
architecture
product design
stakeholder communication
...
```

I cannot master each before touching the next.

So the project is forcing a new strategy:

> **scan broadly → understand enough → integrate → identify where correctness requires depth → go deep there.**

Transactions are currently one of those places where depth is justified.

CSS internals probably are not.

That distinction did not exist naturally for me before this project.

---

# The funniest recurring pattern

Almost every stage has looked like this:

```text
"Oh, I just need to solve X."

        ↓

discover hidden assumption

        ↓

"Oh."

        ↓

learn entirely new field

        ↓

build working version

        ↓

finally understand it

        ↓

"This feels pretty natural now."

        ↓

unlock next hidden assumption

        ↓

"Oh for fuck's sake."
```

And then repeat.

---

# September 5, 2026 status

The most important thing to preserve is this:

**Today, none of this feels finished.**

There are enormous holes.

Concurrency is fuzzy.

Threads/event loops/workers are still fuzzy.

The CRUD input system barely exists.

The forecast engine needs architectural refactoring.

Cloud deployment remains ahead.

Testing and CI are incomplete.

There are probably architectural problems I am currently incapable of seeing.

And that is exactly why this checkpoint matters.

Because several months ago:

* SQLite felt incomprehensible;
* serving architecture felt alien;
* APIs felt alien;
* React felt terrifying;
* tracing errors across layers felt foreign;
* the forecast revision problem looked enormous.

Several of those things now feel increasingly ordinary.

The project has repeatedly made me operate beyond my existing mental models, forced me to construct new ones, and then quietly turned yesterday's terror into today's infrastructure.

So when the next hidden requirement appears—and it almost certainly will—the historically correct reaction is probably:

> **“Ah shit. Here we go again.”**

followed eventually by:

> **“Wait, why did this ever seem so difficult?”**

End of checkpoint.

For now.
