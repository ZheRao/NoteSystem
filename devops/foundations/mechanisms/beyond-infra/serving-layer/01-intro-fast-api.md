# Initial Build Path

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

## 3. First GrowLytics backend

Folder:

```text
growlytics-demo/
  data/
    crop_input_streams.csv
  backend/
    main.py
```

Example CSV:

```csv
month,category,amount
2026-05,Fertilizer,420000
2026-06,Seed,85000
2026-07,Chemical,170000
```

Backend:

```python
# backend/main.py

from pathlib import Path

import pandas as pd
from fastapi import FastAPI

app = FastAPI()

DATA_ROOT = Path(__file__).resolve().parents[1] / "data"


@app.get("/")
def home():
    return {"message": "GrowLytics API is running"}


@app.get("/warnings/upcoming-expenses")
def get_upcoming_expenses():
    df = pd.read_csv(DATA_ROOT / "crop_input_streams.csv")

    warnings = (
        df[df["amount"] >= 100000]
        .sort_values("amount", ascending=False)
    )

    return warnings.to_dict(orient="records")
```

Run from inside `backend/`:

```bash
uvicorn main:app --reload
```

Then open:

```text
http://localhost:8000/
```

You should see:

```json
{"message":"GrowLytics API is running"}
```

Then open:

```text
http://localhost:8000/warnings/upcoming-expenses
```

You should see JSON warnings.

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

## 10. What you should learn first

Learn in this order:

```text
1. Run FastAPI locally
2. Create GET endpoints
3. Read CSV and return JSON
4. Use path parameters
5. Use query parameters
6. Split route logic from service logic
7. Add CORS
8. Let React fetch from FastAPI
```

Do **not** start with authentication, database migrations, OAuth, Docker, or Azure.

Your first victory is:

```text
CSV → FastAPI → JSON → browser
```

That is the serving layer being born.
