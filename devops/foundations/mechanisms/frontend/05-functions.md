# Functions in JavaScript and React: references, calls, and timing

## What a function is

In JavaScript a function is a **value**, just like a number or an object. You can store it in a variable, pass it as an argument, return it from another function, or put it in an object. Almost every function bug in React comes from mixing up two things: **the function itself** and **the result of running it**.

```js
const greet = () => 'hi'

greet      // the function itself (a value you can hand around)
greet()    // runs it now → 'hi'
```

**The rule: parentheses mean "run it now"; no parentheses mean "here is the function."**

## Defining functions

```js
function add(a, b) { return a + b }   // function declaration
const add = (a, b) => a + b           // arrow function, implicit return
const add = (a, b) => { return a + b } // arrow function with a body: needs `return`
```

**The arrow return trap** (JavaScript language):

| Written as | Returns |
|---|---|
| `x => x * 2` | `x * 2` (an expression body returns implicitly) |
| `x => { x * 2 }` | `undefined` (curly braces start a *body*, and there's no `return`) |
| `() => { a: 1 }` | `undefined` (the braces are read as a body, `a:` as a label) |
| `() => ({ a: 1 })` | the object `{ a: 1 }` (parentheses force an expression) |

## Arguments and parameters

```js
const f = (a, b) => [a, b]
f(1)        // [1, undefined]  ← missing arguments become undefined, no error
f(1, 2, 3)  // [1, 2]          ← extra arguments are silently ignored
```

JavaScript never complains about argument counts, so a dropped argument shows up later as an `undefined` somewhere else, far from the cause. That's what happened with `() => submitAdd()`: the event was dropped, and the crash surfaced inside `submitAdd`.

## Functions that return functions (factories)

```js
const makeGreeter = (name) => () => `hi ${name}`

makeGreeter('Zhe')     // returns a function; nothing is greeted yet
makeGreeter('Zhe')()   // 'hi Zhe'
```

A factory's result must itself be called or passed on. If you call a factory and ignore what it returns, nothing happens, and there's no error to tell you so.

## Closures: functions remember where they were created

A function can read the variables that were in scope **where it was defined**, even when it runs later (this is a closure, a JavaScript language feature).

In React, every render calls your component function again, so every render creates **new** handlers that close over **that render's** state values:

```jsx
const [clientId, setClientId] = useState(null)   // this render's snapshot

const enterEdit = (client) => {
  setClientId(client.id)   // schedules the next render
  console.log(clientId)    // still this render's snapshot → null
}
```

- A setter never changes the current variable. It schedules a new render with new variables.
- A handler defined in the component body is recreated every render, so it always sees the snapshot of the render the user is looking at. That's why reading `error` inside `dismissError` is safe.
- A function **stored somewhere long-lived** (in state, in a ref, or registered once in an effect) keeps the snapshot from when it was *created*, which goes stale. That's why functions don't belong in state.

## Methods and `this`

When a function is called *on* an object, `this` refers to that object:

```js
dialog.close()          // `this` is dialog ✅
const c = dialog.close  // detached: just the function, the link to dialog is gone
c()                     // `this` is missing → TypeError: Illegal invocation
```

Built-in DOM methods (`close`, `showModal`, `focus`, `addEventListener`) need their `this`. To pass one as a callback, wrap it: `() => dialog.close()`.

## Timing: render time vs event time (React)

Everything inside JSX `{ }` is evaluated **while the component renders**. A function's *body* runs only **when it's called**.

```jsx
onClick={dialogRef.current.close}           // evaluated at RENDER: ref may be null
onClick={() => dialogRef.current.close()}   // body evaluated at CLICK: ref is attached
onClick={save()}                            // save runs at RENDER, every render ❌
```

A wrapper `() => ...` has three possible effects. Know which one you want:

| Wrapper effect | Example | Good or bad |
|---|---|---|
| **Delays** evaluation to event time | `() => dialogRef.current.close()` | good |
| **Keeps `this`** by calling on the object | `() => dialog.close()` | good |
| **Adds or fixes arguments** | `() => enterEdit(client)` | good |
| **Drops arguments** you needed | `() => submitAdd()` (event lost) | bad |
| **Discards a factory's result** | `() => exitError(type)` | bad |

## Event handler props: what React receives

| Written as | React gets | When it runs |
|---|---|---|
| `onClick={fn}` | `fn` | on click, as `fn(event)` ✅ |
| `onClick={fn()}` | `fn`'s return value | at render ❌ |
| `onClick={() => fn()}` | a wrapper | on click, but `fn` gets no event |
| `onClick={(e) => fn(e)}` | a wrapper | on click, event forwarded ✅ |
| `onClick={() => fn(item)}` | a wrapper | on click, with a fixed argument ✅ (per-row handlers) |
| `onClick={() => makeFn()}` | a wrapper | on click, product discarded ❌ |

## Functions passed to setters (updater functions)

```jsx
setDraft(prev => ({ ...prev, client_name: value }))
```

When you pass a function to a state setter, React calls it **with the latest state** and uses the return value. Use it whenever the new state depends on the old one. It also combines two traps from above: the parentheses around the object literal, and returning a **new** object rather than mutating `prev`.

## Handlers can run more than once

With dialogs, an exit can run twice: once directly, then again through the `close` event. A handler that might run again should be:

- **idempotent:** running it twice gives the same result as running it once
- **null-tolerant:** it reads possibly-cleared state with `?.`

## Pitfalls hit in this project

1. `setDraft({ client_name: value })` replaced the whole object (no merge), so `client_id` was lost.
2. `console.log(clientId)` right after `setClientId(...)` printed the old snapshot.
3. `onSave={() => submitAdd()}` dropped the event, `preventDefault` threw, and the page reloaded.
4. `onClick={dialogRef.current.close}` read a `null` ref at render, and the method was detached from `this`.
5. `onClose={() => exitError(type)}` called a factory and discarded its result, so nothing happened.
6. `error.type` in a handler that ran a second time, after `error` had been cleared, would throw a `TypeError`.
7. `onClose` stored inside the `error` state put a closure where only data belongs, which risks a stale snapshot.

## Tiers

- **Tier 1 (mechanical):** reference vs call; render time vs event time; what each handler prop form receives; setters schedule, they don't mutate; updater functions; the arrow-returning-object parentheses.
- **Tier 2 (purpose and failure modes):** closures and render snapshots; stale closures in long-lived places; `this` and detached methods; factories; idempotent, null-tolerant handlers.
- **Tier 3 (look up when needed):** `bind`, `call`/`apply`, how `this` works in `function` vs arrow functions, `useCallback` (memoising handlers for performance).