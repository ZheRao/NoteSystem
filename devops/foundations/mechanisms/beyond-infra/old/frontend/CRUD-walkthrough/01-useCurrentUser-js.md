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


## Lesson 3 - JavaScript function & `useCallback` & dependency array

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


## Code Appendix
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