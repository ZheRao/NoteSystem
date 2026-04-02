# API Essentials

A public API is just:
> your system exposing a stable contract over the network for other programs to call

Mechanically, you build a server, define routes, validate/authenticate requests, run your logic, and return structured responses.

Let’s make it concrete.

## 1. Smallest mental model

A public API needs 5 things:
1. A **server** listening on the internet
2. **Endpoints** like `/customers` or `/reports/pl`
3. **A request/response contract** for inputs and outputs
4. **Authentication/authorization**
5. **Documentation + versioning**

That is the core.

## 2. Mechanically what happens

Someone else’s code does this:
```http
GET /v1/reports/pl?fiscal_year=2025
Authorization: Bearer abc123
```
Your server:
1. receives the HTTP request
2. checks the token
3. parses query params
4. validates them
5. runs business logic or queries database
6. serializes output to JSON
7. returns response

Example response:
```py
{
  "fiscal_year": 2025,
  "currency": "CAD",
  "rows": [
    {
      "account_id": "4000",
      "account_name": "Grain Sales",
      "amount": 182340.22
    }
  ]
}
```
That is a public API.

## 3. The clean architecture

Do **not** think:
> “I need to expose my internal code”

Think:
> “I need a thin network layer in front of a stable service contract”

Best shape:
```
client
  ↓
HTTP API layer
  ↓
service layer
  ↓
database / files / pipelines / external APIs
```
The API layer should be thin.  
The service layer should hold the real logic.

That separation matters a lot.

## 4. Simplest way to build one in Python

For you, the easiest serious starting point is:
- **FastAPI** for the web server
- **Pydantic** for request/response validation
- **Uvicorn** as the server
- optionally **PostgreSQL** later for persistence

This is the normal modern Python route.

Example:
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI(title="Data Platform API", version="1.0.0")


class PLRow(BaseModel):
    account_id: str
    account_name: str
    amount: float


class PLResponse(BaseModel):
    fiscal_year: int
    currency: str
    rows: list[PLRow]


@app.get("/v1/reports/pl", response_model=PLResponse)
def get_profit_loss(fiscal_year: int):
    if fiscal_year < 2000:
        raise HTTPException(status_code=400, detail="Invalid fiscal_year")

    # pretend this came from your service layer
    rows = [
        PLRow(account_id="4000", account_name="Grain Sales", amount=182340.22)
    ]

    return PLResponse(
        fiscal_year=fiscal_year,
        currency="CAD",
        rows=rows,
    )
```
Run it:
```bash
uvicorn app:app --reload
```
Then users can call:
```
curl "http://127.0.0.1:8000/v1/reports/pl?fiscal_year=2025"
```

## 5. What makes it “public”

Not just that it exists.

It becomes public when:
- it is reachable from the internet
- other people or systems are allowed to call it
- auth is defined
- behavior is documented
- you commit to compatibility expectations

So “public” is partly technical, partly operational.

## 6. The hard part is not the server

The hard part is designing the **contract**.

This is where your QBO pain becomes valuable.

You now know what bad producer design feels like.

So for your own API, aim for:
- flat, explicit fields
- stable naming
- typed outputs
- no ambiguous mixed-purpose objects
- clear error responses
- explicit null behavior
- explicit pagination
- explicit versioning

Bad:
```py
{
  "Row": [
    {
      "type": "Section",
      "Header": {...},
      "Rows": {...},
      "Summary": {...}
    }
  ]
}
```
Better:
```py
{
  "report_type": "profit_and_loss",
  "fiscal_year": 2025,
  "rows": [
    {
      "row_type": "account",
      "section": "Revenue",
      "account_id": "4000",
      "account_name": "Grain Sales",
      "amount": 182340.22,
      "parent_account_id": null,
      "depth": 1
    }
  ]
}
```
That is consumer-oriented.

## 7. Public API design rules you should internalize

### Rule 1: expose semantics, not internals

Do not leak your internal file layout, dataframe quirks, or parser weirdness.

Expose business meaning.

### Rule 2: make invalid states impossible

Use strict validation.  
Reject malformed requests early.

### Rule 3: version from day one

Use `/v1/...`

Not later. Immediately.

### Rule 4: stable field names matter

Changing `accountName` to `account_name` can break clients.

### Rule 5: errors are part of the API

Return consistent status codes and error bodies.

Example:
```py
{
  "error": {
    "code": "INVALID_FISCAL_YEAR",
    "message": "fiscal_year must be >= 2000"
  }
}
```
### Rule 6: design for lists and paging

If an endpoint can return lots of rows, support pagination.

### Rule 7: auth is not optional

Even a simple API key is better than nothing.

## 8. Main pieces you need to decide

Before building, define these.

### A. Who is the consumer?
- your own scripts?
- your future frontend?
- outside companies?
- internal consultants?

This changes design a lot.

### B. What resources exist?

Examples:
- `/clients`
- `/grants`
- `/reports/pl`
- `/eligibility/evaluate`

### C. What operations exist?

Usually:
- `GET` read
- `POST` create/run
- `PUT/PATCH` update
- `DELETE` remove

### D. What auth model?

Common options:
- API keys
- OAuth 2.0
- session/cookie auth for browser apps

For a first serious project, API keys are simplest.

### E. What persistence?
- in-memory for prototype
- files/parquet for internal tools
- PostgreSQL for serious multi-user systems

## 9. Example: your future grant API

You could expose something like:
```
GET  /v1/grants
GET  /v1/grants/{grant_id}
POST /v1/eligibility/check
POST /v1/grants/refresh
```
Example request:
```py
{
  "province": "SK",
  "industry": "Agriculture",
  "employee_count": 18,
  "annual_revenue": 2500000
}
```
Example response:
```py
{
  "matches": [
    {
      "grant_id": "sk-001",
      "grant_name": "Agri Innovation Support",
      "hard_constraint_match": true,
      "confidence": 0.91,
      "reasons": [
        "Province matches Saskatchewan",
        "Industry eligible",
        "Company size within range"
      ]
    }
  ]
}
```
That is a real API product shape.

## 10. How to deploy it publicly

Local FastAPI is not enough.  
To make it public, you need deployment.

Typical path:
1. build app locally
2. containerize with Docker
3. deploy to a cloud host
4. attach domain name
5. enable HTTPS
6. add auth and rate limiting
7. monitor logs/errors

Common hosting options:
- Render
- Railway
- Fly.io
- AWS
- Google Cloud
- Azure

For learning, start with something simple like Render or Railway.  
For long-term serious systems, AWS/GCP/Azure give more control.

## 11. Minimum serious production concerns

Once public, these matter immediately:
- HTTPS
- auth
- rate limiting
- request validation
- logging
- monitoring
- timeouts
- retries
- database connection handling
- secrets management
- versioning
- docs

A public API is not just code.  
It is an operational promise.

## 12. Documentation is part of the product

If people cannot understand your API, it is bad even if the code is perfect.

You need:
- endpoint descriptions
- parameter definitions
- example requests/responses
- auth instructions
- error code meanings
- versioning policy

FastAPI helps here because it auto-generates docs from types.

## 13. Blunt warning

Many people think creating a public API means:
> “wrap my script with a URL”

That is prototype thinking.

A real public API requires:
- stable contracts
- explicit ownership of compatibility
- security
- operational reliability

The code is the easy part.  
The interface discipline is the real work.

## 14. Best learning sequence for you

Given your trajectory, I would do it in this order:
1. build a tiny FastAPI app locally
2. expose 2–3 read-only endpoints
3. use Pydantic models for strict response contracts
4. add API key auth
5. deploy it
6. call it from a separate client script
7. only then add write endpoints

That sequence teaches the right invariants.

## 15. The deepest principle

A public API is you saying:
> “I am freezing a boundary so others can build on top of me”

That is why API design is architecture work.

And that is why your QBO suffering is useful:  
you now understand the cost of a bad boundary.

Lean in — core growth