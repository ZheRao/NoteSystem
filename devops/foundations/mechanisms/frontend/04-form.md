# `<form>`: grouping inputs into one submit action

## What it is

`<form>` is a built-in HTML element (browser layer) that groups inputs and buttons into a single "submit" action. Its default behaviour comes from the pre-JavaScript web: **submitting a form means sending its data to a URL and navigating to the response**, which looks like a full page reload. In a React app, that default is almost never what you want, and most form bugs come from it firing unexpectedly.

## Core concept: the submit event

A form has one central moment, **submit**, and several different user actions trigger it:

| Trigger | Notes |
|---|---|
| Clicking a `<button type="submit">` | the explicit path |
| Clicking a `<button>` with **no `type`** inside a form | a button's default type is `submit` |
| Pressing **Enter** in a text input | implicit submission, even if you never wired a button |
| `form.requestSubmit()` in code | the programmatic equivalent of a click (tier 3) |

All of them converge on one thing: the form fires a `submit` event, and then the browser runs the **default action** for the form's `method`.

## `method` decides the default action

| `method` | Default action after submit |
|---|---|
| `get` (the default) | navigate to the `action` URL (the current page if omitted) with the data in the query string, which reloads the page |
| `post` | send the data in a request body and navigate to the response, which also reloads |
| `dialog` | **no navigation**; closes the enclosing `<dialog>` and sets its `returnValue` to the clicked button's `value` |

So `<form>` with nothing specified means "GET to this same page," which shows up as a reload. React state is wiped, and the console clears.

## Button types (inside a form)

| `type` | Behaviour |
|---|---|
| `submit` (the **default**) | triggers submit |
| `button` | does nothing on its own; only your `onClick` runs |
| `reset` | resets inputs to their initial HTML values. Rarely useful with React-controlled inputs. |

Rule: every `<button>` inside a form should have an explicit `type`, so you're never relying on the default.

## Implementing it in React

**1. Handle the form, not the button.** Put the logic on the form's `onSubmit`, so every trigger (button click, Enter key) goes through one handler:

```jsx
<form onSubmit={submitAdd}>
  ...
  <button type="submit">Save</button>
  <button type="button" onClick={exitAdd}>Cancel</button>
</form>
```

**2. Cancel the default first.**

```jsx
const submitAdd = (event) => {
  event.preventDefault()   // stop the browser's navigation/reload
  // validate, commit draft, then transition state
  exitAdd()
}
```

`event.preventDefault()` (browser layer) cancels the default action for *this* event. Without it, your handler runs, and then the page reloads underneath it.

**3. Inputs stay controlled.** `value` comes from state, and `onChange` updates state. The form doesn't own the data; React state does. The form only provides the submit moment.

## Combining with `<dialog>`

There are two valid patterns. Pick one per dialog, and don't mix them.

| Pattern | Who closes the dialog | Who decides save vs cancel |
|---|---|---|
| `method="dialog"` + `onClose` reads `returnValue` | the **browser**, on submit | `returnValue` (`'save'` / `'cancel'` / `''`) |
| `onSubmit` + `preventDefault()` | **your state**: `exitAdd()` → `workflow = 'view'` → the effect calls `.close()` | the handler itself (submit = save; Cancel's `onClick` = cancel) |

Note: `preventDefault()` also cancels the `method="dialog"` close. In the second pattern the dialog only closes because your state transition tells the effect to close it. Keep `onClose={exitAdd}` anyway, since Esc still closes the dialog natively and needs to reach state.

## Native validation (useful soon)

Attributes like `required`, `minLength`, and `pattern` on inputs make the browser **block submit** and show a message when the input is invalid. The `submit` event never fires, so your handler never runs. That's a zero-code answer to the "Save with an empty name" question:

```jsx
<input required ... />
```

Tier 2: know that it exists and that it blocks submit. The message styling and the details are tier 3.

## Pitfalls hit in this project

1. **`<form onSubmit={onSave}>` with no `preventDefault()`.** Pressing Enter in the input submitted the form, and the page reloaded. This happened even though Save was `type="button"`, because Enter triggers submit regardless of the buttons.
2. **Save wired twice** (the form's `onSubmit` *and* the button's `onClick`). That's two entry points for one action, so the paths can drift apart. Use one path: `type="submit"` plus the form's `onSubmit`.

## Tiers

- **Tier 1 (mechanical):** submit is one event with many triggers (including Enter); a button's default type is `submit`; `onSubmit` on the form plus `event.preventDefault()`; explicit `type` on every button.
- **Tier 2 (purpose and failure modes):** `method` decides the default action (`get`/`post` navigate, `dialog` closes); the two dialog patterns and why you don't mix them; native validation blocks submit.
- **Tier 3 (look up when needed):** `action`, `FormData` and `name` attributes, `requestSubmit()`, `type="reset"`, custom validation messages.

# common bug

## The same bug is hiding in your submit handlers

```jsx
const submitModify = (event) => {
  if (request !== 'idle' || error !== null) return   // ← returns BEFORE preventDefault
  event.preventDefault()
  ...
}
```

If that guard ever fires, the handler returns without cancelling the default, and the form reloads the page. That's the same reload, triggered from a different place. Right now the guard is mostly unreachable, because the inputs and buttons are disabled while pending. But "mostly unreachable" is exactly where reload bugs hide. For example, pressing Enter in the input while the error dialog is open could hit it.

The rule: **`event.preventDefault()` is always the first line of a submit handler**, before any guard or early return. Apply it to `confirmDelete` too. It's worth adding as a bullet to the pitfalls section of your form note.