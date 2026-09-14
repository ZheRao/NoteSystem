# Vite + React Setup

Two paths: scaffold fresh, or wire up an existing folder.

---

## Step 0 — Node version

Check first. This is the most common source of setup failures.

```bash
node -v
```

Current `create-vite` requires Node `^20.19.0 || >=22.12.0`. Node 18 reached end-of-life in April 2025 and will fail.

### Installing / upgrading with nvm

nvm keeps multiple Node versions side by side and lets you switch per project — better than a system-wide install when different projects have different requirements.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

Restart the shell (or `source ~/.bashrc`), then:

```bash
nvm install 22
nvm use 22          # this shell only
nvm alias default 22   # all new shells
```

Confirm with `node -v` before continuing. Upgrading Node also upgrades the bundled npm.

### about the numher `22`

`22` is a Node **major version number**, not anything nvm-specific. `nvm install 22` means "give me the latest 22.x release" — you can equally say `nvm install 24`, or `nvm install 22.11.0` for an exact patch.

**Where the versions stand now.** Node 24 is the Active LTS line. Node 22 has moved into Maintenance — still receiving security fixes, but no longer the default recommendation. Node 18, which you're on, is past EOL entirely.

So for your situation, `nvm install 24` is the better call.

**This convention is going away.** Starting with 27.x, Node moves to one major release per year instead of two, and every release becomes LTS — no more odd/even distinction. Each version will go LTS after its six-month Current phase. Version numbers will align with the calendar year of the initial release: 27.0.0 in 2027, 28.0.0 in 2028.

Practically: today, pick the highest even number that's in Active LTS. From 2027 on, the year tells you the version, and "is it even" stops being a useful heuristic.

**How to check rather than remember.** `nvm ls-remote --lts` lists LTS releases with their status, and `nvm install --lts` grabs the newest without you naming a number. Worth using in place of a hardcoded `22` for exactly the reason you're asking about.

### Pinning a version per project

Drop a `.nvmrc` in the project root:

```
22
```

Then `nvm use` in that directory picks it up with no argument. Useful when one machine hosts projects on different Node versions.

### Staying on an older Node

If upgrading isn't an option, pin the scaffolder to a generation that still supports it:

```bash
npm create vite@5 frontend_system -- --template react
```

Workable, but it means learning on a toolchain two majors behind. Treat it as a stopgap.

---

## Path A — Scaffold (empty or new folder)

```bash
npm create vite@latest frontend_system -- --template react
cd frontend_system
npm install
npm run dev
```

Dev server runs at `http://localhost:5173`.

The `--` passes the flags through to `create-vite` instead of letting npm consume them. Drop it and you land in the interactive prompt.

**Template options**

| Template | What you get |
|---|---|
| `react` | JS, Babel |
| `react-ts` | TypeScript, Babel |
| `react-swc` | JS, SWC — faster on big projects |
| `react-swc-ts` | TypeScript, SWC |

Scaffolding does **not** install dependencies. `npm install` is a required second step.

---

## Path B — Existing folder

Use when the directory already has files. `npm create vite@latest .` will offer to wipe them, which is usually not what you want.

### 1. Ensure `package.json` exists

```bash
npm init -y    # only if you don't already have one — it overwrites
```

### 2. Install

```bash
npm install react react-dom
npm install -D vite @vitejs/plugin-react
```

Runtime code (`react`, `react-dom`) goes in `dependencies` — it ends up in the bundle. Build tooling goes in `devDependencies` via `-D`.

### 3. Add scripts to `package.json`

```json
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "preview": "vite preview"
}
```

### 4. `vite.config.js`

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

### 5. `index.html` — at project root

Not in `public/`. This differs from Create React App.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

`type="module"` and the leading slash both matter — Vite resolves the entry from project root, not relative to the HTML file.

### 6. `src/main.jsx` — entry point

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'

ReactDOM.createRoot(document.getElementById('root')).render(<App />)
```

### 7. `src/App.jsx`

```jsx
export default function App() {
  return <h1>Hello</h1>
}
```

---

## Resulting structure

```
.
├── index.html
├── package.json
├── vite.config.js
└── src
    ├── main.jsx
    └── App.jsx
```

---

## Gotchas

**`EBADENGINE` is a warning, not a stop.** npm prints the engine mismatch and then runs the package anyway. The real failure comes later and looks unrelated — on Node 18 it surfaces as `SyntaxError: The requested module 'node:util' does not provide an export named 'styleText'`, because `styleText` landed in Node 20.12. If a tool crashes right after an `EBADENGINE` warning, check `node -v` before debugging anything else.

**JSX must be in `.jsx`, not `.js`.** Vite's esbuild transform won't parse JSX in `.js` files. The error points at your first `<div>` and reads like a syntax error. Rename the file.

**`index.html` is the entry, not `src/`.** Vite treats the HTML file as the graph root and follows the `<script>` tag from there.

**Static assets** go in `public/` and are served from `/` — `public/logo.png` → `/logo.png`. Files imported from `src/` get hashed and bundled instead.

**Pinned versions in tutorials** (`react@18.3.1`, `vite@5.4.2`) exist so the tutorial's code keeps matching. Unpinned gives you current releases, which is generally what you want when learning.

**Build output** lands in `dist/`. `npm run preview` serves that build locally so you can check it before deploying — it is not a substitute for `npm run dev`.