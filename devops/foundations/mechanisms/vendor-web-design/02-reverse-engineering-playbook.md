# Reverse-Engineering Playbook

## Purpose

When a website export or download flow breaks, the goal is not to panic and re-explore the whole system from scratch. The goal is to identify the **minimum broken segment** in the pipeline and replace only that piece.

This playbook is a practical procedure for reverse-engineering browser-based CSV download workflows, especially when there is no official API documentation and the UI has changed.

It is designed to answer these questions quickly:

* What request actually triggers the export?
* Is the file generated immediately or as an async job?
* Where is authorization carried?
* How is the final file delivered?
* Can this be replayed with `requests`, or does it require a real browser?

## Mental model before touching DevTools

Every export flow can be decomposed into these possible segments:

1. login / session establishment
2. account or entity switching
3. export job creation
4. job polling / status refresh
5. artifact retrieval
6. parsing / storage downstream

The first rule is:

> Never rebuild the whole pipeline unless you are certain the whole pipeline changed.

Most breakages only affect one segment.

Examples:

* login still works, but export endpoint changed
* export still works, but delivery changed from email to signed URL
* file still exists, but auth header format changed
* everything still works in browser, but `requests` now fails due to bot checks

Your job is to isolate which segment changed.

## Step 0 — Decide the objective

Before inspecting anything, decide what you are trying to prove.

Possible objectives:

* prove whether automation is still possible at all
* identify the new export start request
* identify the polling pattern
* identify the final file URL source
* determine whether pure HTTP replay is enough
* determine whether browser automation is required

This prevents wandering through random network calls.

## Step 1 — Reproduce the workflow manually in the browser

Open the exact page where the export occurs and perform the flow manually.

Do not start reverse-engineering from code first. Start from working human behavior.

You want to know:

* what page you are on
* what buttons are clicked
* whether an account/entity must first be switched
* whether filters/date/year selections are active
* whether the browser visibly shows progress
* whether the file downloads directly, after delay, or via email

Write down the human sequence in one sentence.

Example:

> Login → switch account → open grain loads page → click Download CSV → wait for progress 0–100% → CSV downloads

That one sentence becomes the search target for network analysis.

## Step 2 — Open DevTools and record only the relevant window

Use browser DevTools Network tab.

Recommended setup:

* enable **Preserve log**
* enable **Disable cache**
* clear the existing network log
* reproduce only the export flow

This matters because otherwise the noise becomes overwhelming.

You want the cleanest possible slice of traffic around the export event.

Useful filters to try:

* `csv`
* `export`
* `download`
* `graphql`
* `api`
* `blob`
* `events`
* `jobs`
* `report`

Also sort by **time** so you can see causal order.

## Step 3 — Classify the flow shape before reading details

Look at the timeline and decide which broad pattern it resembles.

### Pattern A — One request, one file

You click export and a single request returns CSV bytes.

Signal:

* one request with `content-type: text/csv` or `content-disposition: attachment`

### Pattern B — Start request + polling + final download

You click export, see progress, then later a file downloads.

Signals:

* first request returns JSON
* repeated calls follow
* final request downloads file

### Pattern C — Start request + no visible download request + UI saves file

Signals:

* requests only return JSON
* no CSV response appears
* likely client-side Blob generation

### Pattern D — Start request + email / notification

Signals:

* no file download in network
* response message says email or queued

Get this classification right first. It narrows everything.

## Step 4 — Identify the **start request**

This is the most important request in the entire flow.

Find the request that occurs immediately after clicking export.

For that request, capture these items exactly:

* HTTP method
* URL
* request headers
* request payload/body
* response body
* status code
* initiator (what triggered it)

### Why request payload matters more than headers in modern apps

In GraphQL or JSON APIs, the path may stay constant while the payload changes everything.

Examples:

* `/api/v3/graphql`
* `/api/reports`
* `/exports`

The payload tells you whether the request means:

* start export
* poll event
* fetch file link
* refresh token
* update UI state only

### What to look for in the payload

For REST/JSON:

* `format: csv`
* `output: csv`
* `export: true`
* report name / filter parameters / account ids

For GraphQL:

* `operationName`
* `query`
* `variables`
* `extensions.persistedQuery`

For forms:

* hidden inputs
* CSRF token
* export type / year / filters

### What to look for in the response

* `id`
* `status`
* `message`
* `jobId`
* `eventId`
* `downloadUrl`
* `signedUrl`
* `errors`

The start request tells you the backend contract.

## Step 5 — Separate **real business calls** from support calls

Many requests around an export are not the export itself.

Common support calls:

* token refresh
* analytics / telemetry
* sentry logging
* websocket status pings
* image/font requests
* UI metadata refresh

Do not confuse these with the export workflow.

A good rule:

> The real export calls are the ones carrying export scope, export ids, status transitions, or file URLs.

Everything else is support noise.

## Step 6 — Determine where authorization lives

This is the second most important question.

An export workflow fails in automation more often due to auth misunderstandings than due to the endpoint itself.

Check whether the relevant requests rely on:

### A. Session cookies

Look for `Cookie:` request header and Set-Cookie responses.

Implication:

* preserve cookies in `requests.Session()`

### B. Authorization header

Look for:

* `Authorization: Bearer ...`
* custom lowercase `authorization: ...`

Implication:

* your script must supply the token exactly as expected

### C. CSRF token

Look for:

* hidden HTML input
* `X-CSRF-Token` header
* `authenticity_token`

Implication:

* script may need to parse page HTML or bootstrap JSON first

### D. Signed URL auth

If final file URL has long query parameters like `X-Amz-*`, authorization is embedded in the URL itself.

Implication:

* no cookie/header may be needed for final GET
* must download before expiry

### E. Mixed auth

The site may use cookie auth for page navigation and JWT auth for API/GraphQL calls.

Implication:

* preserve both layers
* browser feeling like “one session” does not mean backend contracts are simple

## Step 7 — Decide whether export is synchronous or asynchronous

This determines the automation design.

### Synchronous export

Indicators:

* one request returns file bytes immediately
* no progress UI
* no job id

Implementation shape:

* replay request
* write response content to file

### Asynchronous export

Indicators:

* status values like `queued`, `processing`, `complete`, `failed`
* event/job/task id returned
* progress indicator in UI
* separate final download request

Implementation shape:

* create job
* poll status endpoint
* stop on terminal state
* when ready, fetch download URL or file bytes

If you see polling, that is good news: the system is exposing explicit state.

## Step 8 — Identify the polling contract

If the workflow is async, identify exactly how progress is checked.

Look for repeated requests after the start request.

Capture:

* polling URL/path
* payload / variables
* stable job identifier
* response fields indicating progress or completion
* polling interval implied by frontend timing

Common forms:

### REST polling

```text
GET /exports/123/status
```

### GraphQL polling

```json
{
  "operationName": "EventManagerQuery",
  "variables": {"id": "..."}
}
```

### Websocket/subscription

Less common for downloads, but possible.

For most automation, simple periodic polling is enough.

### What you need from poll responses

At minimum:

* current status
* terminal success indicator
* terminal failure indicator
* artifact location or way to retrieve it

### Recommended logic

* poll every few seconds
* stop on `complete/succeeded/done`
* raise on `failed/error`
* timeout after reasonable maximum

## Step 9 — Find the **artifact retrieval** mechanism

Once the export completes, how do the bytes actually reach the browser?

There are only a few possibilities.

### A. Direct file response from app backend

Signals:

* final response contains CSV headers
* file bytes come from app domain

### B. App-owned temporary download endpoint

Signals:

* final URL points to app domain like `/downloads/abc`
* may still require session auth

### C. Signed cloud storage URL

Signals:

* final URL points to `s3.amazonaws.com`, GCS, Azure Blob, CDN object storage
* query parameters include signature and expiry

### D. Client-side Blob generation

Signals:

* no file URL exists
* browser builds download from JSON response in JavaScript

### E. Email / out-of-band link

Signals:

* browser does not receive file directly
* email carries future retrieval URL

The retrieval mechanism determines whether automation is simple replay or multi-stage orchestration.

## Step 10 — Save the **minimum reproducible contract**

Do not just save “some headers” or random screenshots.

Save the smallest set of information needed to replay the workflow later.

For each essential request, save:

* method
* URL
* relevant headers only
* payload
* key response fields

### Relevant headers usually include

* `authorization`
* `content-type`
* `referer` if required
* sometimes `origin`
* sometimes CSRF header

### Usually not essential

* `sec-ch-ua*`
* `accept-language`
* telemetry baggage
* user-agent variations, unless anti-bot issue exists

This distinction matters because copying too much browser noise makes scripts fragile.

## Step 11 — Reconstruct with `requests` first

Default to the lightest-weight implementation.

Use `requests` first when:

* login can be replayed or session cookies are available
* API/GraphQL calls are ordinary HTTP requests
* no browser-only JS computation is required for the essential contract
* no anti-bot challenge blocks you

### Typical reconstruction workflow

1. establish session
2. login or reuse authenticated session
3. switch account/entity if needed
4. call start-export request
5. parse job/event id
6. poll until ready
7. retrieve file URL or bytes
8. save artifact
9. parse downstream

This covers a huge percentage of real-world systems.

## Step 12 — When to escalate from `requests` to Playwright

This is an important decision boundary.

Use Playwright when one or more of these is true.

### 1. Login flow depends heavily on JavaScript execution

Examples:

* dynamic tokens created in runtime
* opaque challenge pages
* complex redirects through SSO provider

### 2. Anti-bot protections reject non-browser clients

Signals:

* browser works, `requests` gets blocked
* challenge pages appear
* unexplained 403s despite correct cookies/tokens

### 3. Critical values only exist in browser runtime and are hard to reconstruct

Examples:

* values generated by in-page scripts
* download action triggered by internal JS object state not easily visible in network

### 4. The file is generated client-side from browser-only data flow

If the browser computes the CSV from many JSON calls and JS transformations, it may be faster to automate the browser or directly reproduce the transformation logic.

### 5. MFA requires interactive browser session maintenance

If unattended auth is impossible with plain HTTP, browser automation may preserve a human-established session longer.

### 6. Vendor changes are frequent and brittle

Sometimes a browser-driven workflow is less elegant but more robust against surface API churn because it behaves more like a human.

### Rule of thumb

* **Use `requests` when backend contract is visible and stable enough.**
* **Use Playwright when the browser itself is part of the contract.**

That is the key distinction.

## Step 13 — What Playwright is buying you

Playwright does not magically solve everything. It buys you specific capabilities:

* real browser execution
* real cookie jar
* JS runtime execution
* ability to survive SPA flows and anti-bot checks better
* direct observation of downloads
* easier reproduction of click-based workflows

But it also costs you:

* heavier setup
* slower runs
* more moving parts
* more brittle selectors if implemented carelessly

So it is a powerful last-resort weapon, not the default first tool.

## Step 14 — How to use Playwright intelligently

If you escalate to Playwright, do not immediately script dozens of UI selectors.

Prefer this order:

### Strategy A — Use Playwright mainly for login/session capture

* let browser authenticate naturally
* extract cookies/storage state
* continue API replay with `requests`

This is often the best hybrid pattern.

### Strategy B — Use Playwright to intercept the network contract

* click export in browser
* capture request payloads and responses programmatically
* then move back to `requests`

### Strategy C — Use Playwright for full end-to-end file download

Only when the whole workflow truly depends on browser behavior.

This keeps the heavy weapon reserved for when it is actually needed.

## Step 15 — Common traps during reverse-engineering

### Trap 1 — Copying every browser header blindly

This creates brittle scripts.

### Trap 2 — Ignoring request payloads

In GraphQL, payload is the real endpoint identity.

### Trap 3 — Missing account/entity switch context

The export may depend on prior page or account activation state.

### Trap 4 — Assuming final URL is stable

Signed URLs expire quickly.

### Trap 5 — Confusing refresh-token traffic with export traffic

Support traffic is not the business contract.

### Trap 6 — Overfitting to one run

Some flows vary depending on cache, token age, or file readiness.

### Trap 7 — Rebuilding the whole workflow when only one segment changed

Always preserve working pieces.

### Trap 8 — Not capturing response bodies

Request metadata alone is often insufficient.

### Trap 9 — Chasing UI HTML when network tells the real story

For downloads, the network contract is usually more important than DOM structure.

### Trap 10 — Declaring “impossible” too early

Most systems are not impossible; they are just async, tokenized, or browser-dependent.

## Step 16 — A concrete reverse-engineering checklist

When a CSV export breaks, follow this checklist in order.

### Phase 1 — Observe

1. Reproduce export manually.
2. Write down the exact human flow.
3. Open DevTools Network with Preserve log.
4. Clear network log.
5. Reproduce only the export sequence.

### Phase 2 — Classify

6. Identify the first request triggered by export.
7. Determine sync vs async.
8. Determine direct download vs signed URL vs email vs Blob.
9. Determine auth layers: cookie, JWT, CSRF, signed URL.

### Phase 3 — Capture

10. Save the start request payload.
11. Save the start response body.
12. Save the polling request payload.
13. Save the polling completion response body.
14. Save the final artifact URL pattern and expiry behavior.

### Phase 4 — Rebuild lightly

15. Reuse existing login/session logic if still valid.
16. Recreate only the changed export segment in `requests`.
17. Add polling + timeout + failure handling.
18. Download file immediately after signed URL appears.

### Phase 5 — Escalate only if needed

19. If `requests` fails but browser works, test whether auth or bot checks are the blocker.
20. Escalate to Playwright only when browser execution is part of the necessary contract.

## Step 17 — Logging strategy for future-proofing

When rebuilding an extraction workflow, log enough to diagnose future vendor changes quickly.

Useful logs:

* start export timestamp
* entity/account scope
* export job/event id
* status transitions
* final artifact retrieval success/failure
* response snippets on failure
* token refresh occurrence
* elapsed time to completion

Avoid logging secrets:

* full JWTs
* full signed URLs
* cookies
* refresh tokens

A good compromise:

* log last few characters of ids/tokens only if necessary
* redact query signatures

Good logs turn next breakage from archaeology into patching.

## Step 18 — How to explain findings to stakeholders

When reporting to a manager or sponsor, avoid overwhelming them with protocol details.

Use this structure:

### 1. What changed?

Example:

* vendor replaced email-based export with GraphQL-triggered background export and direct signed download

### 2. Is automation still possible?

* yes / likely yes / uncertain / no without vendor support

### 3. What is the minimal fix?

Example:

* replace old export endpoint with mutation + poll + signed download retrieval

### 4. What are the risks?

Example:

* token refresh mid-export
* short-lived URLs
* possible future anti-bot issues

### 5. What is the effort level?

Example:

* low if current session/auth remains usable
* medium if browser automation becomes necessary

This keeps the conversation decision-oriented.

## Step 19 — Durable principles to remember

### Principle 1

UI changes are often surface changes, not true architectural removal.

### Principle 2

For modern apps, request payloads matter more than endpoint paths.

### Principle 3

Async export systems are usually more automatable than they first appear.

### Principle 4

Signed storage URLs are usually good news, not bad news.

### Principle 5

When the browser itself becomes part of the contract, escalate to Playwright.

### Principle 6

Preserve the working parts of the old pipeline.

### Principle 7

The goal is not to understand everything. The goal is to recover the minimal reproducible contract.

## Final synthesis

Reverse-engineering download workflows is not about memorizing vendor quirks. It is about identifying a small number of invariants:

* how export is initiated
* how state/progress is tracked
* how authorization is carried
* how the final artifact is delivered
* whether the browser is merely a transport surface or an essential runtime dependency

Once you can answer those five questions, most “mysterious” UI changes collapse into a manageable implementation task.

That is the real playbook.
