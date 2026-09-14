# Vite + React Setup

Two paths: scaffold fresh, or wire up an existing folder.

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

**JSX must be in `.jsx`, not `.js`.** Vite's esbuild transform won't parse JSX in `.js` files. The error points at your first `<div>` and reads like a syntax error. Rename the file.

**`index.html` is the entry, not `src/`.** Vite treats the HTML file as the graph root and follows the `<script>` tag from there.

**Static assets** go in `public/` and are served from `/` — `public/logo.png` → `/logo.png`. Files imported from `src/` get hashed and bundled instead.

**Pinned versions in tutorials** (`react@18.3.1`, `vite@5.4.2`) exist so the tutorial's code keeps matching. Unpinned gives you current releases, which is generally what you want when learning.

**Build output** lands in `dist/`. `npm run preview` serves that build locally so you can check it before deploying — it is not a substitute for `npm run dev`.