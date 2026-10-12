# 6. `AddClientForm.jsx`


## Lesson 1 - state vs. derived value & calling async function

Yes. `AddClientForm.jsx` is a great next file because almost everything in it is something we've already learned. Rather than introducing many new React mechanisms, it **combines them into a clean form workflow**.

The whole component is only 38 lines:

```text
AddClientForm
│
├── props
│     disabled
│     onSubmit
│
├── local state
│     name
│     busy
│
├── derived value
│     trimmed
│
├── action
│     handleSubmit()
│
└── JSX
      controlled input
      submit button
```

:chatgpt-content-reference{index="0"}

There's one concept I'd particularly like to emphasize here that we haven't isolated before: the distinction between **state** and a **derived value**.

### 1. Start with the component contract

```js
export default function AddClientForm({ disabled, onSubmit }) {
```

:chatgpt-content-reference{index="1"}

Two props come from the parent:

```text
disabled
    → whether the parent currently allows creation

onSubmit
    → callback through which this component
      requests creation of a client
```

Already notice what isn't passed in:

```text
name
busy
```

Those belong specifically to this form's interaction, so they're owned locally.

Compare that with `ClientRow`:

```text
PARENT-CONTROLLED
client
disabled
onRename
onDelete

LOCAL
mode
draft
busy
```

Here:

```text
PARENT-CONTROLLED
disabled
onSubmit

LOCAL
name
busy
```

Same design philosophy.

---

### 2. Two pieces of state

```js
const [name, setName] = useState('')
const [busy, setBusy] = useState(false)
```

:chatgpt-content-reference{index="2"}

These should now be quite readable.

Initially:

```text
name = ""
busy = false
```

`name` is the current text in the input.

`busy` says whether an asynchronous create operation is underway.

So if the user starts typing:

```text
""
 ↓
"A"
 ↓
"AB"
 ↓
"ABC"
 ↓
"ABC Farms"
```

that's:

```js
setName(...)
```

And when they submit:

```text
busy

false
  ↓
true
  ↓
await server operation
  ↓
false
```

This is nearly identical to `draft` and `busy` in `ClientRow`.

---

### 3. But then there's a third value

```js
const trimmed = name.trim()
```

:chatgpt-content-reference{index="3"}

Notice:

```js
const trimmed = ...
```

not:

```js
const [trimmed, setTrimmed] = useState(...)
```

Why?

Because `trimmed` is completely determined by `name`.

If:

```text
name = "   ABC Farms   "
```

then necessarily:

```text
trimmed = "ABC Farms"
```

There's no independent information here.

That's what I mean by a **derived value**.

---

### 4. State versus derived value

Suppose Claude instead wrote:

```js
const [name, setName] = useState('')
const [trimmed, setTrimmed] = useState('')
```

Now every time `name` changes, we'd somehow need to remember:

```text
name changed
     ↓
update trimmed too
```

And suddenly invalid combinations become possible:

```text
name = "ABC Farms"
trimmed = "XYZ Farms"
```

What does that mean?

Those two values shouldn't be capable of disagreeing because one is mathematically/functionally determined by the other:

```js
trimmed = name.trim()
```

So don't create another source of truth.

Instead:

```text
SOURCE OF TRUTH

name
 │
 │ .trim()
 ↓
DERIVED VALUE

trimmed
```

Every render simply calculates it again.

---

### 5. This is a very useful React principle

A useful default is:

> If a value can be cheaply calculated entirely from existing props/state during rendering, you often don't need separate state for it.

For this component:

```text
STATE
─────────────────────
name
busy


DERIVED
─────────────────────
trimmed = name.trim()
```

This reduces the number of independent states the program has to keep synchronized.

Given your database background, it's somewhat analogous to avoiding redundant stored information when the value can be reliably derived from canonical information—though React state and database normalization are different subjects.

---

### 6. And remember: the entire component function runs again on render

This is where our React execution model matters.

Suppose initially:

```text
name = ""
```

Component executes:

```js
const trimmed = name.trim()
```

therefore:

```text
trimmed = ""
```

User types:

```text
A
```

which eventually calls:

```js
setName("A")
```

React schedules a re-render.

The component executes again from the top:

```js
const [name, setName] = useState('')
```

React gives us the existing state:

```text
name = "A"
```

Then this line executes again:

```js
const trimmed = name.trim()
```

Now:

```text
trimmed = "A"
```

So derived values naturally stay synchronized because they're recalculated during rendering:

```text
name changes
    ↓
render again
    ↓
trimmed recalculated
    ↓
JSX sees new trimmed
```

No effect required.

No second setter required.

---

### 7. Now `handleSubmit()`

```js
async function handleSubmit(event) {
  event.preventDefault()
  if (!trimmed || disabled || busy) return

  setBusy(true)
  try {
    const ok = await onSubmit(trimmed)
    if (ok) setName('')
  } finally {
    setBusy(false)
  }
}
```

:chatgpt-content-reference{index="4"}

This should look extremely familiar after `ClientRow`.

The structure is almost identical to `commitEdit()`:

```text
prevent browser default
        ↓
guard conditions
        ↓
set busy
        ↓
await parent callback
        ↓
respond to result
        ↓
finally clear busy
```

Let's still walk it through because there are some nice details.

---

### 8. `event.preventDefault()`

```js
event.preventDefault()
```

Same as `ClientRow`.

Later we see:

```jsx
<form className="add-form" onSubmit={handleSubmit}>
```

:chatgpt-content-reference{index="5"}

So:

```text
user submits form
      ↓
submit event
      ↓
handleSubmit(event)
      ↓
preventDefault()
```

The browser's normal form behavior is prevented because the application handles creation through JavaScript/React.

Nothing new here.

---

### 9. But look at this guard

```js
if (!trimmed || disabled || busy) return
```

:chatgpt-content-reference{index="6"}

Three invalid situations:

```text
!trimmed
    → no meaningful client name

disabled
    → parent says creation isn't currently allowed

busy
    → a create request is already underway
```

The logical OR:

```js
A || B || C
```

means:

> If any of these is truthy.

So the operation proceeds only when effectively:

```text
meaningful name
AND
not disabled
AND
not already busy
```

---

### 10. Why check `disabled` and `busy` here if the button is disabled anyway?

Excellent defensive design detail.

Later:

```jsx
<button
  type="submit"
  disabled={disabled || busy || !trimmed}
>
```

:chatgpt-content-reference{index="7"}

So ordinarily the UI prevents submission when any condition is invalid.

Why repeat it in:

```js
if (!trimmed || disabled || busy) return
```

Because the **action function should defend its own preconditions** rather than assuming that every possible path to it has been perfectly prevented by UI presentation.

So we have:

```text
UI LAYER

disabled={...}
    ↓
don't offer invalid action normally


HANDLER LAYER

if (...) return
    ↓
don't execute invalid action
even if handler gets invoked
```

Very similar to what we saw in `ClientRow`.

And again, none of this replaces backend validation.

---

### 11. Start asynchronous operation

```js
setBusy(true)
```

Now:

```text
busy = true
```

React will re-render.

What changes?

Two things later depend on `busy`:

```jsx
disabled={disabled || busy}
```

and:

```jsx
{busy ? 'Adding…' : 'Add client'}
```

So the UI immediately transitions from something like:

```text
[ New client name       ] [ Add client ]
```

to:

```text
[ New client name       ] [ Adding… ]
       disabled             disabled
```

while the Promise is pending.

Again:

```text
ASYNC PROGRAM STATE
busy = true

       ↓

DECLARATIVE UI
disable controls
show "Adding…"
```

---

### 12. Then delegate actual creation upward

```js
const ok = await onSubmit(trimmed)
```

:chatgpt-content-reference{index="8"}

This component doesn't know:

```text
POST /client

client_name

modified_by

JSON.stringify

fetch

ApiError
```

It only knows:

> I have a valid client name. Ask my parent to submit it.

Suppose:

```text
name = "  ABC Farms  "
```

Then:

```text
trimmed = "ABC Farms"
```

and:

```js
onSubmit(trimmed)
```

becomes:

```js
onSubmit("ABC Farms")
```

The parent callback returns a Promise.

`await` waits for its result.

Same abstraction boundary as before.

---

### 13. Successful creation has a nice consequence

```js
if (ok) setName('')
```

If the parent reports success:

```text
ok = true
```

the form clears its input.

Why does:

```js
setName('')
```

clear the visible textbox?

Because the textbox is controlled.

Later:

```jsx
value={name}
```

So:

```text
setName("")
    ↓
state changes
    ↓
re-render
    ↓
name = ""
    ↓
value={name}
    ↓
input displays ""
```

There's no:

```js
inputElement.value = ''
```

required.

The state changes; the UI follows.

That's the controlled-input model paying off again.

---

### 14. What if creation fails?

Suppose:

```js
const ok = await onSubmit(trimmed)
```

eventually gives:

```text
ok = false
```

Then:

```js
if (ok) setName('')
```

doesn't execute.

Therefore the user's typed name remains.

That's important UX behavior.

Imagine they typed:

```text
North Valley Agricultural Holdings
```

and the request failed.

You probably don't want to erase their input and make them type it again.

So:

```text
SUCCESS
    ↓
clear form


FAILURE
    ↓
preserve form contents
```

That's a sensible state transition.

---

### 15. And once again `finally`

```js
finally {
  setBusy(false)
}
```

:chatgpt-content-reference{index="9"}

Exactly the invariant we learned in `ClientRow`:

```text
operation begins
      ↓
busy = true
      ↓
whatever happens
      ↓
busy must eventually = false
```

Success:

```text
await resolves
 ↓
maybe clear name
 ↓
finally
 ↓
busy false
```

Failure by exception:

```text
await throws
 ↓
finally STILL executes
 ↓
busy false
 ↓
error continues outward
```

Again, there's no `catch` here, so this component itself isn't responsible for turning thrown errors into error messages.

That responsibility presumably lives above it. We'll verify later.

---

### 16. Now the JSX is almost entirely familiar

```jsx
<form className="add-form" onSubmit={handleSubmit}>
  <input
    type="text"
    placeholder="New client name"
    value={name}
    onChange={(e) => setName(e.target.value)}
    disabled={disabled || busy}
    autoComplete="off"
  />
```

:chatgpt-content-reference{index="10"}

Controlled input:

```text
name state
   ↓
value={name}
   ↓
<input>
   ↓
user types
   ↓
onChange
   ↓
e.target.value
   ↓
setName(...)
   ↓
render
```

At this point, this pattern should hopefully be starting to look almost boring—and that's good.

---

### 17. One keystroke also changes `trimmed`

There's a little extra consequence here.

Suppose:

```text
name = ""
trimmed = ""
```

User types:

```text
A
```

Handler:

```js
setName("A")
```

Then re-render:

```text
name = "A"
      ↓
const trimmed = name.trim()
      ↓
trimmed = "A"
```

Now look at the button:

```jsx
disabled={disabled || busy || !trimmed}
```

If:

```text
disabled = false
busy = false
trimmed = "A"
```

then:

```text
false || false || false
          ↓
        false
```

So the Add button becomes enabled.

Therefore one state update affects multiple pieces of UI:

```text
setName("A")
    ↓
render
    │
    ├── input value becomes "A"
    │
    └── trimmed becomes "A"
              ↓
        button becomes enabled
```

This is an important React idea:

> You don't manually coordinate all affected UI elements. They independently derive their appearance from the same state.

---

### 18. Whitespace is especially illustrative

Suppose user types:

```text
"      "
```

Then:

```text
name = "      "
```

The input **does display those spaces**, because:

```jsx
value={name}
```

uses the raw state.

But:

```js
trimmed = name.trim()
```

produces:

```text
""
```

Therefore:

```jsx
disabled={disabled || busy || !trimmed}
```

becomes true.

So we have two representations for two different purposes:

```text
RAW INPUT STATE

name = "      "
    ↓
preserve exactly what user typed
    ↓
input display


SEMANTIC DERIVATION

trimmed = ""
    ↓
is this a meaningful client name?
    ↓
No → disable submission
```

That's very similar to `useCurrentUser()` returning both raw and trimmed representations.

---

### 19. Why isn't `disabled` combined into `busy`?

We again have two concepts:

```text
disabled
    → external policy from parent

busy
    → this form's local async state
```

Then the component combines them when deciding what UI behavior is allowed:

```jsx
disabled={disabled || busy}
```

That's good separation.

Imagine:

```text
disabled = true
busy = false
```

The form isn't doing anything itself, but its parent says:

> You may not create clients right now.

Versus:

```text
disabled = false
busy = true
```

Parent allows creation generally, but:

> This form already has a create operation underway.

Different causes, same UI consequence:

```text
disable control
```

So:

```js
disabled || busy
```

combines them only at the point where their consequences overlap.

---

### 20. The submit button

```jsx
<button type="submit" disabled={disabled || busy || !trimmed}>
  {busy ? 'Adding…' : 'Add client'}
</button>
```

:chatgpt-content-reference{index="11"}

We now have two separate derived UI decisions.

#### Can the user submit?

```js
disabled || busy || !trimmed
```

#### What should the button say?

```js
busy ? 'Adding…' : 'Add client'
```

So the state matrix is roughly:

```text
disabled=false
busy=false
trimmed="ABC"
────────────────────
button enabled
"Add client"


disabled=false
busy=false
trimmed=""
────────────────────
button disabled
"Add client"


disabled=true
busy=false
────────────────────
button disabled
"Add client"


busy=true
────────────────────
button disabled
"Adding…"
```

Again, UI = function of current state/props.

---

### 21. Let's execute the entire component once

Suppose initially:

```text
disabled = false
name = ""
busy = false
```

Render computes:

```text
trimmed = ""
```

UI:

```text
[ New client name             ] [ Add client ]
                                 ^ disabled
```

User types:

```text
ABC Farms
```

Each keystroke updates `name`.

Eventually:

```text
name = "ABC Farms"
trimmed = "ABC Farms"
busy = false
```

UI:

```text
[ ABC Farms                   ] [ Add client ]
                                 ^ enabled
```

User submits.

```text
handleSubmit(event)
      ↓
preventDefault()
      ↓
guards pass
      ↓
setBusy(true)
```

Re-render:

```text
name = "ABC Farms"
trimmed = "ABC Farms"
busy = true
```

UI:

```text
[ ABC Farms                   ] [ Adding… ]
   disabled                      disabled
```

Then:

```js
await onSubmit("ABC Farms")
```

Suppose it succeeds:

```text
ok = true
```

Then:

```js
setName('')
```

and finally:

```js
setBusy(false)
```

Resulting state:

```text
name = ""
busy = false
trimmed = ""
```

UI:

```text
[ New client name             ] [ Add client ]
                                 ^ disabled
```

That's the complete lifecycle.

---

### 22. Notice what this component does **not** know

This is becoming a recurring architectural theme.

`AddClientForm` doesn't know:

```text
what current user is
what modified_by should be

what POST endpoint exists

what JSON field FastAPI expects

how fetch works

what HTTP 409 means

how errors are displayed

where the client list is stored

how the table gets refreshed
```

Its entire domain contract is essentially:

```text
Give me permission state:

    disabled


Give me a function:

    onSubmit(clientName)
        → Promise<boolean>


I will handle:

    text entry
    whitespace normalization
    submit UX
    busy state
    clearing on success
```

That's a very narrow component.

---

### 23. Compare `AddClientForm` and `ClientRow`

This comparison is useful because we're beginning to see reusable patterns rather than isolated syntax.

```text
ADD CLIENT                       RENAME CLIENT
──────────────────               ──────────────────

name state                       draft state

controlled input                 controlled input

trim()                           trim()

form                             form

preventDefault()                 preventDefault()

busy state                       busy state

await onSubmit(name)             await onRename(name)

success:
clear input                      success:
                                 leave edit mode

failure:
preserve input                   failure:
                                 preserve edit mode

finally:
busy false                       finally:
                                 busy false
```

So these aren't two completely different React problems.

They're instances of the same broader pattern:

```text
LOCAL EDITABLE STATE
       ↓
VALIDATE / NORMALIZE
       ↓
ENTER BUSY STATE
       ↓
DELEGATE DOMAIN OPERATION
       ↓
AWAIT RESULT
       ↓
UPDATE LOCAL UI ON SUCCESS
       ↓
ALWAYS EXIT BUSY STATE
```

That's a pattern worth recognizing.

---

### 24. And now our component hierarchy is almost complete

We've studied:

```text
                 ??? App ???
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
      UserBar   AddClientForm  ClientTable
                                  │
                           clients.map(...)
                                  │
                    ┌─────────────┼─────────────┐
                    ↓             ↓             ↓
                ClientRow     ClientRow     ClientRow
```

The lower-level components now make sense individually.

More importantly, we know the **interfaces they're demanding from whoever sits above them**.

`UserBar` expects approximately:

```text
value
onChange(value)
```

`AddClientForm` expects:

```text
disabled
onSubmit(name) → Promise<boolean>
```

`ClientTable` expects:

```text
clients[]
disabled
onRename(client, newName)
onDelete(client)
```

And `client.js` offers:

```text
listClients()
createClient(...)
updateClient(...)
deleteClient(...)
```

So you can almost see the missing puzzle piece already.

Something has to connect:

```text
UI CALLBACK CONTRACTS
────────────────────────
onSubmit(name)
onRename(client, name)
onDelete(client)


            ↕


API FUNCTIONS
────────────────────────
createClient(...)
updateClient(...)
deleteClient(...)
```

Something also has to own:

```text
clients[]
```

Something has to initially call:

```js
listClients()
```

Something has to own/display API errors.

Something has to combine the current user from:

```js
useCurrentUser()
```

with:

```text
client name
client ID
```

to produce the arguments expected by `client.js`.

And something presumably determines the mysterious:

```text
disabled
```

prop that keeps flowing downward.

That missing layer is almost certainly `App.jsx`.

So we've now earned the ability to study `App.jsx` differently than if we'd opened it first. Instead of seeing a pile of hooks, effects, handlers, and JSX, we should be able to ask of every line:

> **Which lower-level contract is this line satisfying?**

And that is where I think the architecture of the entire frontend is finally going to snap together.


## Code Appendix

```js
import { useState } from 'react'

export default function AddClientForm({ disabled, onSubmit }) {
  const [name, setName] = useState('')
  const [busy, setBusy] = useState(false)

  const trimmed = name.trim()

  async function handleSubmit(event) {
    event.preventDefault()
    if (!trimmed || disabled || busy) return

    setBusy(true)
    try {
      const ok = await onSubmit(trimmed)
      if (ok) setName('')
    } finally {
      setBusy(false)
    }
  }

  return (
    <form className="add-form" onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="New client name"
        value={name}
        onChange={(e) => setName(e.target.value)}
        disabled={disabled || busy}
        autoComplete="off"
      />
      <button type="submit" disabled={disabled || busy || !trimmed}>
        {busy ? 'Adding…' : 'Add client'}
      </button>
    </form>
  )
}

```