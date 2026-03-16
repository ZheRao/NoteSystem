# Website CSV Download Invariants

## Why this matters

CSV download flows look different on the surface, but underneath they usually reduce to a small set of recurring backend patterns. Once you understand those patterns, vendor UI changes stop feeling random. You can reason from first principles:

* Where is the data assembled?
* Who authorizes the download?
* Is the file generated immediately or as a background job?
* Does the browser receive raw CSV bytes, or only a temporary link?
* What state must be preserved across login, export, polling, and download?

This is useful for:

* adapting broken automation when vendors change the UI
* diagnosing whether a change is cosmetic or architectural
* deciding whether `requests` is enough or whether browser automation is needed
* building a fallback mental model when there is no official API

## The core invariant

A CSV download system almost always has **four logical stages**:

1. **Authenticate user context**
2. **Define export scope**
3. **Materialize file bytes**
4. **Deliver bytes to client**

Everything else is just implementation detail.

Even when the UI looks completely different, the site still needs to answer the same questions:

* Which user is requesting the export?
* Which entity/account/report/filter/year does the export represent?
* Where will the CSV be generated?
* How will the browser be allowed to retrieve it?

This is the most important invariant.

## Stage 1 — Authenticate user context

Before a CSV can be downloaded, the server must know who the user is and what they are allowed to see.

Common mechanisms:

### 1. Session-cookie authentication

Traditional server-rendered apps often set a session cookie after login.

Flow:

* browser posts login form
* server validates credentials
* server sets cookie
* subsequent export requests include cookie automatically

Automation implication:

* `requests.Session()` can often replay this easily
* cookies must be preserved across requests

### 2. Bearer/JWT token authentication

Single-page apps often issue an access token and sometimes a refresh token.

Flow:

* login succeeds
* frontend stores token in memory, cookie, or local storage
* export call sends `Authorization: Bearer ...` or another token header

Automation implication:

* script must extract token from page bootstrap JSON, response body, cookie, or storage
* token expiry must be handled

### 3. Hybrid auth

Many apps use both:

* session cookie for page navigation
* JWT for API or GraphQL calls

This is very common. The browser feels like “one login”, but under the hood there are two auth layers.

### 4. SSO / MFA / identity provider flow

Enterprise systems may redirect through Okta, Azure AD, Google, etc.

Automation implication:

* login may become much harder without browser automation
* pure `requests` may not be enough if the flow requires JavaScript or MFA interaction

---

## Stage 2 — Define export scope

The backend must know **what** to export.

Common scope dimensions:

* entity / account / farm / company / tenant
* date range
* crop year / fiscal year
* filters currently applied in UI
* selected columns
* display unit or formatting options
* report type

This information can be passed in several ways.

### Variation A — URL query parameters

Example shape:

```text
/export.csv?year_id=2026&status=active
```

Characteristics:

* simple to inspect
* common in older apps
* easy to automate

### Variation B — POST form body

Example shape:

```text
POST /exports
form-data: year_id=2026, output=csv
```

Characteristics:

* common in classic web apps
* may require CSRF token

### Variation C — JSON REST payload

Example shape:

```json
{
  "entityId": 123,
  "filters": {"year": 2026},
  "format": "csv"
}
```

Characteristics:

* common in SPA/API designs
* easier to evolve without changing URLs

### Variation D — GraphQL mutation variables

Example shape:

```json
{
  "operationName": "ExportLoads",
  "variables": {
    "input": {
      "output": "csv",
      "displayUnit": null
    }
  }
}
```

Characteristics:

* common in modern React apps
* often hides complexity behind a mutation name
* current UI state may also be implied by server-side context, not just variables

### Variation E — Implicit scope from active page/session context

Sometimes the request payload looks surprisingly small because the backend already knows:

* current active entity
* current report page
* default filters stored in session
* role-based restrictions

This is why two identical requests may return different files for different logged-in users or switched accounts.

**Important invariant:** scope is not always explicit. Sometimes it is partly encoded in the current server-side context.

---

## Stage 3 — Materialize file bytes

This is where the actual CSV gets created.

There are two major patterns.

### Pattern 1 — Synchronous generation

The server generates the CSV immediately during the request and returns it in the same response.

Example:

* browser clicks “Export CSV”
* server queries database
* response returns `Content-Type: text/csv`
* browser downloads file directly

Strengths:

* simple
* fewer moving parts

Weaknesses:

* poor for large exports
* request may time out
* ties up web server resources

How it looks in network tab:

* one request
* response headers include `content-type: text/csv` or `content-disposition: attachment`

Automation implication:

* easiest case
* just replay request and save response bytes

### Pattern 2 — Asynchronous background export job

The server creates a job, returns a job/event id, and generates the file in the background.

Example:

* user clicks export
* backend enqueues export job
* frontend polls status
* when job completes, backend returns file URL
* browser downloads file

Strengths:

* works for large datasets
* avoids web request timeout
* scalable

Weaknesses:

* more complex client flow
* must poll or subscribe to progress
* more failure states

How it looks in network tab:

* first request returns JSON with `id`, `status`, `message`
* repeated polling calls
* final download URL appears later

Your Harvest Profit example fits this pattern.

---

## Stage 4 — Deliver bytes to client

Once the file exists, it still needs to reach the browser.

This stage has several common delivery patterns.

### Delivery Pattern A — Direct response download

The same backend request returns the file bytes directly.

Headers often include:

```text
Content-Type: text/csv
Content-Disposition: attachment; filename="report.csv"
```

This is the simplest pattern.

### Delivery Pattern B — Application-owned download endpoint

The backend generates the file, stores it temporarily, then returns a URL like:

```text
/downloads/abc123
```

The browser then performs a second GET to that endpoint.

Characteristics:

* app retains control over authorization
* may require current session cookie

### Delivery Pattern C — Signed object-storage URL

The backend stores the file in S3/GCS/Azure Blob and returns a temporary signed URL.

Example shape:

```text
https://bucket.s3.amazonaws.com/object?X-Amz-Algorithm=...&X-Amz-Expires=300&...
```

Characteristics:

* extremely common in modern systems
* app offloads byte serving to cloud storage
* link usually expires quickly
* no app cookie needed if URL is signed correctly

This is what Harvest Profit appears to use now.

### Delivery Pattern D — Email with later download link

Instead of returning the file immediately, the app emails a download link.

Characteristics:

* often used when exports are very slow
* acts as a human-friendly async queue
* painful for automation unless mailbox is integrated

This was your previous Harvest Profit design.

### Delivery Pattern E — Browser-created CSV client-side

Sometimes the backend returns JSON, and the browser constructs the CSV in JavaScript using `Blob()` and triggers a local download.

Characteristics:

* no file endpoint may exist at all
* network only shows JSON data, not a CSV response
* automation may be easier by calling the underlying JSON endpoint directly

This matters because sometimes the “download” is not actually a backend export system.

---

## The five most common real-world architectures

### Architecture 1 — Classic form submit → direct CSV response

Flow:

1. user logs in
2. user clicks export button
3. browser submits GET/POST with filters
4. server returns CSV bytes immediately

Best for:

* small reports
* legacy apps

Automation difficulty:

* low

### Architecture 2 — REST export endpoint → direct CSV response

Flow:

1. frontend sends authenticated API request
2. backend returns CSV in response

Best for:

* simple SPA systems

Automation difficulty:

* low

### Architecture 3 — Async export job → poll → signed URL

Flow:

1. create export job
2. receive job/event id
3. poll until complete
4. fetch signed storage URL
5. download file

Best for:

* large exports
* scalable systems

Automation difficulty:

* medium

### Architecture 4 — Async export job → email notification → signed URL

Flow:

1. create export job
2. backend generates file later
3. email contains link
4. user or automation downloads

Best for:

* low engineering effort on vendor side
* long-running jobs

Automation difficulty:

* medium to high

### Architecture 5 — Client-side JSON-to-CSV generation

Flow:

1. frontend calls JSON data endpoint
2. JS transforms result to CSV in browser
3. browser downloads Blob

Best for:

* moderate-size data already available in UI

Automation difficulty:

* variable
* often easier to call raw data endpoint than reproduce Blob logic

---

## How to reason from the browser network tab

When reverse-engineering a download flow, ask these questions in order.

### Question 1 — Is the download synchronous or asynchronous?

Evidence for synchronous:

* one request directly returns CSV

Evidence for asynchronous:

* first request returns JSON with status or id
* repeated polling calls
* progress indicator in UI
* final separate download request

### Question 2 — Is the file generated server-side or client-side?

Evidence for server-side:

* network shows CSV response or download URL
* content-disposition header appears

Evidence for client-side:

* network only shows JSON data
* no file endpoint visible
* frontend code mentions `Blob`, `URL.createObjectURL`, or CSV libraries

### Question 3 — How is authorization carried?

Look for:

* cookies
* `Authorization` header
* CSRF token
* signed URL parameters

### Question 4 — Is the final file behind app auth or storage signature?

If final URL is app domain, you may need session auth.
If final URL is signed S3/GCS/Blob, you usually just GET it immediately.

### Question 5 — What is the stable identifier of the export job?

Could be:

* job id
* event id
* Relay node id
* task id

Once found, polling becomes straightforward.

---

## Invariants specific to GraphQL export systems

GraphQL makes things look unfamiliar, but the same logic still applies.

### Common GraphQL export flow

1. mutation starts export
2. mutation returns object with `id`, `status`, maybe `message`
3. query polls that `id`
4. completed object exposes or triggers retrieval of file URL

### What usually matters in the request body

* `operationName`
* `query` or persisted query hash
* `variables`

### Why GraphQL changes feel confusing

Because the route stays the same:

```text
POST /api/v3/graphql
```

Different actions are hidden inside the payload, not the URL.

So the **real endpoint identity** becomes:

* operation name
* query shape
* variables

not just the HTTP path.

This is why headers alone are insufficient.

---

## Common failure modes when vendors change the UI

### 1. Route renamed, business process unchanged

Old `/export_all` becomes new `/graphql` mutation.

Interpretation:

* surface changed
* architecture still supports export

### 2. Sync export changed to async export

User now sees progress bar.

Interpretation:

* backend likely moved to a queued job model
* automation still possible, but needs polling

### 3. Email delivery replaced by direct signed URL

Interpretation:

* automation usually becomes easier
* mailbox dependency disappears

### 4. Filters moved from URL to hidden app state

Interpretation:

* request payload appears incomplete
* active entity/page/session matters more

### 5. Access token expires mid-export

Interpretation:

* need refresh-token logic or retry wrapper

### 6. Bot protection introduced

Interpretation:

* `requests` may fail even though browser works
* browser automation may be required

### 7. Frontend now creates file client-side

Interpretation:

* stop searching for CSV endpoint
* search for raw JSON data endpoint instead

### 8. Download link is short-lived

Interpretation:

* signed URL must be used immediately
* logging it for later is insufficient

---

## Practical automation decision tree

### Case A — Direct CSV response

Use `requests`.

* preserve cookies/auth
* replay request
* save bytes

### Case B — Async job with poll and signed URL

Use `requests`.

* start job
* poll status
* download signed URL immediately

### Case C — App auth + CSRF + REST/GraphQL

Usually still `requests`.

* preserve session
* include CSRF/token headers

### Case D — JS-heavy login or anti-bot challenge

Use Playwright or Selenium.

* drive real browser
* let browser obtain cookies/tokens
* intercept or trigger download

### Case E — File created entirely in browser from JSON

Either:

* call the JSON endpoint directly and reconstruct CSV yourself, or
* use browser automation as last resort

---

## What makes data extraction truly impossible without vendor cooperation?

Usually, almost nothing makes it truly impossible. It becomes a question of cost, fragility, and legality/permissions.

Still, these are the main hard blockers.

### Hard blocker 1 — Strong anti-bot/browser attestation

If the site requires proof of a real browser environment beyond what simple automation can reproduce, pure HTTP replay may fail.

### Hard blocker 2 — Mandatory human MFA at every session

If no reusable long-lived session exists and every login requires a human code approval, unattended automation becomes very hard.

### Hard blocker 3 — Data never leaves server except as rendered pixels

Extremely rare for CSV export, but possible in some locked-down environments.

### Hard blocker 4 — Export only accessible through proprietary desktop client or inaccessible websocket binary protocol

Rare, but increases reverse-engineering cost dramatically.

### Hard blocker 5 — Terms, permissions, or internal governance prohibit automation

This is organizational impossibility, not technical impossibility.

Most of the time the real issue is not impossibility. It is:

* too brittle
* too time-consuming
* too low priority
* too dependent on vendor churn

---

## Last-resort weapon mindset

When there is no official extraction path, your fallback weapons are usually:

1. **HTTP replay with `requests`**
2. **GraphQL mutation/query replay**
3. **Polling job state until file is ready**
4. **Immediate retrieval of signed storage URLs**
5. **Browser automation for JS-heavy or bot-protected flows**
6. **Calling raw JSON endpoints and reconstructing CSV yourself**
7. **Manual export with a deliberately documented procedure until priority increases**

This is the hierarchy from simplest to heaviest.

---

## Harvest Profit mapping example

Based on the observed behavior, Harvest Profit now appears to follow this architecture:

### Old architecture

* authenticated app session
* REST export endpoint
* export triggered by request
* email contains link
* Power Automate extracts link
* script downloads CSV

### New architecture

* authenticated app session + JWT-style API auth
* GraphQL mutation starts export event
* event id returned
* frontend polls event status
* completed export yields short-lived signed S3 URL
* browser downloads CSV directly from S3

### Invariant preserved

The system still does the same four logical stages:

1. authenticate
2. determine scope
3. generate file
4. deliver file

Only the transport mechanism changed.

This is the key lesson: **vendor change often alters surface mechanics, not core architecture.**

---

## Debugging checklist for future breakages

When a website CSV download changes, inspect in this order:

### A. Trigger the download in browser and record network traffic

Filter for:

* `csv`
* `export`
* `download`
* `graphql`
* `blob`
* `events`
* `jobs`

### B. Identify the start request

Capture:

* method
* URL
* headers
* request payload
* response JSON

### C. Decide whether the flow is sync or async

Look for:

* returned `id`
* returned `status`
* progress bar
* repeated polling calls

### D. Identify the final delivery mechanism

Look for:

* direct CSV response
* app download endpoint
* signed object-storage URL
* email link
* no file request at all (client-side generation)

### E. Determine auth dependencies

Record:

* cookies
* bearer token header
* CSRF token
* refresh flow

### F. Implement the minimal replacement path

Prefer replacing only the broken segment, not the whole pipeline.

Example:

* keep login and account-switch logic
* replace email trigger with GraphQL mutation + poll + download

---

## Design lesson for your own systems

A mature export system is not just “a button that downloads CSV.” It is a distributed workflow involving:

* identity
* scope
* compute job orchestration
* storage
* delivery authorization
* retry and expiry behavior

When designing your own systems, think in these layers explicitly.

### Better architectural habits

* separate “start export” from “download artifact”
* expose stable job ids and job states
* make delivery mechanism explicit
* log enough metadata for recovery/debugging
* prefer machine-friendly contracts over email-based workflows
* treat export as artifact production, not just response rendering

Email-based export is convenient for humans but weak for reliable automation. Signed artifact delivery after async generation is usually superior.

---

## Final synthesis

No matter how different website download systems look, most are composed from the same reusable invariants:

* **Authentication context** tells the server who you are.
* **Scope definition** tells the server what data you want.
* **Materialization** creates the CSV, either immediately or in the background.
* **Delivery** moves the bytes to the browser, either directly, through an app endpoint, through email, or through signed storage URLs.

Once you identify those four stages, UI churn becomes less scary. You stop memorizing vendor-specific button behavior and start recognizing the underlying export architecture.

That is the real durable skill.
