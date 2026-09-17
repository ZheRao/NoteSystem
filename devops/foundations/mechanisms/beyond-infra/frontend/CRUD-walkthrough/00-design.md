# Changing from generic React course to study generated CRUD code around `client` API written by me

I think that sounds like a **very good change in learning strategy**, especially for what you're actually trying to accomplish.

The key distinction is that your goal isn't really:

> “Learn React as a subject.”

It's closer to:

> “Understand the frontend well enough that I can build, inspect, debug, modify, and take responsibility for the frontend around Strata.”

Those call for somewhat different learning strategies.

What you're proposing is essentially **project-driven, just-in-time learning**. You already have something unusually valuable: a real API you understand deeply. If Claude generates a small React client around your `client` CRUD endpoints, every unfamiliar React concept suddenly has a concrete reason to exist.

For example, instead of encountering `useEffect` abstractly and wondering *why React invented this weird thing*, you'll encounter something like:

```js
useEffect(() => {
    fetchClients()
}, [])
```

and we can investigate it from the problem outward:

**What problem is this solving? → When does this code execute? → Why can't we just put `fetchClients()` in the component body? → What causes React to render again? → What does `[]` mean? → What happens if we put something inside it? → What bugs can effects create?**

That's a much better way for you to learn than memorizing "`useEffect` is a React Hook for synchronizing with external systems."

Same thing with:

```js
fetch("/api/clients")
    .then(response => response.json())
    .then(data => setClients(data))
    .catch(error => ...)
```

We don't even need to treat that primarily as "React." We can peel it apart:

```text
HTTP request
    ↓
fetch()
    ↓
Promise
    ↓
.then(...)
    ↓
another Promise
    ↓
.then(...)
    ↓
React state update
    ↓
re-render
    ↓
new UI
```

That connects **JavaScript asynchronous execution → browser networking → React state → rendering**, which is the mental model you actually need.

And importantly, I wouldn't make the generated code sacred. Claude is effectively giving us a **specimen to dissect**. We can ask of every piece:

> Why is this here?

> What would happen if I deleted it?

> Is this JavaScript, browser API, or React?

> What state does it manipulate?

> What causes it to execute?

> What assumptions is it making about my FastAPI API?

> How does failure propagate?

> Is this merely convenient code, or something I need to understand for correctness?

That last distinction will keep this from turning into another enormous learning rabbit hole.

### I'd use three levels while we dissect it

**Level 1 — Must understand mechanically.** Things such as components, props, state, events, `useState`, rendering/re-rendering, `useEffect`, forms, controlled inputs, array `.map()`, conditional rendering, `fetch`, promises, `async/await`, and basic module/import syntax. You should be able to look at these and mentally execute roughly what's happening.

**Level 2 — Understand the purpose and failure modes.** Things like effect cleanup, dependency arrays, race conditions in requests, stale state, error/loading states, component boundaries, API abstraction, validation, and frontend/backend contracts. You don't necessarily need encyclopedic knowledge, but because you're responsible for the system, you should recognize where danger lives.

**Level 3 — Look it up when necessary.** Obscure syntax, unusual CSS details, less-common browser APIs, framework conveniences, sophisticated React patterns, optimization tricks, etc. There's very little value in pre-learning all of this.

This also addresses the concern you've had recently about chasing an endlessly expanding frontend knowledge tree. You **don't need to close every theoretical frontend gap before building Strata**. The generated CRUD UI gives you a boundary. We study whatever concepts are necessary to understand *this actual system*, and branch outward only when something exposes an important correctness/design issue.

There is one thing I would *not* do: have Claude generate the whole frontend, verify that it appears to work, and then treat it as solved. That's precisely where AI-generated software can become dangerous—not because Claude can't write decent React, but because you become responsible for behavior encoded in a system you can't reason about.

Your proposed method avoids that. **Let the model produce code quickly; make understanding the code your job.**

And I suspect `useEffect` is actually going to become much less mysterious once we approach it through your CRUD application. It sits right at the intersection of several things you've already been thinking about: state, external systems, lifecycle/timing, synchronization, and side effects.

So when you're ready, paste the generated React code. **Don't explain it to me first.** We can take it literally from the top, and whenever we encounter something like a Promise, arrow function, destructuring, `useState`, or `useEffect`, we'll stop and build the underlying mental model before moving to the next piece.


# The order to walkthrough

Yes. Given that architecture, I would **not start with `App.jsx`**, even though it's the orchestration layer. If we start there, almost every line will point toward concepts you haven't unpacked yet, and we'll constantly jump elsewhere.

I'd study it **bottom-up until the pieces become understandable, then finish top-down with orchestration**.

My proposed order is:

1. **`useCurrentUser.js`** — start here. It's probably small and gives us our first contained exposure to hooks/state without mixing in HTTP, forms, tables, and CRUD simultaneously. We establish what a custom hook is and what the temporary-auth abstraction means.

2. **`client.js`** — then understand the frontend/backend boundary. This is particularly important because *you wrote the FastAPI side*. We can map each function directly onto your endpoints:
   `createClient()` → HTTP → FastAPI endpoint → DB transaction → response/error → Promise resolution/rejection.

   This is where I'd deliberately stop and teach `fetch`, `Promise`, `.then()`, `.catch()`, `async/await`, HTTP response handling, and error propagation. Those are fundamentally **JavaScript/browser concepts**, not React concepts.

3. **`UserBar.jsx`** — first simple presentational component. This lets us understand JSX, components, props, conditional rendering, and the basic relationship:
   `state/data → JSX → rendered DOM`

   without CRUD complexity.

4. **`ClientRow.jsx`** — now introduce a component representing one domain entity. This should naturally expose props, event handlers, callbacks, perhaps editing/deleting, and the crucial idea that a child can *receive data downward and communicate events upward*.

5. **`ClientTable.jsx`** — now composition becomes visible:

   ```text
   ClientTable
        │
        ├── ClientRow
        ├── ClientRow
        └── ClientRow
   ```

   This is where `.map()`, `key`, lists, component composition, and data flow should become concrete.

6. **`AddClientForm.jsx`** — I'd deliberately save this until after the display side. Forms tend to introduce several concepts simultaneously: controlled inputs, `useState`, `onChange`, `onSubmit`, `preventDefault()`, validation, resetting state, callbacks, etc. We'll understand why each exists rather than memorizing syntax.

7. **`App.jsx` — last.** At this point, this should be the payoff. Instead of looking like mysterious React machinery, hopefully you'll see something closer to:

   ```text
                       App
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
     current user    clients       UI state
          │             │
          ↓             ↓
       UserBar      ClientTable
                        │
                        ↓
                    ClientRow

                  AddClientForm
                        │
                        ↓
                     client.js
                        │
                        ↓
                     FastAPI
   ```

And **this is where I'd teach `useEffect` properly**.

That's important because I don't want to teach you `useEffect` as isolated React syntax. By the time we reach `App.jsx`, you'll already understand `client.js`, Promises, state, components, and rendering. Then if we encounter something resembling:

```jsx
useEffect(() => {
    listClients()
        .then(setClients)
        .catch(setError);
}, []);
```

we can ask the much more interesting question:

**Why does React need an effect here at all?**

You already know:

```text
listClients()
      ↓
HTTP GET
      ↓
FastAPI
      ↓
Promise
      ↓
clients returned
      ↓
setClients(...)
      ↓
React state changes
      ↓
App renders again
      ↓
ClientTable receives new clients
      ↓
rows appear
```

The only missing piece is:

> **Who initiates that process, and when?**

And *that* is the problem `useEffect` is addressing.

### One rule I'd use throughout

For every file, let's separate what we're seeing into **four layers**:

**JavaScript → Browser/Web → React → Your application**

For example:

```js
const handleDelete = (id) => {
    deleteClient(id)
        .then(() => setClients(...))
        .catch(err => setError(err.message))
}
```

We wouldn't just say "this deletes a client."

We'd identify:

```text
const / arrow function       → JavaScript
Promise / .then / .catch     → JavaScript
fetch                        → Browser Web API
setClients                   → React
deleteClient                 → your client.js abstraction
DELETE /clients/{id}         → your FastAPI contract
```

I think that classification will be **extremely useful for you**, because one reason React can initially feel incomprehensible is that a 10-line React component can casually mix JavaScript language features, browser APIs, React APIs, and application-specific abstractions.

So: **start with `useCurrentUser.js`. Paste the entire file**, even if it's tiny. We'll dissect it until you can mentally execute it, then move to `client.js`.
