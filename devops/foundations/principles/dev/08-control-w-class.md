# Intro

Yes. This is exactly where `class` becomes useful.

Functions are great when logic is isolated. Classes become useful when you need:

```text
shared state + shared contract + swappable behavior
```

For your platform:

```text
core defines the rules
sources implement the specifics
engine orchestrates them consistently
```

## 1. The simplest class idea

A class is a bundle of:

```text
data/state + methods that operate on that state
```

Instead of passing the same variables into every function:

```python
def fetch(url, headers, timeout):
    ...

def parse(response, schema):
    ...

def save(data, path):
    ...
```

You wrap the shared context:

```python
class HttpClient:
    def __init__(self, timeout: int):
        self.timeout = timeout

    def get(self, url: str):
        print(f"GET {url} with timeout={self.timeout}")
```

Usage:

```python
client = HttpClient(timeout=30)
client.get("https://example.com/data")
```

The `__init__` method runs when the object is created. `self` means “this specific instance.”

So:

```python
client.timeout
```

is the stored state for that object.

## 2. A class as a platform contract

A “contract” means:

```text
Every source must expose the same method names,
even if the internal logic differs.
```

Example:

```python
from abc import ABC, abstractmethod


class BaseExtractor(ABC):
    @abstractmethod
    def extract(self) -> list[dict]:
        """Return raw records from the source."""
        pass
```

This says:

```text
Any extractor must have an extract() method.
```

Now a specific source implements it:

```python
class CsvExtractor(BaseExtractor):
    def __init__(self, path: str):
        self.path = path

    def extract(self) -> list[dict]:
        print(f"Reading from CSV: {self.path}")
        return [
            {"id": 1, "name": "Alice"},
            {"id": 2, "name": "Bob"},
        ]
```

And another source:

```python
class ApiExtractor(BaseExtractor):
    def __init__(self, endpoint: str):
        self.endpoint = endpoint

    def extract(self) -> list[dict]:
        print(f"Calling API: {self.endpoint}")
        return [
            {"id": 10, "name": "Remote Item"},
        ]
```

Now your engine can treat both the same:

```python
def run_ingestion(extractor: BaseExtractor):
    records = extractor.extract()
    print(f"Extracted {len(records)} records")
```

Usage:

```python
csv_extractor = CsvExtractor("data/customers.csv")
api_extractor = ApiExtractor("https://example.com/customers")

run_ingestion(csv_extractor)
run_ingestion(api_extractor)
```

This is the core power.

The engine does **not** care whether the source is CSV, API, QBO, Shopify, Salesforce, or something else.

It only cares:

```text
Can you extract records?
```

## 3. Inheritance: shared default behavior + required custom behavior

Suppose every extractor should log start/end consistently.

You can put the invariant in the base class:

```python
from abc import ABC, abstractmethod


class BaseExtractor(ABC):
    def run(self) -> list[dict]:
        self._log_start()
        records = self.extract()
        self._validate(records)
        self._log_end(records)
        return records

    @abstractmethod
    def extract(self) -> list[dict]:
        pass

    def _validate(self, records: list[dict]) -> None:
        if not isinstance(records, list):
            raise TypeError("Extractor must return a list of dictionaries.")

        for record in records:
            if not isinstance(record, dict):
                raise TypeError("Each record must be a dictionary.")

    def _log_start(self) -> None:
        print(f"Starting extractor: {self.__class__.__name__}")

    def _log_end(self, records: list[dict]) -> None:
        print(f"Finished extractor: {self.__class__.__name__}; records={len(records)}")
```

Then child classes only define the source-specific part:

```python
class CsvExtractor(BaseExtractor):
    def __init__(self, path: str):
        self.path = path

    def extract(self) -> list[dict]:
        return [
            {"id": 1, "source": "csv"},
            {"id": 2, "source": "csv"},
        ]
```

Usage:

```python
extractor = CsvExtractor("customers.csv")
records = extractor.run()
```

Important pattern:

```text
run() = invariant orchestration
extract() = source-specific implementation
```

This is very relevant to your platform.

Your `core` can define:

```text
run ingestion
validate output shape
log metadata
write raw payload
handle failure policy
```

Your `sources/qbo` can define:

```text
how to call QBO
how to paginate QBO
how to parse QBO response
```

## 4. The Template Method pattern

This pattern is probably the most useful for you.

The base class defines the skeleton:

```python
class BaseIngestionJob(ABC):
    def run(self) -> None:
        self.before_run()
        raw_data = self.extract()
        normalized_data = self.normalize(raw_data)
        self.validate(normalized_data)
        self.save(normalized_data)
        self.after_run()

    def before_run(self) -> None:
        print("Starting ingestion")

    @abstractmethod
    def extract(self):
        pass

    @abstractmethod
    def normalize(self, raw_data):
        pass

    def validate(self, data) -> None:
        if data is None:
            raise ValueError("Ingestion output cannot be None")

    @abstractmethod
    def save(self, data) -> None:
        pass

    def after_run(self) -> None:
        print("Finished ingestion")
```

Then a concrete implementation:

```python
class LocalJsonIngestionJob(BaseIngestionJob):
    def __init__(self, input_path: str, output_path: str):
        self.input_path = input_path
        self.output_path = output_path

    def extract(self):
        print(f"Reading raw JSON from {self.input_path}")
        return [{"id": 1}, {"id": 2}]

    def normalize(self, raw_data):
        print("Normalizing data")
        return raw_data

    def save(self, data) -> None:
        print(f"Saving {len(data)} records to {self.output_path}")
```

Usage:

```python
job = LocalJsonInestionJob(
    input_path="input.json",
    output_path="bronze/output.json",
)

job.run()
```

Small typo fixed:

```python
job = LocalJsonIngestionJob(
    input_path="input.json",
    output_path="bronze/output.json",
)

job.run()
```

The child class cannot accidentally skip the platform flow, because the main flow lives in `BaseIngestionJob.run()`.

That is the key architectural move.

## 5. Why this enforces coherence

Without a base class, every ingestion function can drift:

```python
pull_customers()
pull_reports()
pull_items()
pull_invoices()
```

Each one may:

```text
log differently
save differently
validate differently
handle errors differently
name files differently
return different shapes
```

With a base class, the platform says:

```text
Every ingestion job must follow the same lifecycle.
```

For example:

```python
class BaseSourceClient(ABC):
    @abstractmethod
    def fetch(self, request):
        pass
```

```python
class BaseIngestionJob(ABC):
    def run(self):
        request_plan = self.build_request_plan()
        raw_payloads = []

        for request in request_plan:
            response = self.client.fetch(request)
            raw_payloads.append(response)

        self.validate_raw_payloads(raw_payloads)
        self.write_bronze(raw_payloads)
```

Then source-specific classes only fill in:

```text
What requests should be made?
How should response pages be interpreted?
Where should output go?
```

## 6. Inheritance vs composition

This is important.

Use **inheritance** for “is a” relationships:

```text
QboExtractor is a BaseExtractor
CsvExtractor is a BaseExtractor
ApiExtractor is a BaseExtractor
```

Use **composition** for “has a” relationships:

```text
QboExtractor has an HttpClient
IngestionJob has a Writer
IngestionJob has a Validator
```

Example:

```python
class HttpClient:
    def get(self, url: str) -> dict:
        print(f"GET {url}")
        return {"status": "ok"}


class ApiExtractor(BaseExtractor):
    def __init__(self, client: HttpClient, endpoint: str):
        self.client = client
        self.endpoint = endpoint

    def extract(self) -> list[dict]:
        response = self.client.get(self.endpoint)
        return [response]
```

This is cleaner than making `ApiExtractor` inherit from `HttpClient`.

Bad:

```python
class ApiExtractor(HttpClient):
    ...
```

Because an extractor is not an HTTP client.

Better:

```python
class ApiExtractor(BaseExtractor):
    def __init__(self, client: HttpClient):
        self.client = client
```

This will matter a lot in your platform.

## 7. Suggested mental model for your repo

Your structure could evolve like this:

```text
platform/
  core/
    http/
      client.py              # generic request machinery
      auth.py                # generic auth interface, not QBO-specific
    ingestion/
      base.py                # BaseIngestionJob
      writers.py             # BronzeWriter
      validators.py          # contract validators
    engine/
      runner.py              # orchestrates jobs

  sources/
    qbo/
      client.py              # QBO-specific API client
      extractors.py          # QBO-specific extractors
      reports.py             # QBO-specific report logic
      entities.py            # QBO-specific raw table logic
```

The core should contain words like:

```text
HttpClient
Request
Response
Extractor
IngestionJob
Writer
Validator
RetryPolicy
AuthProvider
```

The QBO source should contain words like:

```text
realm_id
minorversion
QueryResponse
ProfitAndLossDetail
GeneralLedger
STARTPOSITION
MAXRESULTS
```

That is the boundary.

## 8. A clean generic example

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any


@dataclass
class SourceRequest:
    name: str
    url: str
    params: dict[str, Any] | None = None


@dataclass
class SourceResponse:
    request_name: str
    payload: dict[str, Any]


class BaseSourceClient(ABC):
    @abstractmethod
    def fetch(self, request: SourceRequest) -> SourceResponse:
        pass


class BaseIngestionJob(ABC):
    def __init__(self, client: BaseSourceClient):
        self.client = client

    def run(self) -> None:
        requests = self.build_requests()

        responses = []
        for request in requests:
            response = self.client.fetch(request)
            responses.append(response)

        records = self.parse_responses(responses)
        self.validate(records)
        self.save(records)

    @abstractmethod
    def build_requests(self) -> list[SourceRequest]:
        pass

    @abstractmethod
    def parse_responses(self, responses: list[SourceResponse]) -> list[dict]:
        pass

    def validate(self, records: list[dict]) -> None:
        if not isinstance(records, list):
            raise TypeError("records must be a list")

        for record in records:
            if not isinstance(record, dict):
                raise TypeError("each record must be a dict")

    @abstractmethod
    def save(self, records: list[dict]) -> None:
        pass
```

A fake concrete source:

```python
class FakeApiClient(BaseSourceClient):
    def fetch(self, request: SourceRequest) -> SourceResponse:
        print(f"Fetching {request.url}")
        return SourceResponse(
            request_name=request.name,
            payload={
                "items": [
                    {"id": 1, "name": "alpha"},
                    {"id": 2, "name": "beta"},
                ]
            },
        )


class FakeApiIngestionJob(BaseIngestionJob):
    def build_requests(self) -> list[SourceRequest]:
        return [
            SourceRequest(
                name="fake_items",
                url="https://example.com/items",
                params={"limit": 100},
            )
        ]

    def parse_responses(self, responses: list[SourceResponse]) -> list[dict]:
        records = []

        for response in responses:
            records.extend(response.payload["items"])

        return records

    def save(self, records: list[dict]) -> None:
        print(f"Saving {len(records)} records")
```

Run it:

```python
client = FakeApiClient()
job = FakeApiIngestionJob(client=client)

job.run()
```

Output conceptually:

```text
Fetching https://example.com/items
Saving 2 records
```

## 9. Why this is an upgrade from your current style

Your old style is function-centered:

```text
_pull_raw(...)
_pull_reports(...)
```

The new style is lifecycle-centered:

```text
client = source-specific communication
job = source-specific ingestion process
base class = platform contract
engine = universal orchestration
```

This is much more scalable.

For example, later your engine can do:

```python
jobs = [
    FakeApiIngestionJob(client),
    AnotherSourceIngestionJob(other_client),
]

for job in jobs:
    job.run()
```

The engine does not need to know their internals.

## 10. The key rule

Do not use classes just to “look advanced.”

Use classes when you have one of these:

```text
1. shared lifecycle
2. shared state
3. interchangeable implementations
4. enforced contracts
5. reusable infrastructure
```

Your `data-platform` has all five.

So yes: this is not cosmetic. This is the correct upgrade.

My recommendation:

```text
Lean in — core growth
```

Start with one base class:

```python
BaseIngestionJob
```

Then one generic dependency:

```python
BaseSourceClient
```

Then one concrete fake example before QBO.

Do not overbuild 12 abstractions immediately. Get the contract shape right first.
