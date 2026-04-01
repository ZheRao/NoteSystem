# `requests.Session()` Core Mechanics Guide

## Why this matters

When you automate a website or API with Python, the hardest part is often not the business logic. It is understanding **how authentication state lives across requests**.

`requests.Session()` is the core tool for that job.

It helps you:

* persist cookies across requests
* reuse headers and connection settings
* behave more like one continuous browser session instead of isolated one-off requests
* inspect what the server actually returned after login
* determine whether auth lives in cookies, JSON body tokens, headers, or a hybrid of all three

This guide focuses on the core mechanics, how to inspect a response, and how to reason about session state clearly.

# 1. Mental model: request → response → session state

A typical authenticated flow looks like this:

1. You create a `Session`
2. You send a login request
3. The server responds with one or more of:

   * cookies via `Set-Cookie`
   * JSON containing tokens
   * headers containing auth-related values
4. `Session` stores cookies automatically
5. Later requests made through that same `Session` automatically resend matching cookies

The most important idea:

> `requests.Session()` is a persistent HTTP client state container, especially for cookies and default headers.

It does **not** magically create authentication for you.
It only preserves authentication artifacts that the server gives you or that you manually attach.

# 2. What `requests.Session()` actually does

## 2.1 Cookie persistence

If the server returns:

```http
Set-Cookie: sessionid=abc123; Path=/; Secure; HttpOnly
```

then `session.cookies` stores it.

On later requests to the same matching domain/path, `requests` automatically sends:

```http
Cookie: sessionid=abc123
```

without you manually building the `Cookie` header.

## 2.2 Default headers persistence

You can do this once:

```python
session.headers.update({"User-Agent": "Mozilla/5.0"})
```

and every future request from that session will reuse it unless overridden.

## 2.3 Connection reuse

Under the hood, `Session` may reuse TCP connections for efficiency. This usually matters less conceptually than cookies, but it is part of why sessions exist.

## 2.4 Shared state across requests

A `Session` behaves like one continuous client identity.

That is why this usually works better than repeated standalone `requests.get()` and `requests.post()` calls when dealing with login flows.

# 3. What a response can contain

After a request like:

```python
resp = session.post(url, ...)
```

you should think of the response as having **four main inspection surfaces**.

## 3.1 Status code

```python
resp.status_code
```

Examples:

* `200` — request succeeded at the HTTP layer
* `302` — redirect happened
* `401` — unauthorized
* `403` — forbidden
* `500` — server-side error

Important:

> HTTP 200 does not necessarily mean login/auth succeeded.

A server can return `200` with an HTML login page, an error JSON body, or a partial success.

## 3.2 Headers

```python
resp.headers
```

This is where you often find:

* `Set-Cookie`
* `Content-Type`
* redirects via `Location`
* auth-related metadata
* caching behavior

## 3.3 Body text / content

```python
resp.text
resp.content
```

Use this when the server returns:

* HTML
* plain text
* JSON that you want to preview quickly
* JavaScript or embedded data

## 3.4 JSON body

```python
resp.json()
```

Use this when the response is JSON.

This is often where access tokens, refresh tokens, user metadata, or GraphQL payloads appear.

# 4. The three most common auth patterns

## 4.1 Cookie-based auth

Login response sets a session cookie.
Future requests work because the cookie is automatically resent.

Example:

```python
session = requests.Session()
resp = session.post(login_url, data=login_form)
print(session.cookies.get_dict())
```

## 4.2 Token-in-body auth

Login response returns JSON like:

```json
{
  "access_token": "...",
  "refresh_token": "..."
}
```

Then you manually attach the token to future headers:

```python
session.headers["Authorization"] = access_token
```

## 4.3 Hybrid auth

This is very common in modern apps.

The server may provide:

* cookies for session continuity
* an access token for API calls
* a refresh token for renewing access

This appears to be the likely pattern for Harvest Profit.

# 5. `BeautifulSoup` vs `pprint`

They are **not** the same thing.

## `BeautifulSoup`

Use `BeautifulSoup` when the response body is **HTML or XML** and you want to parse document structure.

Example use cases:

* find `<input>` hidden tokens in HTML
* inspect forms
* extract links
* search script tags for embedded data

Example:

```python
from bs4 import BeautifulSoup

soup = BeautifulSoup(resp.text, "html.parser")
csrf_input = soup.find("input", {"name": "csrf_token"})
```

## `pprint`

`pprint` stands for “pretty print.”
It does **not parse anything**.
It only formats Python objects so they are easier to read.

Example use cases:

* print nested dicts
* print JSON decoded into Python objects
* inspect headers and metadata

Example:

```python
from pprint import pprint

pprint(resp.headers)
pprint(resp.json())
```

## Practical rule

* Use **BeautifulSoup** to **parse HTML/XML structure**
* Use **`pprint`** to **display Python data structures clearly**

For your current problem, `pprint` is the better default because GraphQL responses are JSON, cookies are Python objects, and headers are mapping-like structures.

# 6. The most useful inspection pattern after login

This should become muscle memory.

```python
import requests
from pprint import pprint

session = requests.Session()

resp = session.post("https://example.com/login", data={
    "email": "your_email",
    "password": "your_password",
}, timeout=30)

print("STATUS:", resp.status_code)
print("FINAL URL:", resp.url)
print("REDIRECT HISTORY:", resp.history)

print("\nHEADERS:")
pprint(dict(resp.headers))

print("\nRESPONSE COOKIES:")
for c in resp.cookies:
    print(c.name, c.value, c.domain, c.path, c.secure, c.expires)

print("\nSESSION COOKIES:")
for c in session.cookies:
    print(c.name, c.value, c.domain, c.path, c.secure, c.expires)

print("\nTEXT PREVIEW:")
print(resp.text[:3000])

try:
    print("\nJSON BODY:")
    pprint(resp.json())
except ValueError:
    print("Response is not JSON.")
```

This one block teaches you almost everything you need to know about what came back.

# 7. Understanding cookies clearly

## 7.1 `resp.cookies`

Cookies set on **this response only**.

```python
resp.cookies
```

Use this when you want to know exactly what this one server reply added.

## 7.2 `session.cookies`

All cookies currently stored in the session cookie jar.

```python
session.cookies
```

Use this when you want to know what future requests will likely send.

## 7.3 Quick dictionary view

```python
session.cookies.get_dict()
```

Example:

```python
{
    "harvest_profit": "...",
    "harvest_profit_refresh": "...",
    "_marketingplan_session": "..."
}
```

This is the easiest first look.

## 7.4 Full cookie inspection

```python
for c in session.cookies:
    print({
        "name": c.name,
        "value": c.value,
        "domain": c.domain,
        "path": c.path,
        "secure": c.secure,
        "expires": c.expires,
        "rest": c._rest,
    })
```

This matters when debugging why a cookie is not being sent.

# 8. How to extract specific cookies

## Simple extraction

```python
access_cookie = session.cookies.get("harvest_profit")
refresh_cookie = session.cookies.get("harvest_profit_refresh")
session_cookie = session.cookies.get("_marketingplan_session")

print(access_cookie)
print(refresh_cookie)
print(session_cookie)
```

## Safer extraction with fallback

```python
def get_cookie(session: requests.Session, name: str) -> str | None:
    return session.cookies.get(name)
```

# 9. How to extract tokens from JSON body

If login returns JSON, do this:

```python
try:
    data = resp.json()
except ValueError:
    data = None
```

Then inspect likely fields:

```python
if isinstance(data, dict):
    for key in ["token", "access_token", "refresh_token", "jwt", "authorization"]:
        if key in data:
            print(key, data[key])
```

For nested structures, use a recursive helper:

```python
def find_token_like_fields(obj, prefix=""):
    if isinstance(obj, dict):
        for k, v in obj.items():
            full_key = f"{prefix}.{k}" if prefix else k
            if isinstance(v, (dict, list)):
                find_token_like_fields(v, full_key)
            else:
                key_l = k.lower()
                if any(word in key_l for word in ["token", "jwt", "auth", "bearer"]):
                    print(f"{full_key} = {v}")
    elif isinstance(obj, list):
        for i, item in enumerate(obj):
            find_token_like_fields(item, f"{prefix}[{i}]")
```

Usage:

```python
if data is not None:
    find_token_like_fields(data)
```

# 10. How to inspect response headers for auth clues

Sometimes auth or redirect clues live in headers.

```python
for k, v in resp.headers.items():
    if "auth" in k.lower() or "token" in k.lower() or "cookie" in k.lower():
        print(k, "=", v)
```

Important header families to inspect:

* `Set-Cookie`
* `Location`
* `Content-Type`
* anything containing `Auth` or `Token`

# 11. Prepared requests: the hidden debugging superpower

This is one of the most useful mechanics in `requests`.

It lets you inspect what the session is **about to send** before actually sending it.

```python
import requests

req = requests.Request(
    "POST",
    "https://www.harvestprofit.com/api/v3/graphql",
    json={"example": 1},
)

prepared = session.prepare_request(req)

print(prepared.method)
print(prepared.url)
print(prepared.headers)
print(prepared.body)
```

Especially useful:

```python
print("Cookie:", prepared.headers.get("Cookie"))
print("Authorization:", prepared.headers.get("Authorization"))
```

This tells you whether your session is actually sending the cookie/token you think it is.

# 12. What `Session()` does **not** do

This is where people often get confused.

## It does not create auth by itself

Wrong mental model:

> I used `requests.Session()`, so now I am logged in.

Correct mental model:

> I used `requests.Session()` so that if the server gives me auth cookies, they will persist.

## It does not automatically parse tokens from JSON into headers

If login response returns:

```json
{"access_token": "abc"}
```

`requests` will not automatically set:

```http
Authorization: abc
```

You must do that yourself.

## It does not behave exactly like a browser

Browsers handle many additional things:

* JavaScript execution
* dynamic token creation
* same-site behavior details
* local/session storage
* CSP and browser-origin behavior

`requests` is just an HTTP client, not a browser runtime.

# 13. Common response types and what to do with them

## 13.1 HTML response

Use:

* `resp.text`
* `BeautifulSoup` if needed

Typical uses:

* login form pages
* CSRF hidden inputs
* error pages
* redirected HTML login screen

## 13.2 JSON response

Use:

* `resp.json()`
* `pprint` for display

Typical uses:

* GraphQL responses
* API auth responses
* token payloads

## 13.3 Redirect response

Inspect:

* `resp.history`
* `resp.url`
* `Location` headers

This often reveals login workflows and intermediate auth steps.

## 13.4 Binary/file response

Use:

* `resp.content`
* possibly inspect `Content-Type`

Typical uses:

* CSV download
* Excel export
* PDF

# 14. A reusable response debugging helper

```python
import requests
from pprint import pprint


def debug_response(resp: requests.Response, session: requests.Session) -> None:
    print("=" * 80)
    print("STATUS:", resp.status_code)
    print("URL:", resp.url)
    print("METHOD:", resp.request.method)
    print("=" * 80)

    print("\nREQUEST HEADERS:")
    pprint(dict(resp.request.headers))

    print("\nREQUEST BODY:")
    body = resp.request.body
    if isinstance(body, bytes):
        print(body[:3000])
    else:
        print(str(body)[:3000])

    print("\nRESPONSE HEADERS:")
    pprint(dict(resp.headers))

    print("\nRESPONSE COOKIES:")
    for c in resp.cookies:
        print(
            f"name={c.name!r}, value={c.value!r}, domain={c.domain!r}, "
            f"path={c.path!r}, secure={c.secure}, expires={c.expires}"
        )

    print("\nSESSION COOKIES:")
    for c in session.cookies:
        print(
            f"name={c.name!r}, value={c.value!r}, domain={c.domain!r}, "
            f"path={c.path!r}, secure={c.secure}, expires={c.expires}"
        )

    print("\nREDIRECT HISTORY:")
    pprint(resp.history)

    print("\nTEXT PREVIEW:")
    print(resp.text[:5000])

    try:
        print("\nJSON BODY:")
        pprint(resp.json())
    except ValueError:
        print("Response is not JSON.")
```

This helper is worth keeping permanently.

# 15. A reusable token/cookie extraction helper

```python
import requests
from typing import Any


def extract_auth_artifacts(resp: requests.Response, session: requests.Session) -> dict[str, Any]:
    artifacts: dict[str, Any] = {}

    # Common cookies
    for name in [
        "harvest_profit",
        "harvest_profit_refresh",
        "_marketingplan_session",
        "sessionid",
        "csrftoken",
        "remember_user_token",
    ]:
        value = session.cookies.get(name)
        if value:
            artifacts[f"cookie:{name}"] = value

    # Interesting headers
    for k, v in resp.headers.items():
        lk = k.lower()
        if "auth" in lk or "token" in lk or "cookie" in lk:
            artifacts[f"header:{k}"] = v

    # Interesting JSON keys
    try:
        data = resp.json()
    except ValueError:
        data = None

    if isinstance(data, dict):
        for k, v in data.items():
            lk = k.lower()
            if isinstance(v, str) and any(word in lk for word in ["token", "jwt", "auth", "bearer"]):
                artifacts[f"json:{k}"] = v

    return artifacts
```

# 16. Typical login-debug workflow

Use this order every time.

## Step 1 — create session

```python
session = requests.Session()
```

## Step 2 — send login request

```python
resp = session.post(login_url, data=payload, timeout=30)
```

## Step 3 — inspect response

* status code
* final URL
* redirect history
* response headers
* response body preview
* response cookies
* session cookies
* JSON body if present

## Step 4 — identify auth artifacts

Ask:

* Did login set cookies?
* Did login return access token in JSON?
* Did login redirect elsewhere?
* Is there a CSRF token?

## Step 5 — prepare next request before sending

```python
req = requests.Request("POST", api_url, json=payload)
prepared = session.prepare_request(req)
print(prepared.headers)
```

Ask:

* Is the `Cookie` header present?
* Is `Authorization` present?
* Is the request body what I expect?

## Step 6 — call the real API endpoint

Only after the session state is understood.

# 17. CSRF: the common hidden blocker

Sometimes login succeeds, cookies exist, but POST requests still fail.

A common reason is CSRF protection.

Typical pattern:

* server sets a CSRF cookie
* server expects a matching header like `X-CSRF-Token`

This often appears in HTML pages or headers.

How to suspect it:

* GET works
* POST fails with 403 or auth-like error
* cookies are present
* request still rejected

What to inspect:

* HTML hidden input fields
* cookies like `csrftoken`
* headers expected by browser requests

# 18. When `BeautifulSoup` is actually necessary

Use it only when the response is HTML and the value you need is embedded in markup.

Examples:

* hidden CSRF token
* login form input names
* inline JSON inside `<script>` tags
* redirect/meta refresh logic

Example:

```python
from bs4 import BeautifulSoup

soup = BeautifulSoup(resp.text, "html.parser")
for inp in soup.find_all("input"):
    print(inp.get("name"), inp.get("value"))
```

For API/GraphQL JSON workflows, `BeautifulSoup` is usually unnecessary.

# 19. Harvest Profit-specific working hypotheses

Based on your observations, the likely auth structure is:

* short-lived access token in `Authorization`
* same or related access token in cookie `harvest_profit`
* refresh token in `harvest_profit_refresh`
* session continuity in `_marketingplan_session`
* possibly remember-me behavior in `remember_user_token`

That suggests a hybrid auth design.

Practical consequence:

* `session.cookies` may capture important state automatically
* but you may still need to explicitly set `session.headers["Authorization"]`

A very useful experiment is:

```python
access_token = session.cookies.get("harvest_profit")
if access_token:
    session.headers["Authorization"] = access_token
```

Then inspect the prepared GraphQL request.

# 20. Common mistakes

## Mistake 1 — assuming HTTP 200 means authenticated success

Always inspect body and cookies.

## Mistake 2 — assuming `Session()` creates cookies by magic

It only stores what the server gives you.

## Mistake 3 — confusing `BeautifulSoup` with pretty-printing

`BeautifulSoup` parses HTML/XML.
`pprint` formats Python objects.

## Mistake 4 — manually hardcoding giant browser `Cookie` headers too early

This is brittle. Prefer `session.cookies` whenever possible.

## Mistake 5 — not inspecting prepared requests

Sometimes the session contains the cookie, but the outgoing request still does not send what you expect.

## Mistake 6 — ignoring redirects

A login flow may “succeed” only through one or more redirects.

# 21. Minimal reference patterns

## Pattern A — inspect cookies after login

```python
session = requests.Session()
resp = session.post(login_url, data=payload)
print(session.cookies.get_dict())
```

## Pattern B — inspect JSON token after login

```python
data = resp.json()
print(data)
access_token = data.get("access_token")
```

## Pattern C — mirror cookie token into Authorization

```python
access_token = session.cookies.get("harvest_profit")
if access_token:
    session.headers["Authorization"] = access_token
```

## Pattern D — inspect what will be sent

```python
req = requests.Request("POST", graphql_url, json=payload)
prepared = session.prepare_request(req)
print(prepared.headers)
print(prepared.body)
```

# 22. Recommended learning sequence

1. Learn to inspect `resp.status_code`, `resp.headers`, `resp.text`, `resp.json()`
2. Learn the difference between `resp.cookies` and `session.cookies`
3. Learn to use `session.cookies.get_dict()`
4. Learn to inspect `resp.history` and `resp.url`
5. Learn to inspect `prepared` requests before sending
6. Only use `BeautifulSoup` when the response is HTML and structural parsing is needed
7. Treat browser headers as noise until proven necessary

# 23. Bottom line

`requests.Session()` is not magic. It is a persistent client state container.

Its most important role is preserving cookies and shared request state across calls.

When reverse-engineering login/auth, always inspect four surfaces:

* status code
* headers
* cookies
* body

And remember:

* use `pprint` to display Python structures clearly
* use `BeautifulSoup` only for HTML/XML parsing
* use prepared requests to verify what your session will actually send

That combination is enough to understand most real-world auth flows at the HTTP layer.
