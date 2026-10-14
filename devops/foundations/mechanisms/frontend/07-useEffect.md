
# `useEffect`: synchronising with things outside React

## What problem it solves

React's job is to turn state into UI: `UI = f(state)`. Rendering is supposed to be **pure**. It calculates what the screen should look like and does nothing else. But some work has to touch things React doesn't control:

- calling `dialog.showModal()` (a DOM method)
- adding a listener to `document`
- starting a timer, opening a connection, fetching data

These are **side effects**. You can't do them during render, because render can run many times, may be thrown away, and must not change the outside world. `useEffect` gives you a place to run code **after** React has updated the screen, to bring the outside world in line with the current state.

The mental model: **an effect isn't an event ("when X happens, do Y"). It's a synchronisation ("keep the outside world matching this state").**

## Comparing the three hooks

| Hook | What it holds or does | Causes a re-render? |
|---|---|---|
| `useState` | a value that drives the UI | yes, when it's set |
| `useRef` | a box whose `.current` persists across renders (e.g. a DOM element) | no |
| `useEffect` | code that runs **after** render, to sync something outside React | no (but it may call setters) |

`useState` and `useRef` *store* something. `useEffect` *does* something, at a specific time, which is why it feels less straightforward.

## Anatomy

```jsx
useEffect(() => {
  // SETUP: start syncing (runs after render)

  return () => {
    // CLEANUP: stop syncing (runs before the next setup, and on unmount)
  }
}, [dependencies])
```

- **Setup:** the function you pass.
- **Cleanup:** the function your setup *returns* (optional).
- **Dependencies:** the values the setup reads. React re-runs the effect only when one of them has changed since the last render.

## When it runs

**The dependency array decides how often:**

| Written as | Setup runs |
|---|---|
| `useEffect(fn, [a, b])` | after the first render, then after any render where `a` or `b` changed |
| `useEffect(fn, [])` | once, after the first render (mount) |
| `useEffect(fn)` (no array) | after **every** render (rarely what you want) |

React compares each dependency with its previous value using `Object.is`, which for objects and arrays means *same reference*, not same contents.

**Cleanup timing is the confusing part.** Cleanup runs at exactly two moments:

1. **Before the effect re-runs**, to undo the previous setup.
2. **When the component unmounts**, to undo the last setup.

So setup and cleanup always come in pairs, and one setup is never left running alongside another.

## Timeline: the Esc-blocking effect

```jsx
useEffect(() => {
  if (!isPending) return                     // nothing to set up, so no cleanup
  const blockEscape = (e) => { if (e.key === 'Escape') e.preventDefault() }
  document.addEventListener('keydown', blockEscape, true)
  return () => document.removeEventListener('keydown', blockEscape, true)
}, [isPending])
```

```
mount, isPending=false   → setup: returns early, nothing registered, no cleanup
isPending → true         → (no previous cleanup) → setup: listener ADDED, cleanup saved
isPending → false        → CLEANUP runs: listener REMOVED → setup: returns early
unmount                  → cleanup only if a listener is currently registered
```

Read this way, the `return` isn't "returning a result." It's handing React the **undo instructions** for what you just did, so that React can call them at the right moment later.

**Without the cleanup,** every time `isPending` became true you'd add *another* listener and never remove any of them. Escape would stay blocked forever, and the listeners would pile up. That's the most common effect bug: a **leak**.

## Which effects need cleanup

| Setup does | Cleanup must |
|---|---|
| `addEventListener` | `removeEventListener` (the *same* function and the same capture flag) |
| `setInterval` / `setTimeout` | `clearInterval` / `clearTimeout` |
| open a connection or subscription | close or unsubscribe |
| start a fetch | ignore or abort the result (so a late response doesn't update stale state) |
| `dialog.showModal()` driven by state | usually none: the `else` branch handles closing |

The test: **if the component vanished right now, would something keep running or keep listening?** If yes, you need a cleanup.

## The dialog effect: sync without cleanup

```jsx
useEffect(() => {
  const dialog = dialogRef.current
  if (open) { if (!dialog.open) dialog.showModal() }
  else      { if (dialog.open) dialog.close() }
}, [open])
```

This needs no cleanup because it isn't *starting* anything ongoing. Each run fully describes the desired state (open or closed), and the next run corrects it. That's the purest form of "sync the outside world to state."

## Dependencies: list everything the setup reads

If the setup reads a prop or state value, that value belongs in the array. If you leave it out, the effect keeps the **snapshot from the render it last ran in** (a stale closure, as in your functions note). ESLint's `react-hooks/exhaustive-deps` rule warns about this. Treat its warnings as real.

## Development-only double run (StrictMode)

In development, React's `<StrictMode>` deliberately runs **setup → cleanup → setup** once on mount. It's a test: if your cleanup truly undoes your setup, the double run is invisible. If you see something happen twice in dev (two listeners, two requests), it's revealing a missing or incomplete cleanup. It doesn't happen in production builds.

## When *not* to use an effect

- **Deriving a value from state or props:** compute it during render instead. Your `findClientName()` is the right pattern. Storing a derived value in state and syncing it with an effect causes an extra render, and the two copies can drift apart.
- **Responding to a user action:** put the logic in the event handler. "When Save is clicked, validate" belongs in `submitModify`, not in an effect watching state.
- **The rule of thumb:** effects are for syncing with **external systems**. If no external system is involved, you probably don't need one.

## Pitfalls hit in this project

1. **The effect's scope surprised me.** An effect in ModifyClientModal ran during Delete, because the component is always mounted and `isPending` is shared. Effects follow the component's lifetime and their dependencies, not visibility. A listener on `document` acts on the whole page.
2. **The keydown handler was on the dialog, but focus had fallen to `<body>`.** Disabling the focused element moved focus outside the dialog, so the handler never saw the key. Document-level listening (capture phase) fixed it.

## Tiers

- **Tier 1 (mechanical):** setup runs after render; the dependency array controls re-runs; cleanup runs before the next setup and on unmount; listeners and timers need cleanup; `removeEventListener` needs the same function.
- **Tier 2 (purpose and failure modes):** effects are synchronisation, not events; leaks from missing cleanup; stale values from missing dependencies; StrictMode's double run as a cleanup test; don't use effects for derived values or user actions.
- **Tier 3 (look up when needed):** `useLayoutEffect` (runs before paint), `AbortController` for fetches, custom hooks that wrap effects (e.g. a reusable `useModalDialog` for your three modals).