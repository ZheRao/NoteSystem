

## Updating one field of object state

**Correct**
```jsx
setDraft(prev => ({ ...prev, client_name: event.target.value }))
```
- A state setter **replaces** the whole value; it never merges.
- `...prev` copies all fields, then the later key overrides just one. Order matters.
- Use the updater function `prev => ...` whenever new state depends on old state.
- Wrap the object literal in `( )` so the arrow returns an object, not a function body.
- Never mutate (`draft.client_name = x`). React only re-renders when it sees a new reference.

**My mistake:** `setDraft({ client_name: value })` silently dropped `client_id` after the first keystroke.

## Reading a property of possibly-`null` state at render

**Correct**
```jsx
value={draft?.client_name ?? ''}
```
- If an element is always rendered (e.g. `<dialog>`), its JSX runs even when the state is `null`.
- `?.` stops and returns `undefined` instead of throwing; `?? ''` supplies the fallback.
- Keep inputs' `value` a string, never `undefined`, to avoid the controlled/uncontrolled warning.
- Keep `null` as a meaningful state ("no draft"); make the render tolerate it rather than hiding the meaning.

**My mistake:** `value={draft.client_name}` with `draft` initialised to `null` threw a `TypeError` on the first render, before anything was clicked.

## Reading state right after setting it

**Correct**
```jsx
setClientId(client.id)
console.log(client.id)   // use the value you already have
console.log(clientId)   // old value
```
- A setter **schedules** the next render; it doesn't change the current variable.
- Each render is a snapshot: `clientId` is a `const` of that render, and the handler closes over it.
- Setters in one handler are batched into a single re-render *after* the handler returns.
- To confirm a new value, log in the component body. To react to a change, use `useEffect` with that state as a dependency.

**My mistake:** I logged `clientId` right after `setClientId(client.id)` and got `null`, assuming the setter updated the variable immediately.

## Passing functions as event handlers

**Correct**
```jsx
onSubmit={submitAdd}                              // pass the function itself
onClick={() => dialogRef.current.close()}          // wrap to delay + keep `this`
```
- `fn` is the function; `() => fn()` is a new function that calls `fn` **with no arguments**.
- JSX `{ }` runs **during render**; a wrapper's body runs **at event time**.
- `obj.method()` calls it *on* `obj`; a bare `obj.method` detaches it and loses `this`.
- Ask what the wrapper changes: the arguments, the timing, or the `this`.

**My mistakes:**
- `onSave={() => submitAdd()}` dropped the event, so `event.preventDefault()` threw and the page reloaded.
- `onClick={dialogRef.current.close}` read a `null` ref at render, and would also lose `this` (Illegal invocation).

## "Dialog closing twice"

**Correct understanding**
- The dialog closes **once**; the **exit transition** runs twice: the direct exit → effect calls `.close()` → `close` event → `onClose` → exit again.
- Cause: two authorities (my state *and* the browser's `close` event) both end the workflow.
- Acceptable when the exit is **idempotent**: setting identical values makes React skip the re-render.
- For exactly one exit, make the `close` event the only path to exit (everything closes the dialog; nothing exits directly).

**My mistake:** I assumed the dialog itself was closing twice, when it was the state transition repeating.

Here's the entry, in the same format as the others:

## Brace-less `if` swallowing the next statement

**Correct**
```js
if (myGeneration !== requestGeneration.current) return   // guard has its own body
if (refreshSucceeded) {
  requestSucceed()
} else {
  setRequest('idle')
}
```
- An `if` without braces takes **the next statement** as its body, even when that statement is a whole `if`/`else` block on the following line.
- A comment where the body should be makes the line *look* complete. The parser ignores it.
- A guard clause must end in `return` (or `throw`), or it isn't a guard.
- Prettier exposes it by re-indenting the nesting, and ESLint's `curly` rule prevents it entirely.

**My mistake:** I wrote a generation-check `if` with no `return`, so it became the body wrapper for the success/failure block. In the normal case (generations equal), `requestSucceed()` never ran, `request` stayed `'pending'`, and the Processing dialog reappeared after loading.
