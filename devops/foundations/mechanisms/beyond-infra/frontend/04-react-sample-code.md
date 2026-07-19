# JSX & the Build-Tool World — a Crash Course

*Anchored in your two files: `serving_frontend.html` (vanilla, runs as-is) and
`ForecastExplorer.jsx` (React, needs a build step). Framed against the ETL /
backend mental models you already work in.*

---

## 0. The one-paragraph mental model

`serving_frontend.html` is **source that is also the runnable artifact** — the
browser executes exactly what you wrote. `ForecastExplorer.jsx` is **source that
must be compiled into a runnable artifact first**. This is the same split you
already live with in your pipeline: the `.py` transform code you write is *not*
what the serving API reads — you *build* `serving.sqlite` from it, and the API
reads the built artifact. `.jsx` is the transform code; the browser is the
serving layer; **Vite is the build step in between** (plus a dev server that
does that build on the fly). Everything else in this document is detail hanging
off that one sentence.

---

## 1. What a `.jsx` file actually is

`.jsx` = **JavaScript + XML**. It is a normal JavaScript file with one extra
syntactic power: you can write HTML-looking tags directly inside the code.

```jsx
const active = cc.category === cat;
return (
  <button className={"fx-catrow" + (active ? " on" : "")}
          onClick={() => pickCat(cc.category)}>
    <span className="fx-catname">{cc.category}</span>
  </button>
);
```

That `<button>...</button>` is **not** a string and **not** real HTML. It is
sugar. A compiler rewrites it into plain function calls before the browser ever
sees it:

```js
React.createElement("button",
  { className: "fx-catrow" + (active ? " on" : ""), onClick: () => pickCat(cc.category) },
  React.createElement("span", { className: "fx-catname" }, cc.category)
);
```

So a `.jsx` file is JavaScript that describes a **tree of UI objects** by
writing them in tag form. The `x` in `jsx` is the only thing a browser can't
parse natively — hence the build step (§4).

Two consequences you'll see all over your file:

- **`className`, not `class`.** `class` is a reserved word in JavaScript, so JSX
  uses `className`. Same reason `for` becomes `htmlFor`.
- **`{ ... }` drops you back into JavaScript** anywhere inside the tags:
  `{cc.category}`, `{money(cc.amount)}`, `{cats.map(...)}`. The braces are the
  seam between "markup mode" and "code mode."

---

## 2. The real difference: imperative DOM vs declarative components

This is the heart of *why the `.jsx` feels more appealing*, and it maps cleanly
onto something you already know deeply.

### Your HTML file is the imperative cursor loop

Look at how a category click flows in `serving_frontend.html`:

```js
async function selectCategory(category){
  state.category = category;                    // 1. mutate global state by hand
  el("sourceCrumb").textContent = `· ${category} · ${state.point.month}`;  // 2. poke a node
  el("sources").innerHTML = `<div class="placeholder">loading…</div>`;     // 3. poke another
  const rows = await api.getSources(...);        // 4. fetch
  renderSources(rows);                           // 5. rebuild innerHTML, re-attach listeners
}
```

You are the one walking the DOM, node by node, telling it *how* to change: set
this text, blank that panel, rebuild this list, re-wire these click handlers.
`el(id)` is `document.getElementById(id)`. **This is a cursor loop.** It's the
front-end equivalent of opening a cursor and issuing row-by-row `UPDATE`s: total
control, but you own every step and every consistency guarantee.

This is also exactly where your earlier crashes came from — the **DOM id
contract**. `el("sources")` only works if an element with `id="sources"` exists.
Rename the `id` in the HTML and forget to update the string, and you get a null
crash. The contract lives in two places (the markup and the string) and nothing
enforces that they agree.

### Your JSX file is the declarative query

React flips it. You never touch the DOM. You write a **pure function of state**:
"given `o`, `mn`, `cat`, `k`, here is what the whole screen should look like."

```jsx
{mn && cat && (
  <div className="fx-level fx-anim">
    <table className="fx-table">
      <tbody>
        {srcs.map((s) => (
          <tr key={s.k} className={"fx-src" + (s.k === k ? " on" : "")}
              onClick={() => pickSrc(s.k)}>
            ...
          </tr>
        ))}
      </tbody>
    </table>
  </div>
)}
```

You declare the *result*. When you call `setCat(category)`, React re-runs this
function, compares the new UI tree to the old one, computes the minimal DOM
changes ("the diff"), and applies them. You describe **what**; the runtime
figures out **how**.

> **The analogy that makes this click:** imperative DOM is a hand-written cursor
> loop; declarative React is **SQL**. In SQL you don't write "seek to this row,
> copy this cell, advance" — you declare the shape of the result you want and the
> engine plans the steps. React is a `SELECT` for your UI. That is the whole
> reason the `.jsx` reads more cleanly: the plumbing is gone.

And notice the id-contract bug **cannot happen** in the JSX file. There are no
string ids to keep in sync — the binding between "the source row" and "the
variable `s`" is the variable itself. An entire class of your past bugs is
designed out of existence.

### Side-by-side summary

| Concern | `serving_frontend.html` (vanilla) | `ForecastExplorer.jsx` (React) |
|---|---|---|
| UI update model | Imperative: you mutate the DOM | Declarative: you set state, runtime diffs |
| State | One mutable global `state` object | `useState` hooks (`o`, `mn`, `cat`, `k`) |
| "Where is the truth?" | Split between `state` + DOM | State only; DOM is derived |
| Wiring events | `addEventListener` after every render | `onClick={...}` inline, always in sync |
| Reuse | Copy/paste render functions | Components (`<Stat/>`, `<Breadcrumb/>`) |
| The chart | Hand-rolled SVG/HTML string | `recharts` `<AreaChart>` component |
| Derived values | Recomputed by hand each render | `useMemo(fn, [deps])` |
| Runs in browser directly? | **Yes** | **No — must be built first** |
| Dependencies | None | `react`, `recharts`, `lucide-react`, … |

---

## 3. State and derived values — the `useMemo` / materialized-view parallel

Your JSX file has this cluster near the top of the component:

```jsx
const [o, setO] = useState(2);          // selected orchestration
const [mn, setMn] = useState(null);     // selected month
const [cat, setCat] = useState(null);   // selected category
const [k, setK] = useState(null);       // selected source key

const flow = useMemo(() => store.cashflow(o), [o]);
const cats = useMemo(() => (mn ? store.month(o, mn) : []), [o, mn]);
const srcs = useMemo(() => (mn && cat ? store.sources(o, mn, cat) : []), [o, mn, cat]);
```

- **`useState`** is a single cell of state that survives re-renders. `[value,
  setValue]` — reading `o` gives the current value; calling `setO(...)` both
  updates it *and* triggers a re-render. This replaces the mutable `state = {...}`
  object in the HTML file. The difference is that a `setState` call is what *drives*
  the re-render — you never manually call a render function.

- **`useMemo(fn, [deps])`** is a **memoized / materialized derivation**. `cats`
  is recomputed only when `o` or `mn` changes; otherwise the cached result is
  reused. This is precisely the materialization instinct you already applied in
  `crop_input_streams`: compute the resolved value once, key it by its inputs,
  don't recompute on every read. The `[o, mn]` array is the cache key.

The whole component is therefore a **dependency graph of derivations over a few
state atoms** — a tiny reactive DAG. Change `mn`, and only the nodes downstream
of `mn` recompute. That should feel familiar.

---

## 4. Why the browser can't run `.jsx` — the build step

A browser understands three things: HTML, CSS, and JavaScript. It does **not**
understand:

1. **JSX tags** (`<button>` inside a `.js` expression) — not valid JavaScript.
2. **Bare module imports** like `import { useState } from "react"`. The browser
   has no idea where `"react"` lives; there's no file called `react` next to your
   code.

So `.jsx` is **source code in a language the runtime doesn't speak**, exactly
like your `.py` transforms are not something SQLite can read. Something has to
translate and package it first. That "something" does two jobs:

- **Transpile** — rewrite JSX into `React.createElement(...)` calls, and rewrite
  modern JS into whatever older browsers need. (Tool: **Babel** or **esbuild**.)
- **Bundle** — follow every `import`, pull `react`, `recharts`, `lucide-react`
  and your own files, and stitch them into a small number of plain `.js` files a
  browser *can* load. (Tool: **Rollup** / **esbuild** / **webpack**.)

The output of that process — a few `.js` + `.css` files plus a tiny
`index.html` — is the **artifact** the browser runs. You author `.jsx`; you ship
built `.js`. This is the same source → artifact boundary as author `.py` →
build `serving.sqlite` → serve it.

```
YOUR PIPELINE:     transforms.py   ──build──▶   serving.sqlite   ──served by──▶  FastAPI
FRONTEND:          ForecastExplorer.jsx ──build──▶  bundle.js/.css ──served by──▶  browser
```

---

## 5. What Vite is (and how it hosts the thing)

**Vite** (French for "fast," rhymes with "street") is the tool that runs that
build step for you. It wears two hats.

### Hat 1 — the dev server (`npm run dev`)

This is the one you'll use constantly, and it's the direct counterpart to your
`python -m http.server 5173`.

- Your HTML file needs `python -m http.server` only to get past the `file://`
  CORS problem — the server just hands over bytes unchanged.
- Vite's dev server hands over bytes **and transforms them on the way out**. When
  the browser requests `ForecastExplorer.jsx`, Vite transpiles it to real JS in
  memory (using **esbuild**, which is extremely fast) and returns that. You never
  see a build folder during development.
- It adds **HMR (Hot Module Replacement)**: a websocket between browser and dev
  server. Save the file, and Vite pushes just the changed module into the running
  page — your drill-down state (`o`, `mn`, `cat`) is preserved, no full reload.
  Compare the HTML workflow: save, then hard-refresh, then re-click your way back
  down to the state you were debugging.

> Fun detail you already stumbled onto: Vite's **default dev port is 5173**.
> Whoever wrote your HTML comment picked `python -m http.server 5173` to mirror
> Vite's convention, so the two worlds line up.

### Hat 2 — the production build (`npm run build`)

For deployment, Vite switches to **Rollup** under the hood and produces the
static artifact: minified, hashed `.js`/`.css` in a `dist/` folder, plus an
`index.html` that points at them. That `dist/` folder is **pure static files** —
no Node.js required to serve it. You can host it with *any* static file server,
including `python -m http.server dist/`, a CDN, or Azure Static Web Apps.

So the lifecycle is:

```
DEV:     npm run dev    → Vite dev server on :5173, transpile-on-request + HMR
BUILD:   npm run build  → dist/ (plain static bundle, browser-ready)
DEPLOY:  serve dist/    → any static host / CDN
```

**Key reassurance:** the production artifact is *more* like your HTML file than
less — it's static files a dumb server streams. The Node/Vite machinery exists
only at authoring and build time, not at runtime. This mirrors your store: the
heavy Python runs at build time; the served thing is a static SQLite file.

---

## 6. What the new format buys you

Concretely, versus `serving_frontend.html`:

- **Components as reusable units.** `<Stat k="Inflow" v={...} c={C.inflow}/>` is
  declared once and used three times. In the HTML file, the three stat tiles are
  hand-built strings. Components are to UI what functions are to your transforms —
  a named, parameterized, testable unit. This is the single biggest scaling win.

- **No manual re-wiring, no id contract.** Events (`onClick`) live on the element
  and can never drift out of sync, so your recurring null-reference bug class is
  gone.

- **A real ecosystem.** `recharts` gave you a gradient-split area chart, tooltips,
  and an interactive active-dot for a few lines of JSX. The HTML file hand-rolls
  its chart. Thousands of such packages exist (tables, date pickers, virtualized
  lists) and install with one command.

- **Declarative flow = fewer states to hand-manage.** You stopped writing "blank
  this panel, show loading, then fill it." You describe each state (`!mn`,
  `mn && !cat`, `mn && cat`) and React shows the right one.

- **Better dev loop.** HMR keeps your app state while you edit — a real
  productivity gain when debugging a 4-level drilldown.

---

## 7. What you give up (the honest compromises)

Nothing is free. Moving from the single HTML file to the JSX/Vite world costs:

- **A toolchain and a `node_modules` folder.** You now need Node.js installed, a
  `package.json`, and an install step. `node_modules` is famously large. This is
  new surface area to maintain and to onboard others onto.

- **You can no longer "just open the file."** There's an indirection between
  what you write and what runs. Errors sometimes point at generated code;
  **source maps** map them back, but it's a layer to understand.

- **Build/deploy complexity.** Instead of copying one `.html`, you run a build
  and deploy a folder. CI has one more job.

- **A bigger baseline payload.** React itself is ~40 KB gzipped before your code.
  For a tiny widget that's pure overhead; the HTML file ships nothing extra. It
  pays off as the app grows, not at hello-world scale.

- **Version churn.** The JS ecosystem moves fast; dependencies need occasional
  updating. Your vanilla file has zero dependencies and will run untouched for
  years.

- **A genuinely new mental model.** Declarative rendering, hooks, the dependency
  arrays of `useMemo`/`useEffect`, and "never touch the DOM" all take a beat to
  internalize coming from imperative code.

Rule of thumb, in your terms: the HTML file is a **one-off script**; the
JSX/Vite setup is a **packaged project**. You already make this call on the
backend — a throwaway `.py` vs. a proper package with `pyproject.toml`, tests,
and a build. Same trade, same reasoning.

---

## 8. The dependency stack, in `pip` terms

This maps almost one-to-one onto Python packaging, which should shortcut a lot:

| JavaScript / Node | Python equivalent | Role |
|---|---|---|
| `node` | `python` | The runtime that runs the tooling |
| `npm` (or `pnpm`, `yarn`) | `pip` | Installs packages |
| `package.json` | `pyproject.toml` | Declares deps + scripts (`dev`, `build`) |
| `package-lock.json` | `poetry.lock` / `requirements.txt` (pinned) | Exact resolved versions |
| `node_modules/` | your virtualenv's `site-packages/` | Installed dependency code |
| `npm run dev` | `make dev` / a console script | Runs a named command |
| a bundler (Vite/Rollup) | building a wheel / container image | Packages source into a deployable artifact |

The `import` lines at the top of `ForecastExplorer.jsx`
(`react`, `recharts`, `lucide-react`) are entries that must exist in
`package.json` and be installed into `node_modules`, exactly as your Python
`import pandas` requires `pandas` in the environment.

---

## 9. Extending this to a production dashboard / website

A single component is the seed. A production GrowLytics dashboard adds layers
*around* it — and each layer has a clean backend analogy:

- **Routing.** Multiple screens (client list, forecast explorer, admin) with real
  URLs. Library: **React Router**, or a meta-framework's built-in file-based
  routing. Think of routes as the URL-to-handler table you already have in
  `api.py`, but for pages.

- **Data fetching as a discipline.** Right now the JSX uses a mock `store`. In
  production you'd use **TanStack Query (React Query)** to call your FastAPI
  endpoints. It gives you caching, loading/error states, retries, and
  deduplication for free — it is essentially a **client-side materialized cache
  with invalidation** over your API. (See §11 for the wiring.)

- **A component library / design system.** Rather than hand-styling everything,
  teams adopt something like **shadcn/ui**, **Radix**, or **MUI** for accessible
  buttons, tables, dialogs. Your CSS-variable palette (`--ink`, `--in`, `--out`)
  is already a proto design system.

- **TypeScript.** `.jsx` → `.tsx`. This is the big one for you: TypeScript puts a
  **schema/contract on the UI**, the same way Pydantic does on your API. Your
  `sources()` response shape would be a `type`, and the compiler catches a typo
  like `s.allocted` before runtime. For someone with a data/backend instinct,
  TypeScript is usually the single highest-value addition.

- **Testing.** **Vitest** (unit, pairs with Vite) + **Playwright** (end-to-end,
  drives a real browser through your drill-down). Analogous to your pytest suite
  over the serving layer.

- **Auth + API integration.** Token handling, protected routes, an API client
  layer. Structurally similar to guarding FastAPI routes.

- **CI/CD + hosting.** `npm run build` in CI, deploy `dist/` to a static host or
  CDN. On Azure this is typically **Azure Static Web Apps** (static frontend) with
  your **FastAPI on App Service / Container Apps** as the API — and your CORS
  allowlist (`GROWLYTICS_CORS_ORIGINS`) now lists the Static Web App's domain
  instead of `localhost:5173`. This is the exact CORS/origin story you already
  worked through, just with production hostnames.

A realistic production layering:

```
Vite + React + TypeScript          ← build tool + UI + type contract
  React Router                      ← pages / URLs
  TanStack Query                    ← data layer over FastAPI (cache + states)
  shadcn/ui (or MUI/Radix)          ← component/design system
  Vitest + Playwright               ← tests
  → build to dist/ → Azure Static Web Apps  (API: FastAPI on App Service)
```

---

## 10. How this evolves going forward

The direction of travel, so today's choices don't surprise you later:

- **Build tools consolidated on esbuild/Rollup-class speed.** Vite won this space
  by being fast; the older, slower webpack-style setups are what it replaced. Vite
  is a safe, mainstream default right now.

- **Meta-frameworks sit on top of Vite/React.** **Next.js** and **Remix/React
  Router** add server-side rendering (SSR), file-based routing, and data-loading
  conventions. You reach for these when you need SEO, fast first paint, or
  server-side data loading. For an internal, authenticated dashboard (your case),
  a plain Vite **SPA** — everything renders in the browser — is usually the right,
  simpler call. Don't adopt a meta-framework until a concrete need pushes you
  there.

- **The client/server line is blurring.** **React Server Components** and "edge"
  rendering move some component execution back onto the server, closer to your
  data — conceptually pulling rendering toward the backend you already own. It's
  worth knowing the term exists; it is not something a POC needs.

- **TypeScript is becoming the default**, not the exception. If you expect this to
  live, starting in `.tsx` saves a migration later.

The stable takeaway: **Vite + React (ideally + TypeScript) as a SPA** is the
low-regret baseline. Add SSR/meta-frameworks only when a real requirement
demands them.

---

## 11. Wiring your mock `store` to the real API

Your `.jsx` was deliberately built so this is a swap, not a rewrite — the comment
at the top says exactly that, and the method shapes already match your endpoints.
The mock computes in the browser:

```jsx
const store = {
  cashflow: (o) => { /* filters DATA.facts in-browser */ },
  month:    (o, mn) => { /* ... */ },
  ...
};
```

To go live, replace the bodies with `fetch`es to FastAPI — i.e. lift the exact
`api` object from `serving_frontend.html`:

```jsx
const API_BASE = "http://localhost:8000";
async function getJSON(path) {
  const res = await fetch(API_BASE + path);
  if (!res.ok) throw new Error(`${res.status} ${(await res.json()).detail ?? res.statusText}`);
  return res.json();
}
const store = {
  cashflow: (o) => getJSON(`/forecast/${enc(orchKeyFor(o))}/cashflow`),
  month:    (o, mn) => getJSON(`/forecast/${enc(orchKeyFor(o))}/month/${mn}`),
  sources:  (o, mn, cat) => getJSON(`/forecast/${enc(orchKeyFor(o))}/month/${mn}/sources?category=${enc(cat)}`),
  lineage:  (k) => getJSON(`/forecast/lineage/${enc(ookFor(k))}`),
};
```

The one real change: these now return **Promises** (they're async). In React you
don't `await` them inside `useMemo`; you use **`useEffect` + `useState`**, or —
better — **TanStack Query**, which turns each endpoint into a cached hook with
built-in loading/error handling. That's the point where the production data layer
from §9 earns its place.

And the invariant you nailed on the backend carries straight over: **money must
aggregate from `crop_input_streams` only; joining `input_output_linkage` during
money aggregation silently double-counts.** Your mock respects this (money never
touches the linkage; lineage is the only place inputs fan out), so as long as the
real endpoints preserve that same grain, the swap is safe.

---

## 12. Glossary

| Term | What it is |
|---|---|
| **JSX** | JavaScript syntax that lets you write HTML-like tags; compiles to `React.createElement` calls |
| **React** | The library that renders a UI tree from state and diffs it against the DOM |
| **Component** | A function returning JSX; a reusable, parameterized UI unit (`<Stat/>`) |
| **Hook** | A `useX` function adding state/behavior to a component (`useState`, `useMemo`, `useEffect`) |
| **`useState`** | One cell of state; `setX` updates it and triggers a re-render |
| **`useMemo`** | Memoized derived value, recomputed only when its dependency array changes |
| **Declarative** | You describe *what* the UI should be; the runtime computes *how* to change the DOM |
| **Imperative** | You issue step-by-step DOM mutations yourself (your HTML file) |
| **Transpile** | Rewrite code from one form to another (JSX → JS, modern JS → older JS); Babel/esbuild |
| **Bundle** | Follow all imports and stitch modules into a few browser-ready files; Rollup/esbuild |
| **Vite** | Dev server (transpile-on-request + HMR) and production build tool; default port 5173 |
| **esbuild** | Very fast transpiler/bundler Vite uses in dev |
| **Rollup** | Bundler Vite uses for the production build |
| **HMR** | Hot Module Replacement — push edited modules into the running page without a reload |
| **SPA** | Single-Page Application — one HTML shell, all views rendered in the browser |
| **SSR** | Server-Side Rendering — HTML rendered on the server first (Next.js, Remix) |
| **`npm`/`node`** | Package manager / runtime; the `pip`/`python` of the JS world |
| **`node_modules`** | Installed dependencies on disk; the `site-packages` of the JS world |
| **`dist/`** | The built static output you deploy |
| **TypeScript / `.tsx`** | JS + static types; the Pydantic-style contract layer for your UI |

---

## 13. Quickstart — running the `.jsx` locally

To see `ForecastExplorer.jsx` actually run (with its mock data, no backend
needed):

```bash
# 1. scaffold a Vite + React project
npm create vite@latest growlytics-ui -- --template react
cd growlytics-ui

# 2. install its deps, plus the two your component imports
npm install
npm install recharts lucide-react

# 3. drop your file in and point the app at it
#    - copy ForecastExplorer.jsx into src/
#    - in src/App.jsx:  import ForecastExplorer from "./ForecastExplorer";
#                       export default function App(){ return <ForecastExplorer/>; }

# 4. run the dev server (opens on http://localhost:5173)
npm run dev
```

Edit the file, save, and watch HMR update the page while your drill-down state
stays put. When you're ready to ship: `npm run build` → serve the `dist/` folder
anywhere. That's the whole loop.

---

*Both files describe the identical four-level drill (chart → month → category →
source → versioned lineage). The HTML file is the imperative cursor-loop version
that runs as-is; the JSX file is the declarative, componentized version that
needs Vite to build it. Same destination, two different philosophies about who
does the work — you, or the runtime.*


# `ForecastExplorer.wired.jsx` — the mechanical, from-zero syntax guide

This assumes you can program (loops, functions, types) but have **never seen
JavaScript, HTML, CSS, or React**. It explains every piece of syntax and every pattern
in the file, building vocabulary from the ground up and then walking the actual file top
to bottom. Where useful it contrasts with Python, since that's the language you know
best.

How to read it: Chapters 0–4 build the language pieces (runtime, JavaScript, HTML, CSS,
React). Chapter 5 walks the real file using that vocabulary. Chapter 6 is a
punctuation/symbol glossary you can keep open as a lookup table. You don't have to read
linearly — but if a symbol looks alien, Chapter 6 will name it and point you back.

---

## Chapter 0 — What actually runs this file

Before any syntax: what is a `.jsx` file, and what executes it?

**The browser** (Chrome, Firefox, …) contains a **JavaScript engine** — a program that
runs JavaScript code, the way the Python interpreter runs `.py` files. JavaScript (JS)
is *the* programming language browsers understand. Nothing else runs natively in a page.

**The DOM** (Document Object Model) is the browser's live, in-memory tree of everything
on the screen — every box, heading, button — as objects the JS engine can read and
change. When you change the DOM, the screen updates. Think of the DOM as a mutable tree
data structure that *is* the rendered page.

**This file is not something the browser can run directly.** It's `.jsx`, not `.js`, and
it contains two things a raw browser doesn't understand:

1. **JSX** — the HTML-looking tags in the middle of the JavaScript (`<div>…</div>`). Not
   legal JavaScript on its own.
2. **ES modules** — the `import` lines at the top, pulling in other packages.

So a **build tool** runs first. In your project that's **Vite**. Vite (via a compiler
called Babel/esbuild) does two jobs:

- **Transpiles** the JSX into plain JavaScript function calls (you'll see exactly what
  `<div>` becomes in Chapter 4).
- **Bundles** — follows every `import`, finds those packages in the `node_modules`
  folder, and stitches the needed code into JavaScript files the browser can load.

The full pipeline:

```
ForecastExplorer.wired.jsx
      │  Vite: transpile JSX → JS, bundle imports
      ▼
plain JavaScript the browser can run
      │  browser's JS engine executes it
      ▼
React builds/updates the DOM tree
      │
      ▼
pixels on screen
```

Two Python contrasts worth stating up front:

- Python is usually run by an interpreter you invoke (`python app.py`). Browser JS is
  fetched over HTTP and run by the engine embedded in the browser; there's no separate
  interpreter you launch. During development, Vite runs a local server that does the
  transpile/bundle on the fly.
- Python `import` reads modules from your environment at *runtime*. JS `import` here is
  resolved at *build time* by Vite and baked into the bundle.

Everything below is about the *plain JavaScript* layer and the HTML/CSS/React concepts
sitting on top of it.

---

## Chapter 1 — JavaScript, the language

Grounded in the file's real lines. If a construct also exists in Python, I say how it
differs.

### 1.1 Statements, semicolons, comments

A JavaScript program is a list of **statements**. Statements end with a semicolon `;`.
(JS can often infer them, but this file writes them, which is the safe habit.)

Comments:

```js
// single-line comment — everything after // on this line is ignored
/* block comment
   spanning multiple lines */
```

Both appear in the file. The big banner comments like `/* ===== STYLES ===== */` are
just block comments used as section dividers.

### 1.2 Values and types

JavaScript values come in a small set of types:

| Type | Examples in the file | Python analog |
|------|----------------------|---------------|
| **number** | `20`, `1.6`, `0.34`, `248` | `int`/`float` — **JS has only one number type**, no int/float split |
| **string** | `"fx-root"`, `` `/orchestrations` `` | `str` |
| **boolean** | `true`, `false` | `True`/`False` (note: lowercase in JS) |
| **null** | `null` | roughly `None` — "intentionally empty" |
| **undefined** | (returned when something is missing) | also roughly `None` — "never set" |
| **object** | `{ ink: "#17241E", ... }` | `dict` (but also serves as instances/records) |
| **array** | `["", "Jan", "Feb", ...]` | `list` |
| **function** | `getJSON`, `money`, arrow functions | `function`/`lambda` |

The one surprise for a Python person: JS has **two** "empty" values. `null` means
"deliberately nothing"; `undefined` means "no value assigned / not found." A missing
object property, a function with no `return`, and an out-of-range lookup all give
`undefined`. `null` is something you assign on purpose (this file uses `null` as the
"nothing selected yet" state, e.g. `useState(null)`).

### 1.3 Variables: `const` and `let`

```js
const API_BASE = "http://localhost:8000";   // line 31
```

`const` declares a variable that **cannot be reassigned**. There's also `let` (used when
you need to reassign) and an older `var` (avoid; this file doesn't use it). This file is
almost entirely `const`.

Two things a Python person must internalize:

- Python has no declaration keyword (`x = 5`). JS requires `const`/`let`. Assigning
  without one is an error in modern setups.
- **`const` blocks reassignment, not mutation.** `const obj = {}` means you can't later
  do `obj = something_else`, but you *can* do `obj.key = 5`. The binding is fixed; the
  contents aren't. Same for arrays: `const a = []; a.push(1)` is fine.

Variables are **block-scoped**: a `const` inside `{ … }` (an `if`, a function, a loop
body) exists only inside that block. Same as Python function scope, but JS scopes to any
`{}` block, not just functions.

### 1.4 Operators

**Arithmetic:** `+  -  *  /  %` as expected. `+` also **concatenates strings**:
`"x" + "%"` → `"x%"` (line 266). `Math.round(...)`, `Math.abs(...)`, `Math.max(...)`,
`Math.min(...)` are functions on the built-in `Math` object.

**Comparison:** use `===` (equal) and `!==` (not equal), **not** `==`/`!=`.

```js
s.output_orchestration_key === ook     // line 291
```

`===` compares value *and* type with no coercion. `==` does surprising type coercion
(`0 == ""` is `true`) and is avoided. Rule: always `===`/`!==`. Also `<  >  <=  >=`.

**Logical:** `&&` (and), `||` (or), `!` (not). These are central to React rendering and
have a subtlety covered in 1.5.

**Ternary (inline if/else):** `condition ? valueIfTrue : valueIfFalse`

```js
color: seasonNet >= 0 ? C.inflow : C.outflow     // line 184
```

Reads: "if `seasonNet >= 0`, use `C.inflow`, else `C.outflow`." It's an *expression*
(produces a value), unlike a Python `if` statement — closer to Python's
`a if cond else b`. Ternaries can chain (line 92):

```js
const off = hi <= 0 ? 0 : lo >= 0 ? 1 : hi / (hi - lo);
// if hi<=0 → 0; else if lo>=0 → 1; else → hi/(hi-lo)
```

**Nullish coalescing `??`:** `a ?? b` gives `a` unless `a` is `null` or `undefined`, in
which case `b`.

```js
detail = (await res.json()).detail ?? detail;   // line 38: use .detail, or fall back
const def = rows.find(...) ?? rows[0];           // line 101: found row, or first row
```

Different from `||`: `??` only falls back on null/undefined, whereas `||` falls back on
*any* falsy value (including `0` or `""`). Use `??` when `0`/`""` are valid.

**Optional chaining `?.`** (not used in this file, but you'll meet it constantly):
`a?.b` reads `a.b` but returns `undefined` instead of crashing if `a` is null/undefined.

### 1.5 Truthiness (why `&&` renders things)

Every value is either **truthy** or **falsy** when used in a boolean context. The
**falsy** values are: `false`, `0`, `""` (empty string), `null`, `undefined`, and `NaN`.
Everything else — including non-empty strings, any object, any array (even `[]`) — is
**truthy**.

The logical operators **return one of their operands**, not a bare boolean:

- `A && B` → if `A` is falsy, result is `A`; otherwise result is `B`.
- `A || B` → if `A` is truthy, result is `A`; otherwise result is `B`.

This is why `{mn && (<div>…</div>)}` in JSX means "render the div only if `mn` is set":
if `mn` is `null` (falsy), the expression is `null` (React draws nothing); if `mn` is a
month number (truthy), the expression is the `<div>`. You're exploiting the return-value
behavior. (One gotcha: `0 && <X/>` renders `0`, because `0` is falsy and returned as-is
— which is why numeric guards sometimes use `count > 0 && …`.)

### 1.6 Strings and template literals

Plain strings use single or double quotes: `"fx-root"`, `'IBM Plex Mono'` (interchangeable).

**Template literals** use backticks `` ` `` and can **interpolate** with `${…}`:

```js
`/forecast/${enc(orch)}/cashflow`                    // line 46
`${res.status} ${detail}`                            // line 39
`${lin.output.output_name} · ${lin.output.crop}`     // line 359
```

This is JS's version of a Python f-string. `${expr}` evaluates `expr` and drops its
string form into the surrounding text. Backtick strings can also span multiple lines —
which is how the entire CSS block is stored as one string (Chapter 5).

### 1.7 Functions

Two forms appear in this file.

**Function declaration** (statement form):

```js
function getJSON(path) {          // line 34 (also async — see 1.13)
  ...
  return res.json();
}
function Stat({ k, v, c }) { ... }  // line 327 — a React component is just a function
```

**Arrow functions** (expression form), JS's compact `lambda`, but not limited to one
expression:

```js
const enc = encodeURIComponent;                         // a function value assigned to a name
const money = (n) => { ...; return ...; };              // line 53: params => { body }
orchestrations.map((o) => (                             // line 164: param => (expression)
  <option ...>...</option>
))
onClick={() => pickCat(cc.category)}                    // line 262: no params
(e) => e && e.activePayload && pickMonth(...)           // line 199: one param `e`
```

Rules for arrow functions:

- `(params) => expression` **implicitly returns** `expression` (no `return` keyword).
  `(params) => { statements }` uses `{}` as a normal body and needs an explicit `return`
  to return anything.
- Parentheses around params are optional for exactly one param (`n => …`), required for
  zero (`() => …`) or multiple (`(a, b) => …`). This file usually keeps them.
- When an arrow returns JSX, you'll see `=> ( … )` — the parentheses are just grouping so
  the multi-line JSX reads cleanly; they don't change meaning.

**Parameters and defaults:** parameters are named like Python. Default values use `=`
(`function f(x = 5)`). This file mostly passes objects and destructures them (1.10).

**Hoisting:** `function` *declarations* are "hoisted" — usable before the line they're
defined on. That's why `ForecastExplorer` (line 69) can use `<Stat/>` and `<FlowTip/>`
even though those functions are defined *below* it (lines 327+). Arrow functions assigned
to `const` are **not** hoisted (they exist only after their line runs), which is why the
helpers `money`/`api` are defined *above* the component.

### 1.8 Objects

An **object** is a set of key→value pairs, written with `{}`. It's JS's dict, and also
its record/struct.

```js
const C = {                              // line 21
  ink: "#17241E", field: "#EDEFE8", panel: "#FBFCF9", line: "#DBDED4",
  inflow: "#2E7D57", outflow: "#B0512B", grain: "#B5872E", muted: "#6E7368",
};
```

Keys are bare identifiers (no quotes needed for simple names), values are any type,
commas separate pairs. Access:

- **Dot notation:** `C.grain` → `"#B5872E"` (line 155). Use when you know the key name.
- **Bracket notation:** `MONTHS[mn]` (line 247) → the element/property at the key held in
  variable `mn`. Use when the key is dynamic. (For arrays this is indexing; arrays are a
  kind of object.)

An object literal can nest objects and arrays freely — the `api` object (line 44) holds
functions as values; the response objects from the server are nested objects.

**Property shorthand:** `{ month_num, month }` is short for `{ month_num: month_num,
month: month }` — when the key name equals the variable name. (You'll see this idea
mirrored in destructuring, 1.10.)

### 1.9 Arrays

An **array** is an ordered list, written with `[]`:

```js
const MONTHS = ["", "Jan", "Feb", ... "Dec"];        // line 25
const REV_ORDER = ["1-pre-seeding", "2-post-seeding", "3-pre-harvest"];  // line 26
```

Indexed from 0: `MONTHS[1]` is `"Jan"`. `array.length` is the count. The empty string at
`MONTHS[0]` is deliberate padding so month number 1 maps to index 1 (`MONTHS[mn]`), a
1-based convenience.

Arrays carry the functional methods in 1.12. Note again: `const` on an array only fixes
the binding; the array's contents can still be mutated (`.push`, index assignment).

### 1.10 Destructuring — unpacking into named variables

This is everywhere in React and is the one piece of syntax most worth mastering.

**Array destructuring** — bind positional elements to names:

```js
const [orchKey, setOrchKey] = useState(null);   // line 71
```

`useState(null)` returns a 2-element array; this names element 0 as `orchKey` and element
1 as `setOrchKey`. (Python analog: `a, b = some_pair`.) The names are your choice; the
*positions* are what matter.

**Object destructuring** — bind properties by name:

```js
function Stat({ k, v, c }) { ... }                        // line 327
function FlowDot({ cx, cy, payload, sel }) { ... }        // line 336
function Breadcrumb({ orch, mn, cat, lin, onRoot, onMonth, onCat }) { ... }  // line 355
```

The `{ … }` in the **parameter list** means "the caller passes one object; pull these
properties out of it by name into local variables." So `<Stat k="Inflow" v="$1" c="#..."
/>` calls `Stat` with an object `{ k:"Inflow", v:"$1", c:"#..." }`, and the parameter
destructuring unpacks it. This is how React components receive their inputs ("props").
(Python analog: like receiving `**kwargs` and naming exactly the keys you expect.)

You can also destructure outside parameter lists: `const { detail } = await res.json()`
would pull `detail` out of the parsed object (the file inlines this differently, but same
idea).

### 1.11 Spread `...`

Three dots means "spread the elements out." In this file it appears as **array spread
into function arguments**:

```js
Math.max(...vals, 0)                          // line 91
Math.max(...cats.map((z) => Math.abs(z.amount)), 1)   // line 258
```

`Math.max(a, b, c)` takes separate arguments; `...vals` spreads an array `[x,y,z]` into
`x, y, z` so `Math.max` sees them individually. (Python analog: `max(*vals, 0)`.) The
trailing `0`/`1` is a floor so `Math.max` never gets an empty argument list.

Spread also copies objects/arrays (`{...obj}`, `[...arr]`) — you'll see that pattern
elsewhere, though this file only uses the argument form.

### 1.12 The array methods (the functional core)

JavaScript's `map`/`filter`/`reduce`/`find` are **methods called on an array** with a
function argument. They're the backbone of both the data layer and the rendering.

**`.map(fn)`** — return a new array, each element transformed by `fn`:

```js
flow.map((r) => r.net)                     // line 90: array of the net field from each row
orchestrations.map((o) => (<option ...>))  // line 164: array of rows → array of JSX elements
```

The mapping callback can receive a second argument, the **index**:

```js
crumbs.map((c, i) => (<span key={i}>...</span>))   // line 362: c is the item, i is its position
```

`.map` is *the* way to turn a list of data into a list of UI elements (Chapter 4).

**`.filter(fn)`** — new array of only the elements where `fn` returns truthy. (Not used
in the wired file — the server does the filtering now — but ubiquitous in JS. Python
analog: `[x for x in xs if cond]`.)

**`.reduce(fn, initial)`** — fold the array to a single value:

```js
flow.reduce((s, r) => s + r.net, 0)        // line 85: sum of net over all rows, start at 0
```

`fn` receives the running accumulator `s` and the current element `r`; its return becomes
the next `s`. `0` is the starting accumulator. (Python analog: `functools.reduce`, or
just `sum(r.net for r in flow)` here.)

**`.find(fn)`** — the first element where `fn` is truthy, or `undefined` if none:

```js
orchestrations.find((o) => o.orchestration_key === orchKey)   // line 83
rows.find((r) => r.orchestration_key.endsWith("3-pre-harvest"))  // line 101
```

**Other built-ins used:**

- `new Set(array)` — a collection of unique values; `.size` is the count of distinct
  items. `new Set(inputs.map((i) => i.revision_name)).size > 1` (line 378) asks "are
  there more than one distinct revision among the inputs?" (Python analog: `len(set(...))`.)
- `string.endsWith("x")` — boolean (line 101).
- `string.split("-")` — array of pieces split on `"-"`; `[0]` takes the first
  (line 403: `rev.split("-")[0]` turns `"1-pre-seeding"` into `"1"`).
- `number.toLocaleString()` — formats with thousands separators (line 56, `money`).
- `(n).toFixed(1)` — string with 1 decimal (line 60, `compact`).
- `isNaN(n)` — "is this Not-a-Number?" guard (line 54).

### 1.13 Asynchronous JavaScript (Promises, async/await, fetch)

This is the concept that makes network code work, and the main new machinery versus a
synchronous program.

A **Promise** is a placeholder for a value that will exist *later* — the result of an
operation still in flight (like an HTTP request). It's an object you can attach
callbacks to: `.then(onSuccess)` runs when the value arrives; `.catch(onError)` runs if
it fails.

```js
api.cashflow(orchKey).then(setFlow).catch((e) => { setFlow([]); setErr(e.message); });
// line 111: when the request resolves, call setFlow with the rows;
//           if it rejects, clear flow and record the error
```

`.then(setFlow)` is shorthand for `.then((rows) => setFlow(rows))` — pass the resolved
value straight into `setFlow`.

**`async` / `await`** is nicer syntax over Promises. An `async` function always returns a
Promise. Inside it, `await` pauses until a Promise resolves and gives you the plain
value:

```js
async function getJSON(path) {                 // line 34
  const res = await fetch(API_BASE + path);    // wait for the HTTP response headers
  if (!res.ok) { ... throw new Error(...); }   // res.ok is false for 4xx/5xx
  return res.json();                           // .json() itself returns a Promise (parse the body)
}
```

- `fetch(url)` is the browser's built-in HTTP client. It returns a Promise for a
  **Response** object. `await` gives you that Response once the server replies.
- `res.ok` is `true` for 2xx status codes. `res.status` is the number (404, 500…).
- `res.json()` reads and parses the response body as JSON — itself asynchronous, so it's
  also a Promise (here returned directly; the caller `await`s or `.then`s it).
- `throw new Error("message")` raises an error (Python analog: `raise Exception(...)`).
  A thrown error inside an async function becomes a **rejected Promise**, which the
  caller catches with `.catch` (line 111) — that's how `getJSON`'s errors reach the UI.
- `try { … } catch (_) { … }` (line 38) is JS's `try/except`; `_` is a throwaway name for
  the caught error.

The practical consequence throughout the component: data does not exist at the moment you
ask for it. You request it, the function keeps going, and the value shows up in a later
callback. That's why fetched data is stored in state and rendered on a *subsequent* pass
(Chapter 4).

---

## Chapter 2 — HTML and the DOM

React produces HTML-like elements, so you need the vocabulary of HTML even though you
won't write a separate `.html` file.

### 2.1 What HTML is

HTML describes a **tree of elements**. An element is written as an **opening tag**, some
**content**, and a **closing tag**:

```html
<div>hello</div>
```

`div` is the **tag name**. Elements nest to form a tree:

```html
<section>
  <h2>Sources</h2>
  <p>some text</p>
</section>
```

Here `section` is the parent; `h2` and `p` are children. This tree is exactly what the
browser turns into the DOM (Chapter 0) and paints.

An element with no content can **self-close**: `<img />`, `<circle />`. (In JSX every
childless element self-closes, e.g. `<Wheat ... />`.)

### 2.2 Attributes

Tags carry **attributes** — `name="value"` pairs that configure the element:

```html
<div class="fx-root">           <!-- class: which CSS rules apply -->
<select value="...">            <!-- value: the selected option -->
<th class="r">                  <!-- multiple classes possible: class="a b c" -->
```

Attributes you'll see: `class` (CSS hook — becomes `className` in JSX, see 4.3),
`id`, `value`, `href`, `width`/`height`, `tabindex` (keyboard focusability), plus
SVG-specific ones like `cx`/`cy`/`r`/`fill`.

### 2.3 The elements used in this file

| Tag | Meaning / role |
|-----|----------------|
| `<div>` | Generic block-level container (a box). The workhorse grouping element. |
| `<span>` | Generic **inline** container (flows within text, doesn't break the line). |
| `<header>` `<main>` `<section>` `<nav>` | "Semantic" containers — same box behavior as `div` but named for what they hold (page header, main content, a section, navigation). Improves accessibility/meaning. |
| `<h2>` | A level-2 heading. |
| `<p>` | A paragraph. |
| `<label>` | A caption bound to a form control; clicking it focuses the control. Wraps the `<select>` here. |
| `<select>` / `<option>` | A dropdown and its choices. |
| `<button>` | A clickable button. |
| `<table>` `<thead>` `<tbody>` `<tr>` `<th>` `<td>` | Table, header section, body section, table row, header cell, data cell. |
| `<svg>` primitives: `<circle>` | Vector graphics; `<circle cx cy r fill>` is drawn by `FlowDot`. |

### 2.4 The DOM and events

Once rendered, the tree is **live**: the browser fires **events** as the user interacts
— `click`, `change` (a form control's value changed), `keydown` (a key was pressed),
`mouseover`, etc. Traditionally you'd attach handler functions to nodes to respond. In
React you declare handlers as attributes (`onClick={…}`) and never touch the DOM directly
(Chapter 4). The **event object** passed to a handler carries details: `e.target` (the
element), `e.target.value` (a control's current value), `e.key` (which key), and
`e.preventDefault()` (cancel the browser's default reaction).

---

## Chapter 3 — CSS

CSS ("Cascading Style Sheets") controls appearance and layout. In this file the entire
stylesheet is one big string in the `CSS` constant (Chapter 5), injected via a `<style>`
tag.

### 3.1 The shape of a rule

```css
.fx-panel{ background:var(--panel); border:1px solid var(--line); border-radius:14px; }
│         │                                                                            │
selector  └──────────────── declarations (property:value; …) inside braces ───────────┘
```

A **rule** = a **selector** (what to match) + a block of **declarations** (what to set).
Each declaration is `property: value;`.

### 3.2 Selectors (how elements are matched)

| Selector | Matches |
|----------|---------|
| `.fx-panel` | any element with `class="fx-panel"` (a **class selector** — the dot means "class") |
| `h2` | every `<h2>` element (a **type/element selector**) |
| `.fx-level-head h2` | an `<h2>` that is **inside** an element with class `fx-level-head` (**descendant**, note the space) |
| `.fx-table th.r` | a `<th>` that *also* has class `r`, inside `.fx-table` (`th.r` = both, no space) |
| `.fx-tick.chosen .fx-dot` | `.fx-dot` inside an element that has **both** classes `fx-tick` and `chosen` |
| `.fx-src:hover` | `.fx-src` while the mouse is over it (a **pseudo-class** — a state) |
| `.fx-picker select:focus-visible` | a focused-via-keyboard `<select>` inside `.fx-picker` |
| `.fx-root *` | **every** element inside `.fx-root` (`*` is the universal selector) |
| `.fx-table th.r, .fx-table td.r` | comma = "or" — matches either selector |

"Cascading" means when several rules match one element, they combine, and conflicts are
resolved by specificity/order. You mostly won't fight this here; classes keep it simple.

### 3.3 The box model

Every element is a box with four concentric layers, inside → out: **content**,
**padding** (space inside the border), **border**, **margin** (space outside the border).

```
margin  ┌─────────────────────────┐
        │ border                   │
        │  ┌────────────────────┐  │
        │  │ padding            │  │
        │  │  ┌──────────────┐  │  │
        │  │  │  content     │  │  │
        │  │  └──────────────┘  │  │
        │  └────────────────────┘  │
        └─────────────────────────┘
```

Properties: `padding: 20px 22px` (vertical 20, horizontal 22 — the shorthand takes
1–4 values), `margin: 14px 0 4px`, `border: 1px solid var(--line)` (width style color),
`border-radius: 14px` (rounded corners). `box-sizing: border-box` (line 432, applied to
`*`) makes `width`/`height` *include* padding and border, which makes layout math sane —
a near-universal reset.

### 3.4 Layout: flexbox and grid

Two layout systems appear.

**Flexbox** (`display:flex`) lays children in a row (or column) with control over spacing
and alignment:

```css
.fx-head{ display:flex; align-items:center; justify-content:space-between; }
```

- `display:flex` — turn this element into a flex container; its direct children become
  flex items in a row.
- `flex-direction:column` — stack vertically instead (line 440).
- `align-items` — cross-axis alignment (`center`, `baseline`, `flex-start`/`flex-end`).
- `justify-content` — main-axis distribution (`space-between` pushes items to the ends).
- `gap:11px` — space between items.
- `flex:1` on a child — "grow to fill available space" (line 486, makes the three stat
  boxes equal width).

**Grid** (`display:grid`) lays children on a 2-D grid:

```css
.fx-grid{ display:grid; grid-template-columns:minmax(340px,2fr) 3fr; gap:22px; }
.fx-catrow{ display:grid; grid-template-columns:120px 1fr 120px; ... }
```

- `grid-template-columns` defines the columns. `120px 1fr 120px` = a fixed 120px column,
  a flexible middle column, another fixed 120px. `2fr 3fr` = two columns splitting space
  2:3. `fr` is a **fractional unit** — a share of leftover space. `minmax(340px, 2fr)`
  means "at least 340px, otherwise a 2fr share."

### 3.5 CSS custom properties (variables)

```css
.fx-root{ --ink:#17241E; --grain:#B5872E; ... }   // define, on line 428
color: var(--ink);                                 // use, on line 430
```

`--name` **defines** a custom property (only on elements matching the selector and their
descendants). `var(--name)` **reads** it. This is why the palette is declared once on
`.fx-root` and referenced everywhere inside. It's the same palette as the JS `C` object;
it exists in *both* places because CSS rules use `var(--…)` while the chart's SVG (drawn
by JS) needs real JS strings from `C` (Chapter 5 explains the split).

### 3.6 Values, units, colors

- **Lengths:** `px` (pixels), `%` (percent of parent), `em` (relative to font size),
  `ch` (width of a "0" — `max-width:52ch` ≈ 52 characters, line 476), `vh` (percent of
  viewport height — `min-height:100vh` = at least the full screen, line 431), `fr`
  (grid fraction).
- **Colors:** hex `#17241E`, or `rgba(0,0,0,.18)` (red,green,blue,alpha) where alpha is
  opacity 0–1. `999px` border-radius makes a pill; `50%` makes a circle.
- **Fonts:** `font-family:'IBM Plex Sans', system-ui, sans-serif` — a fallback list; the
  browser uses the first available. `font:inherit` copies the parent's font (used to make
  `<button>`/`<select>` stop looking like OS defaults).

### 3.7 At-rules, transitions, animations

- `@import url('https://fonts.googleapis.com/...')` (line 426) — pull in external CSS
  (here, web fonts) before the rest.
- `@media(max-width:900px){ … }` (line 447) — apply the enclosed rules only when the
  condition holds (screen ≤ 900px wide). This is **responsive design** — the layout
  collapses to one column on narrow screens.
- `@media(prefers-reduced-motion:reduce){ … }` (line 542) — respect the OS "reduce
  motion" accessibility setting by disabling animations.
- `transition:width .5s cubic-bezier(.4,0,.2,1)` (line 499) — animate the `width`
  property smoothly over 0.5s whenever it changes; `cubic-bezier(...)` is the easing
  curve.
- `@keyframes fxIn{ from{...} to{...} }` (line 541) + `animation:fxIn .34s ... both`
  (line 540) — define a named animation (start/end states) and run it. This is the
  panels' fade-and-rise entrance.

---

## Chapter 4 — React

Now the framework that ties JS + HTML + CSS together.

### 4.1 The core idea: `UI = f(state)`

Traditional ("imperative") DOM code finds nodes and mutates them step by step. React is
**declarative**: you write a function that, given the current data (**state**), *returns
a description of the entire UI*. When state changes, React re-runs your function, diffs
the new description against the old, and patches only the DOM nodes that actually
changed. You never look up or mutate DOM nodes yourself.

If you know materialized outputs: the screen is a pure function of its inputs. You don't
hand-edit the output; you change the inputs and let it recompute. Same discipline, aimed
at pixels.

### 4.2 Components

A **component** is a function that returns UI. It starts with a Capital letter (so JSX
can tell it apart from plain HTML tags):

```js
export default function ForecastExplorer() { ...; return ( <div>…</div> ); }  // line 69
function Stat({ k, v, c }) { return ( <div>…</div> ); }                        // line 327
```

`export default` marks the file's main export — what another module gets when it
`import`s this file. The smaller components (`Stat`, `FlowDot`, `FlowTip`, `Breadcrumb`,
`LineageInputs`) are helpers used inside the main one.

### 4.3 JSX — the HTML-in-JavaScript

JSX is the tag syntax inside the `return (...)`. It is **not** HTML and **not** a string;
it's syntax that Vite compiles into function calls that build "element" objects. For
intuition:

```jsx
<div className="fx-root">hi</div>
```

compiles to roughly:

```js
React.createElement("div", { className: "fx-root" }, "hi")
```

i.e. a plain JS object describing "a div with these props and these children." React
later turns that description into real DOM. You almost never think about the compiled
form — but knowing JSX is *just function calls producing objects* explains every rule
below.

**Rule 1 — one root, or a Fragment.** A component returns a single top element. The whole
`ForecastExplorer` return is one `<div className="fx-root">…</div>`. If you need siblings
with no wrapper, you use a **Fragment** `<>…</>` (empty tags). This file wraps instead of
using fragments.

**Rule 2 — `className`, not `class`.** `class` is a reserved word in JS, so JSX spells the
CSS-class attribute `className`:

```jsx
<div className="fx-panel">          // line 182  → <div class="fx-panel"> in the DOM
```

**Rule 3 — `{ }` drops into JavaScript.** Inside JSX, curly braces mean "evaluate this JS
expression and insert the result here." Two contexts:

```jsx
<div>{money(seasonNet)}</div>                       // line 185: text content from an expression
<option value={o.orchestration_key}>               // line 165: an attribute value from an expression
<Wheat size={20} color={C.grain} />                // lines 155: number & object-field as props
```

Anything in `{}` is JS: a variable, a function call, a ternary, a `.map(...)`. Attribute
values that are strings use quotes (`className="fx-panel"`); values that are JS use braces
(`size={20}`, `value={orchKey}`).

**Rule 4 — `style` is a JS object, camelCased.** Inline styles take an object, not a CSS
string, so you see **double braces** (outer `{}` = "JS expression", inner `{}` = "an
object literal"), and property names are camelCase:

```jsx
style={{ color: seasonNet >= 0 ? C.inflow : C.outflow }}          // line 184
style={{ margin: "14px 26px 0", borderRadius: 8, fontSize: 13 }}  // line 174 (borderRadius, not border-radius)
```

**Rule 5 — attributes/props are camelCase and take JS.** `onClick`, `onChange`,
`onKeyDown`, `strokeWidth`, `tabIndex`, `dataKey`. Numbers/booleans/objects go in braces:
`tickLine={false}`, `size={20}`, `margin={{ top: 8 }}`.

**Rule 6 — conditional rendering uses expressions, not statements.** You can't put an
`if` inside JSX, so you use `&&` and ternaries (Chapter 1.4–1.5):

```jsx
{!mn && ( <div className="fx-empty">…</div> )}      // line 236: render only when mn is falsy
{mn && ( <div className="fx-level">…</div> )}        // line 243: render only when mn is truthy
{mn && cat && ( … )}                                 // line 278: only when both set
{c.on && !c.active ? <button…/> : <span…/>}          // line 365: choose one of two elements
```

Rendering rules for values: `null`, `undefined`, `false`, and `true` render **nothing**;
strings and numbers render as text; an element renders itself; an **array of elements**
renders each in order (that's what `.map` produces).

**Rule 7 — lists via `.map(...)` with a `key`.** To render N items, map an array of data
to an array of elements, and give each a **`key`** — a stable unique identifier:

```jsx
{orchestrations.map((o) => (
  <option key={o.orchestration_key} value={o.orchestration_key}>
    {o.client_name} · {o.revision_name} · {o.forecast_year}
  </option>
))}                                                    // lines 164–168
```

`key` lets React match elements across re-renders so it can patch efficiently instead of
rebuilding the list. Keys should be stable IDs from your data (here the
`orchestration_key`). Using the array index as key (`key={i}`, line 363, `key={idx}`,
line 390) is acceptable only when the list never reorders — true for these fixed lists.

**Rule 8 — comments inside JSX.** `{/* like this */}` (line 180). (A normal `//` comment
can't sit between tags.)

**Rule 9 — components are used as tags and receive props.**

```jsx
<Stat k="Inflow" v={money(selRow.inflow)} c={C.inflow} />   // line 250
```

Each attribute becomes a property of the object passed to the component. `Stat` reads them
via its destructured parameters `{ k, v, c }`. A component invoked with no children
self-closes (`<Stat … />`).

**Rule 10 — you can pass an element *as* a prop.**

```jsx
<Area dot={<FlowDot sel={mn} />} ... />                 // line 218
<Tooltip content={<FlowTip />} />                        // line 216
```

Here `<FlowDot .../>` isn't rendered directly; it's handed to recharts as data, and
recharts calls it internally with extra props (the dot's `cx`, `cy`, `payload`). That's
why `FlowDot`/`FlowTip` receive props they were never explicitly given at this call site
— the library injects them.

### 4.4 Props — inputs to a component

**Props** are the object of attributes a parent passes to a child. They are **read-only**
inside the child (a component never mutates its own props). Props are a component's
signature/contract, exactly like a function's parameters or a Pydantic model's fields.
Data flows **down** (parent → child via props) and events flow **up** (child calls a
function the parent passed in, e.g. `onRoot`, `onMonth`, `onCat` on `Breadcrumb`, lines
231–234).

### 4.5 Hooks: `useState` and `useEffect`

**Hooks** are special functions (names start with `use`) that let a component have memory
and side effects across re-renders. Two rules govern all hooks: call them only at the
**top level** of a component (never inside loops/conditions), and always in the **same
order** every render. React relies on call order to keep each hook's data straight.

#### `useState` — a value React remembers, that triggers re-render when changed

```js
const [orchKey, setOrchKey] = useState(null);   // line 71
```

`useState(initial)` returns `[currentValue, setterFunction]` (array destructuring, 1.10):

- `orchKey` is the current value for this render (a fixed snapshot during the render).
- `setOrchKey(newValue)` is the **only** legal way to change it. Calling a setter tells
  React "this changed — schedule a re-render," and on the next render `useState` hands
  back the new value.
- You never assign `orchKey = …` directly; React wouldn't know, and wouldn't re-render.

Because a setter schedules a *future* render, `orchKey` does not change on the current
line after you call `setOrchKey(x)` — it updates on the next pass. (This trips up
everyone at first; treat state as immutable within a render.)

#### `useEffect` — run a side effect after render, re-run when dependencies change

```js
useEffect(() => {
  if (!ook) { setLin(null); return; }
  api.lineage(ook).then(setLin).catch((e) => { setLin(null); setErr(e.message); });
}, [ook]);                                        // lines 127–130
```

`useEffect(effectFn, depsArray)` means: "after this render commits, run `effectFn`; and
run it again on any later render where a value in `depsArray` changed." A **side effect**
is anything that reaches outside the pure "compute UI from state" world — here, making an
HTTP request. Fetching *must* live in an effect, not in the render body, because the
render must stay pure and synchronous.

- The **dependency array** is the trigger list. `[ook]` = "re-run when `ook` changes."
  `[orchKey, mn]` (line 118) = "re-run when either changes." `[]` (line 105) = "run once,
  after the first render, never again" (nothing in `[]` ever changes).
- An effect can **return a cleanup function**, run before the next effect or when the
  component is removed (used for canceling requests/timers). This file doesn't, but it's
  where an `AbortController` would go.

The lifecycle of one interaction, mechanically:

```
setState(x)                        // e.g. a click handler runs setMn(3)
   │  React schedules a re-render
   ▼
component function runs again      // now mn === 3, but fetched data may still be old/empty
   │  React commits the new DOM
   ▼
matching useEffect runs            // deps changed → api.month(...) fires (async)
   │  ... network ...
   ▼
Promise resolves → setMb(data)     // another setState
   │  schedules another re-render
   ▼
component runs again               // now mb has data → categories appear
```

So there's always a render where the selection is set but its data hasn't arrived yet —
which is why fetched state starts as `[]`/`null` and the code guards for "not loaded."

### 4.6 Controlled inputs

```jsx
<select value={orchKey} onChange={(e) => pickOrch(e.target.value)}>   // line 163
```

A **controlled component** ties a form control's displayed value to state (`value={orchKey}`)
and updates state on change (`onChange`). The state is the single source of truth; the
control merely reflects it. `e.target.value` is the newly selected option's value (a
string). This is the "state down, events up" loop in one line.

### 4.7 Keyboard/accessibility bits

```jsx
<tr tabIndex={0}
    onClick={() => pickSrc(...)}
    onKeyDown={(e) => { if (e.key === "Enter" || e.key === " ") { e.preventDefault(); pickSrc(...); } }}>
```

Lines 291–293: a table row isn't natively focusable/clickable, so `tabIndex={0}` puts it
in the keyboard tab order, and `onKeyDown` makes Enter/Space act like a click
(`e.preventDefault()` stops Space from scrolling the page). This is manual accessibility
for a custom interactive element.

---

## Chapter 5 — The file, top to bottom

Now a mechanical pass using the vocabulary above. Line numbers are approximate anchors.

### 5.1 Imports (lines 1–6)

```jsx
import { useState, useEffect } from "react";
import { AreaChart, Area, XAxis, YAxis, Tooltip, ResponsiveContainer, ReferenceLine, ReferenceDot } from "recharts";
import { Wheat, CornerDownRight, ArrowUpRight, ArrowDownRight } from "lucide-react";
```

`import { names } from "package"` pulls **named exports** out of a package (Chapter 0
bundles them). The `{ }` here is import syntax, not an object. `useState`/`useEffect` are
React's hooks; the `recharts` names are chart components; the `lucide-react` names are
icon components. Everything imported is used later as either a function (`useState(...)`)
or a JSX tag (`<AreaChart>`, `<Wheat/>`).

### 5.2 The banner comment (lines 8–19)

A block comment documenting that this is the wired build and how to run it (uvicorn on
:8000, the dev server, the CORS requirement). Comments have no effect on execution.

### 5.3 Module constants (lines 21–26)

```jsx
const C = { ink:"#17241E", ... };         // palette as JS values (object literal, 1.8)
const MONTHS = ["", "Jan", ... "Dec"];    // 1-indexed labels (array, 1.9)
const REV_ORDER = ["1-pre-seeding", "2-post-seeding", "3-pre-harvest"];
```

Defined once when the module loads. `C` mirrors the CSS variables because the chart's SVG
needs real JS color strings (`fill={C.grain}`) while CSS rules use `var(--grain)`.
`MONTHS[0]` is `""` so `MONTHS[mn]` works with 1-based month numbers. `REV_ORDER` fixes
the revision timeline the lineage track walks.

### 5.4 The `api` layer (lines 28–50)

```jsx
const API_BASE = "http://localhost:8000";
const enc = encodeURIComponent;            // give the built-in a short name

async function getJSON(path) {             // async function (1.13)
  const res = await fetch(API_BASE + path);        // + concatenates the base and path strings
  if (!res.ok) {                                    // 4xx/5xx
    let detail = res.statusText;                    // `let` because it may be reassigned
    try { detail = (await res.json()).detail ?? detail; } catch (_) {}   // best-effort parse FastAPI's detail
    throw new Error(`${res.status} ${detail}`);     // template literal builds the message
  }
  return res.json();
}

const api = {                              // an object whose values are arrow functions
  orchestrations: () => getJSON(`/orchestrations`),
  cashflow: (orch) => getJSON(`/forecast/${enc(orch)}/cashflow`),
  month: (orch, m) => getJSON(`/forecast/${enc(orch)}/month/${m}`),
  sources: (orch, m, cat) => getJSON(`/forecast/${enc(orch)}/month/${m}/sources?category=${enc(cat)}`),
  lineage: (ook) => getJSON(`/forecast/lineage/${enc(ook)}`),
};
```

Each `api.*` method is a short arrow function that builds a URL (template literal +
`encodeURIComponent` to make data safe inside a URL) and calls `getJSON`, which does the
fetch, checks the status, and parses JSON — returning a Promise of the parsed body. This
object is the seam between the component and the server; every method returns the exact
JSON shapes the server sends.

### 5.5 Formatting helpers (lines 52–64)

```jsx
const money = (n) => {
  if (n == null || isNaN(n)) return "—";              // guard: null/undefined/NaN → em dash
  const sign = n < 0 ? "-" : "";                       // ternary picks the sign string
  return sign + "$" + Math.round(Math.abs(n)).toLocaleString();  // concat + method chain
};
const compact = (n) => { ... "$3.7M" / "$4K" ... };
const num = (n) => (n == null || isNaN(n) ? "—" : Number(n).toLocaleString());
```

Pure functions: number in, formatted string out. Note `n == null` is the one place `==`
(not `===`) is idiomatic — it's a deliberate shorthand that's true for **both** `null`
and `undefined`. Method chaining reads left to right: `Math.abs(n)` → `Math.round(...)` →
`.toLocaleString()`.

### 5.6 The component: state (lines 69–81)

```jsx
export default function ForecastExplorer() {
  const [orchestrations, setOrchestrations] = useState([]);   // fetched picker list; starts empty
  const [orchKey, setOrchKey]   = useState(null);             // selection; null until loaded
  const [flow, setFlow]         = useState([]);               // cashflow series
  const [mn, setMn]             = useState(null);             // selected month number
  const [selRow, setSelRow]     = useState(null);             // the clicked chart point
  const [mb, setMb]             = useState(null);             // month breakdown object
  const [cat, setCat]           = useState(null);             // selected category
  const [srcs, setSrcs]         = useState([]);               // sources table rows
  const [ook, setOok]           = useState(null);             // selected source key
  const [chosenRev, setChosenRev] = useState(null);           // resolved revision of that source
  const [lin, setLin]           = useState(null);             // lineage object
  const [err, setErr]           = useState(null);             // last error message
```

Twelve `useState` calls, each destructured into a value + setter (1.10, 4.5). Empty
collections (`[]`) and `null` are the "nothing yet" defaults, which the render and guards
rely on. Selections change on clicks; the fetched data changes when requests resolve.

### 5.7 The component: cheap derived values (lines 83–92)

```jsx
const orch = orchestrations.find((o) => o.orchestration_key === orchKey);  // the selected picker row (object) or undefined
const cats = mb ? mb.categories : [];                                       // the category list, or [] if not loaded
const seasonNet = flow.reduce((s, r) => s + r.net, 0);                      // totals via reduce
const seasonIn  = flow.reduce((s, r) => s + r.inflow, 0);
const seasonOut = flow.reduce((s, r) => s + r.outflow, 0);

const vals = flow.map((r) => r.net);            // array of nets
const hi = Math.max(...vals, 0), lo = Math.min(...vals, 0);  // spread into max/min, floored at 0
const off = hi <= 0 ? 0 : lo >= 0 ? 1 : hi / (hi - lo);      // where zero falls on the y-range (0..1)
```

Plain synchronous derivations off already-loaded state — recomputed each render (cheap,
so no caching). Two comma-separated `const`s on one line (`hi`, `lo`) is legal shorthand.
`off` is the gradient's zero-crossing, used by the chart fill.

### 5.8 The component: effects (lines 94–130)

Five `useEffect`s, one per drill level (4.5). Each: guard, then fetch, then `setX` on
success / clear + `setErr` on failure. Read the dependency arrays as the trigger:

```jsx
useEffect(() => { api.orchestrations().then(rows => { setOrchestrations(rows); setOrchKey(default); }).catch(...); }, []);          // once
useEffect(() => { if (!orchKey) return; /* reset all below */ api.cashflow(orchKey).then(setFlow).catch(...); }, [orchKey]);       // orch → cashflow
useEffect(() => { if (!orchKey || !mn) { setMb(null); return; } api.month(orchKey, mn).then(setMb).catch(...); }, [orchKey, mn]);   // month → breakdown
useEffect(() => { if (!orchKey || !mn || !cat) { setSrcs([]); return; } api.sources(...).then(setSrcs).catch(...); }, [orchKey, mn, cat]); // cat → sources
useEffect(() => { if (!ook) { setLin(null); return; } api.lineage(ook).then(setLin).catch(...); }, [ook]);                          // src → lineage
```

The mount effect (`[]`) also picks a default orchestration:
`rows.find((r) => r.orchestration_key.endsWith("3-pre-harvest")) ?? rows[0]` — first a
matching row, else the first row (`??` fallback, 1.4). The `[orchKey]` effect clears every
deeper piece of state before loading, so a stale drill can't survive an orchestration
change.

### 5.9 The component: event handlers (lines 132–135)

```jsx
function pickOrch(v)        { setOrchKey(v); }
function pickMonth(m, point){ if (!m) return; setMn(m); setSelRow(point ?? null); setCat(null); setOok(null); setChosenRev(null); setLin(null); }
function pickCat(c)         { setCat(c); setOok(null); setChosenRev(null); setLin(null); }
function pickSrc(kk, rev)   { setOok(kk); setChosenRev(rev); }
```

Small functions that only set selection state; the effects react and fetch. `pickMonth`
stashes the clicked chart `point` (so the inflow/outflow/net trio needs no request) and
resets deeper levels. `pickSrc` carries the row's `chosen_revision` into `chosenRev` for
the lineage track. `if (!m) return` is an early exit (recharts can fire a click with no
point).

### 5.10 The component: the loading gate (lines 137–147)

```jsx
if (!orch) {
  return (
    <div className="fx-root">
      <style>{CSS}</style>
      <p style={{ padding: 28, color: "var(--muted)", fontFamily: "..." }}>
        {err ? `Couldn't reach the API: ${err}` : "Loading forecast…"}
      </p>
    </div>
  );
}
```

An **early `return`** before the main return. On the first render(s) `orch` is
`undefined` (no data yet), so reading `orch.client_name` below would crash; this returns a
minimal screen instead. Once the mount effect loads data and sets `orchKey`, a re-render
makes `orch` defined and the gate is skipped. It doubles as the top-level error display.

### 5.11 The main return — the JSX tree (lines 149–323)

One root `<div className="fx-root">`. Reading it as a tree:

- `<style>{CSS}</style>` (line 151) injects the whole stylesheet string (5.13).
- `<header className="fx-head">` (153): the brand block (`<Wheat/>` icon + two text
  divs) and the **picker** `<label><span>…</span><select>…</select></label>`. The
  `<select value={orchKey} onChange=…>` is the controlled dropdown (4.6); its
  `<option>`s come from `orchestrations.map(...)` (Rule 7). `·` is just a literal middle-
  dot character in the text.
- `{err && (<div style={{…}}>API error: {err} …</div>)}` (173): the conditional error
  banner (Rule 6), styled inline.
- `<main className="fx-grid">` (179) holds two `<section>`s side by side (CSS grid, 3.4):

  **Left — `<section className="fx-context">`** (181): the season panel. `<div
  className="fx-hero-net" style={{ color: seasonNet >= 0 ? C.inflow : C.outflow }}>
  {money(seasonNet)}</div>` shows the big net figure, colored by sign. The
  season-split shows `{compact(seasonIn)}`/`{compact(seasonOut)}` with lucide arrow
  icons. Then the chart:

  ```jsx
  <ResponsiveContainer width="100%" height={248}>
    <AreaChart data={flow}
      onClick={(e) => e && e.activePayload && pickMonth(e.activePayload[0].payload.month_num, e.activePayload[0].payload)}>
      <defs><linearGradient id="netFill" ...>
        <stop offset={0}   stopColor={C.inflow}  stopOpacity={0.34} />
        <stop offset={off} stopColor={C.inflow}  stopOpacity={0.04} />
        <stop offset={off} stopColor={C.outflow} stopOpacity={0.04} />
        <stop offset={1}   stopColor={C.outflow} stopOpacity={0.30} />
      </linearGradient></defs>
      <XAxis dataKey="month" ... />
      <YAxis tickFormatter={compact} ... />
      <ReferenceLine y={0} ... />
      <Tooltip content={<FlowTip />} ... />
      <Area type="monotone" dataKey="net" fill="url(#netFill)" dot={<FlowDot sel={mn} />} ... />
      {selRow && <ReferenceDot x={selRow.month} y={selRow.net} ... />}
    </AreaChart>
  </ResponsiveContainer>
  ```

  Mechanically: `ResponsiveContainer` makes the chart fill its box. `AreaChart data={flow}`
  feeds it the array; children configure it. `dataKey="month"`/`dataKey="net"` tell each
  piece which **field** of each row to read (this is why the row field names matter). The
  `<defs><linearGradient>` is an SVG gradient (four color stops; `offset={off}` is the
  computed zero-crossing) referenced by `fill="url(#netFill)"`. `dot={<FlowDot sel={mn}/>}`
  and `content={<FlowTip/>}` pass elements the library renders per point (Rule 10). The
  chart's `onClick` reads the clicked point's payload and calls `pickMonth(month_num,
  point)`. `{selRow && <ReferenceDot .../>}` draws a marker only when a month is selected.

  **Right — `<section className="fx-drill">`** (230): the drill path.
  - `<Breadcrumb orch={orch} mn={mn} cat={cat} lin={lin} onRoot={()=>{…}} onMonth={…}
    onCat={…} />` — the child gets state as props and callbacks to reset levels (events up).
  - `{!mn && (<div className="fx-empty">…</div>)}` — the "pick a point" placeholder.
  - `{mn && (<div className="fx-level fx-anim">…</div>)}` — the month level: the trio of
    `<Stat/>`s (fed from `selRow`, no fetch), then `cats.map(...)` building category
    `<button>`s. Inside the map, `const max = Math.max(...cats.map(z => Math.abs(z.amount)), 1)`
    computes the bar scale, and the fill width is `style={{ width: (Math.abs(cc.amount)/max*100) + "%" }}`
    — a JS-computed percentage string.
  - `{mn && cat && (<div>…<table className="fx-table">…</table></div>)}` — the sources
    level: `<thead>` with `<th>` headers, `<tbody>` with `srcs.map(s => <tr …>…</tr>)`.
    Each row is keyed by `s.output_orchestration_key`, highlights when it equals `ook`,
    is keyboard-accessible (4.7), and shows fields via `<td>{s.output_name}</td>` etc.
    `{Math.round(s.allocation_pct * 100)}%` renders the allocation as a percent.
  - `{lin && (<div className="fx-level fx-anim fx-lineage">…</div>)}` — the lineage level:
    a header paragraph reading `lin.output.*`, then `<LineageInputs inputs={lin.inputs}
    chosen={chosenRev} />`.

The four `{cond && (…)}` blocks are literally your drill levels appearing as state fills
in: none → month → +sources → +lineage.

### 5.12 The sub-components (lines 326–420)

Same JSX rules, smaller scope.

- **`Stat({ k, v, c })`** (327): returns two divs (a label and a value colored by `c`).
  Pure presentation, props destructured.
- **`FlowDot({ cx, cy, payload, sel })`** (336): a recharts custom dot. `if (cx == null)
  return null` — returning `null` renders nothing (Rule 6). Returns an SVG `<circle>`
  whose radius is `0` for the selected month (`payload.month_num === sel`) else `2.6`.
- **`FlowTip({ active, payload })`** (342): the custom tooltip. Guards
  `if (!active || !payload || !payload.length) return null`, then reads the hovered row
  `const d = payload[0].payload` and renders in/out/net.
- **`Breadcrumb({...})`** (355): builds a plain array `crumbs` from state (`push`ing a
  crumb per active level), then `crumbs.map((c, i) => …)` renders each — with a separator
  for `i > 0` and either a `<button>` (clickable ancestor) or a `<span>` (current/inert),
  chosen by the ternary on line 365. A compact "derive data, then map to UI" example.
- **`LineageInputs({ inputs, chosen })`** (376): the revision track.
  `const chosenIdx = REV_ORDER.indexOf(chosen)` finds the resolved revision's position;
  `const multi = new Set(inputs.map(i => i.revision_name)).size > 1` asks "more than one
  distinct revision?" Then `inputs.map((inp, idx) => …)` renders each input, and *inside*
  that, `REV_ORDER.map((rev, j) => …)` renders the timeline ticks — a nested map. Each
  tick's className is assembled by string concatenation with ternaries:
  `"fx-tick" + (isChosen ? " chosen" : "") + (isLater ? " later" : "") + (beyondOutput ? " beyond" : "")`,
  and CSS (5.13) styles those state classes. `{inp.memo && <span>…</span>}` shows the memo
  only if present.

### 5.13 The stylesheet (lines 422–543)

```jsx
const CSS = `
@import url('https://fonts.googleapis.com/...');
.fx-root{ --ink:#17241E; ...; color:var(--ink); background:var(--field); min-height:100vh; }
.fx-root *{ box-sizing:border-box; }
...
`;
```

The whole stylesheet is one **template-literal string** (backticks, multi-line, 1.6),
assigned to `CSS`, and dropped into the DOM by `<style>{CSS}</style>` in the render. Every
rule follows Chapter 3: `.fx-…` class selectors matching the `className`s in the JSX,
custom properties on `.fx-root`, flexbox/grid layout, `@media` responsive breakpoints, a
`@keyframes fxIn` entrance animation, and state selectors like `.fx-tick.chosen .fx-dot`
that light the chosen revision gold. Nothing here is React-specific; it's plain CSS that
happens to travel inside the component file as a string.

---

## Chapter 6 — Symbol & pattern glossary (quick lookup)

When a piece of punctuation looks alien, find it here.

| Symbol / pattern | Name | Means |
|------------------|------|-------|
| `//` , `/* */` | comments | ignored text |
| `;` | semicolon | ends a statement |
| `const` / `let` | declarations | fixed binding / reassignable binding |
| `===` `!==` | strict equality | equal/unequal by value **and** type (always prefer over `==`) |
| `==` `!=` | loose equality | coercing equality — avoided, except idiomatic `x == null` |
| `&&` `\|\|` `!` | logical and/or/not | return an operand, not a bare boolean (drives conditional rendering) |
| `? :` | ternary | inline if/else expression |
| `??` | nullish coalescing | left value unless it's null/undefined, then right |
| `?.` | optional chaining | safe property access (returns undefined instead of crashing) — not in this file |
| `` `...${x}...` `` | template literal | string with `${}` interpolation (like an f-string) |
| `=>` | arrow function | compact function; `(a) => expr` implicitly returns `expr` |
| `{ }` (in JS) | block / object | statement block **or** object literal, by context |
| `{ }` (in JSX) | expression escape | "evaluate this JS and insert it" |
| `{{ }}` (in JSX) | object in braces | outer = JS-expression, inner = object literal (e.g. `style={{…}}`) |
| `[ ]` | array literal / index | make a list, or read `a[i]` |
| `...x` | spread | expand an array/object into elements/args |
| `[a, b] = …` | array destructuring | bind by position (e.g. `const [v, setV] = useState()`) |
| `{ a, b } = …` | object destructuring | bind by property name (used in component params) |
| `x.y` / `x["y"]` | member access | dot for known keys, brackets for dynamic keys |
| `.map(fn)` | array map | transform each element → new array (used to render lists) |
| `.filter(fn)` | array filter | keep elements where fn is truthy |
| `.reduce(fn, init)` | array reduce | fold to a single value |
| `.find(fn)` | array find | first matching element or `undefined` |
| `new Set(arr)` | set | unique values; `.size` counts distinct |
| `async` / `await` | async syntax | function returns a Promise; `await` unwraps one |
| `.then()` / `.catch()` | Promise methods | run on resolve / on reject |
| `fetch(url)` | HTTP client | returns a Promise of a Response |
| `throw` / `try`/`catch` | errors | raise / handle |
| `<Tag />` | JSX element | self-closing element (HTML tag or a component) |
| `<Tag>…</Tag>` | JSX element | element with children |
| `<>…</>` | Fragment | group siblings with no wrapper element — not in this file |
| `className=` | JSX attr | the CSS class (JSX's `class`) |
| `key=` | JSX attr | stable id for a `.map`-ed list item |
| `onClick=` `onChange=` `onKeyDown=` | JSX events | event handler props (camelCase) |
| `{cond && <X/>}` | conditional render | render `<X/>` only when `cond` is truthy |
| `{cond ? <A/> : <B/>}` | conditional render | one element or the other |
| `{list.map(...)}` | list render | render many elements from an array |
| `{/* … */}` | JSX comment | comment between tags |
| `useState(init)` | React hook | `[value, setter]`; setter triggers re-render |
| `useEffect(fn, [deps])` | React hook | run side effect after render; re-run when deps change |
| `.fx-x` (CSS) | class selector | match `class="fx-x"` |
| `a b` (CSS) | descendant | `b` inside `a` |
| `a.b` (CSS) | compound | element with both classes |
| `:hover` `:focus-visible` (CSS) | pseudo-class | style a state |
| `*` (CSS) | universal | match every element |
| `--x` / `var(--x)` (CSS) | custom property | define / read a CSS variable |
| `@import` `@media` `@keyframes` (CSS) | at-rules | import / conditional rules / animations |

---

## A suggested learning path

You don't have to absorb all of this at once. A concrete order that builds confidence:

1. **Read Chapter 1** (JavaScript) with the file open; find each construct in the file as
   you meet it. This is 70% of the unfamiliarity.
2. **Read Chapter 4.3–4.5** (JSX rules + the two hooks). This is the React core.
3. **Do the mechanical pass in Chapter 5** slowly, one section per sitting.
4. **Keep Chapter 6 open** as a lookup whenever a symbol stops you.
5. **Then make one tiny change** — e.g. change the empty-state text on line 239, or add a
   fourth `<Stat/>` — and watch it update. Editing one line and seeing the result is the
   fastest way to convert this from "read" to "known."

Everything here sits on models you already own: pure functions of inputs (materialized
outputs), contracts at boundaries (Pydantic/props), and dependency-driven recompute
(cache invalidation / effect deps). The syntax is new; the structure underneath is not.

# Reading `ForecastExplorer.wired.jsx` from scratch

This is the companion to the earlier mock walkthrough, rewritten for the **live-API
build** — the one you'd actually deploy. It's self-contained: you don't need the old
doc open. But here's the single most useful fact to carry in from it, because it makes
this whole file easier:

> **The mock and the wired build are byte-for-byte identical from the render tree
> downward.** Everything you already learned about JSX, state, the sub-components, the
> revision track, and the CSS is unchanged. The *only* thing that differs is the
> **data layer** — how bytes get into the component. So this doc re-teaches the file
> top to bottom (so it stands alone), but it slows down and does its real work in
> Parts 0.5, 4, and 6c, which are the parts that are genuinely new.

Read Part 0 and Part 0.5 first. Together they're the two ideas that make every other
part obvious — one is about React, one is the answer to the question that started this
whole thread.

---

## 0. The one idea that changes everything

In your vanilla JS demo, rendering was a **sequence of instructions you ran by hand**:

```js
const el = document.getElementById("net-value");  // find the node
el.textContent = "$1,234";                          // mutate it
// ...later, when data changes, find it again and mutate again
```

That's *imperative*: you describe **the steps** to change the screen. It's also exactly
why renaming an `id` without updating every `el("...")` silently killed downstream
rendering — the contract between "the DOM has this node" and "my JS looks up this node"
is invisible and unchecked, and one broken lookup throws and halts the rest of the
script.

React is *declarative*. You never look up a node and mutate it. Instead you write **one
function that returns a description of what the whole UI should look like, given the
current data**:

```
UI = f(state)
```

You change the *data* (`state`); React re-runs your function, compares the new
description to the old one, and patches only the DOM nodes that actually changed. You
never touch the DOM yourself.

This should feel familiar from your backend work. A materialized output is a **pure
function of its inputs** — you don't hand-patch the output table, you change inputs and
recompute. `crop_input_streams` isn't edited in place; it's *derived*. React's render is
the same discipline applied to the screen:

| Backend / ETL you know          | React equivalent                                    |
|---------------------------------|-----------------------------------------------------|
| Inputs (source rows)            | **state** (`useState`)                              |
| Materialized output             | **the returned JSX** (what's on screen)             |
| "recompute when inputs change"  | **re-render when state changes**                    |
| A transform's fetch/IO step     | **`useEffect`** (a side effect, run after render)   |
| A transform function's signature| **a component's props**                             |

Hold that table in your head. Note the one row that's different from the mock doc: the
mock derived its data with `useMemo`; the wired build **fetches** it, and fetching is a
*side effect*, which is what `useEffect` is for. That single swap is 80% of what's new.

---

## 0.5 The three shapes — and where the reshaping lives

This is the section that answers the question that kicked off everything: *"is there a
step to convert HTTP data into the shape the JSX uses?"* The short answer is **no, not
in the browser** — and understanding *why* is the whole mental model.

There are three shapes in play across the system. They are **not** all the same, and
that's by design:

| Shape | Where it lives | Field names look like | Structure |
|-------|----------------|-----------------------|-----------|
| **DB columns** | SQLite `crop_input_streams`, `crop_plan` | `crop_sys`, `perc`, `total_value`, `revision_name` | rows in tables |
| **API contract** | `schema.py` (Pydantic), the JSON on the wire | `crop`, `allocation_pct`, `annualized`, `chosen_revision` | denormalized response objects |
| **What the component reads** | this JSX file | `crop`, `allocation_pct`, `annualized`, `chosen_revision` | same as the contract |

The bottom two rows are **the same shape**. That's the trick. The component reads the
API contract directly. There is no fourth "JSX shape" that needs a translation step.

So where does DB → contract reshaping happen? **On the server.** It's exactly the `AS`
aliases in `queries.py` (`crop_sys AS crop`, `perc AS allocation_pct`,
`total_value AS annualized`) plus Pydantic serializing `schema.py` models to JSON.

Here's the part that explains your confusion when you read the *old* walkthrough. That
doc described the mock's `DATA` constant, which is in the **DB-columns** shape. But the
component renders the **contract** shape. Those genuinely differ — and in the mock, the
gap was bridged *in the browser* by the `store` object, which did the same aliasing
`queries.py` does. You were reading a doc that described the raw fixture (`DATA`), then
looking at a component that consumed the reshaped contract, and correctly noticed they
didn't line up. The reshaping layer was the missing middle.

The two builds, side by side:

```
MOCK:   DATA (DB shape)  ──store reshapes──▶  contract shape  ──▶  component renders
WIRED:  SQLite (DB shape) ──queries.py + Pydantic reshape──▶  contract JSON over HTTP  ──▶  component renders
```

Same reshaping, moved from the browser (`store`) to the server (`queries.py`), which is
where it always belonged. **In the wired build the component receives the contract shape
already finished**, so it renders it as-is. No client-side conversion step.

The one place a conversion *could* ever go — if the backend drifted from `schema.py`, or
you wanted a null fallback like `client_name ?? client_id`, or runtime validation — is
the `api` object (Part 4). It's the seam, the anti-corruption boundary, the same role
`store` played. Today it's empty of transforms because the shapes already agree.

Keep this picture. The rest of the file is just this idea plus React syntax.

---

## 1. Imports (top of file)

```jsx
import { useState, useEffect } from "react";
import {
  AreaChart, Area, XAxis, YAxis, Tooltip, ResponsiveContainer,
  ReferenceLine, ReferenceDot,
} from "recharts";
import { Wheat, CornerDownRight, ArrowUpRight, ArrowDownRight } from "lucide-react";
```

In your HTML demo you pulled libraries in with `<script src="...">` tags that dumped
globals onto `window`. Here there's no HTML doing that. **Vite** (your bundler) reads
these `import` lines, finds the packages in `node_modules`, and stitches exactly the
pieces you named into one bundle. Nothing becomes a global; each name is scoped to this
file.

- `useState`, `useEffect` — the two React **hooks** this build uses. (The mock imported
  `useMemo` instead of `useEffect`; that swap is the headline change — see Part 6.)
- `recharts` — the charting library; `AreaChart`, `XAxis`… are **components** you drop
  into JSX like tags.
- `lucide-react` — icon set; each name renders one SVG icon.

The `{ curly braces }` mean *named* imports. `import X from "..."` without braces would
be the *default* export.

---

## 2. What is *not* here: the `DATA` constant

In the mock, this spot held a giant `const DATA = {...}` literal — an inlined snapshot
of the serving store so the component could run with no backend. **In the wired build
it's gone entirely.** The data now arrives over HTTP.

That single deletion is worth pausing on, because it's the concrete form of Part 0.5.
The fixture existed only to stand in for a database + API that weren't running yet.
Once the real ones are running, the fixture is dead weight, so it's removed and the
`api` layer (Part 4) fetches the same shapes from the live endpoints instead. Think of
it like deleting a seed CSV once the real table is populated — the tests (here, the UI)
don't change, only the source of the rows.

---

## 3. Module constants

```jsx
const C = { ink:"#17241E", field:"#EDEFE8", ... };            // the palette
const MONTHS = ["", "Jan", "Feb", ... "Dec"];                  // 1-indexed labels
const REV_ORDER = ["1-pre-seeding", "2-post-seeding", "3-pre-harvest"];
```

Plain top-level constants, defined once when the module loads.

- `C` is the spruce/sage/harvest-gold palette as **JS values**, needed inline in the
  chart where CSS variables can't reach (recharts SVG props take real strings).
- `MONTHS` has an empty string at index 0 so `MONTHS[mn]` works with your 1-based month
  numbers — a deliberate off-by-one convenience. It's still here because the drill tag
  and breadcrumb label months by number even though cashflow rows now carry a `month`
  string from the API.
- `REV_ORDER` encodes the revision timeline; the lineage track (Part 7) uses it to know
  which revision comes before which.

Nothing here touches the network — these are the same in both builds.

---

## 4. The `api` layer — your serving layer, over HTTP

This is the wired build's version of the mock's `store`, and it's the anchor for
everything else. Where `store` filtered an in-memory array, `api` makes a real request.

```jsx
const API_BASE = "http://localhost:8000";
const enc = encodeURIComponent;

async function getJSON(path) {
  const res = await fetch(API_BASE + path);
  if (!res.ok) {
    let detail = res.statusText;
    try { detail = (await res.json()).detail ?? detail; } catch (_) {}
    throw new Error(`${res.status} ${detail}`);
  }
  return res.json();
}

const api = {
  orchestrations: () => getJSON(`/orchestrations`),                        // -> list[Orchestration]
  cashflow: (orch) => getJSON(`/forecast/${enc(orch)}/cashflow`),          // -> list[MonthPoint]
  month: (orch, m) => getJSON(`/forecast/${enc(orch)}/month/${m}`),        // -> MonthBreakdown
  sources: (orch, m, cat) => getJSON(`/forecast/${enc(orch)}/month/${m}/sources?category=${enc(cat)}`), // -> list[SourceRow]
  lineage: (ook) => getJSON(`/forecast/lineage/${enc(ook)}`),              // -> Lineage
};
```

Read `api` next to `store` and the parallel is exact:

| `store` (mock)                         | `api` (wired)                                  | `api.py` route |
|----------------------------------------|------------------------------------------------|----------------|
| `orchestrations()` filters `DATA`      | `getJSON('/orchestrations')`                   | `GET /orchestrations` |
| `cashflow(o)` reduces facts by month   | `getJSON('/forecast/{orch}/cashflow')`         | `GET /forecast/{orch}/cashflow` |
| `month(o, mn)` groups by category      | `getJSON('/forecast/{orch}/month/{mn}')`       | `GET .../month/{month_num}` |
| `sources(o, mn, cat)`                  | `getJSON('.../sources?category=..')`           | `GET .../sources` |
| `lineage(ook)`                         | `getJSON('/forecast/lineage/{ook}')`           | `GET /forecast/lineage/{ook}` |

Same five methods, same argument order, **same JSON shapes coming back** — that's the
whole reason the component below is unchanged. Three things are new versus the mock, and
all three are about "this is now real IO":

1. **`async` / `await` / Promises.** `getJSON` is `async`, so it returns a *Promise* —
   a value that arrives *later*. The data is no longer sitting in memory; you request it
   and wait. This is the single fact that ripples through the whole component (Part 6).

2. **Error handling at the boundary.** `if (!res.ok)` catches a non-2xx response and
   throws with FastAPI's `detail` field pulled out of the body (falling back to the HTTP
   status text). That thrown error is caught later by each effect's `.catch`, which sets
   the `err` state (Part 6e). This is the same instinct as raising in your query layer so
   callers can handle it — errors surface at the seam, not deep in the render.

3. **`enc = encodeURIComponent`** wraps every path segment that comes from data
   (`orch`, `ook`, `cat`). An `output_orchestration_key` or a category name with a space
   or slash would otherwise break the URL. It's contract-at-the-edge hygiene — the URL is
   a string boundary, so you sanitize what you interpolate into it.

`API_BASE` points at your uvicorn origin. Because this app is served from a *different*
origin (say `:5173`) than the API (`:8000`), the browser enforces CORS: that
`:5173` origin must be listed in `GROWLYTICS_CORS_ORIGINS` or every call is blocked
before it leaves the browser. This is exactly the `config.py` / CORS mechanics you set
up — the frontend is now the client that machinery was built for.

**The takeaway for learning:** this layer *is* your three-tier architecture, unchanged.
`api` calls the routes in `api.py`, which call `queries.py`, which read the store. React
only ever replaced the fourth thing you did by hand — turning JSON into DOM. Everything
above the component you already own.

---

## 5. Formatting helpers

```jsx
const money   = (n) => ...;   // 1234.5  -> "$1,235"
const compact = (n) => ...;   // 3673575 -> "$3.7M"
const num     = (n) => ...;   // 99709   -> "99,709"
```

Pure functions, no React, no network. Number in, string out. Called all over the JSX
(`{money(seasonNet)}`). Keeping formatting in small pure functions is the same instinct
as keeping transforms out of your query layer: the component stays about *structure*,
these stay about *presentation of a value*. Identical to the mock.

---

## 6. The component function — the heart

```jsx
export default function ForecastExplorer() {
  ...state (lots of it now)...
  ...cheap derived values...
  ...effects (the new part: data loading)...
  ...event handlers...
  ...a loading gate...
  return ( ...JSX... );
}
```

A React component **is a function**. It takes props (this one takes none) and returns a
description of UI. `export default` means "this is the main thing this file provides."

The single most important thing to internalize, same as the mock: **this whole function
re-runs top to bottom every time state changes.** Each run produces a fresh description;
React diffs it against the last and patches the real DOM.

### 6a. State — selections *and* fetched data

Here's the first structural difference from the mock, and it follows directly from
Part 0.5 + async. In the mock, only the drill *selections* were state; the *data*
(`flow`, `cats`, `srcs`, `lin`) was **derived** with `useMemo` because it was already in
memory. In the wired build the data can't be derived synchronously — it arrives later
over HTTP — so **the fetched data becomes state too**:

```jsx
// selections — what the user has drilled into
const [orchKey, setOrchKey]   = useState(null);   // which orchestration (null until loaded)
const [mn, setMn]             = useState(null);   // which month   (L1)
const [cat, setCat]           = useState(null);   // which category(L2)
const [ook, setOok]           = useState(null);   // which source  (L3)
const [chosenRev, setChosenRev] = useState(null); // resolved rev, carried from the clicked row

// fetched data — what came back over HTTP
const [orchestrations, setOrchestrations] = useState([]);   // the picker list
const [flow, setFlow]     = useState([]);   // cashflow series (L0)
const [selRow, setSelRow] = useState(null); // the clicked point (its trio: in/out/net)
const [mb, setMb]         = useState(null); // month breakdown (L1 categories)
const [srcs, setSrcs]     = useState([]);   // sources table   (L2)
const [lin, setLin]       = useState(null); // lineage         (L3)

// one shared error slot
const [err, setErr]       = useState(null);
```

`useState(initial)` returns `[value, setter]`. The two rules from the mock still hold and
matter even more here: you **never assign directly** (`flow = rows` is invisible to
React — you call `setFlow(rows)`), and within one render each value is a **fixed
snapshot** (a setter schedules the *next* render, it doesn't mutate the current one).

Why the split into "selections" vs "fetched data"? Because they change for different
reasons. Selections change from **user clicks** (synchronous, immediate). Fetched data
changes when **a request resolves** (asynchronous, later). The selections are the
*inputs*; the fetched data is what the effects *load in response to* those inputs. If you
know all of these values, you know exactly what's on screen — `UI = f(state)` still
holds; there's just more state now because the data lives in state instead of being
recomputed on the fly.

### 6b. Cheap derived values (no `useMemo` needed)

```jsx
const orch      = orchestrations.find((o) => o.orchestration_key === orchKey);
const cats      = mb ? mb.categories : [];
const seasonNet = flow.reduce((s, r) => s + r.net, 0);
const seasonIn  = flow.reduce((s, r) => s + r.inflow, 0);
const seasonOut = flow.reduce((s, r) => s + r.outflow, 0);

const vals = flow.map((r) => r.net);
const hi = Math.max(...vals, 0), lo = Math.min(...vals, 0);
const off = hi <= 0 ? 0 : lo >= 0 ? 1 : hi / (hi - lo);
```

These are still plain derivations — computed fresh each render from state that's already
in hand — but notice there's **no `useMemo`** anywhere in this file. That's a deliberate
teaching simplification, and it's worth understanding when it's fine and when it isn't.

`useMemo` is an *optimization*: it caches a result and skips recomputing it unless a
dependency changed. You reach for it when a computation is expensive enough that redoing
it on every render would be wasteful. Here every derivation is a reduction over ~12
cashflow rows — trivially cheap — so recomputing them each render costs nothing, and the
code is simpler without the ceremony. `orch` is a `.find` over six items; `cats` is a
single property read; the season totals are three 12-element reductions.

The mental model: **`useMemo` is your "recompute the materialized output only when its
inputs changed" discipline.** You apply that discipline when recompute is expensive
(the mock used it because `store.cashflow` did real filter/reduce work you didn't want
to redo on every hover). When recompute is free, you skip the cache — same as you
wouldn't materialize a column that's a trivial rename. If one of these ever got
expensive (say `srcs` became a big client-side sort), you'd wrap it:
`const srcs2 = useMemo(() => heavySort(srcs), [srcs])`. The dependency array is the
lineage; a missing dep is a stale value, the frontend version of a cache you forgot to
invalidate.

Note `cats = mb ? mb.categories : []`: `mb` is the whole `MonthBreakdown` object from
the API (Part 0.5's contract shape), and the category list the UI maps over is its
`.categories` field. Before the month has loaded, `mb` is `null`, so `cats` is `[]` and
the category list renders empty — which is the correct "not loaded yet" state.

### 6c. Effects — data loading, one per drill level (the new core)

This is the part that's genuinely new, and it's where your `fetch`/`await` knowledge
meets its React tool. In the mock, selecting a month *derived* the categories
immediately. Here, selecting a month must *fetch* them, and a fetch is a **side effect**
— it reaches outside the pure `UI = f(state)` world (it talks to the network, it
resolves later). Side effects live in `useEffect`.

```jsx
useEffect(fn, [deps])
```

reads as: *"after the component renders, run `fn`; and run it again whenever anything in
`[deps]` changes."* The dependency array is the same idea as a `useMemo` key — it's the
lineage of the effect, the list of inputs that should trigger a re-run.

There are five effects, one per drill level, each keyed on the selection it depends on.
Read them as a cascade — each level's effect fires when its selection (or an ancestor's)
changes:

**(1) Load the picker once, on mount:**

```jsx
useEffect(() => {
  api.orchestrations()
    .then((rows) => {
      setOrchestrations(rows);
      const def = rows.find((r) => r.orchestration_key.endsWith("3-pre-harvest")) ?? rows[0];
      if (def) setOrchKey(def.orchestration_key);
    })
    .catch((e) => setErr(e.message));
}, []);
```

The `[]` dependency array means "run this **once**, after the first render, and never
again" (nothing in `[]` ever changes). It fetches the orchestration list, stores it, and
picks a sensible default — a `3-pre-harvest` revision to match the demo, falling back to
the first row. `.then` handles success; `.catch` funnels any thrown error (including the
ones `getJSON` throws on non-2xx) into the `err` state.

**(2) Orchestration → cashflow, and reset the whole drill below it:**

```jsx
useEffect(() => {
  if (!orchKey) return;
  setMn(null); setSelRow(null); setMb(null); setCat(null); setSrcs([]); setOok(null); setChosenRev(null); setLin(null);
  api.cashflow(orchKey).then(setFlow).catch((e) => { setFlow([]); setErr(e.message); });
}, [orchKey]);
```

`[orchKey]` means "re-run whenever `orchKey` changes." When it does, this **clears every
deeper level** (month, category, source, and all their fetched data) before loading the
new cashflow. This is your coherence invariant, enforced in the effect: a deeper
selection can never survive a change to a shallower one it depends on. You'd recognize it
as the same reasoning behind "highest input revision ≤ output revision ≤ orchestration
revision" — the state can only ever sit in a *coherent* configuration.
`.then(setFlow)` is shorthand for `.then((rows) => setFlow(rows))`.

**(3) Month → category decomposition:**

```jsx
useEffect(() => {
  if (!orchKey || !mn) { setMb(null); return; }
  api.month(orchKey, mn).then(setMb).catch((e) => { setMb(null); setErr(e.message); });
}, [orchKey, mn]);
```

Fires when either `orchKey` or `mn` changes. The guard `if (!orchKey || !mn)` handles the
"nothing selected yet" case by clearing `mb` and bailing — you don't fetch a month that
isn't chosen. This is the **one new network call** when you open a month; the
inflow/outflow/net trio does *not* come from here (see the next point), preserving your
"one call per level, and the trio is free" design.

**(4) Category → sources:**

```jsx
useEffect(() => {
  if (!orchKey || !mn || !cat) { setSrcs([]); return; }
  api.sources(orchKey, mn, cat).then(setSrcs).catch((e) => { setSrcs([]); setErr(e.message); });
}, [orchKey, mn, cat]);
```

**(5) Source → lineage:**

```jsx
useEffect(() => {
  if (!ook) { setLin(null); return; }
  api.lineage(ook).then(setLin).catch((e) => { setLin(null); setErr(e.message); });
}, [ook]);
```

Each effect follows the identical shape: **guard → fetch → set data on success → clear +
record error on failure.** The guards mean an effect only fetches once its whole
selection path is present; the dependency arrays mean an effect re-fires exactly when one
of its inputs changes and no more often.

The timing is the thing to internalize, because it's the opposite of the mock's
synchronous world. The order of events on any selection is:

1. A setter runs (`setMn(3)`), scheduling a re-render.
2. The component **re-renders immediately** with the new selection but the *old* (or
   empty) data — the fetch hasn't happened yet.
3. **After** that render commits, the matching effect runs and calls `api.*` — the
   request goes out.
4. Sometime later the Promise resolves, `.then(setData)` runs, scheduling another
   re-render.
5. The component re-renders again, now with the fetched data, and it appears on screen.

So there's always a moment where the selection is set but the data for it hasn't
arrived. That's why fetched data starts as `[]`/`null`, and why you need the loading gate
(Part 6e) and empty-until-loaded derivations. Handling "not yet" is the tax of real IO —
the mock never paid it because its data was always already there.

### 6d. Event handlers — set selection, let effects do the fetching

```jsx
function pickOrch(v)        { setOrchKey(v); }
function pickMonth(m, point){ if (!m) return; setMn(m); setSelRow(point ?? null); setCat(null); setOok(null); setChosenRev(null); setLin(null); }
function pickCat(c)         { setCat(c); setOok(null); setChosenRev(null); setLin(null); }
function pickSrc(kk, rev)   { setOok(kk); setChosenRev(rev); }
```

Compare to the mock: the handlers are now *thinner*. They only change **selection
state**. They don't fetch anything — the **effects** watch the selections and fetch in
response. This is a clean separation you'll appreciate: clicks change *what you want to
see*; effects reconcile *the data* to match. (`pickOrch` doesn't even reset the deeper
levels itself, because effect (2) already does that whenever `orchKey` changes — the
reset lives in one place.)

Two details worth calling out:

- **`pickMonth(m, point)` carries the clicked point.** The chart hands the whole data
  point to the handler, and it's stashed in `selRow`. That point *is* a `MonthPoint` from
  the API, so it already contains `inflow`, `outflow`, and `net`. That's how the month
  panel shows the trio instantly with **no extra request** — it reuses the row the chart
  already had. Your "the trio comes from the clicked point, not a call" invariant,
  intact.
- **`pickSrc(kk, rev)` carries the resolved revision** (`chosen_revision`) from the
  clicked source row into `chosenRev` state. The lineage track needs to know which
  revision "won" to light it up, and rather than re-derive it, the row already knew it, so
  it rides along. (This is the change we made so the component never has to reach back
  into a data snapshot — the reason the wired build needs no `DATA`.)

The `pick*` handlers plus the effects together are the entire "state changes" surface:
clicks set selections, effects set data, everything else is a pure function of both.

### 6e. The loading gate — handling "not loaded yet"

```jsx
if (!orch) {
  return (
    <div className="fx-root">
      <style>{CSS}</style>
      <p style={{ padding: 28, color: "var(--muted)", ... }}>
        {err ? `Couldn't reach the API: ${err}` : "Loading forecast…"}
      </p>
    </div>
  );
}
```

An **early return**. On the very first render, `orchestrations` is `[]` and `orchKey` is
`null`, so `orch` (the `.find`) is `undefined`. If the main render tree tried to read
`orch.client_name` it would crash. The gate stops before that: if there's no
`orch` yet, render a small loading (or error) message and return. Once the mount effect
loads orchestrations and sets `orchKey`, a re-render happens, `orch` is now defined, the
gate passes, and the full UI renders.

This is the declarative way to handle async: you don't hide/show nodes imperatively, you
`return` a *different description* while data is missing. It also doubles as the
top-level error surface — if the very first call fails, `err` is set and the gate shows
it, since without orchestrations there's nothing else to draw. The mock had no gate
because its data was never missing.

### 6f. The return — declaring the UI in JSX

Everything above computed values and loaded data. `return (...)` describes what to draw.
**From here down, this file is identical to the mock** — so the four JSX rules are worth
a quick recap, but they behave exactly as you already learned.

JSX looks like HTML but is really JavaScript; Vite compiles each tag into a function call
producing a plain JS object (a "React element" — a description, not a DOM node). Four
differences from HTML:

**1. `className`, not `class`** (`class` is reserved in JS).

**2. `{ }` drops back into JavaScript.**

```jsx
<div className="fx-hero-net" style={{ color: seasonNet >= 0 ? C.inflow : C.outflow }}>
  {money(seasonNet)}
</div>
```

`{money(seasonNet)}` calls your formatter and inserts the string. `style={{...}}` is two
braces: outer = "JS expression", inner = "a JS object"; keys are camelCase.

**3. Conditional rendering uses JS operators, not `if`.**

```jsx
{!mn && ( <div className="fx-empty">...pick a point...</div> )}
{mn   && ( <div className="fx-level">...month detail...</div> )}
{mn && cat && ( <div>...sources table...</div> )}
{lin  && ( <div>...lineage...</div> )}
```

`A && B` renders `B` only when `A` is truthy. Read top to bottom, these four blocks *are*
your drill levels appearing as state fills in: nothing → month → +sources → +lineage.
There's also, just below the header, an error banner in the same style:

```jsx
{err && ( <div style={{...}}>API error: {err} — is uvicorn on :8000 and this origin in CORS?</div> )}
```

— a persistent surface for errors that happen *after* the initial load (a month that
404s, say), since those don't hit the loading gate.

**4. Lists are `.map()` returning elements, with a `key`.**

```jsx
{orchestrations.map((o) => (
  <option key={o.orchestration_key} value={o.orchestration_key}>
    {o.client_name} · {o.revision_name} · {o.forecast_year}
  </option>
))}
```

The `key` is a stable unique id per item so React can track which is which across
re-renders. Note this maps over `orchestrations` (loaded state) using **contract field
names** (`orchestration_key`, `client_name`, `revision_name`, `forecast_year`) — the
exact shape from Part 0.5. Same pattern for the category bars (`cats.map`), sources rows
(`srcs.map`), and lineage inputs.

**Events are props, camelCase, taking a function:**

```jsx
<select value={orchKey} onChange={(e) => pickOrch(e.target.value)}>
```

`<select value={orchKey}>` is a *controlled component* — the state `orchKey` is the
source of truth, the select just reflects it, `onChange` is the only way it changes.
State down, events up.

**The chart wires the click that starts the whole drill:**

```jsx
<AreaChart data={flow}
  onClick={(e) => e && e.activePayload && pickMonth(e.activePayload[0].payload.month_num, e.activePayload[0].payload)}>
  <XAxis dataKey="month" ... />
  <Area dataKey="net" dot={<FlowDot sel={mn} />} ... />
  {selRow && <ReferenceDot x={selRow.month} y={selRow.net} ... />}
</AreaChart>
```

You hand `AreaChart` your `flow` array; it reads `month`/`net` off each point (the
contract field names) and renders the SVG. Its `onClick` pulls the clicked point and
calls `pickMonth(month_num, point)` — passing both the number (to set `mn`) and the whole
point (to set `selRow`, the free trio). That click sets state → re-render → the month
effect fires → categories fetch → `{mn && (...)}` fills in. The full loop, one gesture.

---

## 7. The sub-components

Below the main function are five smaller components — **identical to the mock**. Each is
a function that takes props (destructured in the parameter list) and returns JSX.

```jsx
function Stat({ k, v, c }) {
  return (
    <div className="fx-stat">
      <div className="fx-stat-k">{k}</div>
      <div className="fx-stat-v" style={{ color: c }}>{v}</div>
    </div>
  );
}
```

`{ k, v, c }` is **props destructuring** — the caller wrote `<Stat k="Inflow" v="$1,234"
c="#2E7D57" />`. A component's props are its **signature / contract**, the same way a
function's parameters or a Pydantic model's fields are. `Stat` is pure presentation,
reused for Inflow/Outflow/Net (fed from `selRow`, the free trio).

- **`FlowDot`** / **`FlowTip`** are *render props* for recharts — you hand a component to
  `dot={<FlowDot .../>}` and `content={<FlowTip />}`, and recharts calls them with
  chart-specific props (`cx`, `cy`, `payload`, `active`). `FlowDot` returns `null` when
  `cx == null` — returning `null` means "render nothing." They read `payload.month_num`
  and `payload.month` — contract fields.

- **`Breadcrumb`** builds an array of crumb objects from current state (`orch` always;
  month if `mn`; category if `cat`; the output name if `lin`), then maps it to
  clickable/inactive spans. "Derive a small data structure from state, then map it to UI"
  — the whole React loop in miniature.

- **`LineageInputs`** is your signature revision track, and it's the clearest place your
  domain invariant becomes pixels:

```jsx
function LineageInputs({ inputs, chosen }) {
  const chosenIdx = REV_ORDER.indexOf(chosen);
  const multi = new Set(inputs.map((i) => i.revision_name)).size > 1;
  ...
  {inputs.map((inp, idx) => {
    const ri = REV_ORDER.indexOf(inp.revision_name);
    return (
      ...
      {REV_ORDER.map((rev, j) => {
        const isChosen     = j === ri;          // this input resolved to this rev
        const isLater      = j > ri;            // revs after the one that won
        const beyondOutput = j > chosenIdx;     // revs after the output's own rev
        return <div className={"fx-tick" + (isChosen ? " chosen" : "") + ...}>...</div>;
      })}
    );
  })}
}
```

For each input it walks the full `REV_ORDER` timeline and tags each tick as chosen /
later / beyond-the-output; the CSS lights the chosen dot gold and strikes the later
ones. This is "highest input revision ≤ output revision" rendered as a visual — the
resolution logic you *materialized* in `crop_input_streams` is being *traversed and
displayed*, not recomputed. Note it reads contract field names (`revision_name`,
`input_type`, `value`, `last_modified`) — the same names the API sends. The `chosen`
prop is the `chosenRev` your `pickSrc` carried in from the clicked row.

These components are defined *after* the main component but used *inside* it — JS
function declarations are hoisted, so order doesn't matter.

---

## 8. Styling — the CSS string + `<style>`

```jsx
const CSS = `
@import url('https://fonts.googleapis.com/...');
.fx-root{ --ink:#17241E; --field:#EDEFE8; ... }
...
`;
```

…and the first thing inside `<div className="fx-root">` is `<style>{CSS}</style>`. This
ships styles with the component: a template-literal string of plain CSS injected via a
`<style>` tag. Class names (`fx-root`, `fx-panel`…) match the `className`s in the JSX. The
CSS variables (`--ink`, `--grain`) are declared on `.fx-root`, which is why the palette
*also* exists as the JS `C` object — CSS uses the variables, but recharts SVG props need
real JS values, so the palette lives in both worlds. Identical to the mock; larger apps
would use CSS Modules / Tailwind / styled-components, but the mental model is the same:
class names on elements, rules in a stylesheet.

---

## 9. One full interaction, start to finish (now async)

Trace a session through the machine, watching the async beats:

1. **First render.** State is all empty: `orchestrations=[]`, `orchKey=null`, so `orch`
   is `undefined`. The **loading gate** fires and returns "Loading forecast…". Nothing
   else runs.

2. **Mount effect runs** (after that first render). `api.orchestrations()` goes out over
   HTTP. The function is done; the screen shows the loading text.

3. **Response lands.** `.then` runs: `setOrchestrations(rows)` and `setOrchKey("…3-pre-harvest")`.
   Two setters → a re-render is scheduled.

4. **Re-render.** Now `orch` is defined, so the gate passes and the full UI draws — but
   `flow` is still `[]`, so the chart is empty for an instant.

5. **The `[orchKey]` effect runs** (orchKey just went from null → a value). It resets the
   (already-empty) drill and fires `api.cashflow(orchKey)`. Response lands →
   `setFlow(rows)` → re-render → **the chart draws**. Screen: chart + "pick a point."

6. **You click the March point.** The chart's `onClick` reads `payload.month_num === 3`
   and calls `pickMonth(3, marchPoint)`. `setMn(3)` and `setSelRow(marchPoint)` (plus
   resets) → re-render.

7. **Re-render with `mn=3`.** The `{mn && (...)}` block appears; the inflow/outflow/net
   **trio shows immediately** from `selRow` — no call. `cats` is still `[]` (month not
   fetched yet), so the category bars are momentarily empty.

8. **The `[orchKey, mn]` effect runs** (mn changed). `api.month(orchKey, 3)` → response →
   `setMb(breakdown)` → re-render → `cats = mb.categories` is now populated → **category
   bars appear.**

9. **Click a category** → `pickCat` sets `cat` → the `[orchKey, mn, cat]` effect fires →
   `setSrcs(rows)` → **sources table appears.** Click a row → `pickSrc(ook, rev)` sets
   `ook` + `chosenRev` → the `[ook]` effect fires → `setLin(lineage)` → **the revision
   track appears.** Same loop, four levels.

At no point did anyone look up a DOM node and mutate it. Selections and fetched data
changed state; the function re-ran; React reconciled. The only difference from the mock's
trace is that data changes arrive on a **second** render after each fetch resolves —
which is exactly what the empty-defaults, guards, and loading gate exist to absorb.

---

## 10. What you're *not* hand-rolling forever

The five `useEffect` blocks are the raw, honest version of data fetching, and it's the
right thing to understand first. But two rough edges are worth knowing about, because in
a real app you'd address them:

- **Stale responses.** If you click orchestration A then quickly B, A's request might
  resolve *after* B's and overwrite the screen with A's data. The fix is an
  `AbortController` (or an "is this still the latest?" flag) in the effect's cleanup —
  `useEffect` can `return` a cleanup function that runs before the next effect or on
  unmount:

  ```jsx
  useEffect(() => {
    const ctrl = new AbortController();
    fetch(url, { signal: ctrl.signal }).then(...);
    return () => ctrl.abort();   // cancel the in-flight request if deps change first
  }, [orchKey]);
  ```

- **Per-section loading/error UI.** Right now there's one `err` slot and one top-level
  loading gate; a bigger app shows a spinner in *each* panel while its call is in flight.

In practice most teams don't hand-roll any of this — they use a data-fetching library
(**TanStack Query** / React Query, or **SWR**) that wraps fetch + caching + loading/error
+ cancellation into a hook like `useQuery(['cashflow', orchKey], () => api.cashflow(orchKey))`.
But understand this raw `useEffect` version first; the libraries are just ergonomics over
exactly this cascade. Everything else — state, the drill invariants, JSX, the
components — stays identical.

---

## 11. Vanilla JS → React cheat sheet (wired)

| You did this in the HTML/JS demo            | React (wired) does this                          |
|---------------------------------------------|--------------------------------------------------|
| `<script src>` for libraries                | `import` (bundled by Vite)                       |
| `getElementById` + set `.textContent`       | put `{value}` in JSX; change **state**           |
| Hand-mutate the DOM on every change         | Change state → component re-runs → React patches |
| A global holding "current month"            | `const [mn, setMn] = useState(null)`             |
| A global holding "the fetched rows"         | `const [flow, setFlow] = useState([])` (data is state) |
| `fetch().then(render)` touching the DOM     | `useEffect(() => api.x().then(setState), [deps])` |
| Re-fetch when a dropdown changes            | `useEffect(fetch, [dep])` — dep array is the trigger |
| `if (!loaded) show spinner` imperatively    | early `return <Loading/>` while data is missing  |
| `try/catch` around a fetch, set an error div| `.catch((e) => setErr(e.message))` + `{err && ...}` |
| `for` loop building HTML strings            | `array.map((x) => <El key=.../>)`                |
| `addEventListener("click", fn)`             | `onClick={fn}` prop                              |
| `class="..."` in HTML                       | `className="..."` in JSX                         |

---

## 12. The invariants worth keeping

- **`UI = f(state)`.** The screen is a pure function of the selections **and** the
  fetched data. There's no truth in the DOM; if you know the state you know the screen.
- **Data is state now, not a synchronous derivation.** Because it arrives over HTTP, it
  lands via `setState` inside an effect, on a render *after* the one that requested it.
  Empty defaults + guards + the loading gate exist to make "not yet" a coherent state.
- **Effects are lineage.** Each `useEffect` dependency array lists exactly the selections
  its fetch depends on — the frontend cache-invalidation key. A missing dep means a call
  that doesn't re-fire when it should (stale data); an extra dep means a redundant call.
- **The reshaping lives on the server.** DB columns → API contract happens in
  `queries.py` + Pydantic; the component receives the contract shape finished. The `api`
  object is the seam where a client-side adapter would go if the contract ever drifted —
  today it's transform-free because the shapes agree.
- **Deeper drill state never outlives shallower state.** Effect (2) clears everything
  below on any orchestration change; the `pick*` handlers clear dependents on month/
  category changes. State is always coherent — your revision-ordering instinct applied to
  UI state.
- **Handlers set selections; effects reconcile data.** Clicks say *what you want to see*;
  effects make the data match. Keeping those two jobs separate is why the handlers are
  thin and the data flow is easy to follow.

---

## 13. How the mock and the wired build relate (the capstone)

You now have two files that are the same program with two different data sources, and
seeing the diff between them *is* the lesson:

- **Identical, top to bottom:** the palette/constants, the formatting helpers, the entire
  render tree, all five sub-components, the revision track, and every line of CSS. The UI
  is one thing; it doesn't know or care where its data came from.
- **Different, and only here:** the data layer. `DATA` + `store` (synchronous, in-memory,
  reshapes DB→contract in the browser) becomes `api` (asynchronous, over HTTP, with the
  reshaping done server-side). `useMemo` derivations become `useState` + `useEffect`
  because the data now arrives later. A loading gate and error slot appear because "not
  yet" and "it failed" are now real states.

That's the whole arc of this thread in one sentence: **once the mock's `store` emitted
the same contract shape the API emits, swapping in `fetch` stopped touching the
component.** The confusion you hit at the start — "the doc describes `DATA`, but the
component reads a different shape" — was the sound of a reshaping layer doing its job
quietly. Now you can see it: it's `store` in one file and `queries.py` + Pydantic in the
other, and the component has been speaking the contract the entire time.

Take it one panel at a time. If you want, a good next exercise is to add a per-section
loading spinner to just the sources table — it touches state, an effect, and one
conditional in the render, and nothing else, so it's a small, safe way to feel the whole
loop move under your hands.

# Code Appendix

## `ForecastExplorer.wired.jsx`

```js
import { useState, useEffect } from "react";
import {
  AreaChart, Area, XAxis, YAxis, Tooltip, ResponsiveContainer,
  ReferenceLine, ReferenceDot,
} from "recharts";
import { Wheat, CornerDownRight, ArrowUpRight, ArrowDownRight } from "lucide-react";

/* ==========================================================================
   WIRED BUILD — no embedded data. Each method that the demo's `store` object
   faked is now a real fetch to the FastAPI serving layer (see `api` below).
   The endpoints and their response contracts are exactly the schema.py shapes
   the mock produced, so everything from the component down is byte-for-byte the
   same as the mock build.

   RUN (from the serving dir, matching serving_frontend.html):
     uvicorn api:app --port 8000      # the API   (origin :8000)
     npm run dev                      # this app  (origin :5173, say)
   :5173 must be in GROWLYTICS_CORS_ORIGINS (config.py) or the browser blocks it.
   ========================================================================== */

const C = {
  ink: "#17241E", field: "#EDEFE8", panel: "#FBFCF9", line: "#DBDED4",
  inflow: "#2E7D57", outflow: "#B0512B", grain: "#B5872E", muted: "#6E7368",
};
const MONTHS = ["", "Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];
const REV_ORDER = ["1-pre-seeding", "2-post-seeding", "3-pre-harvest"];

/* ----- the API: one real fetch per method, mirroring the routes in api.py.
   Each returns the schema.py contract verbatim — the same shapes the mock
   store returned. Point API_BASE at your uvicorn origin. --------------------- */
const API_BASE = "http://localhost:8000";
const enc = encodeURIComponent;

async function getJSON(path) {
  const res = await fetch(API_BASE + path);
  if (!res.ok) {
    let detail = res.statusText;
    try { detail = (await res.json()).detail ?? detail; } catch (_) {}
    throw new Error(`${res.status} ${detail}`);
  }
  return res.json();
}

const api = {
  orchestrations: () => getJSON(`/orchestrations`),                                        // -> list[Orchestration]
  cashflow: (orch) => getJSON(`/forecast/${enc(orch)}/cashflow`),                          // -> list[MonthPoint]
  month: (orch, m) => getJSON(`/forecast/${enc(orch)}/month/${m}`),                        // -> MonthBreakdown
  sources: (orch, m, cat) => getJSON(`/forecast/${enc(orch)}/month/${m}/sources?category=${enc(cat)}`), // -> list[SourceRow]
  lineage: (ook) => getJSON(`/forecast/lineage/${enc(ook)}`),                              // -> Lineage
};

/* ----- formatting ---------------------------------------------------------- */
const money = (n) => {
  if (n == null || isNaN(n)) return "—";
  const sign = n < 0 ? "-" : "";
  return sign + "$" + Math.round(Math.abs(n)).toLocaleString();
};
const compact = (n) => {
  const a = Math.abs(n), s = n < 0 ? "-" : "";
  if (a >= 1e6) return s + "$" + (a / 1e6).toFixed(1) + "M";
  if (a >= 1e3) return s + "$" + Math.round(a / 1e3) + "K";
  return s + "$" + Math.round(a);
};
const num = (n) => (n == null || isNaN(n) ? "—" : Number(n).toLocaleString());

/* ==========================================================================
   COMPONENT
   ========================================================================== */
export default function ForecastExplorer() {
  const [orchestrations, setOrchestrations] = useState([]);
  const [orchKey, setOrchKey] = useState(null);
  const [flow, setFlow] = useState([]);
  const [mn, setMn] = useState(null);
  const [selRow, setSelRow] = useState(null);   // the clicked point: inflow/outflow/net live here — no extra call
  const [mb, setMb] = useState(null);
  const [cat, setCat] = useState(null);
  const [srcs, setSrcs] = useState([]);
  const [ook, setOok] = useState(null);
  const [chosenRev, setChosenRev] = useState(null);
  const [lin, setLin] = useState(null);
  const [err, setErr] = useState(null);

  const orch = orchestrations.find((o) => o.orchestration_key === orchKey);
  const cats = mb ? mb.categories : [];
  const seasonNet = flow.reduce((s, r) => s + r.net, 0);
  const seasonIn = flow.reduce((s, r) => s + r.inflow, 0);
  const seasonOut = flow.reduce((s, r) => s + r.outflow, 0);

  // gradient split at the zero crossing
  const vals = flow.map((r) => r.net);
  const hi = Math.max(...vals, 0), lo = Math.min(...vals, 0);
  const off = hi <= 0 ? 0 : lo >= 0 ? 1 : hi / (hi - lo);

  // --- data loading: one effect per drill level, keyed on the selection state --- //

  // GET /orchestrations once; default to a 3-pre-harvest revision like the demo.
  useEffect(() => {
    api.orchestrations()
      .then((rows) => {
        setOrchestrations(rows);
        const def = rows.find((r) => r.orchestration_key.endsWith("3-pre-harvest")) ?? rows[0];
        if (def) setOrchKey(def.orchestration_key);
      })
      .catch((e) => setErr(e.message));
  }, []);

  // orchestration -> cashflow (and reset the whole drill below it)
  useEffect(() => {
    if (!orchKey) return;
    setMn(null); setSelRow(null); setMb(null); setCat(null); setSrcs([]); setOok(null); setChosenRev(null); setLin(null);
    api.cashflow(orchKey).then(setFlow).catch((e) => { setFlow([]); setErr(e.message); });
  }, [orchKey]);

  // month -> category decomposition (the trio came from the clicked point; this is the one new call)
  useEffect(() => {
    if (!orchKey || !mn) { setMb(null); return; }
    api.month(orchKey, mn).then(setMb).catch((e) => { setMb(null); setErr(e.message); });
  }, [orchKey, mn]);

  // category -> sources
  useEffect(() => {
    if (!orchKey || !mn || !cat) { setSrcs([]); return; }
    api.sources(orchKey, mn, cat).then(setSrcs).catch((e) => { setSrcs([]); setErr(e.message); });
  }, [orchKey, mn, cat]);

  // source -> lineage
  useEffect(() => {
    if (!ook) { setLin(null); return; }
    api.lineage(ook).then(setLin).catch((e) => { setLin(null); setErr(e.message); });
  }, [ook]);

  function pickOrch(v) { setOrchKey(v); }
  function pickMonth(m, point) { if (!m) return; setMn(m); setSelRow(point ?? null); setCat(null); setOok(null); setChosenRev(null); setLin(null); }
  function pickCat(c) { setCat(c); setOok(null); setChosenRev(null); setLin(null); }
  function pickSrc(kk, rev) { setOok(kk); setChosenRev(rev); }

  // nothing to show until the picker (and its first cashflow) have loaded
  if (!orch) {
    return (
      <div className="fx-root">
        <style>{CSS}</style>
        <p style={{ padding: 28, color: "var(--muted)", fontFamily: "'IBM Plex Sans',system-ui,sans-serif" }}>
          {err ? `Couldn't reach the API: ${err}` : "Loading forecast…"}
        </p>
      </div>
    );
  }

  return (
    <div className="fx-root">
      <style>{CSS}</style>

      <header className="fx-head">
        <div className="fx-brand">
          <Wheat size={20} strokeWidth={1.6} color={C.grain} />
          <div>
            <div className="fx-brand-name">GrowLytics</div>
            <div className="fx-brand-sub">Forecast Explorer</div>
          </div>
        </div>
        <label className="fx-picker">
          <span>Forecast revision</span>
          <select value={orchKey} onChange={(e) => pickOrch(e.target.value)}>
            {orchestrations.map((o) => (
              <option key={o.orchestration_key} value={o.orchestration_key}>
                {o.client_name} · {o.revision_name} · {o.forecast_year}
              </option>
            ))}
          </select>
        </label>
      </header>

      {err && (
        <div style={{ margin: "14px 26px 0", padding: "10px 14px", border: "1px solid var(--outflow)", borderRadius: 8, color: "var(--outflow)", background: "var(--panel)", fontSize: 13 }}>
          API error: {err} — is uvicorn on :8000 and this origin in GROWLYTICS_CORS_ORIGINS?
        </div>
      )}

      <main className="fx-grid">
        {/* -------- LEFT: persistent season context -------- */}
        <section className="fx-context">
          <div className="fx-panel">
            <div className="fx-eyebrow">{orch.client_name} · {orch.forecast_year} season</div>
            <div className="fx-hero-net" style={{ color: seasonNet >= 0 ? C.inflow : C.outflow }}>
              {money(seasonNet)}
            </div>
            <div className="fx-hero-label">projected net cash position</div>

            <div className="fx-season-split">
              <div><ArrowUpRight size={14} color={C.inflow} /> <span className="fx-mono">{compact(seasonIn)}</span> in</div>
              <div><ArrowDownRight size={14} color={C.outflow} /> <span className="fx-mono">{compact(seasonOut)}</span> out</div>
            </div>

            <div className="fx-chart">
              <ResponsiveContainer width="100%" height={248}>
                <AreaChart
                  data={flow}
                  margin={{ top: 8, right: 6, bottom: 0, left: -8 }}
                  onClick={(e) => e && e.activePayload && pickMonth(e.activePayload[0].payload.month_num, e.activePayload[0].payload)}
                >
                  <defs>
                    <linearGradient id="netFill" x1="0" y1="0" x2="0" y2="1">
                      <stop offset={0} stopColor={C.inflow} stopOpacity={0.34} />
                      <stop offset={off} stopColor={C.inflow} stopOpacity={0.04} />
                      <stop offset={off} stopColor={C.outflow} stopOpacity={0.04} />
                      <stop offset={1} stopColor={C.outflow} stopOpacity={0.30} />
                    </linearGradient>
                  </defs>
                  <XAxis dataKey="month" tick={{ fontSize: 11, fill: C.muted, fontFamily: "IBM Plex Mono" }}
                    axisLine={{ stroke: C.line }} tickLine={false} interval={0} />
                  <YAxis tickFormatter={compact} width={54}
                    tick={{ fontSize: 10, fill: C.muted, fontFamily: "IBM Plex Mono" }}
                    axisLine={false} tickLine={false} />
                  <ReferenceLine y={0} stroke={C.line} strokeWidth={1} />
                  <Tooltip cursor={{ stroke: C.grain, strokeWidth: 1, strokeDasharray: "3 3" }}
                    content={<FlowTip />} />
                  <Area type="monotone" dataKey="net" stroke={C.ink} strokeWidth={1.6}
                    fill="url(#netFill)" dot={<FlowDot sel={mn} />} activeDot={{ r: 5, fill: C.grain, stroke: C.panel, strokeWidth: 2 }}
                    animationDuration={650} />
                  {selRow && <ReferenceDot x={selRow.month} y={selRow.net} r={6}
                    fill={C.grain} stroke={C.panel} strokeWidth={2} />}
                </AreaChart>
              </ResponsiveContainer>
              <div className="fx-chart-hint">Select a month to trace where the money moved.</div>
            </div>
          </div>
        </section>

        {/* -------- RIGHT: the drill path -------- */}
        <section className="fx-drill">
          <Breadcrumb orch={orch} mn={mn} cat={cat} lin={lin}
            onRoot={() => { setMn(null); setCat(null); setOok(null); }}
            onMonth={() => { setCat(null); setOok(null); }}
            onCat={() => setOok(null)} />

          {!mn && (
            <div className="fx-empty">
              <CornerDownRight size={18} color={C.muted} />
              <p>Pick a point on the chart to open its month. From there, drill category → source → the exact versioned inputs behind any figure.</p>
            </div>
          )}

          {mn && (
            <div className="fx-level fx-anim">
              <div className="fx-level-head">
                <h2>Where the money moved</h2>
                <span className="fx-tag">{MONTHS[mn]}</span>
              </div>
              <div className="fx-trio">
                <Stat k="Inflow" v={money(selRow.inflow)} c={C.inflow} />
                <Stat k="Outflow" v={money(selRow.outflow)} c={C.outflow} />
                <Stat k="Net" v={money(selRow.net)} c={C.ink} />
              </div>
              <div className="fx-nocall">these three came from the chart point — no extra call</div>

              <div className="fx-cats">
                {cats.map((cc) => {
                  const max = Math.max(...cats.map((z) => Math.abs(z.amount)), 1);
                  const active = cc.category === cat;
                  return (
                    <button key={cc.category} className={"fx-catrow" + (active ? " on" : "")}
                      onClick={() => pickCat(cc.category)}>
                      <span className="fx-catname">{cc.category}</span>
                      <span className="fx-cattrack">
                        <span className="fx-catfill" style={{
                          width: (Math.abs(cc.amount) / max * 100) + "%",
                          background: cc.amount >= 0 ? C.inflow : C.outflow,
                        }} />
                      </span>
                      <span className="fx-catamt fx-mono">{money(cc.amount)}</span>
                    </button>
                  );
                })}
              </div>
            </div>
          )}

          {mn && cat && (
            <div className="fx-level fx-anim">
              <div className="fx-level-head">
                <h2>Sources</h2>
                <span className="fx-tag">{cat} · {MONTHS[mn]}</span>
              </div>
              <table className="fx-table">
                <thead>
                  <tr><th>Output</th><th>Crop</th><th>Resolved</th>
                    <th className="r">Annualized</th><th className="r">Alloc</th><th className="r">This month</th></tr>
                </thead>
                <tbody>
                  {srcs.map((s) => (
                    <tr key={s.output_orchestration_key} className={"fx-src" + (s.output_orchestration_key === ook ? " on" : "")}
                      tabIndex={0} onClick={() => pickSrc(s.output_orchestration_key, s.chosen_revision)}
                      onKeyDown={(e) => { if (e.key === "Enter" || e.key === " ") { e.preventDefault(); pickSrc(s.output_orchestration_key, s.chosen_revision); } }}>
                      <td>{s.output_name}</td>
                      <td className="fx-crop">{s.crop}</td>
                      <td><span className="fx-revpill">{s.chosen_revision}</span></td>
                      <td className="r fx-mono">{money(s.annualized)}</td>
                      <td className="r fx-mono">{Math.round(s.allocation_pct * 100)}%</td>
                      <td className="r fx-mono strong">{money(s.allocated)}</td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          )}

          {lin && (
            <div className="fx-level fx-anim fx-lineage">
              <div className="fx-level-head">
                <h2>How this figure was built</h2>
                <span className="fx-tag gold">lineage</span>
              </div>
              <p className="fx-linhdr">
                <b>{lin.output.output_name}</b> · {lin.output.crop} · {lin.output.category}
                <span className="fx-linann"> annualized {money(lin.output.annualized)}</span>
              </p>
              <LineageInputs inputs={lin.inputs} chosen={chosenRev} />
            </div>
          )}
        </section>
      </main>
    </div>
  );
}

/* ----- small pieces -------------------------------------------------------- */
function Stat({ k, v, c }) {
  return (
    <div className="fx-stat">
      <div className="fx-stat-k">{k}</div>
      <div className="fx-stat-v" style={{ color: c }}>{v}</div>
    </div>
  );
}

function FlowDot({ cx, cy, payload, sel }) {
  if (cx == null) return null;
  const on = payload.month_num === sel;
  return <circle cx={cx} cy={cy} r={on ? 0 : 2.6} fill={C.ink} opacity={0.5} />;
}

function FlowTip({ active, payload }) {
  if (!active || !payload || !payload.length) return null;
  const d = payload[0].payload;
  return (
    <div className="fx-tip">
      <div className="fx-tip-m">{d.month}</div>
      <div><span style={{ color: C.inflow }}>in</span> <b className="fx-mono">{money(d.inflow)}</b></div>
      <div><span style={{ color: C.outflow }}>out</span> <b className="fx-mono">{money(d.outflow)}</b></div>
      <div className="fx-tip-net"><span>net</span> <b className="fx-mono">{money(d.net)}</b></div>
    </div>
  );
}

function Breadcrumb({ orch, mn, cat, lin, onRoot, onMonth, onCat }) {
  const crumbs = [{ label: orch.revision_name, on: onRoot, active: !mn }];
  if (mn) crumbs.push({ label: MONTHS[mn], on: onMonth, active: !cat });
  if (cat) crumbs.push({ label: cat, on: onCat, active: !lin });
  if (lin) crumbs.push({ label: `${lin.output.output_name} · ${lin.output.crop}`, on: null, active: true });
  return (
    <nav className="fx-crumbs">
      {crumbs.map((c, i) => (
        <span key={i}>
          {i > 0 && <span className="fx-crumb-sep">›</span>}
          {c.on && !c.active
            ? <button className="fx-crumb" onClick={c.on}>{c.label}</button>
            : <span className={"fx-crumb" + (c.active ? " active" : "")}>{c.label}</span>}
        </span>
      ))}
    </nav>
  );
}

/* the signature element: each input shown against the season's revision
   timeline, with the resolved revision lit and later ones struck. */
function LineageInputs({ inputs, chosen }) {
  const chosenIdx = REV_ORDER.indexOf(chosen);
  const multi = new Set(inputs.map((i) => i.revision_name)).size > 1;
  return (
    <div>
      <p className="fx-linnote">
        {multi
          ? `Inputs resolved to ${new Set(inputs.map((i) => i.revision_name)).size} different revisions — each field takes the newest revision at or before ${chosen}.`
          : `All inputs resolved to ${chosen}.`}
      </p>
      <div className="fx-inputs">
        {inputs.map((inp, idx) => {
          const ri = REV_ORDER.indexOf(inp.revision_name);
          return (
            <div className="fx-input" key={idx}>
              <div className="fx-input-top">
                <span className="fx-input-type">{inp.input_type}</span>
                <span className="fx-input-val fx-mono">{num(inp.value)}{inp.unit ? " " + inp.unit : ""}</span>
              </div>
              <div className="fx-track">
                {REV_ORDER.map((rev, j) => {
                  const isChosen = j === ri;
                  const isLater = j > ri;
                  const beyondOutput = j > chosenIdx;
                  return (
                    <div key={j} className={"fx-tick" + (isChosen ? " chosen" : "") + (isLater ? " later" : "") + (beyondOutput ? " beyond" : "")}>
                      <span className="fx-dot" />
                      <span className="fx-ticklabel">{rev.split("-")[0]}</span>
                    </div>
                  );
                })}
                <div className="fx-trackline" />
              </div>
              <div className="fx-input-meta">
                <span className="fx-revpill sm">{inp.revision_name}</span>
                <span className="fx-input-mod">modified {inp.last_modified}</span>
                {inp.memo && <span className="fx-input-memo">“{inp.memo}”</span>}
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}

/* ==========================================================================
   STYLES
   ========================================================================== */
const CSS = `
@import url('https://fonts.googleapis.com/css2?family=Newsreader:opsz,wght@6..72,400;6..72,500;6..72,600&family=IBM+Plex+Sans:wght@400;500;600&family=IBM+Plex+Mono:wght@400;500&display=swap');

.fx-root{ --ink:#17241E; --field:#EDEFE8; --panel:#FBFCF9; --line:#DBDED4;
  --inflow:#2E7D57; --outflow:#B0512B; --grain:#B5872E; --muted:#6E7368;
  font-family:'IBM Plex Sans',system-ui,sans-serif; color:var(--ink);
  background:var(--field); min-height:100vh; letter-spacing:.005em; }
.fx-root *{ box-sizing:border-box; }
.fx-mono{ font-family:'IBM Plex Mono',ui-monospace,monospace; }

.fx-head{ display:flex; align-items:center; justify-content:space-between;
  padding:18px 26px; border-bottom:1px solid var(--line); background:var(--panel); }
.fx-brand{ display:flex; align-items:center; gap:11px; }
.fx-brand-name{ font-family:'Newsreader',serif; font-weight:600; font-size:19px; line-height:1; }
.fx-brand-sub{ font-size:11px; letter-spacing:.14em; text-transform:uppercase; color:var(--muted); margin-top:3px; }
.fx-picker{ display:flex; flex-direction:column; gap:4px; align-items:flex-end; }
.fx-picker span{ font-size:10.5px; letter-spacing:.12em; text-transform:uppercase; color:var(--muted); }
.fx-picker select{ font:inherit; font-size:13px; padding:7px 11px; border:1px solid var(--line);
  border-radius:8px; background:var(--field); color:var(--ink); cursor:pointer; }
.fx-picker select:focus-visible{ outline:2px solid var(--grain); outline-offset:1px; }

.fx-grid{ display:grid; grid-template-columns:minmax(340px,2fr) 3fr; gap:22px; padding:22px 26px 48px; max-width:1240px; }
@media(max-width:900px){ .fx-grid{ grid-template-columns:1fr; } }

.fx-panel{ background:var(--panel); border:1px solid var(--line); border-radius:14px; padding:20px 22px; }
.fx-context{ position:sticky; top:22px; align-self:start; }
@media(max-width:900px){ .fx-context{ position:static; } }

.fx-eyebrow{ font-size:11px; letter-spacing:.13em; text-transform:uppercase; color:var(--muted); }
.fx-hero-net{ font-family:'Newsreader',serif; font-size:46px; font-weight:500; line-height:1.05; margin-top:8px; letter-spacing:-.01em; }
.fx-hero-label{ font-size:12.5px; color:var(--muted); margin-top:2px; }
.fx-season-split{ display:flex; gap:18px; margin:14px 0 4px; font-size:13px; color:var(--muted); }
.fx-season-split div{ display:flex; align-items:center; gap:5px; }
.fx-season-split .fx-mono{ color:var(--ink); }

.fx-chart{ margin-top:12px; }
.fx-chart-hint{ font-size:11.5px; color:var(--muted); font-style:italic; margin-top:6px; text-align:center; }
.fx-tip{ background:var(--ink); color:#EDEFE8; border-radius:9px; padding:9px 11px; font-size:12px; box-shadow:0 6px 18px rgba(0,0,0,.18); }
.fx-tip-m{ font-family:'IBM Plex Mono',monospace; font-size:11px; opacity:.7; margin-bottom:5px; letter-spacing:.05em; }
.fx-tip div{ display:flex; justify-content:space-between; gap:16px; }
.fx-tip-net{ border-top:1px solid rgba(255,255,255,.16); margin-top:5px; padding-top:5px; }

.fx-drill{ min-height:300px; }
.fx-crumbs{ display:flex; flex-wrap:wrap; align-items:center; gap:2px; margin-bottom:16px; font-size:12.5px; }
.fx-crumb{ background:none; border:none; font:inherit; font-size:12.5px; color:var(--muted); cursor:pointer; padding:2px 4px; border-radius:5px; }
button.fx-crumb:hover{ color:var(--ink); background:rgba(0,0,0,.04); }
.fx-crumb.active{ color:var(--ink); font-weight:600; }
.fx-crumb-sep{ color:var(--line); margin:0 2px; }

.fx-empty{ display:flex; gap:12px; align-items:flex-start; background:var(--panel); border:1px dashed var(--line);
  border-radius:14px; padding:26px; color:var(--muted); font-size:14px; line-height:1.55; }
.fx-empty p{ margin:0; max-width:52ch; }

.fx-level{ background:var(--panel); border:1px solid var(--line); border-radius:14px; padding:18px 20px; margin-bottom:16px; }
.fx-level-head{ display:flex; align-items:baseline; gap:10px; margin-bottom:14px; }
.fx-level-head h2{ font-family:'Newsreader',serif; font-weight:600; font-size:17px; margin:0; }
.fx-tag{ font-size:11px; font-family:'IBM Plex Mono',monospace; color:var(--muted); background:var(--field);
  padding:2px 8px; border-radius:999px; border:1px solid var(--line); }
.fx-tag.gold{ color:var(--grain); border-color:var(--grain); }

.fx-trio{ display:flex; gap:10px; }
.fx-stat{ flex:1; border:1px solid var(--line); border-radius:10px; padding:9px 12px; }
.fx-stat-k{ font-size:10.5px; letter-spacing:.09em; text-transform:uppercase; color:var(--muted); }
.fx-stat-v{ font-size:19px; font-weight:600; margin-top:3px; }
.fx-nocall{ font-size:11px; color:var(--muted); margin:9px 0 2px; font-style:italic; }

.fx-cats{ margin-top:14px; display:flex; flex-direction:column; gap:8px; }
.fx-catrow{ display:grid; grid-template-columns:120px 1fr 120px; align-items:center; gap:12px;
  background:none; border:1px solid transparent; border-radius:9px; padding:6px 8px; cursor:pointer; font:inherit; text-align:left; }
.fx-catrow:hover{ background:var(--field); }
.fx-catrow.on{ border-color:var(--grain); background:var(--field); }
.fx-catname{ font-size:13px; font-weight:500; }
.fx-cattrack{ height:20px; background:var(--field); border-radius:5px; overflow:hidden; border:1px solid var(--line); }
.fx-catrow.on .fx-cattrack{ background:var(--panel); }
.fx-catfill{ display:block; height:100%; border-radius:5px 0 0 5px; transition:width .5s cubic-bezier(.4,0,.2,1); }
.fx-catamt{ text-align:right; font-size:13px; }

.fx-table{ width:100%; border-collapse:collapse; font-size:12.5px; }
.fx-table th{ text-align:left; font-size:10.5px; letter-spacing:.05em; text-transform:uppercase;
  color:var(--muted); font-weight:600; padding:6px 8px; border-bottom:1px solid var(--line); }
.fx-table th.r, .fx-table td.r{ text-align:right; }
.fx-table td{ padding:8px; border-bottom:1px solid var(--line); vertical-align:middle; }
.fx-src{ cursor:pointer; transition:background .12s; }
.fx-src:hover{ background:var(--field); }
.fx-src.on{ background:var(--field); box-shadow:inset 3px 0 0 var(--grain); }
.fx-src:focus-visible{ outline:2px solid var(--grain); outline-offset:-2px; }
.fx-crop{ text-transform:capitalize; color:var(--muted); }
.strong{ font-weight:600; }
.fx-revpill{ font-family:'IBM Plex Mono',monospace; font-size:11px; background:var(--field);
  border:1px solid var(--line); border-radius:6px; padding:2px 7px; white-space:nowrap; }
.fx-revpill.sm{ font-size:10.5px; padding:1px 6px; }

.fx-lineage{ border-color:var(--grain); box-shadow:0 0 0 1px rgba(181,135,46,.12); }
.fx-linhdr{ font-size:14px; margin:0 0 3px; }
.fx-linann{ color:var(--muted); font-family:'IBM Plex Mono',monospace; font-size:12px; margin-left:6px; }
.fx-linnote{ font-size:12px; color:var(--grain); margin:6px 0 16px; line-height:1.5; }

.fx-inputs{ display:flex; flex-direction:column; gap:14px; }
.fx-input{ border:1px solid var(--line); border-radius:11px; padding:13px 15px; background:var(--field); }
.fx-input-top{ display:flex; justify-content:space-between; align-items:baseline; }
.fx-input-type{ font-size:13.5px; font-weight:600; }
.fx-input-val{ font-size:14px; }
.fx-track{ position:relative; display:flex; justify-content:space-between; margin:16px 6px 6px; }
.fx-trackline{ position:absolute; top:5px; left:5px; right:5px; height:1px; background:var(--line); z-index:0; }
.fx-tick{ position:relative; z-index:1; display:flex; flex-direction:column; align-items:center; gap:6px; }
.fx-dot{ width:11px; height:11px; border-radius:50%; background:var(--panel); border:1.5px solid var(--line); }
.fx-ticklabel{ font-size:10px; font-family:'IBM Plex Mono',monospace; color:var(--muted); text-transform:capitalize; }
.fx-tick.chosen .fx-dot{ background:var(--grain); border-color:var(--grain); box-shadow:0 0 0 3px rgba(181,135,46,.2); }
.fx-tick.chosen .fx-ticklabel{ color:var(--grain); font-weight:600; }
.fx-tick.later .fx-dot{ opacity:.4; }
.fx-tick.later .fx-ticklabel{ opacity:.4; text-decoration:line-through; }
.fx-input-meta{ display:flex; flex-wrap:wrap; align-items:center; gap:9px; margin-top:4px; font-size:11.5px; color:var(--muted); }
.fx-input-mod{ font-family:'IBM Plex Mono',monospace; }
.fx-input-memo{ font-style:italic; }

.fx-anim{ animation:fxIn .34s cubic-bezier(.2,.7,.2,1) both; }
@keyframes fxIn{ from{ opacity:0; transform:translateY(6px); } to{ opacity:1; transform:none; } }
@media(prefers-reduced-motion:reduce){ .fx-anim{ animation:none; } .fx-catfill{ transition:none; } }
`;

```