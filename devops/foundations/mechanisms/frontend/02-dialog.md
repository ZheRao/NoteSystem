
# `<dialog>`: the native popup element

## What it is

`<dialog>` is a built-in HTML element (browser layer) for popups. Its value is that the browser does the hard parts for you: putting the popup above everything else, blocking the background, focus handling, and Esc-to-close. Building these by hand with `div`s and `z-index` is notoriously error-prone.

## Core concepts

**Modal vs non-modal.** How you open the dialog decides which kind it is.

| Opened with | Kind | Background | Backdrop | Esc closes |
|---|---|---|---|---|
| `dialog.showModal()` | modal | **inert** (no clicks, no Tab focus) | yes | yes |
| `dialog.show()` or the `open` attribute | non-modal | still interactive | no | no |

For "block everything until the user decides," always use `showModal()`.

**Top layer.** A modal dialog renders in a special browser layer above the entire page. No `z-index` is needed, and no parent's `overflow: hidden` can clip it.

**Inert background.** While a modal is open, everything outside it can't be clicked, focused, or tabbed into. You get this for free.

**`::backdrop`.** A pseudo-element behind a modal dialog, covering the page. It exists only in modal mode, and you style it to dim the page.

## The element's API (on the DOM element)

| Member | Kind | Purpose |
|---|---|---|
| `.showModal()` | method | open as modal. **Throws** if the dialog is already open. |
| `.close(value?)` | method | close it; the optional argument becomes `returnValue` |
| `.open` | boolean property | read it to check the current state; never assign it |
| `.returnValue` | string property | why it closed (see forms below) |
| `close` event | event | fires on **every** close, for any reason |
| `cancel` event | event | fires when Esc is pressed, just before the close |

## Structure

All popup content is ordinary markup inside the tag:

```html
<dialog>
  <form method="dialog">
    <h3>Title</h3>
    <label>Name <input /></label>
    <button value="save">Save</button>
    <button value="cancel">Cancel</button>
  </form>
</dialog>
```

**`<form method="dialog">`** is the key companion. Submitting the form *closes the dialog* instead of reloading the page, and the clicked button's `value` becomes `dialog.returnValue`. Pressing Enter in the input submits using the first button. Esc leaves `returnValue` as `''`.

So `returnValue` tells you the exit path:
- `'save'`: the user confirmed
- `'cancel'`: the user explicitly cancelled
- `''`: the user pressed Esc

## Implementing it in React

**The core tension.** React is declarative (UI = f(state)), but `<dialog>` is imperative: you open and close it by *calling methods*. So React state is the source of truth, and the DOM has to be synced to follow it.

**1. Get a handle with `useRef`** (React layer)

```jsx
const dialogRef = useRef(null)
<dialog ref={dialogRef}> ... </dialog>
```

After rendering, React puts the real element into `dialogRef.current`. You never assign `dialogRef.current` yourself; you only call through it: `dialogRef.current.showModal()`.

**2. Sync DOM to state with `useEffect`** (React layer)

```jsx
useEffect(() => {
  const dialog = dialogRef.current
  if (workflow === 'add') {
    if (!dialog.open) dialog.showModal()   // guard: showModal on open dialog throws
  } else {
    if (dialog.open) dialog.close()        // guard: makes the effect safe to re-run
  }
}, [workflow])
```

The effect runs *after* render, so the element exists, and it re-runs whenever `workflow` changes.

**3. Always render the dialog, never conditionally.** With `{workflow === 'add' && <dialog>}`, the element wouldn't exist yet when you need the ref. A closed dialog is invisible anyway.

**4. Funnel every exit through `onClose`.**

```jsx
<dialog ref={dialogRef} onClose={handleClose}>
```

The browser can close the dialog *behind React's back* (Esc, form submit). Without `onClose`, the dialog closes but `workflow` stays `'add'`. The state then disagrees with the screen, and clicking Add again does nothing, because setting the same value doesn't re-run the effect. Handling `onClose` means every exit path (Esc, Cancel, Save, the effect itself) goes through one transition back to `'view'`. Inside it, read `returnValue` to decide whether to commit the draft.

The full cycle looks like this:

```
click Add → setWorkflow('add') → render → effect → showModal()
user exits (Esc / Cancel / Save) → browser closes → close event
→ onClose → (if 'save': commit) → setWorkflow('view') → effect → already closed, guard skips
```

## Styling hooks (deferred)

- `dialog { ... }` styles the box itself (padding, border, width). The browser centres it by default.
- `dialog::backdrop { background: rgba(0,0,0,.6); }` styles the overlay.
- Closing when the backdrop is clicked isn't default behaviour. The newer `closedby="any"` attribute does it, but support is uneven (tier 3).

## Pitfalls hit in this project

1. **Bare `showModal()`** throws a `ReferenceError`. It must be called on the element.
2. **`dialogRef.current = open` / `= close`** throws *no* error, because `open` and `close` are global browser functions (`window.open`, `window.close`). It silently replaces your element in the ref with a function.
3. **Reading `draft.client_name` while `draft` is `null`** crashes on the first render, because the dialog is always rendered. Use `draft?.client_name ?? ''`.
4. **Forgetting `onClose`** desyncs state from the DOM after Esc.

## Tiers

- **Tier 1 (mechanical):** `showModal()` vs `close()`, the ref holds the element, the effect syncs DOM to state, always render, `onClose` as the single exit.
- **Tier 2 (purpose and failure modes):** top layer, inert background, `returnValue` and `method="dialog"`, why the guards exist.
- **Tier 3 (look up when needed):** `cancel` event details, `closedby`, focus-restoration specifics, `::backdrop` styling.

If you'd like, I can turn this into a doc you can keep alongside the other concept notes as you add them.