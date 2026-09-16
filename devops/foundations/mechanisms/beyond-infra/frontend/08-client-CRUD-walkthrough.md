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

# 1. `useCurrentUser.js`

## Lesson 1 - `read()`

Perfect. This is actually an excellent first file because it's only ~30 lines, but it contains several important React/JavaScript ideas without any HTTP complexity. 

I suggest we **do not explain the whole file at once**. Let's mentally execute it from top to bottom and stop whenever we encounter a concept worth understanding.

The first three meaningful lines are:

```js
import { useCallback, useState } from 'react'

const STORAGE_KEY = 'experiment.currentUser'

function read() {
  try {
    return localStorage.getItem(STORAGE_KEY) ?? ''
  } catch {
    return ''
  }
}
```

The import can wait briefly. `STORAGE_KEY` is straightforward. So I think our **first real object of study should be `read()`**, because there's already an important distinction hiding here between **JavaScript** and the **browser environment**. 

---
### 1. What is `localStorage`?

This is **not React**, and it isn't actually part of the JavaScript language either.

`localStorage` is a **browser Web API**. Conceptually, the browser gives JavaScript a persistent little key-value store:

```text
Browser
┌──────────────────────────────┐
│ localStorage                 │
│                              │
│ "experiment.currentUser"     │
│            ↓                 │
│          "Zhe"               │
└──────────────────────────────┘
```

So:

```js
localStorage.getItem(STORAGE_KEY)
```

effectively becomes:

```js
localStorage.getItem('experiment.currentUser')
```

If that key exists, the browser gives us its stored string.

If you've previously stored:

```text
experiment.currentUser → "Zhe"
```

then:

```js
localStorage.getItem(STORAGE_KEY)
```

returns:

```js
"Zhe"
```

The important property is **persistence**. A normal JavaScript variable disappears when you reload the page. `localStorage` survives page reloads and browser restarts until it is removed.

That's exactly why Claude chose it for your temporary authentication substitute. Your frontend needs to remember:

> "Who is supposedly making these CRUD changes?"

even though you don't yet have actual authentication.

---

### 2. Then we encounter this unfamiliar-looking operator

```js
return localStorage.getItem(STORAGE_KEY) ?? ''
```

`??` is JavaScript's **nullish coalescing operator**.

Read it roughly as:

> Use the thing on the left unless it is `null` or `undefined`; otherwise use the thing on the right.

So:

```js
"Zhe" ?? ''
```

produces:

```js
"Zhe"
```

while:

```js
null ?? ''
```

produces:

```js
''
```

And that's relevant because `localStorage.getItem()` returns `null` when the requested key doesn't exist.

Therefore this line guarantees:

```text
localStorage contains user
        ↓
      "Zhe"

localStorage doesn't contain user
        ↓
       null
        ↓
      ?? ''
        ↓
        ""
```

So the rest of your application doesn't have to deal with both `null` and strings. This function normalizes "no user" into the empty string.

---

### 3. Why the `try/catch`?

The file itself actually tells us the intent later: storage might be unavailable (`localStorage` can throw before you ever get a value back), while the React state should still work in memory. 

So:

```js
function read() {
  try {
    return localStorage.getItem(STORAGE_KEY) ?? ''
  } catch {
    return ''
  }
}
```

has a very nice contract:

> **`read()` always gives the application a string.**

It doesn't make the rest of the application care about why persistent storage failed.

That is already a small example of something you've been doing extensively in your backend work: **put messy infrastructure behavior behind a narrow interface**.

The caller doesn't need:

```text
Does localStorage exist?
Did access fail?
Was the key missing?
```

It just gets:

```text
"Zhe"
```

or:

```text
""
```

---

And now we arrive at the **first genuinely React-specific line**:

```js
export function useCurrentUser() {
  const [currentUser, setState] = useState(read)
```



I want to stop **right here**, because this one line deserves significant attention.

There are actually **four separate concepts** packed into it:

```text
export function useCurrentUser()
│      │        │
│      │        └── React custom-hook naming convention
│      └─────────── normal JavaScript function
└────────────────── ES module export


const [currentUser, setState] = useState(read)
      └────────────┘   │       │
        destructuring  │       └── initializer function
                       └────────── React state Hook
```

And `useState` is foundational. If we properly understand what **state means, why a normal variable isn't sufficient, what `setState` actually causes React to do, and why this passes `read` rather than `read()`**, much of the rest of the generated frontend will become easier.

So **that's where I'd go next**: dissect

```js
const [currentUser, setState] = useState(read)
```

from first principles, especially the relationship between **state → render → state update → re-render**.

That is essentially the doorway into React itself.

## Lesson 2 - React Hook

Absolutely. Let's stay on this line until its mechanics are clear:

```js
const [currentUser, setState] = useState(read)
```

This is line 18 of your hook. 

### 1. First: why does React need "state" at all?

Forget `useState` syntax for a moment.

Suppose we wrote:

```js
let currentUser = "Zhe"
```

and somewhere later:

```js
currentUser = "Alice"
```

JavaScript is perfectly happy with this. `currentUser` now contains `"Alice"`.

But React has another problem:

> **How does React know that the screen needs to change?**

Imagine:

```jsx
function UserBar() {
    let currentUser = "Zhe"

    return <div>Current user: {currentUser}</div>
}
```

React renders:

```text
Current user: Zhe
```

Later:

```js
currentUser = "Alice"
```

The JavaScript variable changed.

But that by itself doesn't tell React:

> "Hey, something affecting the UI changed. Execute this component again and figure out what the UI should look like now."

That's one of the fundamental jobs of state.

---

### 2. `useState` gives React ownership of this changing value

Consider:

```js
const [currentUser, setState] = useState("Zhe")
```

Conceptually, React gives us **two things**:

```text
useState("Zhe")
       │
       ├── current value ───────→ "Zhe"
       │
       └── update function ─────→ setState(...)
```

So you can roughly imagine:

```js
const [currentUser, setState] = ...
```

as:

```text
currentUser = "What is the value currently stored by React?"

setState = "Tell React that this state should change."
```

This distinction is important.

We **don't** do:

```js
currentUser = "Alice"
```

Instead:

```js
setState("Alice")
```

means approximately:

> React, update this piece of state to `"Alice"` and schedule the relevant UI to render again.

So the flow becomes:

```text
currentUser = "Zhe"

        ↓

UI rendered using "Zhe"

        ↓

setState("Alice")

        ↓

React stores "Alice"

        ↓

React re-renders

        ↓

component executes again

        ↓

currentUser is now "Alice"

        ↓

UI rendered using "Alice"
```

This **state → render → state change → re-render** cycle is one of the most important mental models in React.

---

### 3. So what is this `[currentUser, setState]` syntax?

This part isn't React at all:

```js
const [currentUser, setState] = ...
```

It's JavaScript **array destructuring**.

Suppose:

```js
const stuff = ["Zhe", someFunction]
```

We could write:

```js
const currentUser = stuff[0]
const setState = stuff[1]
```

JavaScript gives us shorthand:

```js
const [currentUser, setState] = stuff
```

Same idea.

And `useState()` returns something conceptually like:

```js
[value, updaterFunction]
```

Therefore:

```js
const result = useState(...)
```

then:

```js
const currentUser = result[0]
const setState = result[1]
```

is equivalent in spirit to:

```js
const [currentUser, setState] = useState(...)
```

So don't associate the square brackets with some magical React syntax.

React provides the array.

**JavaScript destructuring unpacks it.**

---

### 4. Why call it `setState` rather than `setCurrentUser`?

Normally you'd see:

```js
const [currentUser, setCurrentUser] = useState(...)
```

But your code deliberately does:

```js
const [currentUser, setState] = useState(read)
```

Why?

Because later the hook creates its **own** `setCurrentUser`:

```js
const setCurrentUser = useCallback((value) => {
    setState(value)

    try {
        if (value) localStorage.setItem(STORAGE_KEY, value)
        else localStorage.removeItem(STORAGE_KEY)
    } catch {
        // ...
    }
}, [])
```



That's actually a useful little design.

There are two layers:

```text
Raw React state setter
setState("Alice")
        │
        └── changes React state only


Application's setter
setCurrentUser("Alice")
        │
        ├── setState("Alice")
        │
        └── localStorage.setItem(...)
```

The application shouldn't normally bypass the persistence behavior and directly call the raw React setter.

So the hook wraps it.

We'll come back to this when we reach `useCallback`.

---

### 5. Now the weirdest part: `useState(read)`

You might reasonably expect:

```js
useState(read())
```

because we want the result of:

```js
read()
```

But the code actually says:

```js
useState(read)
```

Those mean very different things.

#### `read()`

Means:

> Execute `read` **right now** and give `useState` its result.

```text
read()
  ↓
localStorage
  ↓
"Zhe"
  ↓
useState("Zhe")
```

#### `read`

Means:

> Give React the **function itself**.

```text
read
 ↓
function reference
 ↓
useState(read)
```

React recognizes that when the initial argument to `useState` is a function, it should use that function as an **initializer**.

Conceptually:

```text
React needs initial state
        ↓
"Hey, they gave me a function"
        ↓
call read()
        ↓
localStorage.getItem(...)
        ↓
"Zhe"
        ↓
initial currentUser = "Zhe"
```

This is called **lazy initialization**.

---

### 6. Why bother?

Because your component/hook can execute repeatedly.

That's something I want you to get accustomed to:

> **A React component function isn't something that necessarily runs once. React runs it again when it renders again.**

Suppose we had:

```js
const [currentUser, setState] = useState(read())
```

JavaScript has to evaluate the argument before calling `useState`.

So every execution evaluates:

```js
read()
```

even though React only needs the value for **initialization**.

Conceptually:

```text
FIRST RENDER

read()
  ↓
localStorage
  ↓
"Zhe"

useState("Zhe")
       ↓
React initializes state


SECOND RENDER

read()                 ← unnecessary work
  ↓
localStorage
  ↓
"Zhe"

useState("Zhe")
       ↓
React already has state;
initial value isn't needed


THIRD RENDER

read()                 ← unnecessary again
...
```

Whereas:

```js
useState(read)
```

lets React decide when `read()` needs to be invoked for initialization.

Conceptually:

```text
FIRST initialization

useState(read)
         ↓
       read()
         ↓
       "Zhe"


LATER RENDER

useState(read)
         ↓
React already has this state
         ↓
don't initialize it again
```

So this line:

```js
const [currentUser, setState] = useState(read)
```

means something close to:

> **Create a React state variable called `currentUser`. The first time this state is initialized, obtain its initial value by calling `read()`. Also give me a function called `setState` with which I can update that state.**

That is the important interpretation.

---

### 7. Now we can understand why `read()` exists separately

The architecture is already becoming visible:

```text
                 Browser localStorage
                        │
                        ↓
                     read()
                        │
                 initial value only
                        ↓
                    useState
                        │
                        ↓
                  currentUser
```

Once React has initialized the state, **React state becomes the live value used by the application**.

`localStorage` is persistence across page loads.

That distinction matters:

```text
React state
    │
    └── What does the application currently believe?
        Fast, live, drives rendering.


localStorage
    │
    └── What should survive if the browser reloads?
```

This also explains why later `setCurrentUser` updates **both**.

---

### 8. Let's mentally execute an actual session

Suppose you've never opened this application.

There is no stored user.

The hook executes:

```js
useState(read)
```

React calls:

```js
read()
```

which calls:

```js
localStorage.getItem("experiment.currentUser")
```

Browser returns:

```js
null
```

therefore:

```js
null ?? ''
```

becomes:

```js
''
```

So:

```text
currentUser = ""
```

Later you type:

```text
Zhe
```

and eventually the application calls:

```js
setCurrentUser("Zhe")
```

That will do two things:

```text
setState("Zhe")
      │
      ↓
React state becomes "Zhe"
      │
      ↓
React re-renders UI


localStorage.setItem(
    "experiment.currentUser",
    "Zhe"
)
      │
      ↓
browser persists "Zhe"
```

Now close the browser.

Tomorrow, open the application again.

Everything in JavaScript memory from yesterday is gone.

React starts fresh:

```js
useState(read)
```

React calls:

```js
read()
```

Browser says:

```text
experiment.currentUser → "Zhe"
```

Therefore the **new React application reconstructs its initial state from persistent browser storage**.

That's the entire purpose of the first half of this hook.

---

### Next line: this is where things get especially interesting

Now we're prepared for:

```js
const setCurrentUser = useCallback((value) => {
    setState(value)
    try {
        if (value) localStorage.setItem(STORAGE_KEY, value)
        else localStorage.removeItem(STORAGE_KEY)
    } catch {
        // private mode / storage disabled — in-memory state still works
    }
}, [])
```



And I think the right way to attack this is **not to start with `useCallback`**.

First we should temporarily pretend it says:

```js
const setCurrentUser = (value) => {
    ...
}
```

and understand **arrow functions**, `value`, `setState(value)`, the `if/else`, and why the code deliberately updates React state *before* touching `localStorage`.

Then we'll put `useCallback(..., [])` back around it and ask the much deeper React question:

> Why does React care about the **identity of a function**?

That will prepare us nicely for the dependency-array concepts you'll later encounter with `useEffect`.


## Lesson 3 - JavaScript function & `useCallback` * dependency array

Good. Let's temporarily remove the React-specific wrapper and look at the underlying function:

```js
const setCurrentUser = (value) => {
    setState(value)
    try {
        if (value) localStorage.setItem(STORAGE_KEY, value)
        else localStorage.removeItem(STORAGE_KEY)
    } catch {
        // private mode / storage disabled — in-memory state still works
    }
}
```

Then we'll put `useCallback` back around it. This is the core of lines 20–28. 

### 1. `(value) => { ... }` is just a function

This:

```js
(value) => {
    setState(value)
}
```

is JavaScript **arrow-function syntax**.

A close equivalent using traditional syntax is:

```js
function (value) {
    setState(value)
}
```

So this:

```js
const setCurrentUser = (value) => {
    setState(value)
}
```

means essentially:

```js
function setCurrentUser(value) {
    setState(value)
}
```

There are some technical differences between arrow functions and traditional functions, particularly around `this`, but they aren't relevant to what we're doing here. Don't spend learning bandwidth on that yet.

The important thing is simply:

```text
(value) => { ... }
   ↑
parameter

(value) => {
    setState(value)
    └───────────── function body
}
```

So later somebody can call:

```js
setCurrentUser("Zhe")
```

and during that execution:

```text
value = "Zhe"
```

---

### 2. First thing it does: update React

```js
setState(value)
```

Remember where `setState` came from:

```js
const [currentUser, setState] = useState(read)
```

So if:

```js
value === "Zhe"
```

then:

```js
setState(value)
```

is:

```js
setState("Zhe")
```

We're telling React:

> Change the `currentUser` state associated with this hook to `"Zhe"`.

That will cause the relevant React rendering to update.

Notice something subtle about the architecture now.

The rest of the application doesn't receive `setState`.

It will eventually receive:

```js
setCurrentUser
```

The hook is deliberately hiding the primitive operation:

```text
setState
```

behind an application-specific operation:

```text
setCurrentUser
```

That's very similar to something like:

```python
def set_current_user(value):
    ...
```

wrapping lower-level persistence machinery.

---

### 3. Then it synchronizes that value to browser storage

```js
if (value) localStorage.setItem(STORAGE_KEY, value)
else localStorage.removeItem(STORAGE_KEY)
```

This is normal JavaScript `if/else`, just compressed because each branch contains only one statement.

Expanded:

```js
if (value) {
    localStorage.setItem(STORAGE_KEY, value)
} else {
    localStorage.removeItem(STORAGE_KEY)
}
```

Suppose:

```js
value = "Zhe"
```

JavaScript evaluates:

```js
if ("Zhe")
```

A non-empty string is **truthy**, so:

```js
localStorage.setItem(STORAGE_KEY, value)
```

runs.

Given:

```js
const STORAGE_KEY = 'experiment.currentUser'
```

that's effectively:

```js
localStorage.setItem(
    'experiment.currentUser',
    'Zhe'
)
```

Browser storage becomes:

```text
experiment.currentUser → "Zhe"
```

---

### 4. What happens if `value` is empty?

Suppose the user clears their identity:

```js
setCurrentUser("")
```

First:

```js
setState("")
```

React's state becomes empty.

Then:

```js
if (value)
```

means:

```js
if ("")
```

An empty string is **falsy** in JavaScript.

So the `else` executes:

```js
localStorage.removeItem(STORAGE_KEY)
```

Rather than storing:

```text
experiment.currentUser → ""
```

the application removes the key entirely.

That's why the earlier `read()` function needed:

```js
localStorage.getItem(STORAGE_KEY) ?? ''
```

The two pieces fit together:

```text
WRITE

setCurrentUser("Zhe")
        ↓
localStorage:
experiment.currentUser → "Zhe"


setCurrentUser("")
        ↓
removeItem(...)
        ↓
no key exists


READ AFTER RELOAD

getItem(...)
        ↓
null
        ↓
null ?? ''
        ↓
""
```

So `"no current user"` has one consistent representation inside the React application:

```js
""
```

even though browser storage represents it by the **absence of the key**.

That's a nice little boundary normalization.

---

### 5. Why is `setState(value)` outside the `try`?

This is worth noticing:

```js
setState(value)

try {
    if (value) ...
    else ...
} catch {
    ...
}
```

rather than:

```js
try {
    setState(value)
    ...
} catch {
    ...
}
```

The comment tells us the intended behavior:

```js
// private mode / storage disabled — in-memory state still works
```



There are really two capabilities here:

```text
CORE BEHAVIOR
React remembers current user
        ↓
setState(value)


OPTIONAL PERSISTENCE
Browser remembers it across reloads
        ↓
localStorage
```

If browser persistence fails, Claude's generated implementation has decided:

> That's okay. The application should continue functioning for the current session.

So:

```text
setCurrentUser("Zhe")
        │
        ├── setState("Zhe")        ✓
        │
        └── localStorage.setItem   ✗ throws error
                                      │
                                      ↓
                                    catch
                                      │
                                      ↓
                                 ignore failure
```

The UI still knows:

```text
currentUser = "Zhe"
```

but if you refresh the browser, `"Zhe"` may be lost.

That's a **degradation policy**, not merely syntax.

---

### 6. Now let's put `useCallback` back

The real code is:

```js
const setCurrentUser = useCallback((value) => {
    ...
}, [])
```

And now we hit another fundamental React idea.

At first glance, this seems bizarre.

Why not simply write:

```js
const setCurrentUser = (value) => {
    ...
}
```

?

It would work.

So why `useCallback`?

To understand that, we need one JavaScript fact first.

#### Functions are values — and function identity matters

Consider:

```js
const a = () => {}
const b = () => {}
```

They contain identical code.

But:

```js
a === b
```

is:

```js
false
```

Why?

Because JavaScript created **two different function objects**.

Think of it similarly to:

```text
a ──────→ Function object #1

b ──────→ Function object #2
```

Same behavior doesn't mean same object identity.

And here's where React rendering makes this interesting.

Remember:

> React executes component functions again when they render.

So imagine:

```jsx
function App() {
    const setCurrentUser = (value) => {
        ...
    }

    ...
}
```

First render:

```text
App() executes
    ↓
create function object #1
    ↓
setCurrentUser → function #1
```

Then state changes and React renders again:

```text
App() executes again
    ↓
execute:
(value) => {...}
    ↓
create function object #2
    ↓
setCurrentUser → function #2
```

Even though the source code is unchanged:

```js
(value) => { ... }
```

these are different function objects:

```text
RENDER 1                 RENDER 2

setCurrentUser            setCurrentUser
      │                         │
      ↓                         ↓
function #1               function #2

        #1 !== #2
```

---

### 7. `useCallback` lets React preserve that function identity

Now:

```js
const setCurrentUser = useCallback(
    (value) => {
        ...
    },
    []
)
```

roughly means:

> React, remember this callback and give me the same function reference on later renders unless its dependencies change.

So:

```text
FIRST RENDER

useCallback(...)
     ↓
function #1
     ↓
setCurrentUser


SECOND RENDER

useCallback(...)
     ↓
same function #1
     ↓
setCurrentUser


THIRD RENDER

useCallback(...)
     ↓
same function #1
```

Instead of:

```text
#1 → #2 → #3 → #4...
```

we retain:

```text
#1 → #1 → #1 → #1...
```

This is called **memoizing the callback**.

---

### 8. And what the hell is `[]`?

Now we've arrived at something that will become extremely important when we eventually study `useEffect`.

```js
useCallback(
    function,
    []
)
```

The second argument is the **dependency array**.

Conceptually:

> Keep this callback as long as none of these dependencies change.

Here there are **zero dependencies**:

```js
[]
```

So React doesn't have anything that could trigger creation of a new callback.

That's why its identity remains stable.

Compare conceptually:

```js
useCallback(() => {
    doSomething(clientId)
}, [clientId])
```

Now React has been told:

> This function depends upon `clientId`.

So:

```text
clientId = 10

callback #1
    ↓
re-render
    ↓
clientId still 10
    ↓
reuse callback #1


clientId changes to 11
    ↓
dependency changed
    ↓
create callback #2
```

That's the beginning of the React **dependency mental model**.

We'll get much deeper into it with `useEffect`.

---

### 9. But why doesn't this callback depend on `setState`?

Look inside:

```js
(value) => {
    setState(value)
    ...
}
```

Surely it uses `setState`.

Why isn't this:

```js
[setState]
```

?

Because React guarantees that the setter returned from `useState` has a **stable identity**.

So React isn't going to suddenly give you a different `setState` function on every render.

And these:

```js
STORAGE_KEY
localStorage
```

aren't changing React state/props captured from a render either.

So the generated code can reasonably use:

```js
[]
```

---

### 10. Is `useCallback` actually necessary here?

This is an important distinction between:

> **What React requires**

and:

> **What someone chose as an optimization/design decision**

This code could simply be:

```js
const setCurrentUser = (value) => {
    setState(value)
    ...
}
```

and the fundamental functionality would still work.

`useCallback` is about preserving function identity.

Whether preserving that identity matters depends on how `setCurrentUser` is subsequently used — for example, whether it's passed to children whose rendering/memoization depends on reference equality, or used as a dependency elsewhere.

So I don't want you leaving this section thinking:

> "React setter wrapper = must use `useCallback`."

Absolutely not.

Instead:

> "`useCallback` is available when stable callback identity matters."

We'll see whether it was genuinely useful once we inspect the consumers of this hook.

---

### 11. We're down to the final line

The hook finishes with:

```js
return [currentUser.trim(), currentUser, setCurrentUser]
```



There's **no React magic in the `return` itself**. It's returning an ordinary JavaScript array containing three values:

```text
index 0 → currentUser.trim()
index 1 → currentUser
index 2 → setCurrentUser
```

And there is something very deliberate here:

```js
currentUser.trim()
```

versus:

```js
currentUser
```

The hook exposes **both a normalized version and the raw value**.

For example:

```js
currentUser = "  Zhe  "
```

then:

```js
currentUser.trim()
```

is:

```js
"Zhe"
```

but the original `currentUser` remains:

```js
"  Zhe  "
```

So the caller might do something like:

```js
const [modifiedBy, currentUser, setCurrentUser] = useCurrentUser()
```

which would give:

```text
modifiedBy       → "Zhe"
currentUser      → "  Zhe  "
setCurrentUser   → function
```

That makes sense architecturally: **the raw string can drive the input UI, while the trimmed string can be sent to your FastAPI API as `modified_by`.**

And notice how this completes our first custom hook:

```text
                    useCurrentUser()
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ↓             ↓              ↓
        modifiedBy    currentUser    setCurrentUser
        (trimmed)       (raw)          (function)
             │             │              │
             ↓             ↓              ↓
        API writes      UI input       React state
                                      + localStorage
```

The entire file is therefore an **abstraction over temporary user identity**. React state handles the live application value; `localStorage` makes it survive reloads; the custom hook packages those mechanics so the rest of your application doesn't have to care about them. 

At this point, I think we've extracted the important mechanics from `useCurrentUser.js`. **Next I would move to `client.js`**, because that's where we can build the second major mental model: `fetch` → `Promise` → HTTP → FastAPI → response → `.then()` / `.catch()`. That foundation should make the React components substantially easier to read.

---
### 12. Aside: about `useCallback` and re-render

The name is a red herring. `setCurrentUser` isn't a variable that lives somewhere and gets overwritten — it's a local binding inside the function body of `useCurrentUser`, and that function gets *called again* on every render.

So it's not this:

```js
const a = () => {}
const a = () => {}   // wouldn't even be legal with const
```

It's this:

```js
function useCurrentUser() {
  const a = () => {}
  return a
}

const first = useCurrentUser()   // render 1
const second = useCurrentUser()  // render 2
first === second  // false
```

Each call creates a brand-new scope with a brand-new binding named `a`, holding a brand-new closure object. The old one isn't overwritten; it just becomes garbage once nothing references it. Shadowing/overwriting would require the two declarations to be in the *same* scope, and they never are — two renders are two separate invocations.

The same name in source code says nothing about identity at runtime. That's the whole reason `useCallback` exists: it stashes the function object in React's internal hook state for that component instance and, as long as the dep array is unchanged, hands back the *same object* the next time the function body runs.

One caveat worth noticing in this particular hook, though: it returns a fresh array literal every render, so any consumer doing

```js
const user = useCurrentUser()
useEffect(() => { ... }, [user])
```

still gets a new value each time. The memoization only pays off when the setter is destructured out and used as a dep or passed to a `React.memo` child:

```js
const [, , setCurrentUser] = useCurrentUser()
// stable across renders — safe in deps, won't re-render memo'd children
```

Which is the normal usage, so it's fine — but the `useCallback` is buying you something only at that level, not for the tuple as a whole.


# 2. `client.js`

## Lesson 1 - request PATH & custom error handling abstraction & tip of asynchronous function

Absolutely. And `client.js` is where I think things get especially useful because we're leaving React temporarily and learning the **JavaScript/browser machinery that connects React to the FastAPI system you already understand**.

I don't want to jump immediately into `request()`. There's enough packed into it—`async`, `await`, `fetch`, Promises, `throw`, HTTP responses—that we'd risk explaining one unfamiliar thing using three other unfamiliar things.

Instead, let's work downward. The file has four conceptual layers:

```text
Configuration
    BASE
      ↓
Error abstraction
    ApiError
    toApiError()
      ↓
Generic HTTP machinery
    request()
      ↓
Application API
    listClients()
    createClient()
    updateClient()
    deleteClient()
```

That's exactly consistent with its stated purpose: a "thin wrapper over the FastAPI endpoints." 

We'll start with `BASE`, but only spend as much time as it deserves.

---

### 1. `BASE` — where are requests going?

```js
const BASE = (import.meta.env.VITE_API_BASE ?? '').replace(/\/$/, '') || '/api'
```



This line looks much scarier than what it's actually doing.

Let's break it apart:

```text
import.meta.env.VITE_API_BASE
            ↓
           ?? ''
            ↓
     .replace(/\/$/, '')
            ↓
         || '/api'
```

You already encountered:

```js
??
```

in `useCurrentUser.js`.

So:

```js
import.meta.env.VITE_API_BASE ?? ''
```

means:

> If `VITE_API_BASE` isn't `null` or `undefined`, use it. Otherwise use `""`.

`import.meta.env` here is **Vite machinery**, not React.

Vite exposes environment variables beginning with `VITE_` through:

```js
import.meta.env
```

So you might configure something conceptually like:

```text
VITE_API_BASE=http://127.0.0.1:8000
```

and then:

```js
import.meta.env.VITE_API_BASE
```

would provide that value.

---

### 2. What's this `.replace(/\/$/, '')` monstrosity?

```js
.replace(/\/$/, '')
```

This is ordinary JavaScript string replacement using a **regular expression**.

Don't spend much learning bandwidth on the regex itself right now.

It means essentially:

> Remove one `/` if it appears at the end of the string.

So:

```text
"http://127.0.0.1:8000/"
                 ↓
"http://127.0.0.1:8000"
```

Why?

Because later the code does:

```js
`${BASE}${path}`
```

If:

```text
BASE = http://127.0.0.1:8000/
path = /client
```

you'd get:

```text
http://127.0.0.1:8000//client
```

Removing the trailing slash normalizes it:

```text
http://127.0.0.1:8000
                      +
/client
                      ↓
http://127.0.0.1:8000/client
```

Tiny boundary-normalization detail.

---

### 3. But what's `|| '/api'`?

This is worth understanding because `||` and the `??` we just learned look similar but **are not equivalent**.

Remember:

```js
x ?? y
```

means:

> Use `y` only if `x` is `null` or `undefined`.

Whereas:

```js
x || y
```

means:

> Use `y` if `x` is **falsy**.

And JavaScript considers things such as these falsy:

```js
false
0
''
null
undefined
NaN
```

So if there is no configured base:

```js
import.meta.env.VITE_API_BASE
        ↓
undefined

undefined ?? ''
        ↓
''

''.replace(...)
        ↓
''

'' || '/api'
        ↓
'/api'
```

Therefore the default is:

```js
BASE = '/api'
```

That's what the comment at the top is describing: Vite proxies `/api/...` to your locally running FastAPI server. 

So ultimately there are two possible architectures:

```text
DEFAULT DEVELOPMENT SETUP

React/browser
     │
     │ /api/client
     ↓
Vite dev server
     │
     │ proxy
     ↓
FastAPI :8000
```

or, if `VITE_API_BASE` is explicitly configured:

```text
React/browser
     │
     │ http://127.0.0.1:8000/client
     ↓
FastAPI directly
```

The direct version involves CORS, hence the comment about FastAPI enabling CORS for `:5173`.

We don't need to go down the CORS rabbit hole yet.

---

### 4. Now we get an abstraction you should find very familiar

```js
export class ApiError extends Error {
  constructor(message, status, options) {
    super(message, options)
    this.name = 'ApiError'
    this.status = status
  }
}
```



There's no React here.

This is **JavaScript object-oriented syntax**.

And conceptually, it's doing something extremely straightforward:

> Create our own specialized kind of JavaScript error that carries HTTP information.

Normally JavaScript has:

```js
Error
```

Your application defines:

```js
ApiError
```

with:

```js
class ApiError extends Error
```

meaning:

```text
          Error
            ↑
            │ inherits from
            │
         ApiError
```

So an `ApiError` is still an `Error`, but it carries something extra:

```js
status
```

---

### 5. Why does this abstraction exist?

Imagine FastAPI returns:

```text
HTTP 409 Conflict

{
    "detail": "Client name already exists"
}
```

Your frontend could throw a generic:

```js
new Error("Client name already exists")
```

But then you've lost something useful:

```text
409
```

The caller might want to distinguish:

```text
404 → client doesn't exist

409 → conflict / duplicate client name

422 → validation problem

500 → backend failure
```

So this application creates:

```js
new ApiError(
    "Client name already exists",
    409
)
```

Now the object conceptually contains:

```text
ApiError
├── message = "Client name already exists"
├── status  = 409
├── name    = "ApiError"
└── other normal Error behavior
```

This is why the comment says callers can special-case `404 / 409`. 

---

### 6. Let's unpack the constructor syntax

```js
constructor(message, status, options) {
```

When somebody writes:

```js
new ApiError("Client already exists", 409)
```

JavaScript effectively executes:

```text
message = "Client already exists"
status  = 409
options = undefined
```

Then:

```js
super(message, options)
```

calls the parent `Error` constructor.

Remember:

```js
class ApiError extends Error
```

So `super(...)` essentially says:

> Let `Error` initialize the normal Error machinery first.

Conceptually:

```text
new ApiError(...)
       │
       ↓
ApiError constructor
       │
       ↓
super(message, options)
       │
       ↓
Error constructor
```

Then:

```js
this.name = 'ApiError'
this.status = status
```

adds/sets properties on the newly created object.

`this` basically means:

> **this particular `ApiError` object currently being constructed.**

So:

```js
const error = new ApiError("Duplicate client", 409)
```

gives you something conceptually like:

```js
error.message
// "Duplicate client"

error.status
// 409

error.name
// "ApiError"
```

---

### 7. There's a design pattern here worth noticing

Your FastAPI backend has its own representation of failure:

```text
FastAPI
   │
   ↓
HTTP response
   │
   ├── status = 409
   │
   └── JSON:
       {
          "detail": "Client name already exists"
       }
```

Your React application doesn't necessarily want every component dealing directly with that representation.

So `client.js` is beginning to establish a boundary:

```text
        FASTAPI WORLD
             │
             │ HTTP response
             ↓
         client.js
             │
             │ translate
             ↓
       FRONTEND WORLD

ApiError {
    message,
    status
}
```

That's why calling this a **thin wrapper** is useful.

Your components shouldn't ideally need to know:

```js
response.json()
response.status
body.detail
```

They should be able to say something closer to:

```js
createClient(...)
```

and receive either:

```text
Client
```

or:

```text
ApiError
```

That is a much narrower contract.

---

### 8. And now we reach our first major JavaScript asynchronous function

The next function is:

```js
async function toApiError(response) {
  let detail

  try {
    const body = await response.json()
    detail = body?.detail
  } catch {
    // body was empty or not JSON
  }

  ...
}
```



I want to stop here rather than explain the remaining 70 lines.

Because this introduces the machinery that you specifically wanted to understand:

```js
async
await
response.json()
```

and underneath all three sits the concept you mentioned at the beginning:

---
### **Promise**

Before explaining `async` or `await`, I think we should build **Promise from first principles**.

Because otherwise you'll learn rules like:

> "`await` waits for a Promise."

and immediately have the question:

> **Okay... what exactly is a Promise, why do we need one, and what does "wait" actually mean?**

And that's the part worth understanding.

Our next step should therefore temporarily leave `client.js` syntax behind and answer one fundamental problem:

> **JavaScript calls FastAPI over the network. FastAPI might respond in 2 ms, 200 ms, 5 seconds, or never. What should JavaScript do during that unknown amount of time?**

Once we understand that problem, **Promise → `.then()` → `.catch()` → `async` → `await`** becomes a coherent sequence rather than five pieces of syntax to memorize.

### 9. Aside: details of origins, CORS, proxy

## The core idea: origins

A browser identifies a server by its **origin** — the triple `(scheme, host, port)`. So:

- `http://localhost:5173` — your React app
- `http://127.0.0.1:8000` — your FastAPI server

Those are **different origins** (different port, and also different host string — `localhost` and `127.0.0.1` are not the same origin to the browser even though they resolve to the same machine).

The browser enforces the **same-origin policy**: JavaScript loaded from origin A can *send* a request to origin B, but the browser will hide the response from your code unless origin B explicitly says "I allow A to read this." That opt-in mechanism is CORS.

## Why relative URLs matter

In your `request()`:

```js
await fetch(`${BASE}${path}`)
```

If `BASE` is `/api`, the URL is `/api/client` — a **relative** URL. The browser resolves it against the page's own origin, so it actually requests `http://localhost:5173/api/client`. Same origin as the page ⇒ no CORS involved at all.

If `BASE` is `http://127.0.0.1:8000` (via `VITE_API_BASE`), the URL is absolute and points at a different origin ⇒ CORS applies.

## What the Vite proxy does

The Vite dev server isn't just serving your JS bundle; it's a real HTTP server, and you can tell it to forward certain paths elsewhere. In `vite.config.js` there's something like:

```js
server: {
  proxy: {
    '/api': {
      target: 'http://127.0.0.1:8000',
      rewrite: (p) => p.replace(/^\/api/, ''),
    },
  },
}
```

So the flow is:

```
browser ──GET /api/client──> Vite (:5173) ──GET /client──> FastAPI (:8000)
                                          <──── JSON ─────
        <──────── JSON ────────
```

The `rewrite` strips the `/api` prefix, which is why your FastAPI routes are just `/client`, not `/api/client`. The `/api` prefix exists purely as a marker so Vite knows which requests to forward versus which to serve as React app files.

Key point: the proxy hop from Vite to FastAPI is **server-to-server**. No browser, no same-origin policy, no CORS. As far as the browser is concerned it only ever talked to `:5173`.

## Where CORS comes in

CORS is the fallback path for when you *don't* use the proxy. Setting `VITE_API_BASE=http://127.0.0.1:8000` makes the frontend talk to FastAPI directly. Now the browser is making a cross-origin request, and FastAPI has to opt in — presumably `serve/api.py` has something like:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

That's what the comment means by "CORS is enabled there for `:5173`" — the backend has whitelisted the dev server's origin specifically.

There's a wrinkle that matters for your code: your `PATCH` and `DELETE` calls send `Content-Type: application/json`. That makes them **non-simple** requests, so before the real request the browser fires a `OPTIONS` **preflight** asking the server whether that method and header are permitted. If the server doesn't answer the preflight correctly, the actual PATCH never leaves the browser and you get a CORS error in the console — even though the endpoint itself is perfectly fine. This is a common source of "it works in curl but not in the browser" confusion. With the proxy, none of this happens.

## Why default to the proxy

Two practical reasons:

1. **Same-origin in dev mirrors same-origin in prod.** In production you'll typically serve the built static files and the API behind one domain (nginx routing `/api/*` to FastAPI, everything else to the static bundle). Using relative `/api` URLs means the same frontend code works in both places with zero configuration.
2. **Cookies.** Same-origin requests carry cookies naturally. Cross-origin ones need `credentials: 'include'` on the fetch plus `allow_credentials=True` and an explicit (non-wildcard) origin on the server. Not relevant to this file yet, but it bites the moment you add auth.

`VITE_API_BASE` is the escape hatch: useful if you want to point the frontend at a deployed/staging backend, or if you're running the frontend from something that isn't the Vite dev server.

## Reading the error message

```js
throw new ApiError(
  'Could not reach the API. Is the FastAPI server running on port 8000?',
  0,
)
```

`fetch` only rejects on network-level failures — connection refused, DNS failure, or a blocked CORS request. It does *not* reject on 404 or 500; those come back as a normal `response` with `ok === false`, which is why the 4xx/5xx handling is separate below it. Status `0` here is a convention meaning "no HTTP response ever happened."

In proxy mode, that error usually means Vite got connection-refused forwarding to `:8000` — so the message's advice is right. In direct mode it could also be a CORS rejection, where the server is running fine but the browser threw the response away.

## Lesson 2 (away from `client.js`) - 

Good. Let's build the asynchronous model first, because once this clicks, a surprisingly large amount of `client.js` becomes ordinary code.

### 1. The underlying problem: FastAPI doesn't answer instantly

Eventually `client.js` does this:

```js
response = await fetch(`${BASE}${path}`, {
    method,
    headers: body === undefined ? {} : { 'Content-Type': 'application/json' },
    body: body === undefined ? undefined : JSON.stringify(body),
})
```



Forget `await` for now. Focus on `fetch()`.

Suppose the browser sends:

```text
GET /client
```

to your FastAPI application.

There is necessarily a period like this:

```text
Browser                         FastAPI
   │                               │
   │────── GET /client ───────────>│
   │                               │
   │                               │ query DB
   │          ???                  │ serialize response
   │                               │
   │<────── 200 + JSON ────────────│
   │
```

What should JavaScript do during `???`?

A naïve model would be:

```text
Call fetch()
    ↓
STOP EVERYTHING
    ↓
wait...
    ↓
wait...
    ↓
FastAPI responds
    ↓
continue JavaScript
```

That would be terrible in a browser.

Imagine FastAPI takes five seconds. If JavaScript simply blocked the browser's main thread waiting for network I/O, the interface could become unresponsive.

So browser network operations are asynchronous.

Conceptually, JavaScript says:

> Start this network operation, and give me something representing the result that I will receive **in the future**.

That "something" is a **Promise**.

---

### 2. A Promise is a representation of a future result

Suppose we do:

```js
const result = fetch('/api/client')
```

A common beginner intuition is:

```text
result = FastAPI response
```

No.

`fetch()` can't immediately give you the response because it doesn't have it yet.

Instead:

```text
result = Promise
```

Conceptually:

```text
fetch('/api/client')
       │
       ├───────────────> browser starts HTTP request
       │
       ↓
    Promise
       │
       │ "I'll eventually tell you
       │  how this operation went."
       ↓
     result
```

So a Promise is essentially an object representing:

> **an asynchronous operation whose outcome isn't available yet.**

---

### 3. A Promise has states

The useful simplified model is:

```text
                  Promise
                     │
                 PENDING
                /       \
               /         \
              ↓           ↓
         FULFILLED      REJECTED
              │           │
           result        error
```

When `fetch()` first returns:

```text
Promise: pending
```

The HTTP request is underway.

Later, if the operation succeeds:

```text
Promise: fulfilled
value: Response object
```

Or if the network operation itself fails:

```text
Promise: rejected
reason: some error
```

Once a Promise is fulfilled or rejected, we say it is **settled**.

So:

```text
pending
   ↓
fulfilled

OR

pending
   ↓
rejected
```

It doesn't go backward.

---

### 4. `.then()` answers: "What should happen when it succeeds?"

Before `async/await` became common, you'd frequently interact with Promises like this:

```js
fetch('/api/client')
    .then((response) => {
        console.log(response)
    })
```

Read that as:

> Start the request. **When the Promise fulfills**, call this function with the resulting value.

Remember our arrow functions:

```js
(response) => {
    console.log(response)
}
```

is just a function receiving one parameter.

So:

```text
fetch('/api/client')
        ↓
Promise pending

JavaScript continues doing other work
        ↓
...
        ↓
FastAPI responds
        ↓
Promise fulfills
        ↓
Response object
        ↓
pass Response into callback
        ↓

(response) => {
    console.log(response)
}
```

That is the basic meaning of `.then()`.

---

### 5. `.catch()` answers the opposite question

We could write:

```js
fetch('/api/client')
    .then((response) => {
        console.log(response)
    })
    .catch((error) => {
        console.log(error)
    })
```

Conceptually:

```text
                 fetch()
                    │
                 Promise
                /       \
               /         \
        fulfilled        rejected
            │                │
            ↓                ↓
         .then()          .catch()
            │                │
       response            error
```

So these aren't magical React constructs.

They are JavaScript **Promise APIs**.

---

### 6. Here's something extremely important: `fetch()` has a particular idea of "success"

Suppose your FastAPI endpoint responds:

```text
HTTP 404 Not Found
```

You might expect:

```text
fetch Promise
      ↓
   rejected
      ↓
   .catch()
```

But that's generally **not** what happens.

The browser successfully made an HTTP request and successfully received an HTTP response.

So `fetch()` fulfills its Promise with a `Response` object even for HTTP statuses such as:

```text
400
404
409
422
500
```

This distinction explains a major piece of your `request()` function.

Your code has:

```js
if (!response.ok) throw await toApiError(response)
```



Because these are two different kinds of failure:

```text
NETWORK/TRANSPORT FAILURE

Browser can't reach FastAPI
        ↓
fetch Promise rejects
        ↓
catch (cause)


HTTP APPLICATION FAILURE

Browser reaches FastAPI
        ↓
FastAPI intentionally responds 409
        ↓
fetch Promise fulfills!
        ↓
Response.status = 409
Response.ok = false
        ↓
your code detects !response.ok
        ↓
your code throws ApiError
```

That distinction is fundamental to understanding this entire wrapper.

---

### 7. Now another twist: `.then()` itself returns a Promise

Suppose you write:

```js
fetch('/api/client')
    .then((response) => response.json())
    .then((data) => console.log(data))
```

Why can we have **two `.then()` calls**?

Because:

```js
somePromise.then(...)
```

returns another Promise.

So it's actually a chain:

```text
fetch(...)
    ↓
Promise #1
    ↓
.then(response => response.json())
    ↓
Promise #2
    ↓
.then(data => console.log(data))
    ↓
Promise #3
```

This is why Promise code can be chained.

And there's another important fact hiding here:

```js
response.json()
```

is itself asynchronous and returns a Promise.

Why?

Because receiving an HTTP `Response` and consuming/parsing its body aren't represented as the same operation in the Fetch API.

So:

```js
response.json()
```

does **not** immediately mean:

```text
give me JavaScript object
```

It means roughly:

> Read the response body and parse it as JSON, giving me a Promise for the resulting JavaScript value.

Therefore:

```js
response.json()
```

returns:

```text
Promise
```

which eventually fulfills with something like:

```js
[
    {
        client_id: 1,
        client_name: "Farm A",
        ...
    },
    {
        client_id: 2,
        client_name: "Farm B",
        ...
    }
]
```

---

### 8. Let's connect this to FastAPI

Suppose your endpoint returns:

```json
[
    {
        "client_id": 1,
        "client_name": "ABC Farms"
    }
]
```

The journey is approximately:

```text
FastAPI/Pydantic
      ↓
serialize response
      ↓
HTTP response body
      ↓
bytes/text travelling over HTTP
      ↓
browser Response
      ↓
response.json()
      ↓
Promise
      ↓
JSON parsing
      ↓
JavaScript value
      ↓
[
  {
    client_id: 1,
    client_name: "ABC Farms"
  }
]
```

This is an important boundary.

**HTTP JSON is not itself a JavaScript object.**

The browser receives an HTTP response body. `response.json()` consumes that body and produces the corresponding JavaScript value asynchronously.

---

### 9. Now we can understand `async` and `await`

Promise chaining works:

```js
fetch('/api/client')
    .then((response) => response.json())
    .then((data) => {
        console.log(data)
    })
    .catch((error) => {
        console.log(error)
    })
```

But it can become difficult to read when workflows get more complicated.

JavaScript gives us another syntax for working with Promises:

```js
async
await
```

So instead of:

```js
fetch('/api/client')
    .then((response) => {
        ...
    })
```

we can often write:

```js
const response = await fetch('/api/client')
```

Now here's the mental model I want you to be careful with.

`await` does **not** mean:

> Freeze the entire JavaScript/browser environment until FastAPI responds.

Rather, conceptually:

> This async function cannot proceed past this point until this Promise settles. Yield control so other work can continue; resume this function when the result becomes available.

Think:

```text
async function request() {

    code A
       ↓
    await fetch(...)
       │
       ├──── Promise pending ────────┐
       │                             │
       │    this function pauses     │
       │                             │
       │    browser/JS can do        │
       │    other work               │
       │                             │
       │<── Promise fulfills ────────┘
       ↓
    code B

}
```

So **the function is suspended**, not the entire browser.

That distinction matters enormously.

---

### 10. `await` essentially unwraps the Promise result for us

Without `await`:

```js
const response = fetch('/api/client')
```

means:

```text
response = Promise
```

With:

```js
const response = await fetch('/api/client')
```

conceptually:

```text
fetch(...)
    ↓
Promise<Response>
    ↓
await
    ↓
Response
    ↓
response
```

That's why after this line your code can do:

```js
response.ok
response.status
response.statusText
response.json()
```

because `response` is now the actual Fetch `Response` object, not the Promise representing its future arrival.

---

### 11. Why does the containing function need `async`?

You can't ordinarily put:

```js
await ...
```

inside an arbitrary normal function.

So your code declares:

```js
async function request(...) {
```

and:

```js
async function toApiError(...) {
```

 

This tells JavaScript:

> This function performs asynchronous work and may use `await`.

But there's an even more important consequence:

### An `async` function itself returns a Promise.

This is the piece that connects everything.

Suppose:

```js
async function example() {
    return 42
}
```

Calling:

```js
example()
```

does **not** directly return:

```text
42
```

It returns conceptually:

```text
Promise
    ↓ eventually fulfilled with
42
```

So:

```js
const x = example()
```

means:

```text
x = Promise
```

while:

```js
const x = await example()
```

eventually gives:

```text
x = 42
```

---

### 12. And now suddenly the bottom of `client.js` makes sense

Look at:

```js
export function listClients() {
  return request('/client')
}
```



Notice:

```js
listClients()
```

is **not declared `async`**.

But that's completely fine.

Why?

Because:

```js
request('/client')
```

already returns a Promise, since `request()` is an `async` function.

Therefore:

```text
listClients()
      ↓
request('/client')
      ↓
Promise
      ↓
listClients returns that Promise
```

This is why later React code can presumably do either:

```js
listClients()
    .then(...)
    .catch(...)
```

or:

```js
const clients = await listClients()
```

Same Promise.

Two different ways of consuming it.

---

### 13. `.then()` and `await` are therefore not two unrelated mechanisms

This is the connection I especially want you to retain.

Given:

```js
const promise = listClients()
```

you can consume it with:

```js
promise.then((clients) => {
    console.log(clients)
})
```

or, inside an async function:

```js
const clients = await promise
console.log(clients)
```

These are two styles built around the **same underlying Promise mechanism**.

Conceptually:

```text
                   Promise<Client[]>
                          │
              ┌───────────┴───────────┐
              ↓                       ↓
          .then(...)                 await
              │                       │
              ↓                       ↓
       callback receives       function resumes with
          Client[]                  Client[]
```

So don't build two mental models called "Promise chaining" and "async programming."

Build **one**:

> asynchronous operations are represented by Promises.

Then JavaScript gives you different syntax for consuming them:

```text
Promise
├── .then()
├── .catch()
└── async/await
```

---

### 14. Now return to `toApiError()`

We can finally read the beginning without hand-waving:

```js
async function toApiError(response) {
  let detail
  try {
    const body = await response.json()
    detail = body?.detail
  } catch {
    // body was empty or not JSON
  }
```



Let's mentally execute a real FastAPI error.

FastAPI sends:

```json
{
    "detail": "Client name already exists"
}
```

We already have:

```text
response = Fetch Response object
```

Then:

```js
response.json()
```

returns:

```text
Promise<parsed body>
```

Then:

```js
await response.json()
```

suspends `toApiError()` until parsing completes.

When it fulfills:

```js
const body = {
    detail: "Client name already exists"
}
```

Then:

```js
detail = body?.detail
```

gives:

```js
detail = "Client name already exists"
```

---

### 15. `?.` is another JavaScript operator

This:

```js
body?.detail
```

is **optional chaining**.

Very roughly:

```js
body?.detail
```

means:

> If `body` isn't `null` or `undefined`, access `.detail`; otherwise produce `undefined` instead of crashing.

For example:

```js
const body = {
    detail: "Duplicate client"
}

body?.detail
// "Duplicate client"
```

versus:

```js
const body = null

body?.detail
// undefined
```

Whereas:

```js
body.detail
```

when `body === null` would throw an error.

This is another defensive boundary-normalization detail.

---

### 16. Why the `try/catch` around `response.json()`?

Suppose FastAPI—or a proxy/server somewhere between you and FastAPI—responded with something that wasn't JSON.

Perhaps an empty body.

Then:

```js
await response.json()
```

can reject/throw.

And here's another major connection:

With Promise syntax you might think:

```text
Promise rejected
      ↓
.catch(...)
```

With `await`, rejection behaves like a thrown exception at the `await` point:

```js
try {
    const body = await response.json()
} catch {
    ...
}
```

So:

```text
Promise style                 async/await style

something()                   try {
   .then(...)                     await something()
   .catch(...)                 } catch {
                                   ...
                               }
```

Again: **same underlying asynchronous failure, different syntax for handling it.**

---

And I want to stop here, because we've now built the machinery necessary to understand the rest of this file rather than merely translating syntax.

Our current model is:

```text
FastAPI
   ↓
HTTP
   ↓
fetch()
   ↓
Promise<Response>
   ↓
await
   ↓
Response
   ↓
response.json()
   ↓
Promise<JavaScript value>
   ↓
await
   ↓
JavaScript object/array
```

And for failure:

```text
NETWORK FAILURE
fetch Promise rejects
        ↓
await throws
        ↓
catch


HTTP 409 / 404 / 422 / 500
fetch Promise fulfills
        ↓
Response.ok === false
        ↓
our code deliberately converts it
        ↓
ApiError
```

That second distinction is going to become **the heart of `request()`**.

Next, I'd continue through the rest of `toApiError()` line by line—especially this beast:

```js
detail = detail
  .map((e) => `${(e.loc ?? []).slice(1).join('.')}: ${e.msg}`)
  .join('; ')
```

because it's a beautiful little piece of JavaScript once decomposed, and it's directly translating **Pydantic's structured validation errors into a human-readable frontend error message**. 


## Lesson 3 (back to `client.js`) -

# 3. `UserBar.jsx`



# 4. `ClientRow.jsx`



# 5. `ClientTable.jsx`



# 6. `AddClientForm.jsx`


# 7. `App.jsx`


# Code Appendix

## 1. `useCurrentUser.js`

```js
import { useCallback, useState } from 'react'

const STORAGE_KEY = 'experiment.currentUser'

function read() {
  try {
    return localStorage.getItem(STORAGE_KEY) ?? ''
  } catch {
    return ''
  }
}

/**
 * The name sent as `modified_by` on every write. There's no auth yet, so this
 * is just a self-declared identity persisted in localStorage.
 */
export function useCurrentUser() {
  const [currentUser, setState] = useState(read)

  const setCurrentUser = useCallback((value) => {
    setState(value)
    try {
      if (value) localStorage.setItem(STORAGE_KEY, value)
      else localStorage.removeItem(STORAGE_KEY)
    } catch {
      // private mode / storage disabled — in-memory state still works
    }
  }, [])

  return [currentUser.trim(), currentUser, setCurrentUser]
}
```

## 2. `client.js`

```js
/**
 * Thin wrapper over the FastAPI endpoints in serve/api.py.
 *
 * By default requests go to /api/... which the Vite dev server proxies to
 * http://127.0.0.1:8000 (see vite.config.js). Set VITE_API_BASE to call the
 * backend directly instead — CORS is enabled there for :5173.
 */

const BASE = (import.meta.env.VITE_API_BASE ?? '').replace(/\/$/, '') || '/api'

/** Error carrying the HTTP status so callers can special-case 404 / 409. */
export class ApiError extends Error {
  constructor(message, status, options) {
    super(message, options)
    this.name = 'ApiError'
    this.status = status
  }
}

/** Pull FastAPI's `detail` out of an error body; fall back to the status text. */
async function toApiError(response) {
  let detail
  try {
    const body = await response.json()
    detail = body?.detail
  } catch {
    // body was empty or not JSON
  }

  if (Array.isArray(detail)) {
    // pydantic validation errors
    detail = detail
      .map((e) => `${(e.loc ?? []).slice(1).join('.')}: ${e.msg}`)
      .join('; ')
  }

  const message =
    (typeof detail === 'string' && detail.trim()) ||
    `${response.status} ${response.statusText}`

  return new ApiError(message, response.status)
}

async function request(path, { method = 'GET', body } = {}) {
  let response
  try {
    response = await fetch(`${BASE}${path}`, {
      method,
      headers: body === undefined ? {} : { 'Content-Type': 'application/json' },
      body: body === undefined ? undefined : JSON.stringify(body),
    })
  } catch (cause) {
    throw new ApiError(
      'Could not reach the API. Is the FastAPI server running on port 8000?',
      0,
      { cause },
    )
  }

  if (!response.ok) throw await toApiError(response)
  if (response.status === 204) return null
  return response.json()
}

/** GET /client -> ClientRead[], newest-modified first. */
export function listClients() {
  return request('/client')
}

/** POST /client -> ClientRead. 409 if the name is taken. */
export function createClient({ clientName, modifiedBy }) {
  return request('/client', {
    method: 'POST',
    body: { client_name: clientName, modified_by: modifiedBy },
  })
}

/** PATCH /client/{id} -> ClientRead. 404 if missing, 409 if the name is taken. */
export function updateClient({ clientId, newClientName, modifiedBy }) {
  return request(`/client/${clientId}`, {
    method: 'PATCH',
    body: { new_client_name: newClientName, modified_by: modifiedBy },
  })
}

/** DELETE /client/{id} -> null. 404 if missing. */
export function deleteClient({ clientId, modifiedBy }) {
  return request(`/client/${clientId}`, {
    method: 'DELETE',
    body: { modified_by: modifiedBy },
  })
}
```

# 3