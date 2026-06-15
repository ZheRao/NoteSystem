# 00 — Intuition: Beyond Infrastructure

## Purpose

This folder documents the layer beyond data infrastructure.

The forecast engine, lineage architecture, medallion layers, contracts, and orchestration system prepare trustworthy data. However, trustworthy data alone is not a client-facing platform.

A real platform needs additional layers:

```text
Data / Forecast Infrastructure
        ↓
Serving Layer
        ↓
Frontend Layer
        ↓
User Experience
```

The goal of `beyond-infra` is to understand how a backend data system becomes an interactive application that users can operate, question, inspect, and trust.

---

## 1. Core System Decomposition

### 1.1 Infrastructure / Data System

The infrastructure layer is responsible for correctness.

It answers:

```text
Can we stand behind the numbers?
```

Responsibilities:

* ingest source data
* validate schemas
* normalize semantics
* compute forecast outputs
* maintain lineage
* preserve revisions
* support deterministic rebuilds
* store outputs in usable formats
* make data queryable by downstream systems

Examples:

```text
crop_plan
output
crop_input_streams
input_output_linkage
orchestration outputs
revision metadata
```

This layer should not primarily care about screens, buttons, pages, or user interface design.

Its job is to produce trustworthy structured data.

---

### 1.2 Serving Layer

The serving layer sits between the data system and the frontend.

It answers:

```text
How does the application expose system capability to users?
```

Responsibilities:

* expose APIs
* read from CSV, Parquet, SQLite, PostgreSQL, or cloud databases
* apply thin business logic
* generate warnings
* filter data by client, year, revision, category, and user role
* return JSON to the frontend
* enforce authentication and authorization
* protect sensitive data
* provide stable contracts for the frontend

Example:

```text
GET /forecast/summary?client_id=c0001&year=2026
GET /warnings/upcoming-expenses?client_id=c0001&revision=post-seeding
GET /lineage/output/{output_key}
GET /revisions/compare?base=rev1&target=rev2
```

The serving layer should generally be thin.

It should not redo the entire forecast engine. It should consume trusted forecast outputs and expose them through controlled APIs.

---

### 1.3 Frontend Layer

The frontend layer is what the user interacts with.

It answers:

```text
How does the user understand and operate the system?
```

Responsibilities:

* display forecast summaries
* show warnings and alerts
* provide filters and navigation
* allow drill-through into lineage
* compare revisions
* visualize monthly cash flow
* guide user attention
* make the platform feel usable

The frontend should not be responsible for deep forecast logic.

It should ask the serving layer for data, then render that data clearly.

---

## 2. Local Development Mental Model

A local platform can run entirely on one machine.

```text
Your PC
│
├── Frontend server
│   └── React app at http://localhost:5173
│
├── Backend server
│   └── FastAPI app at http://localhost:8000
│
└── Local data
    └── CSV / Parquet / SQLite files
```

The browser loads the frontend.

The frontend calls the backend.

The backend reads local data.

The backend returns JSON.

The frontend displays the result.

```text
Browser
   ↓
React frontend
   ↓ fetch()
FastAPI API
   ↓
CSV / SQLite / forecast outputs
   ↓
FastAPI returns JSON
   ↓
React renders UI
```

Local development is conceptually the same as cloud deployment.

Local:

```text
localhost:5173
localhost:8000
local files
```

Cloud:

```text
app.growlytics.com
api.growlytics.com
cloud database / storage
```

The hosting changes. The architecture pattern remains.

---

## 3. What Power BI Can and Cannot Do

Power BI is useful for reporting.

It can show:

* tables
* charts
* filters
* summaries
* historical trends
* financial statements

But Power BI becomes limited when the goal shifts from reporting to platform behavior.

A platform may need:

* warning centers
* custom workflows
* interactive drill-through
* revision comparison
* lineage explanation
* guided user experiences
* role-specific views
* application-like navigation
* API-driven behavior
* future writeback and approvals

Power BI visualizes data.

A platform operationalizes data.

---

## 4. Initial GrowLytics Platform Target

The first goal is not to build the full production system.

The first goal is to prove:

```text
Can the forecast engine drive a useful application experience?
```

Minimum viable local platform:

```text
CSV / SQLite
    ↓
FastAPI
    ↓
JSON
    ↓
React
    ↓
Browser UI
```

Initial pages:

### 4.1 Forecast Overview

Shows:

* cash position by month
* total revenue
* total expenses
* debt service
* ending cash position

### 4.2 Warning Center

Shows:

* upcoming large expenses
* negative cash months
* unusual month-over-month changes
* missing assumptions
* major revision impacts

### 4.3 Lineage Explorer

Allows user to click a number and inspect:

```text
frontend warning
    ↓
orchestration key
    ↓
output key
    ↓
input_output_key
    ↓
source assumptions
```

### 4.4 Revision Compare

Allows user to compare:

```text
revision A vs revision B
```

Questions:

* What changed?
* Which assumptions changed?
* What was the financial impact?
* Which outputs were affected?

---

## 5. Tool Map

### 5.1 HTML

HTML defines the structure of a web page.

It answers:

```text
What elements exist on the page?
```

Examples:

```html
<h1>Forecast Overview</h1>
<button>View Lineage</button>
<table></table>
```

Learning priority:

```text
High, but shallow.
```

You do not need to master raw HTML deeply before React. You need to understand basic page structure, elements, forms, buttons, tables, and semantic layout.

Use case in GrowLytics:

* page headings
* tables
* buttons
* forms
* cards
* navigation

---

### 5.2 CSS

CSS controls visual presentation.

It answers:

```text
How should the page look?
```

Examples:

* spacing
* colors
* fonts
* grid layouts
* responsive design
* hover effects

Learning priority:

```text
Medium.
```

For the first prototype, visual perfection is not the goal. Clarity is.

Use case in GrowLytics:

* warning cards
* table layout
* dashboard panels
* navigation sidebar
* readable spacing

Possible future tools:

* Tailwind CSS
* CSS Modules
* Styled Components
* MUI
* Chakra UI
* Ant Design

MUI is a popular React component library that provides ready-made interface components, which can help move faster than building every button/table/card from scratch.

---

### 5.3 JavaScript

JavaScript is the language of the browser.

It answers:

```text
How does the frontend behave?
```

Responsibilities:

* react to user clicks
* call APIs
* handle returned JSON
* update screen state
* manage forms and filters

Learning priority:

```text
Very high.
```

React is written in JavaScript or TypeScript. You need enough JavaScript to understand:

* variables
* functions
* arrays
* objects
* imports/exports
* promises
* async/await
* fetch
* map/filter/reduce

Use case in GrowLytics:

```javascript
const response = await fetch("http://localhost:8000/warnings");
const data = await response.json();
```

---

### 5.4 TypeScript

TypeScript is JavaScript with static types.

It answers:

```text
What shape should this data have?
```

Example:

```typescript
type Warning = {
  month: string;
  category: string;
  amount: number;
  severity: string;
};
```

Learning priority:

```text
Medium now, high later.
```

For the very first prototype, JavaScript is acceptable. For a serious platform, TypeScript becomes valuable because API contracts and frontend data structures need to stay reliable.

Use case in GrowLytics:

* define warning shapes
* define forecast summary types
* define lineage response types
* reduce frontend bugs

Roadmap position:

```text
After basic React works.
```

---

### 5.5 React

React is a library for building user interfaces from components. React’s official docs describe it as a library for web and native user interfaces built from individual pieces called components.

It answers:

```text
Given this data, what should the screen look like?
```

Core concepts:

* components
* props
* state
* effects
* conditional rendering
* lists
* event handlers

Example:

```jsx
function WarningCard({ warning }) {
  return (
    <div>
      <h3>{warning.category}</h3>
      <p>{warning.month}</p>
      <p>{warning.amount}</p>
    </div>
  );
}
```

Learning priority:

```text
Very high, after FastAPI endpoints exist.
```

Use case in GrowLytics:

* dashboard pages
* warning center
* lineage drill-through
* revision comparison
* filters and selectors

Minimum concepts needed:

```text
components
props
useState
useEffect
fetch
map
conditional rendering
```

Do not start with advanced React.

Avoid initially:

* Redux
* Next.js
* complex state libraries
* server-side rendering
* advanced routing
* animation libraries

---

### 5.6 Node.js

Node.js runs JavaScript outside the browser.

It answers:

```text
How do frontend tools run on my machine?
```

For this project, Node.js is mainly needed because React tooling uses it.

When you run:

```bash
npm run dev
```

Node starts the frontend development server.

Learning priority:

```text
Medium.
```

You do not need to become a Node backend developer right away.

Use case in GrowLytics:

* install frontend packages
* run React development server
* build frontend assets
* manage frontend dependencies

Key tools:

* npm
* package.json
* Vite
* node_modules

---

### 5.7 Vite

Vite is a modern frontend build tool commonly used to create React apps.

It answers:

```text
How do I run and build the frontend during development?
```

Use case:

```bash
npm create vite@latest frontend
npm run dev
```

Local frontend usually runs at:

```text
http://localhost:5173
```

Learning priority:

```text
High for setup, shallow for internals.
```

You need to know how to create, run, and build a React app. You do not need to master bundler internals immediately.

---

### 5.8 FastAPI

FastAPI is a Python web framework for building APIs using type hints. Its official docs describe it as a modern, high-performance framework for building APIs with Python based on standard Python type hints.

It answers:

```text
How does the frontend request data from Python?
```

Example:

```python
@app.get("/warnings")
def get_warnings():
    return [{"message": "Large fertilizer expense"}]
```

Learning priority:

```text
Highest immediate priority.
```

This is the most natural bridge from your current Python/data background to platform development.

Use case in GrowLytics:

* expose forecast summary
* expose warning center
* expose lineage explorer
* expose revision comparison
* eventually enforce authentication and role permissions

Minimum concepts needed:

* `FastAPI()`
* route decorators
* GET endpoints
* query parameters
* path parameters
* Pydantic models
* CORS
* service-layer separation
* error responses
* dependency injection later

---

### 5.9 Uvicorn

Uvicorn is the server process that runs FastAPI locally.

It answers:

```text
What program actually listens for HTTP requests?
```

Command:

```bash
uvicorn main:app --reload
```

Meaning:

```text
main = main.py
app = FastAPI object inside main.py
--reload = restart when code changes
```

Learning priority:

```text
High, but shallow.
```

Use case:

* run local API at `localhost:8000`
* test backend endpoints
* serve JSON to React

---

### 5.10 Pydantic

Pydantic validates and structures data in FastAPI.

It answers:

```text
What shape should request and response data have?
```

Example:

```python
from pydantic import BaseModel

class Warning(BaseModel):
    month: str
    category: str
    amount: float
    severity: str
```

Learning priority:

```text
High.
```

Use case in GrowLytics:

* enforce response schemas
* validate request payloads
* document API contracts
* reduce frontend/backend mismatch

---

### 5.11 SQL

SQL is the language for querying relational databases.

It answers:

```text
How do I retrieve structured data efficiently?
```

Example:

```sql
SELECT *
FROM crop_input_streams
WHERE amount > 100000;
```

Learning priority:

```text
High.
```

You already understand dataframes. SQL gives you the query language used by most production databases.

Use case in GrowLytics:

* read forecast outputs
* filter by client/year/revision
* join lineage tables
* query warning candidates
* support future application state

---

### 5.12 SQLite

SQLite is a lightweight database stored in a local file.

It answers:

```text
How can I have a real database locally without setting up a server?
```

Use case:

```text
growlytics.db
```

Good for:

* local proof of concept
* small app state
* warnings
* comments
* user preferences
* scenario metadata

Learning priority:

```text
High for local platform.
```

SQLite is a strong next step after CSV because it introduces database thinking without cloud complexity.

---

### 5.13 PostgreSQL

PostgreSQL is a production-grade relational database.

It answers:

```text
What database should a serious web application use?
```

Learning priority:

```text
Medium now, high later.
```

Use case in GrowLytics later:

* production app database
* user accounts
* client metadata
* permissions
* saved scenarios
* warning statuses
* audit logs

Roadmap position:

```text
After local FastAPI + SQLite works.
```

---

### 5.14 DuckDB

DuckDB is an analytical database that can query local files such as CSV and Parquet efficiently.

It answers:

```text
How do I query analytical forecast outputs without loading everything manually into pandas?
```

Use case in GrowLytics:

* query partitioned forecast outputs
* query Parquet files
* perform analytical joins
* support local serving over file-based outputs

Learning priority:

```text
High for file-based forecast outputs.
```

DuckDB is especially useful if the forecast engine writes Parquet/CSV outputs and the serving layer needs to read them efficiently.

---

### 5.15 MongoDB

MongoDB is a document database.

It answers:

```text
How do I store flexible JSON-like documents?
```

Use case:

* semi-structured data
* flexible documents
* event logs
* loosely structured application data

Learning priority:

```text
Low for GrowLytics initial platform.
```

For GrowLytics, relational structure and lineage matter heavily. SQL databases are likely a better first fit than MongoDB.

MongoDB is worth knowing, but it is not the first database I would choose for this platform.

---

### 5.16 Django

Django is a full-stack Python web framework.

It answers:

```text
How do I build a complete web application with many batteries included?
```

Django includes:

* routing
* ORM
* admin panel
* authentication
* templates
* forms
* migrations

Learning priority:

```text
Medium later.
```

FastAPI is better for your immediate goal because you need an API layer over existing Python data logic.

Django may be useful later if you want:

* built-in admin interfaces
* traditional web application structure
* mature authentication patterns
* database-backed business apps

For now:

```text
FastAPI first.
Django later for comparison.
```

---

### 5.17 API Development

API development is the design of controlled interfaces between systems.

It answers:

```text
How should software components communicate?
```

Key ideas:

* endpoints
* HTTP methods
* status codes
* request payloads
* response schemas
* versioning
* error handling
* pagination
* filtering
* authentication
* authorization

Use case in GrowLytics:

```text
Frontend should not read raw forecast files.
Frontend should call APIs.
```

Example:

```text
Bad:
React reads crop_input_streams.csv directly.

Good:
React calls GET /warnings/upcoming-expenses.
```

Learning priority:

```text
Extremely high.
```

---

### 5.18 REST

REST is the most common API style for web applications.

It answers:

```text
How do I model resources through URLs and HTTP methods?
```

Examples:

```text
GET /clients
GET /clients/c0001
GET /clients/c0001/forecast
GET /outputs/{output_key}/lineage
POST /scenarios
```

Learning priority:

```text
Very high.
```

REST should be learned before GraphQL.

---

### 5.19 GraphQL

GraphQL is an API query language where the client asks for exactly the data shape it wants.

It answers:

```text
How can the frontend request flexible nested data from the backend?
```

Potential use case:

```graphql
query {
  warning(id: "w001") {
    amount
    month
    lineage {
      outputKey
      inputs {
        inputOutputKey
        source
      }
    }
  }
}
```

Learning priority:

```text
Low now, medium later.
```

GraphQL can be powerful for complex frontend data requirements, but it adds complexity.

For GrowLytics:

```text
Start with REST.
Consider GraphQL only after REST pain becomes obvious.
```

---

### 5.20 OWASP API Security

OWASP API Security is a framework for understanding common API risks. The OWASP API Security Top 10 includes risks such as Broken Object Level Authorization, where APIs expose object IDs and fail to check whether the caller should access the object.

It answers:

```text
How can APIs be abused?
```

Critical risks for GrowLytics:

* user accesses another client’s data
* user changes `client_id=c0001` to `client_id=c0002`
* user accesses outputs they should not see
* endpoint exposes too much data
* weak authentication
* missing authorization checks
* excessive data returned
* insecure third-party integrations

Learning priority:

```text
High, after basic APIs work.
```

Important principle:

```text
Authentication answers: Who are you?
Authorization answers: What are you allowed to access?
```

GrowLytics must eventually enforce both.

---

### 5.21 Web Application Security

OWASP also maintains a broader Web Application Top 10, described as a standard awareness document for critical web application security risks.

Learning priority:

```text
Medium now, high before production.
```

Topics:

* injection
* broken authentication
* access control
* security misconfiguration
* vulnerable dependencies
* logging and monitoring failures

For local prototype, security is not the first blocker.

For client-facing production, security becomes mandatory.

---

### 5.22 Jest

Jest is a JavaScript testing framework.

It answers:

```text
How do I test frontend logic?
```

Use case:

* test formatting functions
* test warning display logic
* test utility functions
* test simple components

Learning priority:

```text
Low now, medium later.
```

For React testing, also learn:

* React Testing Library
* Vitest

Since Vite-based projects often use Vitest naturally, Vitest may be more convenient than Jest for a new React/Vite project.

---

### 5.23 Pytest

Pytest is Python’s most common testing framework.

It answers:

```text
How do I test backend logic?
```

Learning priority:

```text
Very high.
```

Use case in GrowLytics:

* test warning rules
* test lineage retrieval
* test API responses
* test edge cases
* test permissions later

For your background, backend tests should come earlier than frontend tests.

---

### 5.24 GitHub

GitHub is a platform for hosting Git repositories and collaborating on code.

It answers:

```text
Where does the code live, and how do changes get reviewed?
```

Learning priority:

```text
High.
```

Use case:

* source control
* branches
* pull requests
* issues
* documentation
* project history
* CI/CD with GitHub Actions

You should document architectural decisions directly in the repo.

---

### 5.25 GitHub Actions

GitHub Actions runs automated workflows.

It answers:

```text
What happens automatically when code changes?
```

Use case:

* run tests on push
* lint code
* build frontend
* check formatting
* eventually deploy app

Learning priority:

```text
Medium.
```

Roadmap position:

```text
After backend/frontend skeleton exists.
```

---

### 5.26 Docker

Docker packages applications into containers. Docker’s docs describe a container as a runnable instance of an image that can be started, stopped, moved, connected to networks, and attached to storage.

It answers:

```text
How do I make the app run consistently across machines?
```

Use case:

* package FastAPI backend
* package frontend
* run PostgreSQL locally
* avoid “works on my machine” issues
* prepare for cloud deployment

Learning priority:

```text
Medium now, high before deployment.
```

Do not start here.

First make the app work locally without containers.

Then containerize.

---

### 5.27 Docker Compose

Docker Compose runs multiple containers together.

It answers:

```text
How do I run backend + frontend + database together?
```

Example future local stack:

```text
FastAPI container
React container
PostgreSQL container
Redis container
```

Learning priority:

```text
Medium later.
```

Roadmap position:

```text
After FastAPI + React + SQLite/PostgreSQL basics.
```

---

### 5.28 NGINX

NGINX is a web server and reverse proxy.

It answers:

```text
How does external traffic get routed to the right internal service?
```

Example:

```text
api.growlytics.com → FastAPI
growlytics.com → React frontend
```

Use case:

* reverse proxy
* HTTPS termination
* static file serving
* routing
* load balancing

Learning priority:

```text
Low now, medium before production deployment.
```

You do not need NGINX for the first local prototype.

---

### 5.29 Redis

Redis is an in-memory data store.

It answers:

```text
How do I store fast temporary data?
```

Use cases:

* caching
* background job queues
* sessions
* rate limiting
* temporary computation results

Learning priority:

```text
Low now, medium later.
```

Potential GrowLytics use:

* cache expensive forecast summaries
* store background task status
* speed up repeated API calls
* queue long-running computations

Do not use Redis until you have a clear performance or background-job need.

---

### 5.30 Background Jobs

Some work should not happen inside a normal API request.

Example:

```text
User clicks rebuild forecast.
Forecast rebuild takes 3 minutes.
```

Bad:

```text
API request waits 3 minutes.
```

Better:

```text
API starts background job.
Frontend shows job status.
```

Tools:

* Celery
* RQ
* Dramatiq
* FastAPI BackgroundTasks
* cloud queues later

Learning priority:

```text
Low now, high later for rebuild workflows.
```

---

### 5.31 Authentication

Authentication answers:

```text
Who are you?
```

Examples:

* username/password
* OAuth
* SSO
* magic links
* Microsoft login
* Google login

Learning priority:

```text
Medium now, high before any real users.
```

For local prototype, skip authentication.

For production, it is mandatory.

Potential tools:

* Auth0
* Clerk
* Firebase Auth
* Microsoft Entra ID
* Azure AD B2C
* Django auth
* FastAPI Users

---

### 5.32 Authorization

Authorization answers:

```text
What are you allowed to access?
```

This is more important and more difficult than authentication.

Example risk:

```text
GET /clients/c0001/forecast
```

A user changes URL to:

```text
GET /clients/c0002/forecast
```

If the API returns c0002 data, the system is broken.

Learning priority:

```text
High before production.
```

GrowLytics will need:

* client-level access
* role-level access
* employee-level access
* admin vs viewer roles
* possibly field/farm/entity-level permissions

---

### 5.33 Firebase

Firebase is a Google-backed app platform.

It can provide:

* authentication
* database
* hosting
* serverless functions
* file storage

Learning priority:

```text
Low for GrowLytics.
```

Firebase is useful for fast consumer-app prototypes, but GrowLytics likely needs stronger relational data modeling, lineage, and backend control.

Good to know. Not first priority.

---

### 5.34 Supabase

Supabase is an open-source Firebase-like platform built around PostgreSQL.

It provides:

* PostgreSQL database
* authentication
* storage
* APIs
* edge functions

Learning priority:

```text
Medium later.
```

It may be more relevant than Firebase because PostgreSQL fits structured business applications better.

Potential use:

* quick authenticated app backend
* prototype app state
* hosted Postgres

---

### 5.35 Cloud Platforms

Eventually, local development must become deployed infrastructure.

Relevant platforms:

* Azure
* AWS
* Google Cloud
* Render
* Railway
* Fly.io
* Vercel
* Netlify

For GrowLytics, Azure may be strategically relevant because Microsoft ecosystem alignment is common in business settings.

Learning priority:

```text
Low now, high after local proof of concept.
```

Do not pay for cloud before local architecture works.

---

### 5.36 Azure-Specific Future Tools

Potential Azure tools:

* Azure App Service
* Azure Container Apps
* Azure Functions
* Azure Static Web Apps
* Azure SQL Database
* Azure PostgreSQL
* Azure Blob Storage
* Azure Data Lake Storage
* Azure Key Vault
* Azure Monitor
* Microsoft Entra ID

Learning priority:

```text
Later.
```

First understand the local version.

Cloud is the same architecture with managed hosting, identity, storage, security, monitoring, and scaling.

---

### 5.37 Internet and Networking

Networking explains how systems talk.

Core concepts:

* IP address
* DNS
* ports
* HTTP
* HTTPS
* request/response
* headers
* cookies
* CORS
* TLS
* firewalls
* reverse proxies

Learning priority:

```text
High, but practical.
```

Immediate concepts:

```text
localhost
port
HTTP method
URL path
query string
request
response
JSON
CORS
```

Future concepts:

```text
DNS
HTTPS
TLS certificates
reverse proxy
load balancer
VNet
firewall
```

---

## 6. Additional Tools To Be Aware Of

### 6.1 npm

JavaScript package manager.

Used to install frontend dependencies.

### 6.2 pnpm

Alternative package manager to npm.

Often faster and cleaner.

Not required initially.

### 6.3 ESLint

Finds JavaScript/TypeScript code issues.

### 6.4 Prettier

Automatically formats frontend code.

### 6.5 Ruff

Fast Python linter/formatter.

Useful for backend quality.

### 6.6 SQLAlchemy

Python ORM / database toolkit.

Useful when FastAPI talks to SQL databases.

### 6.7 SQLModel

A library combining SQLAlchemy and Pydantic patterns, often used with FastAPI.

Useful later for database-backed FastAPI apps.

### 6.8 Alembic

Database migration tool.

Useful when schema changes need to be versioned.

### 6.9 OpenAPI

API specification format.

FastAPI automatically generates OpenAPI docs.

Important for frontend/backend contracts.

### 6.10 Postman / Insomnia / Bruno

API testing tools.

Useful for manually calling endpoints.

FastAPI `/docs` can cover early needs.

### 6.11 Playwright

End-to-end browser testing tool.

Useful later for testing full user workflows.

### 6.12 TanStack Query

React library for server-state management.

Useful when frontend makes many API calls.

Not needed for first page, useful later.

### 6.13 React Router

Handles frontend navigation.

Useful when you have multiple pages:

```text
/overview
/warnings
/lineage
/revisions
```

### 6.14 Recharts / Plotly / ECharts

Charting libraries.

Useful for financial visualizations.

### 6.15 Component Libraries

Useful for faster UI development.

Options:

* MUI
* Chakra UI
* Ant Design
* Mantine
* shadcn/ui

Do not spend too long choosing. Pick one when raw HTML/CSS slows progress.

### 6.16 Logging

Backend should log important events.

Python tools:

* logging
* structlog
* loguru

Future production needs:

* request logs
* error logs
* audit logs
* security logs

### 6.17 Observability

Observability answers:

```text
What is the system doing in production?
```

Tools:

* OpenTelemetry
* Prometheus
* Grafana
* Azure Monitor
* Sentry

Later priority.

### 6.18 Secrets Management

Secrets include:

* API keys
* database passwords
* tokens
* connection strings

Local:

```text
.env files
```

Production:

```text
Azure Key Vault
AWS Secrets Manager
environment variables
```

Do not hardcode secrets.

---

## 7. Recommended Learning Roadmap

### Phase 0 — Mental Model

Goal:

```text
Understand how frontend, backend, and data communicate.
```

Learn:

* localhost
* ports
* HTTP requests
* JSON
* browser dev tools
* API endpoints
* CORS

Build:

```text
GET /health
```

---

### Phase 1 — FastAPI Serving Layer

Goal:

```text
Expose forecast data through Python APIs.
```

Learn:

* FastAPI basics
* route decorators
* query parameters
* path parameters
* Pydantic models
* error handling
* service-layer separation

Build:

```text
GET /forecast/summary
GET /warnings/upcoming-expenses
GET /lineage/{output_key}
```

Data source:

```text
CSV first
SQLite or DuckDB next
```

---

### Phase 2 — React Frontend Basics

Goal:

```text
Display API data in a browser.
```

Learn:

* JavaScript basics
* components
* props
* state
* effects
* fetch
* conditional rendering
* lists
* simple CSS

Build:

```text
Forecast Overview page
Warning Center page
```

---

### Phase 3 — Application Experience

Goal:

```text
Move from dashboard to platform.
```

Learn:

* React Router
* component structure
* frontend state
* UI component library
* charting library

Build:

```text
Lineage drill-through
Revision comparison
Warning detail page
```

---

### Phase 4 — Data Persistence

Goal:

```text
Move beyond read-only files.
```

Learn:

* SQLite
* SQL
* SQLAlchemy or SQLModel
* migrations later
* PostgreSQL later

Build:

```text
saved warnings
comments
scenario metadata
user preferences
```

---

### Phase 5 — Testing and Quality

Goal:

```text
Make the system safe to change.
```

Learn:

* pytest
* FastAPI TestClient
* frontend testing with Vitest or Jest
* API contract tests
* linting and formatting

Build:

```text
tests for warning rules
tests for lineage retrieval
tests for API schemas
```

---

### Phase 6 — Security Foundations

Goal:

```text
Avoid dangerous application design mistakes.
```

Learn:

* authentication
* authorization
* OWASP API Security Top 10
* OWASP Web Top 10
* CORS
* rate limiting
* secret management
* least privilege

Build:

```text
mock user roles
client-level access checks
permission guard functions
```

---

### Phase 7 — Containerization

Goal:

```text
Make the app portable.
```

Learn:

* Docker
* Dockerfile
* Docker Compose
* environment variables
* local Postgres container

Build:

```text
docker-compose up
```

running:

```text
frontend
backend
database
```

---

### Phase 8 — Deployment

Goal:

```text
Make the app accessible outside your machine.
```

Learn:

* Azure App Service or Container Apps
* Azure Static Web Apps
* database hosting
* HTTPS
* DNS
* logging
* monitoring
* CI/CD

Build:

```text
deployed GrowLytics prototype
```

---

## 8. What To Avoid Initially

Avoid starting with:

* Kubernetes
* microservices
* GraphQL
* Redis
* NGINX
* OAuth
* cloud deployment
* complex frontend state management
* perfect UI design
* production authentication
* multi-tenant security implementation

These matter later.

They are not needed to prove the first platform slice.

---

## 9. First Concrete Build Target

The first real milestone:

```text
CSV forecast output
    ↓
FastAPI endpoint
    ↓
JSON response
    ↓
React page
    ↓
warning cards displayed in browser
```

Example endpoint:

```text
GET /warnings/upcoming-expenses?client_id=c0001&year=2026&threshold=100000
```

Example frontend:

```text
Warning Center

High Severity
- Fertilizer: $420,000 due May 2026
- Chemical: $170,000 due July 2026
```

Once this works, the conceptual wall breaks.

The platform exists.

Small, ugly, local, incomplete — but real.

---

## 10. Guiding Principle

Do not learn tools as isolated technologies.

Learn each tool only when it answers a concrete platform question.

```text
Need to expose Python logic? Learn FastAPI.
Need to display returned data? Learn React.
Need to store app state? Learn SQLite.
Need consistent deployment? Learn Docker.
Need real users? Learn authentication.
Need production hosting? Learn Azure.
Need secure APIs? Learn OWASP.
```

The goal is not to collect technologies.

The goal is to build a trustworthy client-facing application on top of reliable data infrastructure.

The infrastructure makes the numbers trustworthy.

The serving layer makes the numbers accessible.

The frontend makes the numbers understandable.

The platform makes the numbers actionable.


# 00a — Intuition (Refinement): Timeframes, Invariants, and Required Scripts

## How this relates to `00-intuition.md`

This document sharpens the *conceptual spine* of `00-intuition.md` and adds the
parts that were missing once the serving layer was understood in detail.

It refines, rather than replaces:

```text
refines  → Section 1 (layer decomposition)
refines  → Section 2 (local development mental model)
refines  → Section 9 (first concrete build target)
adds     → System invariants
adds     → Build-time vs run-time timeframes
adds     → Universal module / file structure
adds     → Required scripts (name, purpose, I/O, language, details)
```

The Tool Map (Section 5), Additional Tools (Section 6), Roadmap (Section 7),
and What To Avoid (Section 8) remain valid as written and are not repeated here.

The central correction this document makes:

```text
The original doc treats Infrastructure → Serving → Frontend as three places.
They are also two times.
Everything left of the store happens in batch (build time).
Everything right of the store happens per request (run time).
They meet at exactly one artifact: the store.
```

---

## A. The Two Timeframes

The single most important distinction the original doc was missing is *when*
things happen. The three layers are not only three components; they straddle
two completely different timeframes with two different triggers.

```text
BUILD TIME (batch)                RUN TIME (per request)
triggered by new forecast data    triggered by a user click
writes the store                  only reads the store
runs occasionally                 runs constantly
```

These never share a trigger. A user click must never cause a rebuild; a
rebuild must never sit on the request path.

The store is the pivot where the two timeframes meet:

```text
forecast engine output                              user
        │                                            │
        ▼  (build time, batch)                       ▼  (run time, click)
   build_serving_store.py                         frontend
        │ writes                                     │ HTTP request
        ▼                                            ▼
   ┌──────────────────────── the store ────────────────────────┐
   │  written by ONE batch job · read by MANY requests          │
   │  SQLite file now → managed database later                  │
   └────────────────────────────────────────────────────────────┘
        ▲ reads
        │
   serving layer (api.py → queries.py)
```

This resolves the "can we pre-define the queries?" confusion: **defining** the
queries happens at build time (you author the question surface in code);
**executing** them happens at run time (the click loop fills in values). Same
functions, two timeframes — exactly like an ETL job whose transforms are
*defined* once and *run* on each batch.

---

## B. The Three Layers, Sharpened (data shapes at each hop)

The original 1.1–1.3 described *responsibilities*. The missing piece is the
*shape of the data* as it crosses each boundary. Each layer speaks exactly two
representations and knows nothing two steps away.

```text
LAYER            speaks outward        speaks inward         knows nothing about
Frontend         pixels / clicks       HTTP / JSON           SQL
Serving layer    HTTP / JSON           query calls / rows    pixels
Store            (none)                SQL / rows            HTTP, JSON, pixels
```

One drill-down click, traced with the shape on every hop:

```text
1. user clicks "March"                          → a gesture
2. frontend builds an HTTP request              → GET /sources?orch=...&month=2026-03
3. serving layer parses + validates params      → (orch, month) as values
4. serving layer calls the query function       → SQL text + parameter tuple
5. store executes the query                     → rows as tuples / keyed rows
6. serving layer serializes rows                → JSON array of objects
7. frontend renders + wires the next click      → pixels + a new handler
```

Key fact about the store's output shape (this is what makes it feel
"shapeless" coming from pandas): a query returns **a list of rows, each row a
tuple whose positions match the SELECT columns in order**. Setting a row
factory turns each row into a name-keyed object, and a name-keyed object is
exactly what becomes JSON at step 6. The data lives in the store; only the
small result set travels back.

---

## C. The Query Surface: Fixed and Parameterized

The serving layer does **not** accept arbitrary queries from the frontend.
That would be both a security hole and an un-maintainable contract.

```text
fixed at build time   →  the STRUCTURE of every query (the SELECT, the WHERE columns)
supplied at run time  →  the VALUES only (which client, which month, which source)
```

The parameter placeholder is the literal seam between the two timeframes:

```text
WHERE orchestration_key = ?  AND  month = ?
      └── build-time structure ──┘     └── run-time value ──┘
```

This directly answers the security risk already noted in Section 5.20 (a user
swapping `client_id=c0001` to `client_id=c0002`). Because the query structure
is fixed and the client key is a *value*, the serving layer can — and must —
authorize that value against the caller before running the query. Fixed
structure is what makes the authorization check possible in one place.

---

## D. System Invariants

These must be true for the platform to work. Tools may vary; local or cloud may
vary; these do not.

```text
1.  Build and serve are separate triggers.
    fixed: the separation       varies: cron vs Airflow; CLI vs web server

2.  The store is a disposable, rebuildable projection — never the source of truth.
    fixed: regenerable + idempotent loader   varies: SQLite vs Postgres vs warehouse

3.  Exactly one writer; the serving path is read-only.
    fixed: single-writer / read-only split   varies: enforced physically vs by discipline

4.  The query surface is finite, fixed at build time, and parameterized.
    fixed: the fixed parameterized menu       varies: raw SQL vs ORM; SQL dialect

5.  Each layer translates between two representations and knows only its neighbors.
    fixed: separation of concerns             varies: FastAPI vs Flask; React vs other

6.  The response shape is an explicit contract with one source of truth.
    fixed: one authoritative schema           varies: Pydantic vs TypedDict vs JSON schema

7.  Every servable slice is addressable by stable keys, and the data carries them.
    fixed: addressability + keys in every row varies: key naming

8.  Row-level granularity and provenance are preserved; aggregates are computed at query time.
    fixed: keep the grain + lineage columns   varies: column names, storage format

9.  Where the store lives is configuration, injected — never hardcoded.
    fixed: location-agnostic code             varies: file path vs DSN vs secret vault
    (this is the invariant that makes local → cloud cheap)

10. The read path is stateless per request.
    fixed: statelessness + scoped connections varies: one handle vs a pool; one process vs many
```

---

## E. Local → Cloud Is a Configuration Swap

The original Section 2 says "the hosting changes, the architecture remains."
The refinement makes that precise: **only two files change**, and the query
logic does not move.

```text
COMPONENT            INVARIANT   LOCAL (today)              CLOUD (later)
store connection     9, 10       connect to a file          pool to a managed database
store engine         2, 3        single SQLite file         managed Postgres / warehouse
config               9           a path + .env              DSN + secret from a vault
loader trigger       1           run by hand                scheduler / pipeline hook
api runtime          10          one local process          many stateless instances + load balancer
queries + schema     4,5,6       unchanged                  unchanged (modulo minor SQL dialect)
```

The honest caveat: SQL dialect differences may force small edits in the query
file. Invariant 9 is what guarantees those edits stay confined to one file
instead of scattering across the codebase.

---

## F. Universal Module / File Structure

Each file exists to protect an invariant. The crown jewel is the query module:
pure, transport-agnostic, lifecycle-agnostic — imported by both sides, married
to neither.

```text
project/
├── config.py                 # inv 9    — where the store is; reads env, hardcodes nothing
├── schema.py                 # inv 6    — response shapes (one source of truth)
├── store.py                  # inv 9,10 — open/close a connection; the local↔cloud swap point
├── queries.py                # inv 4,5  — the fixed, parameterized query functions (pure)
│
├── build/
│   └── build_serving_store.py  # inv 1,2,3 — forecast output → store. BUILD TIME. only writer.
│
├── serve/
│   ├── api.py                  # inv 1,5,6,10 — routes → queries → JSON. RUN TIME. read only.
│   └── rules.py                # inv 5    — thin warning/business logic (the only run-time compute)
│
├── frontend/
│   ├── apiClient.ts            # the only module that talks HTTP; knows the base URL
│   ├── types.ts                # inv 6    — TS types mirroring schema.py
│   ├── App.tsx                 # app shell + routing
│   └── pages/                  # ForecastOverview, WarningCenter, LineageExplorer, RevisionCompare
│
├── tests/
│   └── test_queries.py         # verify query shapes with no browser / no network
│
└── .env(.example)              # inv 9    — config source; never commit real secrets
```

Three lifecycle buckets, and every file belongs to exactly one:

```text
BUILD TIME (batch)   → build/build_serving_store.py
RUN TIME (request)   → serve/api.py  (and the frontend it serves)
RUN BY NEITHER       → config.py, schema.py, store.py, queries.py
                       (imported by both sides; this spine does not move to the cloud)
```

---

## G. Required Scripts

For each script: purpose, when it runs, input → output, language, the
invariants it protects, and notes. No implementation code — this is a
blueprint, not a build.

### Build-time

#### `build/build_serving_store.py`
- **Purpose:** project the forecast engine's outputs into the read-optimized
  serving store, and build the indexes the UI drills on. (This is the demo
  `build_db.py`, generalized.)
- **Runs at:** build time (batch) — triggered whenever the forecast engine
  publishes a new revision.
- **Input:** forecast output files (CSV / Parquet) + `config` (store location).
- **Output:** the serving store, idempotently rebuilt (drop-and-replace), with
  row-level grain and lineage columns preserved and indexes on
  `orchestration_key`, `month`, `source`, `output_key`.
- **Language:** Python (pandas + `sqlite3`, or SQLAlchemy later).
- **Protects:** 1, 2, 3, 7, 8.
- **Notes:** the **only** writer of the forecast projection. Must be safe to
  re-run. Not importable by the request path.

### Shared core (imported by both sides; run by neither directly)

#### `config.py`
- **Purpose:** the single source of "where things are" and environment settings.
- **Runs at:** imported at both build and run time; never executed alone.
- **Input:** environment variables / `.env`.
- **Output:** config values (store location or DSN, CORS origins, thresholds).
- **Language:** Python.
- **Protects:** 9.
- **Notes:** the local↔cloud difference lives here and nowhere else.

#### `store.py`
- **Purpose:** acquire and release a connection/handle to the store; the
  backend-swap boundary.
- **Runs at:** imported at both build and run time.
- **Input:** `config` (location/credentials).
- **Output:** a connection (and a per-request acquire/release context manager);
  sets the row factory so rows come back keyed by column name.
- **Language:** Python (`sqlite3` now; a SQLAlchemy engine + pool later).
- **Protects:** 9, 10.
- **Notes:** the **one file** that changes when SQLite → Postgres. Connections
  are scoped per unit of work, never held as global mutable state.

#### `queries.py`
- **Purpose:** the fixed, parameterized query functions — the entire question
  surface of the serving layer (level-0 monthly, level-1 sources,
  level-2 lineage, revision compare, etc.).
- **Runs at:** imported at both build and run time.
- **Input:** a connection + parameter values (`orchestration_key`, `month`,
  `source`, revision keys).
- **Output:** rows (lists of name-keyed rows / dicts).
- **Language:** Python + SQL.
- **Protects:** 4, 5.
- **Notes:** pure and transport-agnostic — **no FastAPI import, no HTTP**. The
  `?` placeholders are the build-time/run-time seam. SQL may need minor dialect
  tweaks on cloud, but they stay confined to this file.

#### `schema.py`
- **Purpose:** the single source of truth for response shapes — the JSON keys
  and types each endpoint returns.
- **Runs at:** imported at run time (and by tests).
- **Input:** none (definitions).
- **Output:** typed models used to validate and serialize responses; the source
  of the auto-generated OpenAPI spec.
- **Language:** Python (Pydantic).
- **Protects:** 6.
- **Notes:** `frontend/types.ts` mirrors this. Changing a shape is a deliberate,
  versioned contract change — not an accident.

### Run-time serving (per request; read-only)

#### `serve/api.py`
- **Purpose:** the FastAPI app. Defines routes, maps each route to a query
  function, validates parameters, authorizes the client key, serializes rows to
  JSON, handles errors and CORS.
- **Runs at:** run time — this is the process "triggered at serving time"
  (`uvicorn api:app`).
- **Input:** HTTP requests (path + query parameters).
- **Output:** JSON responses shaped by `schema.py`.
- **Language:** Python (FastAPI), served by Uvicorn.
- **Protects:** 1, 5, 6, 10.
- **Notes:** thin (no forecast logic), stateless per request, read-only.
  Acquires a connection from `store.py` per request and releases it. As it
  grows, split into one router file per page — each still only maps
  HTTP → query → JSON.

#### `serve/rules.py`
- **Purpose:** the thin business logic the serving layer is allowed to have —
  e.g. flag negative-cash months, threshold breaches, large month-over-month
  swings. (Backs the Warning Center.)
- **Runs at:** run time, called by `api.py`.
- **Input:** rows from `queries.py`.
- **Output:** derived/annotated rows (warnings with severity).
- **Language:** Python.
- **Protects:** 5.
- **Notes:** the **only** place run-time computation beyond pure querying is
  allowed. Must stay thin (never re-run the forecast engine) and deterministic,
  so it is unit-testable in isolation.

### Frontend (run time, in the browser)

#### `frontend/apiClient.ts`
- **Purpose:** the single module that knows the API base URL and how to call
  each endpoint.
- **Runs at:** run time, in the browser.
- **Input:** function arguments (filters: client, year, month, source).
- **Output:** typed JS objects matching `types.ts`.
- **Language:** TypeScript / JavaScript.
- **Notes:** the only place the frontend talks HTTP. Base URL comes from a
  frontend env value (`localhost:8000` now → `api.<domain>` later) — the
  frontend's version of injected config.

#### `frontend/types.ts`
- **Purpose:** TypeScript types mirroring the backend response contracts.
- **Runs at:** compile/run time.
- **Input:** none.
- **Output:** types used across components.
- **Language:** TypeScript.
- **Notes:** must stay in sync with `schema.py`; can be auto-generated from the
  OpenAPI spec later.

#### `frontend/pages/*` (e.g. `ForecastOverview`, `WarningCenter`, `LineageExplorer`, `RevisionCompare`)
- **Purpose:** render the data for each page and capture the next user action.
- **Runs at:** run time, in the browser.
- **Input:** data from `apiClient.ts`.
- **Output:** rendered UI + user events that trigger the next call (each drill
  click calls the next endpoint).
- **Language:** TypeScript / React (JSX).
- **Notes:** presentational only — no SQL, no business logic.

#### `frontend/App.tsx`
- **Purpose:** the app shell and navigation between pages.
- **Runs at:** run time.
- **Input:** the current URL.
- **Output:** the routed page.
- **Language:** React + React Router.
- **Notes:** maps `/overview`, `/warnings`, `/lineage`, `/revisions` to pages.

### Development / operations support

#### `tests/test_queries.py`
- **Purpose:** run the query functions directly against a built store and assert
  their shapes (the formalized version of the demo's `__main__` block).
- **Runs at:** development / CI.
- **Input:** a built store.
- **Output:** pass/fail + the observed shapes.
- **Language:** Python (pytest).
- **Protects:** 6.
- **Notes:** verifies the contract with no browser and no network — the fastest
  feedback loop you have.

#### `.env` / `.env.example`
- **Purpose:** hold environment values (store location, CORS origins, secrets)
  read by `config.py`.
- **Runs at:** read at startup of build and serve.
- **Input:** —
- **Output:** environment variables.
- **Language:** dotenv.
- **Protects:** 9.
- **Notes:** commit only `.env.example`; real secrets move to a vault in cloud.

#### `migrations/` (Alembic) — later only
- **Purpose:** version schema changes once the store gains *application-state*
  tables (saved warnings, comments, preferences).
- **Runs at:** build/deploy time, when schema evolves.
- **Input:** schema change definitions.
- **Output:** ordered, reversible migration steps.
- **Language:** Python (Alembic).
- **Notes:** **not** needed for the forecast projection (you just rebuild that).
  Needed only for the app-state domain described in Section H.

---

## H. One Subtlety: Two Data Domains in the Store

Section 4 of the roadmap (saved warnings, comments, user preferences) hides a
trap. Those are written by *users at run time*, which appears to violate
Invariant 3 (single writer, read-only serving path). The resolution: the store
holds **two domains with opposite lifecycles**, and they must be kept separate.

```text
DOMAIN A — forecast projection            DOMAIN B — application state
written by: the batch loader              written by: users, at run time
lifecycle: rebuildable, disposable        lifecycle: durable, NOT rebuildable
on change: drop and rebuild               on change: migrate (never drop)
read at serve time: yes                   read + written at serve time: yes
needs migrations: no                      needs migrations: yes
```

Invariant 3 ("read-only serving path") applies to **Domain A only**. Domain B
is genuinely read-write at run time and needs transactions and migrations.
Keep them in separate tables (ideally a separate store), so a rebuild of the
forecast projection can never destroy user-authored data. Conflating them is
the most likely architectural mistake at the persistence phase.

---

## I. Refined First Concrete Build Target

The original Section 9 target stands, with the timeframes made explicit:

```text
BUILD TIME (once, by hand):
  forecast CSV  ──build_serving_store.py──▶  the store (SQLite)

RUN TIME (every click):
  React page ──fetch──▶ api.py ──queries.py──▶ store ──rows──▶ api.py ──JSON──▶ React
```

Done means: a warning card rendered in the browser whose number can be clicked
down to the individual streams and the revision that produced them — proving
the projection, the fixed query surface, the contract, and the drill path all
hold together. Small, local, real.