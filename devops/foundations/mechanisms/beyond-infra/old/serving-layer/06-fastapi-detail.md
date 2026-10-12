# FastAPI Parameters — Path, Query, Body, Header

The reason these categories exist is **not really FastAPI**. FastAPI is exposing the structure already built into HTTP.

Think from first principles: an HTTP request has several places where information can live, and each place communicates a different **meaning**.

Suppose Strata eventually exposes an endpoint for a client's forecast:

```http
PUT /clients/42/forecasts/2026?recalculate=true
Authorization: Bearer abc123
Content-Type: application/json

{
    "acres": 1200,
    "yield": 55,
    "price": 7.25
}
```

There are four fundamentally different kinds of information here.

| Type                 | Example                      | Conceptual meaning                      |
| -------------------- | ---------------------------- | --------------------------------------- |
| **Path parameter**   | `/clients/42/forecasts/2026` | **Which resource?**                     |
| **Query parameter**  | `?recalculate=true`          | **How do I want to operate/query it?**  |
| **Body parameter**   | `{ "acres": 1200, ... }`     | **What data am I sending?**             |
| **Header parameter** | `Authorization: Bearer ...`  | **Metadata/context about this request** |

That's the distinction worth understanding.

## 1. Path parameters identify the resource

Imagine:

```http
GET /clients/42
```

Here `42` isn't merely some input to a Python function. It is part of the **identity/address of the resource**.

```python
@app.get("/clients/{client_id}")
def get_client(client_id: int):
    ...
```

Conceptually:

```text
/clients/42
         ^^
      which client?
```

And you can compose identities:

```text
/clients/42/forecasts/2026
         ^^           ^^^^
       client         year
```

This reads naturally as:

> Give me the 2026 forecast belonging to client 42.

For Strata, you might eventually have routes like:

```text
/clients/{client_id}
/clients/{client_id}/forecasts/{year}
/clients/{client_id}/forecasts/{year}/crop-plan
/clients/{client_id}/forecasts/{year}/debt-schedule
```

The path describes **what thing you're talking about**.


## 2. Query parameters modify the request

Now suppose I ask:

```http
GET /clients/42/forecasts/2026?revision=3
```

`revision=3` doesn't necessarily identify a fundamentally different resource hierarchy. It's a **qualification on what I want returned**.

Other classic examples:

```text
GET /transactions?limit=100
GET /transactions?sort=date
GET /transactions?category=fertilizer
GET /transactions?start_date=2026-01-01
```

Think:

```text
PATH
"What thing?"

/transactions

QUERY
"What subset/version/view of that thing?"

?category=fertilizer&limit=100
```

This is why search, filtering, sorting, pagination, and optional controls commonly become query parameters.

In FastAPI:

```python
@app.get("/transactions")
def get_transactions(
    category: str | None = None,
    limit: int = 100,
):
    ...
```

FastAPI recognizes these as query parameters because they aren't declared in the path.


## 3. Body parameters contain the actual payload

Suppose the frontend needs to save:

```text
x = 10
y = 20
```

You *could* technically design:

```http
POST /values?x=10&y=20
```

But now imagine your real Strata input:

```text
crop
acres
yield
price
production_cost
inventory
allocation_schedule
...
```

Stuffing all of that into the URL becomes absurd:

```text
POST /crop-plan?crop=wheat&acres=1200&yield=55&price=7.25&...
```

Instead HTTP gives you a **body**, which is designed for the payload:

```http
POST /crop-plans

{
    "crop": "wheat",
    "acres": 1200,
    "yield": 55,
    "price": 7.25
}
```

FastAPI + Pydantic make this particularly natural:

```python
class CropPlan(BaseModel):
    crop: str
    acres: float
    yield_: float
    price: float


@app.post("/crop-plans")
def create_crop_plan(crop_plan: CropPlan):
    ...
```

Conceptually:

> **Path:** where should this data go?
> **Body:** what data am I sending there?

That's a very useful distinction for the CRUD system you're building.

For example:

```http
PUT /clients/42/forecasts/2026/crop-plan

{
    "crop": "wheat",
    "acres": 1200,
    "yield": 55
}
```

The path establishes **where**:

```text
client 42
  └── forecast 2026
        └── crop plan
```

The body establishes **what new state/data**:

```text
crop = wheat
acres = 1200
yield = 55
```

## 4. Headers describe the request itself

Headers are different again.

Suppose:

```http
GET /clients/42/forecasts/2026
Authorization: Bearer abc123
```

The authorization token isn't part of the forecast.

It isn't:

```text
forecast data
    authorization = abc123
```

Rather, it describes **the context in which you're making the request**:

> Here is who I am / my authorization credentials.

Headers commonly carry things like:

```http
Authorization: Bearer ...
Content-Type: application/json
Accept: application/json
User-Agent: ...
```

And applications can define their own headers.

So:

```python
from fastapi import Header

@app.get("/forecast")
def get_forecast(
    authorization: str = Header()
):
    ...
```

The mental distinction is:

```text
RESOURCE
    /clients/42/forecast

REQUEST OPTIONS
    ?revision=3

DATA
    {
        "acres": 1200,
        "yield": 55
    }

REQUEST METADATA
    Authorization: Bearer ...
```

## Why not just put everything in one place?

This gets to your actual question.

Technically, HTTP could have been designed so everything was just one giant dictionary:

```json
{
    "client_id": 42,
    "year": 2026,
    "revision": 3,
    "authorization": "abc123",
    "acres": 1200
}
```

But then the semantics become muddy.

What does `client_id` mean?

Is it identifying the resource?

Is it something we're updating?

Is it a filter?

What does `authorization` have to do with the business object?

Separating the request into components creates a **protocol-level vocabulary**:

```text
HTTP REQUEST
│
├── Path
│     └── What resource am I addressing?
│
├── Query string
│     └── How am I selecting/configuring the request?
│
├── Headers
│     └── What metadata/context accompanies the request?
│
└── Body
      └── What payload am I transmitting?
```

And then there is another dimension you just learned about earlier: the **HTTP method**.

```text
METHOD       PATH                         QUERY          HEADERS       BODY
  │            │                            │               │           │
  ↓            ↓                            ↓               ↓           ↓
PUT     /clients/42/forecast/2026     ?recalculate=true   Auth...    {...}
 │              │                            │               │          │
what        which thing?                  options         context    new data
operation?
```

Together, these pieces form the meaning of the request.

So when you write:

```python
@app.put("/clients/{client_id}/forecasts/{year}")
def update_forecast(
    client_id: int,
    year: int,
    data: ForecastInput,
    recalculate: bool = True,
    authorization: str = Header(),
):
```

those aren't just four arbitrary ways FastAPI lets you pass Python arguments.

They're mapping the **structure of an HTTP request** into a Python function:

```text
HTTP                                      Python
────────────────────────────────────────────────────────

PUT                                       @app.put(...)

.../clients/42/forecasts/2026       →     client_id = 42
                                          year = 2026

?recalculate=true                   →     recalculate = True

Authorization: Bearer xyz           →     authorization = ...

{"acres": 1200, ...}                →     data = ForecastInput(...)
```

That's the deeper model I'd keep in your head. Once you see FastAPI as a translator from **HTTP semantics → typed Python function calls**, a lot of its API design starts feeling much less arbitrary.
