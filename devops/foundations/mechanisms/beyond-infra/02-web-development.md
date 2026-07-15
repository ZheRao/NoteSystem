 Web Application Development — A Course for Backend Engineers

*Written for someone who already thinks in data pipelines, schemas, invariants,
and serving layers. Every concept is anchored to something you already own on
the backend, and to GrowLytics (FastAPI + SQLite store + React/Vite dashboard)
where it helps.*

---

## How to read this

This is a map, not a manual. It goes: **substrate → languages → rendering
models → frontend architecture → the backend seam → tooling → production →
scaling GrowLytics → a learning path**. Each part stands alone; skim to what you
need. The goal is that when you hear a term or hit a decision, you know *which
box it lives in and what problem it solves* — the thing that's genuinely hard
when the whole field is new at once.

**The single reframe that makes web dev tractable for a backend person:**

> A web application is a **distributed system in which one of the nodes is a
> computer you don't control, don't trust, and can't see** — the user's browser.
> Everything strange about frontend work (why state is duplicated, why security
> is paranoid, why "just read from the DB" is impossible) falls out of that one
> fact. You already build systems where trust boundaries and data grain matter.
> This is the same discipline with the boundary drawn at the network edge.

---

## Part 1 — The substrate: how the web actually works

Before any framework, understand the pipe. This is the "how does a query
physically execute" layer, and most frontend confusion is really confusion here.

### 1.1 Client and server, and the browser as a runtime

Two programs talk over a network. The **server** (your FastAPI process) holds the
data and the trust. The **client** (the browser) renders UI and issues requests.
The browser is not a document viewer — it is a **sandboxed runtime**: it has a
JavaScript engine, a rendering engine, storage, a network stack, and a security
model. Think of it as an application server you ship code *to* on every visit,
except you have no shell access, the hardware varies wildly, and a fraction of
users are actively hostile.

### 1.2 What happens when someone opens your app

```
type URL → DNS lookup → TCP connect → TLS handshake → HTTP request
        → server responds with HTML → browser parses it
        → browser sees <script>/<link>, fires more requests (JS, CSS, images)
        → JS runs → may fire more requests (your API: /forecast/.../cashflow)
        → pixels
```

- **DNS** turns `app.growlytics.com` into an IP address (a distributed key-value
  lookup).
- **TCP** opens a reliable byte stream.
- **TLS** encrypts it — this is the `https`. Certificates prove the server is who
  it claims. TLS termination is usually done at a load balancer/CDN, not your app.
- **HTTP** is the request/response protocol on top. This is the part you already
  know from FastAPI.

The key insight: **the first response is just the beginning.** A modern app
fires dozens of follow-up requests. Your drilldown's four levels are four such
requests, issued as the user clicks.

### 1.3 HTTP anatomy (you know most of this already)

A request is a **method + path + headers + optional body**; a response is a
**status code + headers + body**.

- **Methods** carry intent: `GET` (read, safe, cacheable), `POST` (create/act),
  `PUT`/`PATCH` (update), `DELETE`. You use these in `api.py`.
- **Status codes** are the contract: `2xx` success, `3xx` redirect, `4xx` client
  error (`400` bad request, `401` unauthenticated, `403` forbidden, `404` not
  found, `422` validation — FastAPI's favorite), `5xx` server error. Your HTML
  file's `getJSON` throws on non-2xx; that's this contract in action.
- **Headers** are metadata: `Content-Type`, `Authorization`, `Cache-Control`,
  `Set-Cookie`, the CORS headers. This is where a lot of web-specific behavior
  hides.

### 1.4 State on a stateless protocol — cookies and sessions

HTTP has no memory; each request is independent. Yet apps need to know "who is
this." Two mechanisms bridge that gap:

- **Cookies**: small key-values the server sets (`Set-Cookie`) and the browser
  automatically returns on every subsequent request to that origin. This is how
  classic **sessions** work: the cookie holds a session id, the server looks up
  the session server-side. Cookies are automatic (sent by the browser without
  your JS doing anything) — which is exactly why they enable CSRF (§7.4).
- **Tokens** (e.g. JWT in `Authorization: Bearer ...`): the client stores a token
  and explicitly attaches it. Not automatic, so not CSRF-prone, but you have to
  store it somewhere and storage has its own risks.

The taxonomy of "where does state live" is the recurring theme of frontend work
(§4.2). Auth state is the first instance of it.

### 1.5 The same-origin policy and CORS

The browser enforces the **same-origin policy**: JavaScript on
`app.growlytics.com` may not freely read responses from `api.growlytics.com`
unless the API opts in. An *origin* is scheme + host + port — so
`localhost:5173` and `localhost:8000` are different origins, which is precisely
why your dev setup needs CORS.

**CORS** (Cross-Origin Resource Sharing) is the API's opt-in: your FastAPI reads
`GROWLYTICS_CORS_ORIGINS` and emits `Access-Control-Allow-Origin` headers saying
"these origins may call me." You already solved this. The mental model:
**CORS is a server-declared allowlist that the browser enforces on the server's
behalf.** It protects users, not servers — it stops a malicious site from
silently using a logged-in user's credentials against your API.

### 1.6 Caching — the performance substrate

The web is caches all the way down: browser cache, CDN edge cache, reverse-proxy
cache, your API's cache, the DB's buffer pool. `Cache-Control`, `ETag`, and
friends are how servers tell each layer how long something is fresh. For a data
dashboard the interesting caches are the **CDN** (for static assets — your built
JS/CSS) and **client-side server-state caches** (§4.3, e.g. React Query holding
your `/cashflow` response). You already cache-and-invalidate on the backend
(`crop_input_streams` is a materialized cache); the web just has more tiers.

---

## Part 2 — The three languages and what each is *for*

The browser runs exactly three languages. Each owns one concern. Keeping them in
their lanes is the oldest discipline in frontend.

### 2.1 HTML — structure and semantics

HTML is the **document schema**: it declares *what things are* — a heading, a
list, a table, a button, a form field. It is not about looks. Good HTML is
**semantic**: `<button>` for actions, `<table>` for tabular data, `<nav>` for
navigation, headings in order. Semantics are what screen readers, search engines,
and browsers rely on — so accessibility (§7.2) and SEO are downstream of HTML
quality.

Backend analogy: HTML is your **table definition** — the structural contract.
CSS and JS are indexes and stored procedures layered on top.

### 2.2 CSS — presentation

CSS (Cascading Style Sheets) controls *appearance and layout*: color, spacing,
typography, and how boxes arrange on screen. The parts that trip up newcomers:

- **The box model**: every element is a box with content, padding, border,
  margin. Most layout confusion is box-model confusion.
- **Layout engines**: **Flexbox** (1-dimensional: a row or column — your stat
  trio) and **Grid** (2-dimensional: your `grid-template-columns: 1fr 380px`).
  Learn these two well; they replaced a generation of hacks.
- **The cascade + specificity**: when multiple rules match, which wins? Rules by
  origin, specificity, and order. This is CSS's "query planner," and it's the
  source of "why is my style not applying."
- **Responsive design**: `@media` queries adapt layout to screen size (your HTML
  file collapses to one column under 880px). Mobile is not optional.
- **Design tokens / variables**: `--ink`, `--in`, `--out` in your files. A
  proto design system — named values reused everywhere, so a rebrand is one edit.

Backend analogy: CSS is a **declarative rule engine** with a conflict-resolution
policy (the cascade). You declare intent; the engine resolves and paints.

### 2.3 JavaScript — behavior

JS is the only programming language the browser runs. It manipulates the page,
responds to events, and talks to your API. Things worth internalizing coming
from Python:

- **The event loop and async.** JS is single-threaded with an event loop.
  `async/await`, Promises, and callbacks are how it stays responsive while
  waiting on the network. Your `getJSON` is `async` for this reason. Same idea as
  Python's asyncio, and FastAPI's async, so this should be familiar.
- **The DOM.** The **Document Object Model** is the live, in-memory tree
  representing the page. `document.getElementById` reads it; setting `.innerHTML`
  mutates it. Frameworks (Part 4) exist largely to stop you touching the DOM by
  hand.
- **Events.** Clicks, input, submit, scroll — the app is a loop reacting to
  these. Your drilldown is an event pipeline: click → fetch → update.
- **Modules.** `import`/`export` split code into files. The browser needs these
  resolved/bundled (your JSX crash-course covered this).

Backend analogy: JS is your **application logic layer**, the DOM is a **mutable
in-memory data structure**, and events are the **request queue** driving it.

---

## Part 3 — The rendering spectrum (the most important axis to understand)

"How and where does HTML get produced" is *the* architectural decision in web
apps, and the vocabulary here (SPA, SSR, SSG…) is where backend folks feel most
lost. Here is the whole spectrum, from least to most machinery.

### 3.1 Static files
Hand-written `.html` served as-is. No server logic. Fast, trivial, but no
dynamic data. (A brochure site.)

### 3.2 Server-rendered multi-page app (MPA) — the classic model
The server builds a full HTML page per request from templates + data, and the
browser loads a fresh page on every navigation. This is **Django templates,
Rails, PHP, Flask + Jinja**. Data lives on the server; the browser is mostly a
renderer. Pros: simple, SEO-friendly, secure by default. Cons: a full page
reload per click, limited interactivity.

### 3.3 Single-Page Application (SPA) — where GrowLytics lives
The server sends one near-empty HTML shell plus a JS bundle. **JavaScript renders
everything in the browser** and fetches data from an API as needed. Navigation
happens client-side (no full reloads). This is **React/Vue/Svelte + Vite**, your
`ForecastExplorer`. Pros: rich interactivity, clean API/UI separation, feels like
an app. Cons: bigger initial JS payload, worse SEO by default, more moving parts,
you re-implement things the browser gave you free (routing, history).

### 3.4 Server-Side Rendering (SSR)
Run the *same* component code on the server to produce real HTML for the first
paint, then "hydrate" it in the browser so it becomes interactive. Best of both:
fast first paint + SEO + full interactivity. This is **Next.js / Remix**. Cost:
you now run a Node server (or edge runtime), and the mental model doubles (code
runs in two places).

### 3.5 Static Site Generation (SSG)
Render pages to HTML **at build time**, deploy the static result. Great for
content that changes rarely (docs, marketing, blogs). This is exactly your
build-time-materialization instinct: do the work once, serve the artifact.

### 3.6 Hybrid / incremental (ISR, streaming, server components)
Modern meta-frameworks blend the above per-route: some pages static, some SSR,
some client-only; regenerate static pages on a schedule (ISR); stream HTML in
chunks; run some components only on the server (**React Server Components**),
close to the data, sending less JS to the client. This is the current frontier —
powerful, and more than a POC needs.

### 3.7 The decision table

| Model | First paint | SEO | Interactivity | Infra | Best for |
|---|---|---|---|---|---|
| Static | fast | good | none | trivial | brochureware |
| MPA (server templates) | fast | good | low | 1 server | forms-y CRUD apps |
| **SPA** | slower | poor* | high | static host + API | **internal dashboards (you)** |
| SSR | fast | good | high | Node/edge server | public, dynamic, SEO-critical |
| SSG | fastest | good | varies | static host | docs, marketing |
| Hybrid/RSC | tunable | good | high | meta-framework | large public products |

\*SEO is rarely a concern for an authenticated internal tool, which is why a
plain SPA is the right, simpler call for GrowLytics. **Don't buy SSR complexity
you don't need.**

---

## Part 4 — Frontend architecture (the SPA world in depth)

Once you're in an SPA, these are the load-bearing concepts. Your `.jsx` already
demonstrates most of them.

### 4.1 Components — the unit of reuse
A component is a function that takes inputs (**props**) and returns UI. It's the
frontend's function/module. Compose small ones into big ones. Your `<Stat/>`,
`<Breadcrumb/>`, `<FlowTip/>` are components; `<ForecastExplorer/>` composes
them. The discipline is the same as writing small pure functions in your
transforms: single responsibility, clear inputs, reusable.

### 4.2 State — the taxonomy that organizes everything
"State" is any data that can change and affects the UI. The crucial skill is
knowing **which kind** you have, because each has a different home:

- **Local/component state** — belongs to one component (a dropdown's open/closed).
  Home: `useState` in that component.
- **Shared/lifted state** — several components need it, so it lives in their
  common parent and passes down as props. Your `o/mn/cat/k` live in
  `ForecastExplorer` and flow down. This is "lifting state up."
- **Global/app state** — needed app-wide (current user, theme). Home: a global
  store — **Context**, **Zustand**, **Redux**, etc.
- **Server-cache state** — data that actually lives in your DB and is *mirrored*
  in the client (your `/cashflow` response). This is the big one (§4.3).
- **URL state** — the current route, query params, the selected month. Often the
  URL *should* be your source of truth so views are shareable/bookmarkable.

> The most common frontend mistake is treating server data as if it were local
> state. It isn't — it's a **cache of a remote source of truth**, with all the
> staleness/invalidation problems you know from materialized views. Which is why:

### 4.3 Data fetching — treat server data as a cache, not as state
Naively you fetch in an effect and stash it in `useState` (fine for a prototype).
In production you use a **server-state library** — **TanStack Query (React
Query)**, **SWR**, or **RTK Query** — which gives you caching, dedup, background
refetch, loading/error states, retries, and invalidation. Conceptually it is a
**client-side materialized cache over your API with an invalidation protocol** —
the exact discipline behind `crop_input_streams`, moved to the browser. For
GrowLytics this is the single highest-value production upgrade to the current
mock `store`.

### 4.4 Routing
Mapping URLs to views. **React Router** (or a meta-framework's file-based
routing) is the URL-to-handler table you already have in `api.py`, but for
screens. Good routing makes the drill state live in the URL
(`/client/c0001/forecast?month=3&cat=Revenue`) so it's shareable and the back
button works.

### 4.5 Forms and validation
Forms are where users write data back, so they need validation, error display,
and submission handling. Libraries: **React Hook Form** for the mechanics,
**Zod** for schema validation. Crucial invariant: **validate on the client for
UX, re-validate on the server for safety.** Client validation is a convenience;
never a trust boundary. (Your FastAPI/Pydantic validation is the real gate.)

### 4.6 Styling approaches
Several schools, pick one per project:

- **Plain CSS / CSS Modules** — what your files use; scoped, simple.
- **Utility CSS (Tailwind)** — style via class names in markup; fast, consistent,
  ubiquitous. (Artifacts here use it.)
- **CSS-in-JS** (styled-components, Emotion) — styles as JS; powerful, some
  runtime cost.
- **Component libraries / design systems** — **shadcn/ui**, **Radix**, **MUI**,
  **Chakra** — pre-built accessible components so you're not hand-rolling
  dialogs and date pickers.

### 4.7 TypeScript — the contract layer (learn this early)
TypeScript adds static types to JS (`.jsx` → `.tsx`). For you this is the
highest-leverage addition: it is **Pydantic for the frontend**. Your
`sources()` response becomes a `type`, and a typo like `s.allocted` is a compile
error, not a runtime surprise. Types also make refactoring safe and document the
shapes flowing through the UI. Strongly recommended as a default for anything
meant to last.

---

## Part 5 — The backend seam (your home turf, from the other side)

You own the API already; here's how the frontend relationship works.

### 5.1 API styles
- **REST** — resources and HTTP verbs; what you built. Simple, cacheable,
  universal. Great default.
- **GraphQL** — client asks for exactly the fields it wants in one request;
  powerful for complex, varied UIs; more server machinery. Consider only if the
  client's data needs get genuinely tangled.
- **RPC / tRPC** — call server functions like local ones, with end-to-end types
  (great in an all-TypeScript codebase). Less relevant with a Python backend.

For GrowLytics, **REST + OpenAPI is the right seam.** FastAPI already emits an
**OpenAPI schema** — you can *generate a typed TypeScript client from it*, so the
frontend's types stay in lockstep with the API. That closes the contract loop
end to end, the same way Pydantic guards the Python side.

### 5.2 Backend-for-Frontend (BFF)
Sometimes you add a thin server layer dedicated to one frontend — to aggregate
calls, hide secrets, handle auth, and shape responses for the UI. A meta-framework
often *is* your BFF. For now your FastAPI is close enough to a BFF; keep it in
mind as the app grows.

### 5.3 Authentication and authorization
- **Sessions (cookies)**: server holds session state; simplest and very safe for
  first-party web apps; needs CSRF protection.
- **Tokens (JWT)**: stateless, common for SPAs and mobile; you manage storage and
  refresh; watch XSS.
- **OAuth 2.0 / OpenID Connect**: "log in with Google/Microsoft/Okta" — delegate
  identity to a provider. OIDC is the identity layer on top of OAuth. This is
  where most production apps land for real users.

**AuthN vs AuthZ**: authentication = *who are you*; authorization = *what may you
do*. Both must be enforced **on the server** — the client can hide a button, but
the API is the real gate. (For GrowLytics: client-side revision filtering is UX;
the API must still refuse data a user shouldn't see.)

---

## Part 6 — The toolchain, organized by the problem it solves

The JS ecosystem's size is intimidating because tools are usually listed flat.
Here they are grouped by *the problem area* so you know what slot each fills. You
do **not** adopt all of these; you pick one per row as needed.

| Problem area | What it does | Representative tools |
|---|---|---|
| **Runtime** | Runs the tooling (not the app) | Node.js (also Bun, Deno) |
| **Package manager** | Installs deps (`pip`) | npm, pnpm, yarn |
| **Build tool / bundler** | Transpile + bundle to browser-ready files | **Vite** (uses esbuild + Rollup), webpack, Turbopack |
| **UI library/framework** | Renders UI from state | **React**, Vue, Svelte, Angular, SolidJS |
| **Meta-framework** | Adds SSR/routing/data conventions | Next.js, Remix, SvelteKit, Nuxt, Astro |
| **Language/types** | Static types (Pydantic-for-UI) | **TypeScript** |
| **Routing** | URL → view | React Router, TanStack Router, framework-native |
| **Server-state / data** | Cache + fetch over your API | **TanStack Query**, SWR, RTK Query |
| **Global state** | App-wide client state | Zustand, Redux Toolkit, Jotai, Context |
| **Styling** | Look & layout | Tailwind, CSS Modules, styled-components |
| **Component library** | Prebuilt accessible UI | shadcn/ui, Radix, MUI, Chakra |
| **Forms & validation** | Inputs + schemas | React Hook Form, **Zod** |
| **Charts / viz** | Data visualization | **Recharts** (yours), Visx, D3, ECharts |
| **Lint / format** | Consistency + catch bugs | ESLint, Prettier, Biome |
| **Unit/component tests** | Fast logic/UI tests | **Vitest**, Jest, Testing Library |
| **End-to-end tests** | Drive a real browser | **Playwright**, Cypress |
| **Type-safe API client** | Generate client from schema | openapi-typescript, orval |
| **Monorepo (later)** | Manage many packages | Turborepo, Nx, pnpm workspaces |

If you internalize only the bold cells, you have a complete, modern, mainstream
stack: **Vite + React + TypeScript + TanStack Query + Recharts + Vitest +
Playwright**, generating a typed client from FastAPI's OpenAPI. That is a
production-grade dashboard stack and a low-regret default.

---

## Part 7 — What a real production web system looks like

A one-page prototype is `browser → your API → SQLite`. Production adds many
layers, each solving a specific failure mode. Here is the whole picture.

### 7.1 The request path in production

```
        Browser
          │  (HTTPS)
     ┌────▼───────────┐
     │      CDN       │  static assets (your built JS/CSS), edge caching, TLS
     └────┬───────────┘
          │
     ┌────▼───────────┐
     │ Load balancer  │  TLS termination, spreads traffic, health checks
     └────┬───────────┘
          │
   ┌──────▼───────┐   ┌───────────────┐
   │  Web / SSR   │   │  (static host │  ← for a pure SPA, the frontend is just
   │  server(s)   │   │   e.g. Azure  │     static files on a CDN; no server
   └──────┬───────┘   │  Static Web   │
          │           │   Apps)       │
   ┌──────▼───────┐   └───────────────┘
   │  API / app   │  your FastAPI (multiple instances, autoscaled)
   │  server(s)   │
   └──┬────────┬──┘
      │        │
 ┌────▼──┐  ┌──▼──────┐
 │ Cache │  │Database │  Redis/etc.       Postgres/managed SQL (SQLite→cloud DB)
 └───────┘  └─────────┘
          │
   ┌──────▼──────────┐
   │  Object storage │  files, exports, backups (blob storage)
   └─────────────────┘
```

Cross-cutting, off to the side: **observability** (logs/metrics/traces/error
tracking), **CI/CD** (build + test + deploy), **secrets/config management**, and
**queues/workers** for background jobs (your ETL batch runs fit here).

### 7.2 Environments, config, and secrets
At least **dev / staging / production**, each with its own config and data. This
is your **configuration-injection invariant**, generalized: the same artifact runs
in every environment; only injected config differs (API URLs, DB connection,
CORS origins, feature flags). Secrets (DB passwords, API keys) live in a secret
manager, never in the repo. You already do this with `GROWLYTICS_CORS_ORIGINS` —
production just formalizes it across every knob.

### 7.3 CI/CD
On every push: **lint → type-check → unit tests → build → e2e tests → deploy**.
The frontend build (`npm run build` → `dist/`) and the API deploy are separate
jobs. Deploys should be automated, reversible (roll back to the last good build),
and gated on green tests. This is the packaging/release discipline you'd apply to
any serious backend service.

### 7.4 The cross-cutting concerns (each is a whole discipline)

- **Performance.** Measured by **Core Web Vitals** (loading, interactivity,
  visual stability). Levers: code-splitting (ship less JS up front), CDN caching,
  image optimization, lazy loading, and *server-state caching* so you don't
  refetch. For a data dashboard, also: paginate/virtualize big tables, and don't
  ship a megabyte of JSON when the user needs a page.

- **Accessibility (a11y).** Keyboard navigation, screen-reader support, color
  contrast, focus management. Not optional and often legally required. It's mostly
  free if your HTML is semantic (§2.1) — your files already add `tabIndex` and
  keydown handlers, which is the right instinct.

- **Security.** The big browser-era threats:
  - **XSS** (cross-site scripting): attacker gets their JS to run in your page,
    usually via unescaped user content. React escapes output by default, which
    defuses most XSS — but `dangerouslySetInnerHTML` and injecting HTML strings
    (like the vanilla file's `innerHTML`) reopen it. **Never inject untrusted
    strings as HTML.**
  - **CSRF** (cross-site request forgery): a malicious site rides a user's
    cookie to hit your API. Mitigated with CSRF tokens / `SameSite` cookies.
    Token-in-header auth sidesteps it.
  - **CORS** (§1.5): the allowlist you already manage.
  - **CSP** (Content-Security-Policy): a header restricting what scripts may run —
    defense-in-depth against XSS.
  - And the classics that live on your side: input validation, authZ on every
    endpoint, no secrets in client code. **Anything in the browser is public** —
    treat all client-side code and data as readable by the user.

- **Internationalization (i18n).** If you'll have non-English users: externalize
  strings, format numbers/dates/currency per locale (relevant for a finance tool
  showing `$` amounts). Libraries: react-i18next, FormatJS.

- **SEO.** Only matters for public pages. For an authenticated internal dashboard,
  skip it — another reason the SPA model fits you.

### 7.5 Observability
You can't fix what you can't see. Production apps ship:
- **Structured logs** (server side, you know these),
- **Metrics** (request rates, latencies, error rates),
- **Traces** (a request's path across services),
- **Error tracking** (e.g. Sentry) capturing frontend exceptions with stack
  traces and user context,
- **RUM** (Real User Monitoring) — actual load/interaction timings from real
  browsers.

The frontend additions (error tracking + RUM) exist because the browser is a node
you don't control (the reframe from the top) — it's the only way to see what
actually happened on the user's machine.

---

## Part 8 — Scaling GrowLytics from prototype to production

Concretely, here's a staged path for *your* app. Each stage is shippable; you
stop wherever the value runs out.

**Stage 0 — where you are.** One `ForecastExplorer.jsx` with a mock `store`,
FastAPI over SQLite, dev via Vite + `uvicorn`. Great for validating the UX.

**Stage 1 — wire to the real API, properly.** Replace the mock `store` with
`fetch` calls, but route them through **TanStack Query** for caching and
loading/error states. Add **TypeScript** and **generate a typed client from
FastAPI's OpenAPI** so the contract is enforced end to end. Add **React Router**
so the drill state lives in the URL.

**Stage 2 — make it a real app shell.** Multiple screens (client list → forecast
explorer → lineage), a component library (shadcn/ui) for consistent, accessible
UI, real forms (React Hook Form + Zod) if users edit inputs, and a global auth
context.

**Stage 3 — harden.** Add **auth** (OIDC via your identity provider) with server
-side authZ on every endpoint. Add **tests**: Vitest for logic/components,
Playwright to drive the four-level drilldown end to end. Add **error tracking**
(Sentry) and structured API logs.

**Stage 4 — productionize the platform.** Move SQLite → a managed cloud DB (your
`store.py` abstraction was designed for exactly this swap). Deploy the frontend
`dist/` to **Azure Static Web Apps** behind a CDN; run FastAPI on **App
Service/Container Apps**, autoscaled, behind a load balancer. Formalize
**dev/staging/prod** with injected config (your CORS/config-injection pattern,
generalized). Automate **CI/CD** with the lint→type→test→build→deploy pipeline.
Move heavy ETL to **background workers/queues**.

**Stage 5 — scale & polish.** Performance (code-splitting, table virtualization
for large forecasts, caching tiers), observability dashboards, feature flags for
safe rollout, i18n if needed. Consider SSR/meta-framework **only** if a concrete
need (public pages, SEO, first-paint on slow devices) ever appears — for an
internal tool it likely never will.

The throughline: **the same invariants you enforce on the backend — single
source of truth, materialized caches with invalidation, config injection, grain
discipline, contracts at every boundary — are exactly what make a frontend
scale.** You're not learning a new philosophy; you're applying yours across the
network edge.

---

## Part 9 — A learning path (order matters)

If you want a sequence rather than a pile:

1. **The substrate (Part 1).** Be fluent in HTTP, origins/CORS, cookies vs
   tokens. You're most of the way there already.
2. **HTML + CSS fundamentals.** Semantic HTML; the box model; Flexbox and Grid;
   responsive `@media`. Build two or three static layouts by hand. This is the
   part backend folks skip and regret.
3. **JavaScript + the DOM + async.** Enough to be dangerous: the event loop,
   Promises/`async`, array methods (`map`/`filter`/`reduce`), modules. Your Python
   makes this fast.
4. **React fundamentals.** Components, props, `useState`, `useEffect`, `useMemo`,
   lists/keys, conditional rendering. You've already seen all of these in your
   `.jsx`; now build one from scratch.
5. **TypeScript.** Convert a small React app to `.tsx`. High leverage for you.
6. **Data & routing.** TanStack Query against a real API; React Router.
7. **Tooling & quality.** Vite config basics, ESLint/Prettier, a few Vitest tests,
   one Playwright flow.
8. **Production concerns.** Auth (OIDC), the security checklist, performance
   basics, a real deploy to Azure.
9. **Meta-frameworks (optional/last).** Next.js or Remix — only once the SPA model
   feels natural and you have a reason (SSR/SEO) to go further.

Skip, for now: CSS-in-JS debates, Redux (Context/Zustand suffice), GraphQL, RSC,
monorepos. They're real, but they're not on the critical path for GrowLytics.

---

## Part 10 — Glossary

| Term | Meaning |
|---|---|
| **Origin** | scheme + host + port; the unit of the same-origin policy |
| **CORS** | server-declared allowlist letting cross-origin JS read responses |
| **DOM** | live in-memory tree of the page that JS manipulates |
| **SPA** | Single-Page App — JS renders everything client-side (GrowLytics) |
| **MPA** | Multi-Page App — server sends a full HTML page per navigation |
| **SSR** | Server-Side Rendering — components run on the server for first paint |
| **SSG** | Static Site Generation — pages rendered to HTML at build time |
| **Hydration** | attaching client interactivity to server-rendered HTML |
| **RSC** | React Server Components — components that run only on the server |
| **Component** | function taking props, returning UI; the unit of reuse |
| **Props** | inputs passed into a component |
| **State** | changeable data that affects the UI (§4.2 taxonomy) |
| **Server state** | client-side cache of data that lives in your DB |
| **Hook** | `useX` function adding behavior to a React component |
| **Bundler** | tool that stitches modules into browser-ready files (Vite/Rollup) |
| **Transpile** | rewrite code form-to-form (JSX→JS, TS→JS) |
| **Meta-framework** | framework atop React adding SSR/routing/data (Next, Remix) |
| **BFF** | Backend-for-Frontend — a thin API layer shaped for one UI |
| **JWT** | signed token used for stateless auth |
| **OAuth/OIDC** | delegated authorization / identity ("log in with…") |
| **AuthN / AuthZ** | authentication (who) / authorization (what you may do) |
| **XSS / CSRF / CSP** | script-injection attack / cookie-riding attack / script-policy header |
| **CDN** | edge network caching static assets near users |
| **Core Web Vitals** | standard metrics for load/interactivity/stability |
| **a11y / i18n** | accessibility / internationalization |
| **Design tokens** | named reusable style values (`--ink`, `--in`) |
| **RUM** | Real User Monitoring — timings from real browsers |

---

*The whole field reduces to one discipline you already have: managing data and
trust across boundaries. Web development just draws one of those boundaries at
the browser — a node you don't own — and everything else (duplicated state,
paranoid security, caching tiers, contracts at every seam) follows from that.
You're extending your backend instincts across the network edge, not starting
over.*

# Frontend Development, Problem Area by Problem Area

*A deep expansion of the Part 6 toolchain table. For each area: **what problem it
solves (what breaks without it)**, **how it actually works**, **the anchor to
your `growlytics_platform`**, and **what you'd reach for**. Read top to bottom
once — the areas build on each other — then use it as a reference.*

Your current system, which every section refers back to:

```
src/growlytics_platform/
├── data_system/                         ## (your ETL producing CSVs)
└── serving_system/
    ├── build/build_serving_store.py     ## CSV  → SQLite  (a BUILD step)
    ├── serve/queries.py                 ## pure query functions
    ├── serve/api.py                     ## FastAPI routes  (the API)
    └── frontend/serving_frontend.html   ## the UI in one file (runs as-is)
        → upgrading to ForecastExplorer.jsx (React, needs a build step)
```

The single most useful thing to hold onto: **you already have a build step, an
artifact, an API, and a hand-written API client.** Frontend tooling is mostly the
same four ideas, applied to the UI, with more specialized tools for each.

---

## GROUP A — The machinery that turns your source into something a browser runs

This is the group you flagged as most opaque ("no idea what this is"), and it's
foundational, so it gets the most detail. The whole group answers one question:
**"I wrote `ForecastExplorer.jsx`; how does that become something running in a
browser?"**

---

### A1. Runtime — *Node.js*

#### The problem it solves
JavaScript was born as a language that only runs *inside a browser*. But all the
frontend tools — the thing that compiles your `.jsx`, the dev server, the test
runner, the linter — are themselves programs, and they're written in JavaScript.
So you need a way to **run JavaScript on your own machine, outside a browser**, to
run those tools. That's what a runtime is.

#### How it works
A browser contains a JavaScript engine (Google's is called **V8**). **Node.js is
that same V8 engine, lifted out of the browser and given access to your file
system, network, and processes.** So Node is "JavaScript as a normal command-line
program," the way `python` is "Python as a normal command-line program."

When you type `npm run dev` or `npx vite build`, you are running Node programs.
Node is the interpreter; the frontend tools are scripts it executes.

#### The anchor to your platform
**Node is to your frontend tooling what the `python` interpreter is to your
backend.** You run `python -m build.build_serving_store` to execute a Python
program; you run `node`/`npm`/`vite` to execute a JavaScript program. Same role.

#### The clarification that removes most of the fear
For a Single-Page App like GrowLytics, **Node is a build-time-only tool. It is
not in your production stack at all.** Node compiles your `.jsx` into plain
static files (`dist/`), and then it's done — those static files are served by a
CDN or `python -m http.server`, with no Node anywhere. Compare: you run Python to
*build* `serving.sqlite`, but SQLite itself doesn't need Python to be read later.
Node is the builder, not the server. (Only if you later adopt SSR — Part B,
meta-frameworks — does Node run in production.)

*(You may also see **Bun** and **Deno** — newer alternatives to Node that do the
same job faster/with more built in. Node is the safe default; ignore the others
for now.)*

---

### A2. Package manager — *npm / pnpm / yarn*

#### The problem it solves
Your app depends on code other people wrote: `react`, `recharts`,
`lucide-react`. You need something to (a) download the right versions of those,
(b) download *their* dependencies recursively, (c) record exactly what got
installed so the next machine gets the identical set, and (d) run project
scripts. That's a package manager. **Without it you'd be manually downloading and
version-matching hundreds of interdependent libraries by hand.**

#### How it works
Four pieces, all of which have exact Python equivalents:

| JS | Python | Role |
|---|---|---|
| `npm` (the command) | `pip` | installs/removes packages, runs scripts |
| `package.json` | `pyproject.toml` | declares your deps + your scripts (`dev`, `build`, `test`) |
| `package-lock.json` | a fully-pinned lockfile (e.g. `poetry.lock`) | the exact resolved versions, so installs are reproducible |
| `node_modules/` | your venv's `site-packages/` | the actual installed dependency code on disk |

`package.json` lists what you *want* ("react, roughly version X"). You run `npm
install`. npm resolves the full dependency tree, downloads everything into
`node_modules/`, and writes the exact result to `package-lock.json`. You commit
`package.json` and the lockfile; you do **not** commit `node_modules/` (it's
regenerated from the lockfile, exactly like you don't commit a venv).

`package.json` also holds **scripts** — named commands. `"dev": "vite"`,
`"build": "vite build"`, `"test": "vitest"`. Then `npm run dev` runs that. This is
the JS equivalent of a `Makefile` or console-script entry points.

#### Your specific question — how does this work with your editable install?
You installed your Python platform with an **editable install** (`pip install -e
.` against `src/growlytics_platform`). That does something specific: instead of
copying your package into `site-packages`, it drops a **link** pointing back at
your source tree, so edits to `data_system`/`serving_system` are picked up
immediately without reinstalling.

The JS world has the exact same concept, called **workspaces** (npm workspaces,
or pnpm/yarn workspaces). A workspace setup lets one repo contain several local
packages and **symlinks them into `node_modules`**, so package A can `import`
package B and edits to B are seen instantly — *precisely what `pip install -e`
does for Python.* The mechanism (a symlink into the dependency folder) is
essentially identical.

Two things to keep straight, though:

1. **Your Python editable install and the frontend are separate dependency
   universes.** Python deps live in your venv/`site-packages` under
   `pyproject.toml`; frontend deps live in `node_modules` under `package.json`.
   They don't mix and don't know about each other. Installing `react` does
   nothing to your Python env and vice versa.

2. **You only need JS workspaces if you split the frontend into multiple local JS
   packages** (e.g. a shared component library + the app). For a single frontend
   app you don't need workspaces at all — just one `package.json`. A realistic
   monorepo layout for you:

```
growlytics_platform/            ## repo root
├── pyproject.toml              ## your Python platform (editable-installed)
├── src/growlytics_platform/    ## data_system, serving_system  (Python)
└── frontend/                   ## the React app — its OWN world
    ├── package.json            ##   frontend deps + scripts
    ├── node_modules/           ##   installed JS deps (git-ignored)
    └── src/ForecastExplorer.jsx
```

The Python side stays exactly as it is; the frontend is a sibling folder with its
own `package.json`. Your existing `pip install -e` doesn't change.

#### What you'd use
**npm** is fine and universal. **pnpm** is faster and more disk-efficient (worth
switching to once you're comfortable). Pick one and stick to it per repo.

---

### A3. Build tool / bundler — *Vite* (this is the big one)

You said "transpile + bundle to browser-ready files" made it *worse*, so let's
build this from the ground up. There are two separate jobs hiding in that phrase —
**transpile** and **bundle** — and it helps to see them apart before seeing the
tool that does both.

#### The root problem: the browser is picky about what it will run
A browser understands exactly three things: HTML, CSS, and a specific dialect of
JavaScript. It does **not** understand:

- **JSX.** `<button onClick={...}>` inside a `.js` file is not valid JavaScript.
  The browser would throw a syntax error immediately.
- **TypeScript.** Types like `const x: number = 5` aren't JavaScript either.
- **Bare imports.** `import { useState } from "react"` — the browser has no idea
  what `"react"` is. There's no file called `react` next to your code; it's a
  package buried in `node_modules`. Browsers can only import by *path or URL*, not
  by package name.
- Sometimes, **brand-new JS syntax** that older browsers haven't caught up to.

So the code you *write* is not code the browser *runs*. Something must sit in
between. That something does two jobs.

#### Job 1 — Transpile (translate the language)
**Transpiling** = automatically rewriting your source into the plain-old
JavaScript a browser accepts, without changing what it does. Two concrete examples
from your own file:

Your JSX:
```jsx
return <span className="fx-catname">{cc.category}</span>;
```
gets transpiled to plain JS:
```js
return React.createElement("span", { className: "fx-catname" }, cc.category);
```

TypeScript:
```ts
const total: number = a + b;
```
gets transpiled by simply **stripping the types**:
```js
const total = a + b;
```

That's all "transpile" means: *same behavior, but rewritten into a dialect the
browser understands.* (The tools that do this are **esbuild**, **Babel**, or
**swc** — you rarely invoke them directly; Vite calls them for you.)

**Backend anchor:** this is a compilation/translation step, exactly like your
`build_serving_store.py` translates CSV (your authoring format) into SQLite (the
format the serving layer can actually query). You author in a convenient format;
a build step translates it into the runnable format. `.jsx` is your CSV; browser
JS is your SQLite.

#### Job 2 — Bundle (assemble many files into few)
Your app is spread across many files that `import` each other, and your imports
also pull in library code from `node_modules`. Two problems if you shipped that
raw:

1. The browser can't resolve `import ... from "react"` (bare specifier, as
   above).
2. Even if it could, a real app is *hundreds to thousands* of little modules.
   Making the browser fetch each one over the network separately would be
   painfully slow.

**Bundling** = starting from your entry file, following every `import` (yours and
libraries'), and stitching the whole graph into a small number of optimized files
the browser can load in a few requests. While it's at it, the bundler also:

- **Minifies** — strips whitespace/comments and shortens names, so the file is
  smaller to download.
- **Tree-shakes** — drops code you imported but never actually use, so you don't
  ship dead weight.
- **Fingerprints/hashes filenames** — outputs `index-a1b2c3.js` so that when you
  deploy a new version the filename changes and the CDN/browser cache can't serve
  a stale copy (cache-busting).

**Backend anchor:** bundling is *exactly* the spirit of
`build_serving_store.py`. You take **many input files** (CSVs / source modules),
resolve the relationships between them, and produce **a few optimized artifacts**
(SQLite / bundled JS) tuned for fast reads at serve time. Same pattern: many
authoring inputs → few optimized serving outputs.

#### The tool: Vite
**Vite** does both jobs and wears two hats depending on whether you're developing
or deploying.

**During development (`npm run dev`):** Vite starts a dev server (default port
**5173** — the same number your HTML file used, deliberately). When your browser
requests a file, Vite transpiles it *on the fly* and serves it, extremely fast.
It also gives you **HMR (Hot Module Replacement)**: a live websocket to the
browser so when you save a file, just that piece is swapped into the running page
without a full reload — and your app's state (which month you'd drilled into) is
preserved. Compare your HTML workflow today: save, hard-refresh, re-click your way
back down. HMR removes all of that.

**For deployment (`npm run build`):** Vite runs the full transpile + bundle +
minify + tree-shake + hash pipeline and writes the result to a **`dist/`** folder:
a tiny `index.html` plus a couple of hashed `.js`/`.css` files. **That `dist/`
folder is plain static files** — no Node, no Vite, nothing special needed to serve
it. You could literally `python -m http.server dist/`, or drop it on a CDN /
Azure Static Web Apps.

#### The whole picture in one diagram
```
AUTHOR                     BUILD (Node + Vite, dev-time only)          RUN
ForecastExplorer.jsx  ─┐
other .jsx/.ts files  ─┼─► transpile (JSX/TS → JS)                 browser
node_modules (react…) ─┘   bundle + minify + tree-shake + hash  ─► loads
                           → dist/  (static .html/.js/.css)         dist/

   your parallel:  CSVs ─► build_serving_store.py ─► serving.sqlite ─► FastAPI reads it
```

Once you see that the frontend has an authoring format, a build step, and a served
artifact — the same three-part shape as your data pipeline — the bundler stops
being mysterious. It's your build step for the UI.

---

## GROUP B — What you actually write the app in

Group A gets your code to the browser. Group B is *the code itself*: the library
that renders UI, the optional framework around it, and the language.

---

### B1. UI library / framework — *React* (and Vue, Svelte, Angular)

#### The problem it solves
In your `serving_frontend.html`, you keep the UI in sync with the data **by hand**:
on every click you mutate a global `state` object, then manually rebuild
`innerHTML` for the affected panels and re-attach event listeners. This is
tedious and it's where your bugs live (the DOM-id-contract crash: rename an `id`,
forget to update the matching `el("...")`, and rendering silently dies).

A UI library solves: **"keep the screen in sync with the data automatically, so I
never touch the DOM by hand."**

#### How it works
You write UI as a **function of state**. In React you declare "given the current
values of `o`, `mn`, `cat`, `k`, here is what the whole screen should look like."
When you change state (`setMn(3)`), React re-runs that function, compares the new
UI description to the old one, computes the minimal set of real DOM changes, and
applies them. You describe the *what*; React handles the *how*.

The unit of code is the **component** — a function taking inputs (**props**) and
returning UI. Your `<Stat/>`, `<Breadcrumb/>`, `<FlowTip/>` are components;
`<ForecastExplorer/>` composes them. Components are to UI what small functions are
to your transforms: named, reusable, single-purpose, testable.

#### The backend anchor
This is the **imperative-vs-declarative** distinction you already know from data:
your HTML file is a hand-written **cursor loop** (seek to this node, update that
cell); React is **SQL** (declare the result you want, let the engine plan the
steps). That's the entire reason `ForecastExplorer.jsx` reads more cleanly than
`serving_frontend.html` — the plumbing is gone, and the id-contract bug class is
gone with it (there are no string ids to keep in sync; the binding *is* the
variable).

#### What you'd use
**React** — largest ecosystem, what your `.jsx` already uses, the safe default.
Alternatives you'll hear about: **Vue** (gentle, very approachable), **Svelte**
(compiles away, minimal runtime, elegant), **Angular** (heavy, batteries-included,
enterprise). Any of them solves the same problem; you've already committed to
React, so stay there.

---

### B2. Meta-framework — *Next.js / Remix* (and why you probably don't need one yet)

#### The problem it solves
React by itself is *just the rendering library*. It doesn't decide how URLs map to
pages, whether pages render on the server or the client, how data loads before a
page shows, or how the whole thing builds and deploys. On a big public product you
want opinionated answers to all of that in one coherent package. A meta-framework
is React **plus** those answers.

#### How it works / what it adds
A meta-framework (built *on top of* React) typically provides: **file-based
routing** (a file `pages/forecast.jsx` automatically becomes the `/forecast`
route), **server-side rendering** (render the page to real HTML on the server for
a fast first paint and SEO), **data-loading conventions** (declare what data a
page needs and the framework fetches it at the right time), and a **build/deploy
story**. Because it renders on the server, a meta-framework runs Node **in
production** (unlike the pure-SPA case from A1).

#### The backend anchor + honest guidance
File-based routing is just "a convention that turns your folder structure into the
URL-to-handler table," similar in spirit to how your `api.py` maps paths to
functions — but for pages instead of endpoints. **However:** meta-frameworks earn
their complexity mainly through SSR/SEO/first-paint, and **an authenticated
internal dashboard like GrowLytics needs none of those.** A plain Vite SPA is
simpler and correct for you. File this away as "the thing to reach for *if* you
ever build public, SEO-critical, content-heavy pages" — and not before.

---

### B3. Language / types — *TypeScript*

#### The problem it solves
Plain JavaScript has no types. Nothing stops you writing `s.allocted` (typo),
passing a string where a number is expected, or assuming an API response has a
field it doesn't. These become **runtime** bugs — the UI crashes or shows garbage
in front of a user. **Without types, the shapes of your data are invisible and
unenforced across the whole UI.**

#### How it works
**TypeScript is JavaScript plus static types** (files become `.ts`/`.tsx`). You
annotate the shapes of your data and function signatures; a **compiler checks them
before the code ever runs** and flags mismatches as errors in your editor. The
types are erased during transpile (B/A3), so the browser runs plain JS — types
cost nothing at runtime, they're purely a compile-time safety net.

Example for GrowLytics — you'd declare the shape your API returns:
```ts
type Source = {
  output_name: string;
  crop: string;
  chosen_revision: string;
  annualized: number;
  allocation_pct: number;
  allocated: number;
};
```
Now if a component tries to read `s.allocted`, or treats `annualized` as a string,
that's a red squiggle in your editor, not a broken screen later.

#### The backend anchor — this one's direct
**TypeScript is Pydantic for the frontend.** Your `serving_system` already defines
Pydantic v2 models so the *server* validates and documents its data shapes;
TypeScript does the identical job on the *client*. And the two connect: because
FastAPI publishes an OpenAPI schema of those Pydantic models, you can
**auto-generate the TypeScript types from it** (see C4), so a change to a Pydantic
model surfaces as a type error in the frontend. That closes the contract loop end
to end — the single highest-leverage thing you can adopt given how you already
think.

#### What you'd use
Just **TypeScript**, from the start of any project meant to last. `.jsx` → `.tsx`.

---

## GROUP C — Structuring the running app and its data flow

Now the app exists and renders. These areas govern how it's organized internally:
how the user moves between views, and how data gets in and out.

---

### C1. Routing — *React Router*

#### The problem it solves
A Single-Page App has one real HTML page but many logical "screens" (client list,
forecast explorer, a specific lineage view). You need: different URLs for
different screens, the browser back/forward buttons to work, and links people can
bookmark and share. Without a router, your whole app is one URL and none of that
works — reloading loses everything, and you can't send a colleague a link to "DJF
Holdings, March, Revenue."

#### How it works
A router maps **URL patterns → which component to show**, and keeps the address
bar in sync with what's rendered — all *without* a full page reload (it
intercepts navigation and swaps components client-side). It also lets the URL
carry state: `/client/c0001/forecast?month=3&category=Revenue`.

#### The backend anchor + why it matters for you specifically
Routing is the **URL-to-handler table you already wrote in `api.py`**, but mapping
to *screens* instead of API endpoints. For GrowLytics the big win is putting your
drill state (`o/mn/cat/k`) **into the URL** instead of only in React state. Then a
forecast view is shareable and bookmarkable, and the back button steps back up the
drilldown — turning your prototype into something that behaves like a real tool.

#### What you'd use
**React Router** (standalone), or a meta-framework's built-in routing if you ever
adopt one.

---

### C2. Server-state / data — first, *what an API client is*, then *TanStack Query*

You said you don't know what an "API client" is, so start there — and the good
news is **you've already built one.**

#### What an "API client" is
An API client is simply **the code that knows how to talk to your API**: which
URLs exist, which HTTP method each uses, how to attach auth, how to send/parse
JSON, and how to turn a non-2xx response into an error. It's a thin layer so the
rest of your app can say `getCashflow(orch)` instead of hand-writing `fetch` and
URL strings everywhere.

This object in your `serving_frontend.html` **is a hand-written API client:**
```js
const api = {
  getOrchestrations: () => getJSON(`/orchestrations`),
  getCashflow: (orch) => getJSON(`/forecast/${enc(orch)}/cashflow`),
  getMonth:    (orch,m) => getJSON(`/forecast/${enc(orch)}/month/${m}`),
  ...
};
```
That's it. An API client is a named set of functions, one per endpoint, that
your UI calls. You already understand the concept; you just didn't have the word.

#### The problem the *server-state library* solves
Calling the API is easy; managing the *results* well is not. Data from your API is
**a copy of something that really lives in the database** — and copies go stale,
need loading and error states, get requested by two components at once, should be
cached so you don't refetch on every click, and must be re-fetched or invalidated
when the underlying data changes. Doing all of that by hand (as your HTML file
does, per-panel, with `innerHTML = "loading…"` and manual error handling) is
repetitive and bug-prone.

#### How it works
A **server-state library** — **TanStack Query** (formerly React Query), or
**SWR** — wraps your API client and gives you, for free: caching keyed by the
request, automatic loading/error state, request de-duplication, background
refetching, retries, and cache invalidation. You describe *what* you want ("the
cashflow for orchestration 2") and it manages the fetching lifecycle.

#### The backend anchor — this will feel familiar
Server data in the client is **a materialized cache of a remote source of truth,
with an invalidation protocol** — which is *exactly* what `crop_input_streams` is
on your backend: a materialized result you read fast and rebuild when inputs
change. TanStack Query is that same discipline, moved to the browser. The most
common beginner mistake — treating fetched data like plain local state — is
exactly the mistake of forgetting it's a cache with staleness. You already have
the right instinct from your serving layer; this just names the tool.

#### What you'd use
**TanStack Query** over a small API client. For GrowLytics this is the number-one
production upgrade to the current mock `store` — swap the mock's methods for real
`fetch` calls, wrap them in TanStack Query, and you get caching + loading/error
handling across the whole drilldown with almost no per-panel code.

---

### C3. Type-safe API client — *generate the client from FastAPI's OpenAPI*

#### The problem it solves
A hand-written API client (like your `api` object, or even a typed one you write
by hand) can **silently drift from the backend.** You rename a field in a Pydantic
model or change an endpoint's shape, and nothing tells the frontend — it just
breaks at runtime in front of a user. The contract between front and back is
maintained by memory and vigilance, which fails.

#### How it works
FastAPI automatically publishes an **OpenAPI schema** — a machine-readable
description of every endpoint, its parameters, and its exact response shapes
(derived from your Pydantic models). A generator tool reads that schema and
**auto-produces a TypeScript API client**: typed functions for every endpoint,
with types for every request and response. Regenerate it whenever the API changes.
Now if the backend renames `annualized`, the generated types change, and every
frontend use of the old name becomes a compile error — the drift is caught at
build time, not in production.

#### The backend anchor
This makes the **Pydantic contract enforceable across the network boundary.**
Your Pydantic models are already the single source of truth for shapes on the
server; generating the client makes them the single source of truth for the
client too. It's the front-to-back version of the grain/contract discipline you
already enforce inside the serving layer.

#### What you'd use
Tools like **openapi-typescript** or **orval** point at FastAPI's
`/openapi.json` and emit the typed client (often wired to work with TanStack
Query). This is a natural Stage-1 addition for GrowLytics precisely because your
backend already emits the schema for free.

---

### C4. Global state — *Context / Zustand / Redux*

#### The problem it solves
Most state belongs to one component (a dropdown's open/closed) or a small subtree
(your `o/mn/cat/k`, which live in `ForecastExplorer` and pass down as props). But
some state is genuinely **app-wide**: the logged-in user, the theme, a global
notification. Passing that down through every intermediate component as props
("prop drilling") becomes miserable. Global state solves: **"make truly app-wide
data available anywhere without threading it through everything."**

#### How it works
A global store holds the shared data in one place; any component can read from it
(and update it) directly, no matter how deep it sits. React's built-in **Context**
covers simple cases (current user, theme). Dedicated libraries — **Zustand**
(simple, modern), **Redux Toolkit** (structured, powerful, more ceremony),
**Jotai** — handle larger or more complex global state with better performance and
tooling.

#### The backend anchor + a caution
Think of it as a small **in-memory singleton** the whole UI can reach — like a
shared config or connection object available process-wide. The important
discipline (one you'll appreciate): **don't put server data in global state.**
Server data belongs in the server-state cache (C2); global state is only for
genuinely client-side, app-wide values. Mixing them is the classic mess. Keep the
buckets separate: local state → `useState`; shared subtree → props; server data →
TanStack Query; truly global client data → a global store.

#### What you'd use
Start with **Context** for the one or two truly-global things (user, theme). Reach
for **Zustand** if that gets unwieldy. You likely won't need Redux for a
dashboard.

---

## GROUP D — What the user actually sees

The app runs and has data. Group D is the visual and interaction layer.

---

### D1. Styling — *CSS Modules / Tailwind / component libraries*

#### The problem it solves
Raw CSS is global: a rule you write for one component can accidentally affect
another across the app (name a class `.card` twice with different intent and they
collide). At scale you need **styles that don't leak**, a consistent design
language, and a way to not rewrite the same spacing/color rules forever.

#### How it works — the main approaches
- **Plain CSS / CSS Modules** — what your files use. CSS Modules scope class names
  to a single component automatically, killing the collision problem. Simple,
  no new mental model.
- **Utility CSS (Tailwind)** — you style by composing tiny predefined class names
  directly in the markup (`class="flex gap-2 text-sm"`). Fast, extremely
  consistent, hugely popular; the tradeoff is busy-looking markup.
- **CSS-in-JS** (styled-components, Emotion) — write styles as JavaScript
  alongside components. Powerful and dynamic, small runtime cost.

#### The backend anchor
Your CSS variables (`--ink`, `--in`, `--out`) are already **design tokens** —
named, reusable values, so a rebrand is one edit instead of a find-replace. That's
the same "single source of truth, change once" instinct you apply to config. You
have the right idea already.

#### What you'd use
You said this matters least, so: keep **CSS Modules** for now (closest to what you
know). Learn **Tailwind** later if/when you adopt a component library that assumes
it. Not a priority.

---

### D2. Component library — *shadcn/ui / Radix / MUI*

#### The problem it solves
Real apps need lots of fiddly, interactive widgets: dropdowns, modals, tabs, date
pickers, tables with sorting, tooltips, comboboxes. Building these *correctly* —
keyboard navigation, focus trapping, screen-reader support, edge cases — is
genuinely hard and easy to get subtly wrong. A component library gives you these
**pre-built, accessible, and consistent**, so you assemble instead of hand-craft.

#### How it works
You install the library and drop in its components (`<Dialog>`, `<Select>`,
`<Tabs>`, `<DataTable>`), styling them to your brand. They've already solved the
accessibility and interaction details. **shadcn/ui** (you copy the component code
into your project and own it — very popular), **Radix** (unstyled, accessible
primitives), **MUI**/**Chakra** (full styled systems) are the common choices.

#### The backend anchor
This is **"use a well-tested library instead of hand-rolling a hard, error-prone
thing"** — the same reason you use FastAPI/SQLite/Pydantic rather than writing an
HTTP server, a storage engine, and a validator yourself. Accessibility and
interaction correctness are the frontend's version of "correctness details you
don't want to reimplement."

#### What you'd use
**shadcn/ui** is the current sweet spot (you own the code, it's accessible, it
looks good). Adopt one when you outgrow hand-built widgets — likely around Stage 2
of your rollout.

---

### D3. Forms & validation — *React Hook Form + Zod*

#### The problem it solves
The moment users *write* data back (editing a crop's acreage, a unit price), you
need to collect inputs, validate them, show clear per-field errors, disable submit
while pending, and handle server rejections. Doing this by hand for every form is
repetitive and easy to get wrong (and getting it wrong means bad data or bad UX).

#### How it works
A **form library** (**React Hook Form**) manages the mechanics — tracking field
values, touched/dirty state, submission, and wiring errors to fields. A **schema
validator** (**Zod**) lets you declare the rules once ("acreage is a positive
number; unit price ≥ 0") and validates against them, producing typed, structured
errors. Together: declare the shape, get validation + error display + typed data.

#### The backend anchor — and a hard rule
Zod schemas are **Pydantic models for the client**: declarative shape + validation
producing a typed result. But the critical invariant, which your backend instinct
already tells you: **client-side validation is for UX convenience, never for
trust.** It gives instant feedback, but a hostile client can bypass it entirely.
**The server must re-validate everything** — your FastAPI/Pydantic layer is the
real gate. Client validation = fast feedback; server validation = the actual
boundary. Never confuse the two.

#### What you'd use
**React Hook Form + Zod**, once GrowLytics has any data-entry (Stage 2+). Pure
read-only dashboards can skip this until then.

---

### D4. Charts / visualization — *Recharts / Visx / D3*

#### The problem it solves
Turning numbers into accurate, interactive visuals — axes, scales, tooltips,
legends, responsive resizing, hover/click interactions — is a lot of finicky
geometry and math you don't want to hand-compute. For a forecasting product,
charts *are* the product, so you want this to be solid.

#### How it works — a spectrum of control vs effort
- **Recharts** (what your `.jsx` uses) — declarative React chart components
  (`<AreaChart>`, `<Area>`, `<XAxis>`). You hand it data and props; it draws.
  Fast to build, covers most needs. Your gradient-split area chart with tooltips
  and an interactive active-dot came from a handful of lines.
- **Visx** — lower-level React + D3 primitives; more control, more work.
- **D3** — the foundational, maximally powerful, maximally manual library that the
  others build on. Reach for it only when you need something bespoke.

#### The backend anchor
Same "library vs hand-roll" calculus as everything else: Recharts is to charting
what FastAPI is to HTTP — it handles the hard, standard parts so you express
intent. Your `serving_frontend.html` hand-rolls its chart as SVG strings; the
`.jsx` delegates to Recharts, which is why it's cleaner and more capable.

#### What you'd use
**Recharts** — you're already on it, and it's the right level for a finance
dashboard. Drop to Visx/D3 only for a visualization Recharts can't express.

---

## GROUP E — Keeping it correct as it grows

A prototype can be correct by hand-checking. A production app changed by multiple
people over months needs automated guarantees. This group is your CI safety net —
and it's the most natural bridge from your backend habits, since you already value
invariants and tests.

---

### E1. Lint / format — *ESLint + Prettier* (or *Biome*)

#### The problem it solves
Two separate annoyances: (1) **formatting** bikeshedding — spaces, quotes, commas
— wastes review time and pollutes diffs; (2) **whole classes of bugs and bad
patterns** (unused variables, a React hook used incorrectly, `==` vs `===`) that a
tool can catch automatically before they cause trouble.

#### How it works
A **formatter** (**Prettier**) rewrites your code to one canonical style on save —
no more debates, clean diffs. A **linter** (**ESLint**) statically analyzes your
code for likely bugs and rule violations and flags them. (**Biome** is a newer
single tool doing both, fast.) Both run in your editor and in CI, so problems are
caught before merge.

#### The backend anchor
This is your Python `black` (formatter) + `ruff`/`flake8` (linter) — the exact
same tools-by-role, different names. You already know why they're worth it.

#### What you'd use
**Prettier + ESLint** (or **Biome**). You flagged this as "maybe later" — that's
fine; it's low-effort to add whenever, and it pays off the moment more than one
person touches the code.

---

### E2. Unit / component tests — *Vitest + Testing Library*

#### The problem it solves
You need to verify that a piece of logic or a single component behaves correctly —
in isolation, fast, on every change — so a refactor or new feature doesn't
silently break existing behavior. You said you know *why* to test but not *how* on
the frontend; here's the how.

#### How it works
- A **test runner** (**Vitest** — pairs naturally with Vite) runs your test files
  and reports pass/fail, exactly like **pytest** runs yours. Same idea: files of
  `test_*` functions with assertions.
- For pure logic (say, a `money()` or `compact()` formatter, or your
  cashflow-grouping function), tests look just like your Python unit tests: call
  the function, assert the output.
- For **components**, **React Testing Library** renders a component into a
  simulated DOM in memory, lets you find elements the way a user would ("the
  button labeled *Revenue*"), simulate a click, and assert what appears. E.g.:
  render `<ForecastExplorer/>`, click the March point, assert the category rows
  show — verifying a drilldown step without a real browser.

#### The backend anchor
Vitest ≈ pytest; component tests ≈ integration tests over a small unit. The
mental model is identical to testing `queries.py`: set up inputs, exercise the
unit, assert the result. The only new part is "rendering a component and
interacting with it," which Testing Library makes read like user actions.

#### What you'd use
**Vitest + React Testing Library.** Start with pure-logic tests (easiest, highest
value — your formatters and any data-shaping functions), then add a few component
tests for critical interactions.

---

### E3. End-to-end (E2E) tests — *Playwright*

#### The problem it solves
Unit tests check pieces; they don't prove the *whole system works together* from
the user's point of view — real browser, real (or realistic) API, real clicks. An
E2E test catches the failures that only appear when everything is wired up: the
build broke, the API contract shifted, a route 404s, the drilldown's third level
doesn't load.

#### How it works
An E2E tool (**Playwright**, or Cypress) launches a **real browser**, navigates to
your running app, and performs a scripted user journey — click this, type that,
assert the page shows the expected result — failing loudly if any step breaks. For
GrowLytics, one E2E test would drive the entire four-level drill: load the app →
click a chart point → click a category → click a source → assert the versioned
lineage renders. If any layer (frontend, build, API, data) is broken, that test
goes red.

#### The backend anchor
E2E is your **full-stack integration/smoke test** — the one that hits the running
API against a real store and asserts an end-to-end result, rather than testing a
single function. It's the highest-confidence, slowest test; you keep a small
number of these for critical paths.

#### What you'd use
**Playwright.** A handful of E2E tests covering your main user journeys, run in CI
before deploy, catches the scary "everything's individually fine but the whole
thing is broken" failures.

---

## How it all fits together (the one-screen recap)

```
GROUP A — get your code to the browser
  Node          runs the tools (build-time only; not in your SPA's production)
  npm/pnpm      installs deps (= pip); package.json (= pyproject); workspaces (= pip -e)
  Vite          transpile (JSX/TS → JS)  +  bundle (many files → few) → dist/ static files

GROUP B — what you write
  React         UI as a function of state (declarative = SQL; vs your HTML's cursor loop)
  meta-framework  React + SSR/routing/data — you don't need one yet
  TypeScript    types = Pydantic for the frontend

GROUP C — structure & data
  React Router  URL → screen (= api.py's routing, for pages); put drill state in the URL
  API client    functions that call your API (your `api={...}` already is one)
  TanStack Query  caches server data (= a client-side crop_input_streams)
  generated client  types from FastAPI's OpenAPI → contract can't drift
  global state  app-wide client data only (user/theme); NOT server data

GROUP D — what the user sees
  styling       CSS Modules now; Tailwind later; your --vars are design tokens
  component lib (shadcn/ui)  prebuilt accessible widgets — don't hand-roll modals
  forms (RHF+Zod)  Zod = Pydantic on the client; server still re-validates (hard rule)
  charts (Recharts)  declarative charts — already in your .jsx

GROUP E — keep it correct
  ESLint+Prettier  = ruff + black
  Vitest + Testing Library  = pytest, plus render-and-click component tests
  Playwright    real-browser E2E = your full-stack smoke test
```

**The through-line:** every one of these areas is a specialized version of
something you already do on the backend — a build step, an artifact, a contract,
a cache, a validator, a test. Frontend isn't a different philosophy; it's your
philosophy applied on the far side of the network, where the runtime is a browser
you don't control. You built the primitive version by hand
(`build_serving_store.py`, `api.py`, the `api={}` client, manual DOM updates);
these tools are the mature, specialized replacements for each of those hand-built
pieces.