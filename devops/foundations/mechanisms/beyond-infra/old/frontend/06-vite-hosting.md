# React / Vite Hosting & Structure Reference (GrowLytics `frontend_system`)

The equivalent of your `python -m http.server` workflow — plus repo layout, where the `.jsx` lives, and the TypeScript migration path.

---

## 0. The one invariant everything else follows from

**The browser only understands three things: HTML, CSS, and plain JavaScript (ES modules).** It does *not* understand:

- **JSX** — `<ForecastExplorer />` is not JavaScript. It's syntax sugar that must be compiled to `React.createElement(...)` calls.
- **Bare imports** — `import React from "react"` names a package, not a URL. The browser needs `import React from "/node_modules/.vite/deps/react.js"` or similar. Something must rewrite it.
- **TypeScript** — `.ts`/`.tsx` types must be stripped out before the browser sees the code.

So the whole question "how do I host a React app" reduces to: **something has to transform your source files into what the browser can actually run.** That "something" is Vite.

With plain HTML, your source files *already are* what the browser needs, so `python -m http.server` just hands them over untouched. With React, there's a transform step in between. That's the entire difference. Everything below is just *when* and *where* that transform happens.

### The mental model you already have

This maps cleanly onto your ETL / materialized-cache distinction:

| Your data world | Frontend world |
|---|---|
| **Compute-on-read** — run the transform per request against source | **`npm run dev`** — Vite transforms each `.jsx` on the fly, per browser request |
| **Materialized output** — run the transform once, serve the frozen result | **`npm run build`** — transform everything ahead of time into a `dist/` folder of plain static files |

`python -m http.server` is a pure "serve materialized output" tool. It has no transform step. That's *fine* for plain HTML (no transform needed) and it's *also* fine for a **built** React app (transform already done — the output in `dist/` is plain HTML/CSS/JS). It is **not** fine for React *source* — there's nothing to do the JSX transform.

---

## 1. Command equivalence (the direct answer)

Your three-terminal HTML workflow, translated. Terminals 1 and 2 (data + API) **do not change at all** — only the frontend terminal changes.

```text
HOW TO RUN — DEV MODE (three terminals)

  Terminal 1  (data layer — UNCHANGED)
    python build/build_serving_store.py          # build the SQLite store

  Terminal 2  (API — UNCHANGED)
    uvicorn api:app --port 8000                  # the API  (origin :8000)

  Terminal 3  (frontend — this is what changes)
    cd src/growlytics_platform/frontend_system
    npm install                                  # first time only (installs node_modules/)
    npm run dev                                  # Vite dev server (origin :5173) + hot reload
```

```text
HOW TO RUN — PRODUCTION PREVIEW (what you'd actually ship)

  Terminals 1 & 2: same as above.

  Terminal 3:
    cd src/growlytics_platform/frontend_system
    npm run build                                # transform ALL source -> dist/ (static files)
    npm run preview                              # serve dist/ locally (origin :4173)

  # dist/ is now plain HTML/CSS/JS. You could ALSO serve it with your old tool:
  #   cd dist && python -m http.server 5173      # works identically — no transform needed anymore
```

### The one-to-one mapping

| Step | Your HTML workflow | React/Vite — dev | React/Vite — prod |
|---|---|---|---|
| 1. Data | `python build/build_serving_store.py` | *(unchanged)* | *(unchanged)* |
| 2. API | `uvicorn api:app --port 8000` | *(unchanged)* | *(unchanged)* |
| 3. Frontend | `python -m http.server 5173` | `npm run dev` → `:5173` | `npm run build` → then `npm run preview` (`:4173`) **or** serve `dist/` with anything, including `python -m http.server` |

Default ports: **Vite dev = 5173**, **Vite preview = 4173**. Both configurable (see §6).

**Key takeaway:** `npm run dev` is your day-to-day equivalent of `python -m http.server` — but it does live transformation + hot module replacement (HMR), so edits to `.jsx` appear in the browser without a manual refresh. `python -m http.server` never did that; it just served bytes.

---

## 2. Repo structure — where the `.jsx` goes

After scaffolding into `frontend_system/`, this is the shape. Annotated with what each piece is *for*.

```text
src/growlytics_platform/frontend_system/
├── index.html               ← THE ENTRY POINT. Not in public/, not in src/ — the root.
│                              Vite reads this first; it's the "front door."
├── package.json             ← declares deps + the dev/build/preview scripts (§5)
├── package-lock.json        ← pins exact dep versions (commit this)
├── vite.config.js           ← Vite config: port, proxy, plugins (§6)
├── node_modules/            ← installed deps. HUGE. gitignore it. Rebuilt by `npm install`.
├── dist/                    ← `npm run build` output. Static, shippable. gitignore it.
├── public/                  ← static assets served AS-IS at the root URL, untouched by Vite.
│   └── favicon.svg            e.g. public/logo.png  ->  http://localhost:5173/logo.png
└── src/                     ← YOUR SOURCE CODE lives here. Everything here gets transformed.
    ├── main.jsx             ← the JS entry module. Mounts React onto <div id="root">.
    ├── App.jsx              ← the root component. Renders your app's top-level tree.
    ├── ForecastExplorer.jsx ← *** YOUR COMPONENT GOES HERE ***  (or src/components/)
    ├── index.css            ← global styles (your spruce/sage/harvest-gold tokens fit here)
    └── App.css              ← component-scoped styles (optional)
```

**Where to put `ForecastExplorer.jsx`:** inside `src/`. Either flat (`src/ForecastExplorer.jsx`) for a small app, or `src/components/ForecastExplorer.jsx` once you have more than a few components. The only hard rule: **it must live under `src/` so Vite transforms it**, and it must be *imported* (directly or transitively) starting from `main.jsx`, or it never gets bundled.

### The entry-point chain (this is the non-obvious part coming from plain HTML)

In plain HTML, JS enters via a `<script>` tag you write by hand. In Vite, there's a fixed chain, and understanding it removes most of the "where does my code even run" confusion:

```text
index.html                         (the front door)
   │  contains:  <div id="root"></div>
   │             <script type="module" src="/src/main.jsx"></script>
   ▼
src/main.jsx                       (JS entry — the "wire it up" file)
   │  createRoot(document.getElementById("root")).render(<App />)
   ▼
src/App.jsx                        (root component)
   │  import ForecastExplorer from "./ForecastExplorer"
   │  return <ForecastExplorer />
   ▼
src/ForecastExplorer.jsx           (your actual UI — fetches from :8000, renders the drill-down)
```

Read it top to bottom: the HTML gives React an empty `<div id="root">` to take over; `main.jsx` grabs that div and hands control to React; `App.jsx` is the top of your component tree; your `ForecastExplorer` hangs off of it. This is the DOM contract, same species as the vanilla-JS `el("...")` null-reference lesson: **if `index.html` has no `<div id="root">`, `main.jsx`'s `getElementById("root")` returns `null` and the whole app silently fails to mount** — identical failure mode, different layer.

A minimal `main.jsx` for reference:

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App.jsx";
import "./index.css";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

---

## 3. Merging your existing `ForecastExplorer.jsx` into a fresh scaffold

Because you already have a component, don't let the scaffolder near it. Safest sequence:

```bash
cd src/growlytics_platform
npm create vite@latest frontend_system -- --template react   # clean scaffold
cd frontend_system
npm install

# copy your existing component in:
cp /path/to/ForecastExplorer.jsx src/ForecastExplorer.jsx
```

Then wire it up by editing `src/App.jsx` to import and render it (replace the demo counter the template ships with):

```jsx
import ForecastExplorer from "./ForecastExplorer.jsx";

export default function App() {
  return <ForecastExplorer />;
}
```

If `frontend_system/` already contains your work, `create-vite` will detect a non-empty directory and prompt you (remove / ignore / cancel) — pick **cancel** and scaffold into a throwaway folder instead, then copy the config files (`package.json`, `vite.config.js`, `index.html`) over manually. Never let it "remove existing files."

---

## 4. The `:5173 → :8000` origin gap (CORS — and a nicer alternative)

Same issue you already solved for the HTML version: the browser tab is served from `:5173` (Vite) but your API is at `:8000`, so every `fetch` is cross-origin and gets blocked unless the server allows it.

**Option A — keep using CORS (what you have).** Just make sure `http://localhost:5173` is in `GROWLYTICS_CORS_ORIGINS`. For the prod preview add `http://localhost:4173` too. Nothing new here.

**Option B — Vite dev proxy (worth knowing).** Vite can proxy API paths so the browser *thinks* everything is same-origin, and CORS never enters the picture in dev. In `vite.config.js`:

```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  server: {
    port: 5173,
    proxy: {
      "/api": {
        target: "http://localhost:8000",  // your FastAPI
        changeOrigin: true,
      },
    },
  },
  plugins: [react()],
});
```

Now a `fetch("/api/forecast")` from your React code goes to `:5173/api/...`, Vite quietly forwards it to `:8000`, and the browser sees a same-origin request. This is the frontend analogue of your **config-injection** habit: the environment (dev vs prod) decides where `/api` points, and your component code just says `/api/...` regardless. In production you'd instead serve `dist/` behind the same host/reverse-proxy as the API (e.g. Azure App Service), so `/api` resolves naturally and, again, no CORS. The proxy is dev-only; it does nothing in `build`/`preview`.

Both options are valid. CORS is the explicit contract; the proxy is the "make the origins match so there's no contract to negotiate" move. Pick one and be consistent per environment.

---

## 5. What the `package.json` scripts actually are

The template gives you these. They're just named shell commands (`npm run <name>`):

```jsonc
{
  "scripts": {
    "dev":     "vite",           // start dev server + HMR (compute-on-read)   -> :5173
    "build":   "vite build",     // transform everything into dist/ (materialize)
    "preview": "vite preview",   // serve the built dist/ locally               -> :4173
    "lint":    "eslint ."        // static checks (optional but recommended)
  }
}
```

- `npm run dev` — daily driver. Live transform, instant reload. **Never ship this** — it's a dev server, not a production one.
- `npm run build` — the transform/materialize step. Produces `dist/`: minified, hashed, plain static assets. This is the only artifact that gets deployed.
- `npm run preview` — a quick local sanity check that the *built* output works (it serves `dist/`). Also not a production server; it's just closer to prod than `dev`.

The deployable artifact is always **`dist/`**. How you serve it in production (Azure static hosting, a CDN, nginx, even `python -m http.server` in a pinch) is orthogonal to Vite — by that point it's just static files.

---

## 6. Configuring the port (to keep `:5173` exactly)

If you want the dev port pinned so it matches your existing CORS/muscle memory, set it in `vite.config.js` (shown in §4) or per-run:

```bash
npm run dev -- --port 5173      # the -- forwards --port to vite, same pattern as scaffolding
npm run preview -- --port 5173  # pin preview to 5173 too if you prefer
```

The `--` here does the same job it did in the scaffold command: it separates npm's flags from the ones you want forwarded to the underlying tool (Vite).

---

## 7. Environment config (the `VITE_` invariant)

Coming from `GROWLYTICS_CORS_ORIGINS`-style env injection, the frontend equivalent has one hard rule you must internalize:

**Only variables prefixed with `VITE_` are exposed to your browser code.** This is a *safety* invariant — it prevents you from accidentally leaking server secrets into a bundle that ships to every user's browser.

```bash
# frontend_system/.env.development
VITE_API_BASE=/api            # using the proxy from §4

# frontend_system/.env.production
VITE_API_BASE=https://your-app.azurewebsites.net/api
```

```jsx
// in your component:
const base = import.meta.env.VITE_API_BASE;
fetch(`${base}/forecast`);
```

`import.meta.env` is Vite's injection channel (not `process.env` — that's Node). Anything *without* the `VITE_` prefix stays server-side and is invisible to the browser. Mental model: `VITE_` is the allowlist boundary between "safe to materialize into the public bundle" and "must never leave the server" — a deliberate contract, exactly like your CORS origin list.

---

## 8. TypeScript — the near-future upgrade

### What actually changes

TypeScript is **the same transform pipeline with type-stripping added, plus a separate type-checking step.** Concretely:

| Aspect | JavaScript (now) | TypeScript (upgrade) |
|---|---|---|
| Scaffold template | `--template react` | `--template react-ts` |
| File extensions | `.jsx`, `.js` | `.tsx` (has JSX), `.ts` (no JSX) |
| Config file | *(none needed)* | `tsconfig.json` + `tsconfig.node.json` |
| Type definitions | — | `@types/react`, `@types/react-dom` (template adds them) |
| `build` script | `vite build` | `tsc -b && vite build` |
| Dev server | `vite` | `vite` *(unchanged!)* |

Your component becomes `ForecastExplorer.tsx`. The entry chain is identical, just renamed: `index.html` → `main.tsx` → `App.tsx` → `ForecastExplorer.tsx`.

### The critical invariant most people miss

**Vite does not type-check. It only strips types.** Vite uses esbuild to turn `.tsx` into `.js`, and esbuild deletes type annotations *without verifying them* — that's why it's fast. So on its own, Vite will happily transform and run code that is riddled with type errors.

Type safety comes from a **separate** tool: `tsc` (the TypeScript compiler) run with `--noEmit` (or build mode `tsc -b`), which checks but produces no output. This is why the `react-ts` template's build script is:

```jsonc
"build": "tsc -b && vite build"
//         └─ type-check first; the && means vite build only runs if types pass
```

Consequences to plan around:

- **`npm run dev` does *not* type-check** (it's still just `vite`). Type errors won't stop the dev server. You catch them via your editor's TS language server, or by running `tsc -b --watch` in a fourth terminal.
- **`npm run build` *does* type-check**, because of the `tsc -b &&` prefix — a bad type will fail the build. This is your CI gate.
- So the "does bad code get through" answer depends on *which command*: dev is permissive, build is strict. Know which one you're relying on.

Think of it as two independent contracts over the same source: **esbuild** owns *"turn it into runnable JS"* (fast, no checking), and **tsc** owns *"prove the types are consistent"* (the actual guarantee). Vite wires up the first; you must not forget to run the second. It's the same separation-of-concerns discipline as keeping your Pydantic validation distinct from your SQL execution — one proves the contract, the other does the work.

### Migration path (incremental — you don't need a big-bang rewrite)

TypeScript allows JS and TS to coexist, so you can convert file by file:

1. Add TS tooling to the existing JS project:
   ```bash
   npm install -D typescript @types/react @types/react-dom
   npx tsc --init          # generates a tsconfig.json to tune
   ```
2. In `tsconfig.json` set `"allowJs": true` so `.jsx`/`.js` files still compile alongside `.tsx`. Start with `"strict": false` and tighten later.
3. Rename one file `.jsx` → `.tsx`, fix the errors it surfaces, commit. Repeat.
4. Add the type-check to your build script once enough is converted: `"build": "tsc -b && vite build"`.
5. Flip `"strict": true` when you're ready for the full guarantee — this is where TS earns its keep (it'll flag every place a `fetch` result is used without validating its shape, which for a data-heavy forecasting UI is exactly the class of bug worth catching).

Because you already think in Pydantic-style contracts, TS will feel native: it's the same "declare the shape, enforce it at the boundary" instinct, moved to the browser. Your L0–L3 API response shapes become `interface`s, and the compiler enforces that your drill-down components consume them correctly — the frontend counterpart to your Pydantic v2 schemas.

---

## 9. Gotchas checklist

- **`node_modules/` and `dist/` → `.gitignore`.** Commit `package.json` and `package-lock.json`; those two reconstruct `node_modules/` deterministically via `npm install`. (`node_modules` is the "rebuildable cache," `package-lock.json` is the "recipe.")
- **`public/` vs `src/`.** Files in `public/` are served verbatim at the URL root and are *not* processed. Files in `src/` are transformed. Put importable code/assets in `src/`; put "must keep this exact filename/path" assets (e.g. a `robots.txt`) in `public/`.
- **Imports are relative to the file**, e.g. `import ForecastExplorer from "./ForecastExplorer"` — the `.jsx`/`.tsx` extension is optional in the import. A wrong relative path is the most common "module not found" cause.
- **The `<div id="root">` contract.** If it's missing or renamed in `index.html`, the app mounts to `null` and silently renders nothing — same failure class as your vanilla-JS `id`/`el("...")` mismatch, one layer up.
- **Dev server ≠ production server.** `npm run dev` and `npm run preview` are for local use only. Ship the contents of `dist/`.
- **Version pinning.** `npm create vite@latest` gives you whatever the current major is; the *commands and workflow above are stable across Vite versions*, so the specific number doesn't matter for this guide. `package-lock.json` freezes the exact versions for reproducibility.

---

## 10. TL;DR

- `npm run dev` (→ `:5173`) is your everyday replacement for `python -m http.server`, but with live JSX transformation + hot reload.
- Your `.jsx` goes in **`src/`**, reached via the chain `index.html → src/main.jsx → src/App.jsx → src/ForecastExplorer.jsx`.
- Terminals 1 (build store) and 2 (uvicorn) don't change; only the frontend terminal does.
- `npm run build` materializes everything into **`dist/`** — plain static files you can serve *anywhere*, including your old `python -m http.server`.
- Keep `http://localhost:5173` (and `:4173` for preview) in `GROWLYTICS_CORS_ORIGINS`, or use Vite's dev proxy to sidestep CORS entirely.
- **TypeScript:** scaffold with `--template react-ts`, files become `.tsx`, and remember the invariant — **Vite strips types but does not check them; `tsc -b` (wired into the `build` script) is what enforces them.** `npm run dev` stays permissive.


# File Structure

## Run it (three terminals, from the serving dir)

```bash
# 1) data layer — UNCHANGED
python build/build_serving_store.py          # build the SQLite store

# 2) API — UNCHANGED
uvicorn api:app --port 8000                  # the API (origin :8000)

# 3) frontend — this replaces `python -m http.server`
cd src/growlytics_platform/frontend_system
npm install                                  # first time only -> creates node_modules/ + package-lock.json
npm run dev                                  # Vite dev server (origin :5173) + hot reload
```

Open http://localhost:5173. Requirements: **Node 20+ (22 recommended — see `.nvmrc`)**.
`:5173` must be in `GROWLYTICS_CORS_ORIGINS`, or switch on the dev proxy in
`vite.config.js` and set `VITE_API_BASE=` empty.

### Ship it

```bash
npm run build     # transform everything -> dist/ (plain static files)
npm run preview   # serve dist/ locally (origin :4173) to sanity-check
```

Deploy the contents of `dist/`. For production set `VITE_API_BASE` to the real
API origin (e.g. via `.env.production`) before `npm run build`.

---

## File map

### ✅ Must-have — the app won't run without these (all included here)

| File | What it is / why it's required |
|---|---|
| `index.html` | The entry point Vite reads first. Holds `<div id="root">` and the module `<script>`. |
| `src/main.jsx` | JS entry. Mounts React onto `#root`. Start of the entry chain. |
| `src/App.jsx` | Root component. Renders `ForecastExplorer`; future home for routing/layout. |
| `src/ForecastExplorer.jsx` | **Your component** — the actual UI + API calls. (One edit vs. your upload — see below.) |
| `src/index.css` | Minimal global reset. Kept tiny on purpose; the component owns its own styles. |
| `vite.config.js` | Registers the React plugin (enables JSX), pins the dev port, holds the proxy option. |
| `package.json` | Declares deps (`react`, `react-dom`, `recharts`, `lucide-react`) + the `dev`/`build`/`preview` scripts. |

### 🤖 Auto-generated — never write or edit these by hand; git-ignored

| File / dir | Created by | Notes |
|---|---|---|
| `package-lock.json` | `npm install` | Pins exact resolved versions. **Do commit this** (it's the reproducible recipe), but never hand-edit it. |
| `node_modules/` | `npm install` | The installed dependency tree. Huge, disposable, rebuilt from the lockfile. Git-ignored. |
| `dist/` | `npm run build` | The materialized static output you deploy. Regenerated every build. Git-ignored. |
| `.vite/` | Vite | Dev dependency-pre-bundling cache. Delete anytime to force a clean rebuild. Git-ignored. |

> Rule of thumb: `package.json` + `package-lock.json` are the **recipe** (commit them); `node_modules/` and `dist/` are the **cache** (rebuildable, ignore them). Same distinction as your source CSVs vs. the built SQLite store.

### 🎁 Nice-to-have — included, but safe to delete; each earns its place

| File | Purpose | Delete if… |
|---|---|---|
| `.env.development` | Injects `VITE_API_BASE` for dev so the API origin isn't hardcoded. | you're fine hardcoding the fallback in the component. |
| `.env.example` | Committed template documenting required env vars (real `.env.*` stay out of git). | you don't use env vars at all. |
| `.gitignore` | Keeps `node_modules/`, `dist/`, local envs, and caches out of the repo. | (keep it — deleting this is the one you'll regret.) |
| `.nvmrc` | Pins the Node version (`22`) so you/CI use a Vite-compatible runtime. | you manage Node versions elsewhere. |
| `eslint.config.js` | Powers `npm run lint`; catches missing hook deps, unused vars, JSX slips. | you don't want linting yet (also drop the `lint` script + its 3 devDeps). |
| `README.md` | This file. | never. 🙂 |

---

## The one change made to your uploaded component

`ForecastExplorer.jsx` line 31 was:

```js
const API_BASE = "http://localhost:8000";
```

changed to:

```js
const API_BASE = import.meta.env.VITE_API_BASE ?? "http://localhost:8000";
```

It still works with zero config (the `?? "http://localhost:8000"` fallback), but
now dev/prod origins come from `.env.*` instead of being baked in — the same
config-injection pattern as `GROWLYTICS_CORS_ORIGINS`. Nothing else was touched.

---

## Entry chain (how a browser request becomes your UI)

```
index.html                     ← browser loads this; it has <div id="root">
  └─ <script src="/src/main.jsx">
       └─ main.jsx              ← createRoot(#root).render(<App/>)
            └─ App.jsx          ← <ForecastExplorer/>
                 └─ ForecastExplorer.jsx   ← fetches :8000, renders L0–L3
```

If `index.html` loses its `<div id="root">`, `main.jsx` mounts to `null` and the
page renders blank — the same null-reference DOM-contract failure as renaming an
`id` without updating `el("...")` in the vanilla-JS demo.















# Code Appendix

## `index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>GrowLytics · Forecast Explorer</title>
  </head>
  <body>
    <!-- React takes over this div. If you rename this id, update main.jsx too,
         or getElementById("root") returns null and nothing mounts. -->
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

## `src/main.jsx`

```javascript
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App.jsx";
import "./index.css";

// The entry chain: index.html -> main.jsx (here) -> App.jsx -> ForecastExplorer.jsx
// createRoot mounts React onto the empty <div id="root"> from index.html.
createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

## `src/App.jsx`

```javascript
import ForecastExplorer from "./ForecastExplorer.jsx";

// The top of your component tree. Right now it just renders the one screen.
// Later, this is where a router, a global header, or a client picker would live.
export default function App() {
  return <ForecastExplorer />;
}
```

## `src/index.css`

```css
/* Deliberately minimal. ForecastExplorer.jsx carries its own full design system
   (the CSS template string + the Google Fonts @import at the bottom of that file),
   so this file only handles what lives OUTSIDE .fx-root. Don't duplicate the
   spruce/sage/harvest-gold tokens here — they're owned by the component. */

html,
body,
#root {
  margin: 0;
  padding: 0;
  min-height: 100%;
}

body {
  /* matches --field so there's no white gutter before the app paints */
  background: #edefe8;
  font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
  -webkit-font-smoothing: antialiased;
  text-rendering: optimizeLegibility;
}
```

## `vite.config.js`

```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

// https://vite.dev/config/
export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173, // pinned so it stays in GROWLYTICS_CORS_ORIGINS; Vite falls forward
    //           to 5174+ if 5173 is taken unless you also set strictPort: true
    strictPort: false,

    // --- WSL note ---------------------------------------------------------
    // If the app is unreachable from your Windows browser, uncomment host:
    // host: true,   // binds 0.0.0.0 so Windows can reach the WSL2 dev server
    // ----------------------------------------------------------------------

    // --- OPTIONAL: dev proxy (alternative to CORS) -------------------------
    // With this on, set VITE_API_BASE="" and the browser sees same-origin
    // requests to :5173, which Vite forwards to :8000 — no CORS needed in dev.
    // Your API routes live at the root (/orchestrations, /forecast/...), so
    // proxy those prefixes explicitly:
    //
    // proxy: {
    //   "/orchestrations": { target: "http://localhost:8000", changeOrigin: true },
    //   "/forecast":       { target: "http://localhost:8000", changeOrigin: true },
    // },
    // ----------------------------------------------------------------------
  },
  preview: {
    port: 4173,
  },
});
```

## `package.json`

```json
{
  "name": "growlytics-frontend",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint ."
  },
  "dependencies": {
    "lucide-react": "^0.400.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "recharts": "^2.13.0"
  },
  "devDependencies": {
    "@eslint/js": "^9.0.0",
    "@vitejs/plugin-react": "^4.3.0",
    "eslint": "^9.0.0",
    "globals": "^15.0.0",
    "vite": "^6.0.0"
  }
}
```

## `.gitignore`

```
# dependencies — rebuilt by `npm install` from package-lock.json
node_modules/

# build output — regenerated by `npm run build`
dist/

# Vite cache
.vite/

# local env overrides (may hold secrets); keep .env.example committed
.env.local
.env.*.local

# logs / editor
npm-debug.log*
*.log
.DS_Store
.idea/
.vscode/*
!.vscode/extensions.json
```