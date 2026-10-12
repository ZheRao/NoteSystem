# 5. `ClientTable.jsx`


## Lesson 1 - collection rendering & component composition & closure & `key`

Yes. `ClientTable.jsx` is much smaller than `ClientRow.jsx`, but architecturally it's important because we move **one level upward in the component tree**.

With `ClientRow`, we understood one client. Now `ClientTable` answers:

> Given an **array of clients**, how do we create one `ClientRow` for each client and wire each row's callbacks back to the correct client?

The entire component is only 29 lines. :chatgpt-content-reference{index="0"}

Its architecture is basically:

```text
ClientTable
│
│ receives clients[]
│
├── table header
│
└── clients.map(...)
      │
      ├── ClientRow(client A)
      ├── ClientRow(client B)
      ├── ClientRow(client C)
      └── ...
```

And there's one genuinely new React concept here—`key`—plus a very useful callback transformation:

```js
onRename={(newName) => onRename(client, newName)}
```

Let's build toward those slowly.

### 1. First line: importing another component

```js
import ClientRow from './ClientRow.jsx'
```

:chatgpt-content-reference{index="1"}

We just studied:

```js
export default function ClientRow(...) {
```

Now we're seeing the other side of that default export.

Remember our distinction:

```text
ClientRow.jsx
────────────────────────────────

export default function ClientRow(...) {
    ...
}


ClientTable.jsx
────────────────────────────────

import ClientRow from './ClientRow.jsx'
```

This is ordinary JavaScript module machinery.

But architecturally something important has happened.

`ClientTable` can now use `ClientRow` as part of its own JSX:

```jsx
<ClientRow ... />
```

So we're seeing **component composition** concretely for the first time.

---

### 2. Component tree versus DOM tree

This distinction is worth establishing now.

In our React source code, we can think in terms of a component tree:

```text
ClientTable
     │
     ├── ClientRow
     ├── ClientRow
     └── ClientRow
```

But those aren't necessarily literal browser DOM elements named `<ClientTable>` and `<ClientRow>`.

Remember:

```jsx
<ClientRow ... />
```

means:

> Use the JavaScript component `ClientRow` to determine what UI belongs here.

And we already know `ClientRow` ultimately returns:

```jsx
<tr>...</tr>
```

So conceptually:

```text
REACT COMPONENT TREE

ClientTable
   │
   ├── ClientRow
   ├── ClientRow
   └── ClientRow


eventually produces


BROWSER DOM

<table>
   │
   ├── <thead>...
   │
   └── <tbody>
          ├── <tr>...</tr>
          ├── <tr>...</tr>
          └── <tr>...</tr>
```

This distinction becomes increasingly important as React applications get larger.

---

### 3. What does `ClientTable` receive?

```js
export default function ClientTable({
    clients,
    disabled,
    onRename,
    onDelete
}) {
```

The actual source puts this on one line. :chatgpt-content-reference{index="2"}

Again, ordinary object destructuring of props.

There are four inputs:

```text
clients
    → collection of client objects

disabled
    → parent-controlled action policy

onRename
    → callback for renaming a client

onDelete
    → callback for deleting a client
```

Notice how this differs slightly from `ClientRow`.

`ClientRow` received:

```text
one client
```

whereas `ClientTable` receives:

```text
many clients
```

That's a clue about the responsibility boundary:

```text
ClientTable
    → collection-level presentation

ClientRow
    → individual-client presentation
```

---

### 4. Something else is missing: there is no state

Look at the entire component.

There is no:

```js
useState
```

No:

```js
useEffect
```

No:

```js
useRef
```

`ClientTable` doesn't own any local state.

It is essentially:

```text
props
  ↓
render transformation
  ↓
JSX
```

That's perfectly normal.

Not every React component needs hooks.

In fact, this component is very close to our simplest model:

> Component = function from props to UI description.

---

### 5. The outer table is ordinary JSX

```jsx
<table className="client-table">
  <thead>
    <tr>
      <th className="col-id">ID</th>
      <th className="col-name">Client name</th>
      <th className="col-meta">Last modified</th>
      <th className="col-meta">Modified by</th>
      <th className="col-actions">Actions</th>
    </tr>
  </thead>
```

:chatgpt-content-reference{index="3"}

There's almost nothing React-specific here.

It's describing an HTML table header:

```text
┌────┬─────────────┬───────────────┬─────────────┬─────────┐
│ ID │ Client name │ Last modified │ Modified by │ Actions │
└────┴─────────────┴───────────────┴─────────────┴─────────┘
```

The interesting part begins inside:

```jsx
<tbody>
```

because that's where the number of rows depends on application data.

---

### 6. `clients.map(...)`

Here's the heart of the file:

```jsx
<tbody>
  {clients.map((client) => (
    <ClientRow
      ...
    />
  ))}
</tbody>
```

:chatgpt-content-reference{index="4"}

We've already encountered `.map()` in `client.js`.

And this is why I said it was worth learning as **JavaScript first**, rather than treating it as some React feature.

`.map()` is ordinary JavaScript array functionality.

Suppose:

```js
const numbers = [1, 2, 3]
```

Then:

```js
numbers.map((number) => number * 10)
```

produces:

```js
[10, 20, 30]
```

The conceptual operation is:

```text
INPUT ARRAY

[1, 2, 3]

     ↓ map(transform)

1 → 10
2 → 20
3 → 30

     ↓

OUTPUT ARRAY

[10, 20, 30]
```

`.map()` means:

> Take every element in this array, transform it using this function, and produce a new array containing the transformed results.

---

### 7. Now apply exactly that to clients

Suppose `clients` contains:

```js
[
  {
    client_id: 1,
    client_name: "Alpha Farms"
  },
  {
    client_id: 2,
    client_name: "Beta Farms"
  },
  {
    client_id: 3,
    client_name: "Gamma Farms"
  }
]
```

Now:

```js
clients.map((client) => ...)
```

means conceptually:

```text
client #1
   ↓ transform

client #2
   ↓ transform

client #3
   ↓ transform
```

But what are we transforming each client **into**?

JSX:

```jsx
<ClientRow ... />
```

So:

```text
ARRAY OF DATA

[
  client 1,
  client 2,
  client 3
]

       ↓ .map()

ARRAY OF UI DESCRIPTIONS

[
  <ClientRow client={client1} ... />,
  <ClientRow client={client2} ... />,
  <ClientRow client={client3} ... />
]
```

That's one of the most fundamental React collection patterns:

> **Data array → `.map()` → array of components/elements.**

---

### 8. JSX can render that array

Remember the braces:

```jsx
{clients.map(...)}
```

mean:

> Evaluate this JavaScript expression and put its result here.

The expression returns an array of React elements.

React knows how to render that collection.

So:

```jsx
<tbody>
  {clients.map(...)}
</tbody>
```

means roughly:

```text
<tbody>

    whatever UI results from
    transforming every client

</tbody>
```

If:

```js
clients = []
```

then:

```js
clients.map(...)
```

produces:

```js
[]
```

and there simply aren't any rows inside `<tbody>`.

If there are 100 clients, `.map()` produces descriptions for 100 `ClientRow`s.

The code itself doesn't need:

```text
if 1 client...
if 2 clients...
if 3 clients...
```

The collection determines the rendered structure.

---

### 9. Let's inspect one iteration

Suppose the current element is:

```js
client = {
    client_id: 42,
    client_name: "ABC Farms",
    modified_by: "Zhe",
    last_modified: "2026-09-17 19:30:00"
}
```

Then the callback:

```js
(client) => (
    <ClientRow ... />
)
```

returns:

```jsx
<ClientRow
  key={client.client_id}
  client={client}
  disabled={disabled}
  onRename={(newName) => onRename(client, newName)}
  onDelete={() => onDelete(client)}
/>
```

:chatgpt-content-reference{index="5"}

So we're creating/configuring one `ClientRow`.

Now let's inspect every piece of that contract.

---

### 10. `client={client}`

This is perhaps the simplest:

```jsx
client={client}
```

The left side:

```text
client=
```

is the prop name that `ClientRow` receives.

The right side:

```js
{client}
```

is the current JavaScript variable from `.map()`.

So:

```text
current client object
       ↓
<ClientRow client={client} />
       ↓
ClientRow receives:
       ↓
function ClientRow({ client, ... })
```

This is the concrete parent → child prop flow we've been talking about.

---

### 11. `disabled={disabled}`

Same idea:

```jsx
disabled={disabled}
```

The `ClientTable` itself received `disabled` from **its parent**.

Now it passes that same value further down.

So we have:

```text
higher parent
     │
     │ disabled
     ↓
ClientTable
     │
     │ disabled
     ↓
ClientRow
     │
     ↓
Rename/Delete buttons
```

This pattern is sometimes described as passing or "drilling" props down through the component tree.

`ClientTable` doesn't necessarily need to reinterpret this value.

It's acting as an intermediary.

And now we understand the `disabled` variable we deliberately didn't speculate about while studying `ClientRow`: `ClientTable` doesn't define its meaning either. It receives it from still higher up. :chatgpt-content-reference{index="6"}

We'll presumably finally discover its origin when we reach `App.jsx`.

---

### 12. Now the most interesting line

```jsx
onRename={(newName) => onRename(client, newName)}
```

:chatgpt-content-reference{index="7"}

This line is worth spending real time on.

At first glance you might ask:

> Why not just write `onRename={onRename}`?

Because `ClientRow` and `ClientTable` operate at **different abstraction levels**.

Let's inspect their contracts.

`ClientTable` apparently has an `onRename` callback that expects something like:

```js
onRename(client, newName)
```

It needs to know:

```text
WHICH client?
+
WHAT new name?
```

But remember what `ClientRow` does:

```js
const ok = await onRename(trimmed)
```

`ClientRow` only supplies:

```text
WHAT new name?
```

Why doesn't `ClientRow` need to supply the client?

Because this particular row already represents exactly one client.

---

### 13. `ClientTable` binds the client into the callback

This:

```js
(newName) => onRename(client, newName)
```

creates a new function.

Suppose the current `.map()` iteration has:

```js
client = {
    client_id: 42,
    client_name: "ABC Farms"
}
```

Then conceptually the function is:

```js
function (newName) {
    return onRename(client42, newName)
}
```

That function is passed into this particular `ClientRow`.

So from the row's perspective, its interface is beautifully simple:

```js
onRename("XYZ Farms")
```

But internally the wrapper converts that into:

```js
onRename(client42, "XYZ Farms")
```

That means the row doesn't need to repeatedly say:

```js
onRename(client, trimmed)
```

Its parent has already **bound the identity of the row** into the callback.

---

### 14. Think of each row getting its own customized function

Suppose:

```js
clients = [
    alpha,
    beta,
    gamma
]
```

The `.map()` effectively constructs:

```text
ClientRow for Alpha
    │
    └── onRename(newName)
            ↓
        parentOnRename(alpha, newName)


ClientRow for Beta
    │
    └── onRename(newName)
            ↓
        parentOnRename(beta, newName)


ClientRow for Gamma
    │
    └── onRename(newName)
            ↓
        parentOnRename(gamma, newName)
```

Each row receives a function with the appropriate `client` captured.

This is JavaScript **closure** behavior.

---

### 15. Let's introduce "closure" carefully

Consider:

```js
function makeGreeter(name) {
    return () => console.log(name)
}
```

Then:

```js
const greetZhe = makeGreeter("Zhe")
const greetAlice = makeGreeter("Alice")
```

Even after `makeGreeter()` finishes, the returned functions retain access to the `name` associated with the execution that created them:

```js
greetZhe()    // uses "Zhe"
greetAlice()  // uses "Alice"
```

That's the key idea of a closure:

> A function can retain access to variables from the surrounding lexical environment in which it was created.

Back to:

```js
clients.map((client) => (
```

Inside each iteration, we create:

```js
(newName) => onRename(client, newName)
```

That function captures the relevant `client`.

So when the user clicks Save much later:

```text
ClientRow #42
      ↓
onRename("XYZ Farms")
      ↓
captured client is #42
      ↓
parent onRename(client42, "XYZ Farms")
```

This is a very important JavaScript mechanism underlying React callback patterns.

---

### 16. Let's trace the rename across two components

Now we can finally connect `ClientRow` and `ClientTable`.

Suppose row 42 is:

```text
ABC Farms
```

User clicks Rename and enters:

```text
XYZ Farms
```

Inside `ClientRow`:

```js
const ok = await onRename(trimmed)
```

where:

```text
trimmed = "XYZ Farms"
```

So:

```js
onRename("XYZ Farms")
```

But the `onRename` prop that row received was:

```js
(newName) => onRename(client, newName)
```

Therefore:

```text
ClientRow

onRename("XYZ Farms")
        │
        ↓
wrapper created by ClientTable

(newName) => onRename(client, newName)
        │
        │ newName = "XYZ Farms"
        │ client = client #42
        ↓
ClientTable's parent callback

onRename(client42, "XYZ Farms")
```

That's the complete translation.

This is exactly like adapter layers you've seen elsewhere:

```text
CHILD'S INTERFACE
onRename(newName)

        ↓ adapter

PARENT'S INTERFACE
onRename(client, newName)
```

---

### 17. Why is that good component design?

Because `ClientRow` already has one job:

> Represent one client.

So its action API can be local:

```text
rename me to X
delete me
```

rather than:

```text
rename client ID 42 to X
delete client ID 42
```

The row doesn't need to identify itself to its parent every time.

Its parent configured it with client-specific callbacks.

You can think of the row's interface as:

```text
ClientRow {
    client: this client's display data

    onRename(newName):
        rename THIS client

    onDelete():
        delete THIS client
}
```

That's a nice narrow component contract.

---

### 18. Delete uses the exact same pattern

```jsx
onDelete={() => onDelete(client)}
```

:chatgpt-content-reference{index="8"}

Remember `ClientRow` does:

```js
const ok = await onDelete()
```

No arguments.

Why?

Because `ClientTable` already created a specialized callback:

```js
() => onDelete(client)
```

Suppose this is row 42.

Then conceptually:

```js
function () {
    return parentOnDelete(client42)
}
```

So:

```text
ClientRow
    │
    │ onDelete()
    ↓
wrapper
    │
    │ captured client42
    ↓
parent
    │
    │ onDelete(client42)
    ↓
...
```

Again:

```text
ROW LANGUAGE
"delete me"

       ↓

TABLE/PARENT LANGUAGE
"delete this specific client object"
```

---

### 19. Now we reach the genuinely new React concept: `key`

```jsx
key={client.client_id}
```

:chatgpt-content-reference{index="9"}

This one is special.

`key` isn't an ordinary prop being passed into `ClientRow` like:

```jsx
client={client}
disabled={disabled}
```

It's information React itself uses when rendering collections.

To understand why, let's imagine there are three rows:

```text
Alpha Farms
Beta Farms
Gamma Farms
```

React renders:

```text
Row
Row
Row
```

Then the data changes.

Perhaps Beta gets deleted:

```text
BEFORE

Alpha
Beta
Gamma


AFTER

Alpha
Gamma
```

React needs to reconcile the old collection with the new collection.

The important question is:

> Which new row corresponds to which old row?

---

### 20. Why position alone isn't enough

Imagine React only thought in positions:

```text
OLD

position 0 → Alpha
position 1 → Beta
position 2 → Gamma


NEW

position 0 → Alpha
position 1 → Gamma
```

If it reasoned only by position, it might conceptually see:

```text
position 0 stayed Alpha

position 1 changed
Beta → Gamma

position 2 disappeared
```

But semantically that's wrong.

What actually happened was:

```text
Alpha stayed
Beta disappeared
Gamma stayed
```

This distinction matters enormously because components can contain local state.

And `ClientRow` does:

```text
mode
draft
busy
```

You absolutely don't want Gamma accidentally inheriting Beta's row identity/state merely because it moved into Beta's previous position.

---

### 21. `key` gives React stable identity

That's why we have:

```jsx
key={client.client_id}
```

Suppose:

```text
Alpha → client_id 1
Beta  → client_id 2
Gamma → client_id 3
```

React sees:

```text
BEFORE

key=1 → Alpha
key=2 → Beta
key=3 → Gamma


AFTER

key=1 → Alpha
key=3 → Gamma
```

Now reconciliation is much clearer:

```text
key=1 still exists
    → same conceptual child

key=2 disappeared
    → remove that child

key=3 still exists
    → same conceptual child
```

So a good initial mental model is:

> **`key` tells React the stable identity of an item within a rendered collection.**

---

### 22. And `client_id` is a very natural key

The code uses:

```js
client.client_id
```

rather than something like the client's position in the array.

That makes sense because the database ID is intended to identify the client record independently of ordering.

Suppose the API changes order:

```text
BEFORE

42 ABC Farms
51 XYZ Farms


AFTER

51 XYZ Farms
42 ABC Farms
```

Their positions changed.

Their identities did not:

```text
client_id 42 remains client 42
client_id 51 remains client 51
```

So:

```jsx
key={client.client_id}
```

lets React track that.

---

### 23. This matters even more because `ClientRow` owns state

Imagine row 42 is currently:

```text
mode = "edit"
draft = "New ABC"
```

Then another client is inserted above it.

If React correctly tracks:

```text
key=42
```

the conceptual identity of that `ClientRow` remains tied to client 42 even though its array position may move.

That's exactly what you want.

This is why keys aren't just about suppressing a React warning.

They're part of React's reconciliation/identity model.

---

### 24. Don't think of `key` as a normal prop

This is worth explicitly separating.

When we write:

```jsx
<ClientRow
    key={client.client_id}
    client={client}
/>
```

the `ClientRow` signature is:

```js
function ClientRow({
    client,
    disabled,
    onRename,
    onDelete
})
```

There is no:

```js
key
```

there.

React consumes `key` for its own bookkeeping.

So conceptually:

```text
key
    → React reconciliation identity


client
disabled
onRename
onDelete
    → actual ClientRow props
```

If `ClientRow` itself needed the client ID, it already gets it through:

```js
client.client_id
```

---

### 25. Now let's mentally execute the entire `.map()`

Suppose:

```js
clients = [
    { client_id: 10, client_name: "Alpha" },
    { client_id: 20, client_name: "Beta" },
    { client_id: 30, client_name: "Gamma" }
]
```

Then:

```js
clients.map((client) => ...)
```

first iteration:

```text
client = Alpha
```

returns conceptually:

```jsx
<ClientRow
    key={10}
    client={Alpha}
    disabled={disabled}
    onRename={(newName) => onRename(Alpha, newName)}
    onDelete={() => onDelete(Alpha)}
/>
```

Second:

```jsx
<ClientRow
    key={20}
    client={Beta}
    disabled={disabled}
    onRename={(newName) => onRename(Beta, newName)}
    onDelete={() => onDelete(Beta)}
/>
```

Third:

```jsx
<ClientRow
    key={30}
    client={Gamma}
    disabled={disabled}
    onRename={(newName) => onRename(Gamma, newName)}
    onDelete={() => onDelete(Gamma)}
/>
```

So `ClientTable` is essentially a **factory/configurator for rows**.

---

### 26. Now we can see responsibility distribution across these two files

This is probably more valuable than memorizing any syntax.

### `ClientTable`

Knows:

```text
I have MANY clients.

I need a table.

For each client:
    create one row.

I know which client each row represents.

I adapt collection-level callbacks
into row-specific callbacks.
```

### `ClientRow`

Knows:

```text
I represent ONE client.

I know how to:
    display it
    enter edit mode
    manage draft
    confirm delete
    represent busy state

When actual domain mutation is needed:
    call my callback.
```

So:

```text
                ClientTable
                     │
        collection responsibility
                     │
         ┌───────────┼───────────┐
         ↓           ↓           ↓
     ClientRow   ClientRow   ClientRow
         │           │           │
      local UI    local UI    local UI
       state       state       state
```

This is much more scalable than putting every row's edit/delete UI state into the table itself.

---

### 27. Let's trace a rename across the component hierarchy

We're now able to follow much farther than before.

Suppose client 42 is `"ABC Farms"`.

```text
CLIENT TABLE
clients = [...]
     │
     │ .map()
     ↓
ClientRow for client 42
     │
     │ user clicks Rename
     ↓
setMode("edit")
     │
     ↓
edit form renders
     │
     │ user types "XYZ Farms"
     ↓
setDraft("XYZ Farms")
     │
     │ Save
     ↓
commitEdit()
     │
     ↓
onRename("XYZ Farms")
```

But remember that callback was constructed by `ClientTable`:

```text
onRename("XYZ Farms")
     │
     ↓
(newName) => onRename(client42, newName)
     │
     ↓
parent-level onRename(
    client42,
    "XYZ Farms"
)
```

And **that's where this file stops**.

`ClientTable` doesn't know what happens afterward.

That callback came from its own parent.

So we're climbing the tree:

```text
??? parent
     │
     │ onRename(client, newName)
     ↓
ClientTable
     │
     │ specializes callback
     ↓
ClientRow
     │
     │ local interaction
     ↓
user
```

When we eventually reach `App.jsx`, we'll find out what's at the top of that chain.

---

### 28. One subtle observation: callbacks flow down even though events conceptually flow up

Earlier I said:

```text
data ↓
events ↑
```

That's a useful conceptual model, but now we can make it more precise.

Technically, the **callback function itself flows downward as a prop**:

```text
App
 │
 │ callback
 ↓
ClientTable
 │
 │ specialized callback
 ↓
ClientRow
```

Then later the child **invokes** that function:

```text
ClientRow
 │
 │ calls callback
 ↑
ClientTable's abstraction
 │
 │ calls callback
 ↑
App
```

So a more mechanically accurate model is:

```text
RENDER/CONFIGURATION TIME

data       ↓
callbacks  ↓


INTERACTION TIME

child invokes callback
       ↑
information travels toward owner
```

That's why React can maintain one-way data flow without needing some special "send event to parent" language feature.

It's ordinary JavaScript functions passed downward and invoked later.

That's an important refinement.

---

### 29. And `ClientTable` itself remains remarkably simple

After all that explanation, look again at the actual implementation:

```jsx
export default function ClientTable({ clients, disabled, onRename, onDelete }) {
  return (
    <table className="client-table">
      ...
      <tbody>
        {clients.map((client) => (
          <ClientRow
            key={client.client_id}
            client={client}
            disabled={disabled}
            onRename={(newName) => onRename(client, newName)}
            onDelete={() => onDelete(client)}
          />
        ))}
      </tbody>
    </table>
  )
}
```

:chatgpt-content-reference{index="10"}

There is essentially no orchestration here.

No async logic.

No API.

No state.

No effects.

It is mostly a **structural transformation**:

```text
array of domain objects
        ↓
table of row components
```

---

### 30. The four things worth retaining from this file

I wouldn't spend memory on every `<th>`.

The important concepts are:

```text
1. COLLECTION RENDERING

clients[]
   ↓
.map(...)
   ↓
<ClientRow />[]
```

Second:

```text
2. COMPONENT COMPOSITION

ClientTable
   ↓
uses ClientRow
   ↓
ClientRow itself produces <tr>
```

Third:

```text
3. CALLBACK SPECIALIZATION / CLOSURES

table-level:
onRename(client, newName)

        ↓ bind/capture client

row-level:
onRename(newName)
```

And fourth:

```text
4. KEY / COMPONENT IDENTITY

key={client.client_id}

tells React:
"This rendered child corresponds
to this stable item."
```

Those are the real lessons of `ClientTable.jsx`.

And there's something nice happening across the files we've studied so far:

```text
useCurrentUser.js
────────────────────
state + persistence


client.js
────────────────────
HTTP/API boundary


UserBar.jsx
────────────────────
props + controlled input
+ callbacks


ClientRow.jsx
────────────────────
local UI state machine
+ async interaction
+ refs/effects


ClientTable.jsx
────────────────────
collections
+ component composition
+ keys
+ callback adaptation
```

We're progressively accumulating exactly the pieces we'll need to understand the orchestration layer rather than jumping straight into it.

Our next planned file is `AddClientForm.jsx`. That should be particularly useful now because it will likely combine several mechanisms we've already learned—controlled input, form submission, async/busy state, callback props—but at the **create-client** boundary rather than editing an existing row. After that, we should finally be equipped to tackle `App.jsx` top-to-bottom and trace the entire system as one connected execution.

## Code Appendix

```js
import ClientRow from './ClientRow.jsx'

export default function ClientTable({ clients, disabled, onRename, onDelete }) {
  return (
    <table className="client-table">
      <thead>
        <tr>
          <th className="col-id">ID</th>
          <th className="col-name">Client name</th>
          <th className="col-meta">Last modified</th>
          <th className="col-meta">Modified by</th>
          <th className="col-actions">Actions</th>
        </tr>
      </thead>
      <tbody>
        {clients.map((client) => (
          <ClientRow
            key={client.client_id}
            client={client}
            disabled={disabled}
            onRename={(newName) => onRename(client, newName)}
            onDelete={() => onDelete(client)}
          />
        ))}
      </tbody>
    </table>
  )
}
```