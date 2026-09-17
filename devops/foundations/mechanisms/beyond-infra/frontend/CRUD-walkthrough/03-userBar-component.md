# 3. `UserBar.jsx`

## Lesson 1 - React props & default/named export & declarative return

Yes. `UserBar.jsx` is only 21 lines, but it introduces several foundational React ideas at once. We'll keep the same slow approach and not explain the entire thing immediately.

Here is its overall shape:

```jsx
export default function UserBar({ value, onChange }) {
  return (
    <div className="user-bar">
      ...
    </div>
  )
}
```



For now, I want to focus only on that outer shell, because it answers a fundamental question:

> **What actually is a React component?**

### 1. Strip away React for a moment

Start with:

```js
function UserBar({ value, onChange }) {
    ...
}
```

At the JavaScript level, `UserBar` is just a **function**.

You've written plenty of functions:

```python
def create_client(client_name, modified_by):
    ...
```

or JavaScript:

```js
function createClient({ clientName, modifiedBy }) {
    ...
}
```

Now we have:

```js
function UserBar({ value, onChange }) {
    ...
}
```

The special React idea isn't that React invented a new kind of function syntax.

React's convention is essentially:

> A component is a function React can call to determine what UI that component should currently represent.

So mentally:

```text
UserBar
   │
   │ ordinary JS function
   │
   │ receives inputs
   ↓
{ value, onChange }
   │
   │ computes UI
   ↓
JSX
```

That gives us a very useful first approximation:

> **React component ≈ function from inputs to UI description.**

We'll refine that model later because state, effects, and rendering introduce important subtleties.

---

### 2. What are the inputs?

This part:

```js
{ value, onChange }
```

should look familiar after `client.js`.

It's object destructuring.

Conceptually React will provide the component an object containing its **props**:

```js
{
    value: ...,
    onChange: ...
}
```

The component immediately destructures it:

```js
function UserBar({ value, onChange })
```

which is approximately:

```js
function UserBar(props) {
    const value = props.value
    const onChange = props.onChange
}
```

So **props are not mysterious React variables**.

At the JavaScript level, they're values passed into your component through an object.

React gives that object the conceptual role of:

> **props = inputs supplied to this component by its parent.**

---

### 3. Where do these props actually come from?

We haven't studied the parent yet, so we shouldn't invent the exact caller.

But eventually somewhere higher in the component tree there will need to be something conceptually like:

```jsx
<UserBar
    value={something}
    onChange={someFunction}
/>
```

React takes those attributes and effectively provides `UserBar` with:

```js
{
    value: something,
    onChange: someFunction
}
```

Then:

```js
function UserBar({ value, onChange })
```

extracts them.

So we have a parent → child data relationship:

```text
Parent component
      │
      │ value
      │ onChange
      ↓
   UserBar
```

And already the two props have different kinds of values.

`value` will presumably be something like:

```js
"Zhe"
```

while `onChange` is a **function**.

Something conceptually like:

```js
(newValue) => {
    ...
}
```

This connects beautifully to what we learned with `useCallback`:

> JavaScript functions are values.

Therefore a function can be passed into another component just like a string or number.

```text
props
│
├── value
│      ↓
│    "Zhe"
│
└── onChange
       ↓
     function object
```

That's going to become extremely important when we reach line 10.

---

### 4. Why is it `UserBar`, capitalized?

This looks trivial, but in JSX it matters.

React/JSX distinguishes roughly between:

```jsx
<div>
<input>
<span>
```

and:

```jsx
<UserBar>
<ClientRow>
<ClientTable>
```

Lowercase names represent built-in/platform HTML elements.

Capitalized names represent your own React components.

So:

```jsx
<div />
```

means conceptually:

> Give me a DOM `<div>`.

Whereas:

```jsx
<UserBar />
```

means conceptually:

> Use the JavaScript component called `UserBar`.

That naming convention is therefore semantically meaningful in JSX.

---

### 5. Now `export default`

The complete first line is:

```js
export default function UserBar({ value, onChange }) {
```



This part isn't React either.

It's JavaScript's module system.

We already saw named exports in `client.js`:

```js
export function listClients() { ... }
export function createClient(...) { ... }
```

Those are **named exports**.

So another file imports them by name:

```js
import { listClients, createClient } from './client'
```

Here Claude instead used:

```js
export default function UserBar(...)
```

meaning this module has a **default export**.

A caller can therefore do conceptually:

```js
import UserBar from './UserBar'
```

Notice no `{ }`.

Compare:

```text
NAMED EXPORT

export function createClient() {}

        ↓

import { createClient } from './client'


DEFAULT EXPORT

export default function UserBar() {}

        ↓

import UserBar from './UserBar'
```

This is JavaScript module syntax, not a React concept.

---

### 6. Now we reach the actual React-ish part: `return (...)`

```jsx
return (
  <div className="user-bar">
    ...
  </div>
)
```



Here's the mental transition.

A normal function might calculate and return:

```js
function add(a, b) {
    return a + b
}
```

So:

```text
inputs
 ↓
function
 ↓
number
```

A React component calculates and returns a **description of UI**:

```text
props
 ↓
UserBar()
 ↓
JSX
 ↓
description of what UI should look like
```

That JSX looks suspiciously like HTML:

```jsx
<div className="user-bar">
```

but it is **not literally an HTML file**.

It's JSX syntax embedded in JavaScript.

That's why your file extension is:

```text
.jsx
```

rather than simply `.js`.

---

### 7. JSX is a way to describe UI using JavaScript

Take:

```jsx
<div className="user-bar">
    ...
</div>
```

Your browser does not natively execute JSX as HTML inside JavaScript.

The Vite/React build tooling transforms JSX into JavaScript representations React understands.

You do **not** need to memorize the exact transformed output right now.

The important model is:

```text
JSX

<div>
   ...
</div>

       ↓
build transformation

JavaScript representation
of this desired UI

       ↓
React

       ↓
actual browser DOM
```

So when `UserBar()` executes, it isn't directly issuing an imperative browser command like:

```text
create div
create input
insert span
attach this
modify that
```

Instead it returns a **description of what the UI should be**.

That's a huge philosophical difference.

---

### 8. This is declarative rather than imperative

Imagine doing UI manually.

An imperative approach might conceptually say:

```js
const div = document.createElement('div')
const input = document.createElement('input')

input.value = value

div.appendChild(input)

document.body.appendChild(div)
```

You're telling the browser:

> Do this. Then do this. Then mutate this. Then attach that.

React encourages you to say:

```jsx
<div>
    <input value={value} />
</div>
```

which means:

> Given the current application state/props, **this is what the UI should look like**.

Then React handles reconciling that description with the existing DOM.

So:

```text
IMPERATIVE

"Change the UI by performing
these DOM operations."


DECLARATIVE / REACT

"Given the current data,
the UI should look like this."
```

This connects directly to our earlier state discussion.

Remember:

```text
state
  ↓
render
  ↓
UI

state changes
  ↓
render again
  ↓
new UI description
```

We're now actually seeing what "render" means in concrete code.

---

### 9. Suppose `value = "Zhe"`

Imagine React invokes:

```js
UserBar({
    value: "Zhe",
    onChange: someFunction
})
```

Inside:

```text
value = "Zhe"
onChange = someFunction
```

The component executes from the beginning.

Then it returns JSX describing the UI.

Later, suppose `value` changes to:

```text
"Alice"
```

React can render the component again:

```js
UserBar({
    value: "Alice",
    onChange: someFunction
})
```

The function executes again.

It produces a new UI description based on:

```text
value = "Alice"
```

This is why our earlier statement was so important:

> React component functions execute again during re-renders.

It also explains why we spent time on `useCallback`.

If the parent recreates ordinary functions every render, then:

```js
onChange
```

could be a new function object each time.

`useCallback` can stabilize that identity where useful.

We're now seeing where those callback functions actually **go**: they're passed down as props into components like this one.

---

### 10. Let's look only at the first JSX element

```jsx
<div className="user-bar">
```

Why:

```jsx
className
```

instead of HTML's:

```html
class
```

?

In JSX, React uses:

```jsx
className="user-bar"
```

to represent the element's CSS class.

Eventually the DOM element corresponds conceptually to:

```html
<div class="user-bar">
```

So somewhere in the project's CSS there may be a rule like:

```css
.user-bar {
    ...
}
```

and that controls presentation.

Again, separate the layers:

```text
<div>
    → HTML/DOM element concept

className
    → JSX/React-facing property

"user-bar"
    → application-defined CSS class name
```

---

### 11. Then the first child

```jsx
<label htmlFor="current-user">Acting as</label>
```



This will give us another useful JSX distinction.

The plain text:

```text
Acting as
```

is simply rendered text.

But:

```jsx
htmlFor="current-user"
```

corresponds to the HTML relationship between a `<label>` and a form control.

Later the input has:

```jsx
id="current-user"
```



So conceptually:

```text
<label htmlFor="current-user">
             │
             │ references
             ↓
<input id="current-user">
```

This associates:

```text
"Acting as"
```

with that input, which is useful for accessibility and normal label behavior.

Why `htmlFor` instead of HTML's `for`?

Same general reason as `className`: JSX uses JavaScript-facing property names for some HTML attributes.

---

### Now we have enough foundation for the really important part

The next block is:

```jsx
<input
  id="current-user"
  type="text"
  placeholder="your name"
  value={value}
  onChange={(e) => onChange(e.target.value)}
  autoComplete="off"
/>
```



And **this is where I want to slow down significantly**, because this tiny input introduces one of the most important React patterns you'll encounter:

```text
controlled input
```

In particular, these two lines form a feedback loop:

```jsx
value={value}
onChange={(e) => onChange(e.target.value)}
```

At first glance it seems almost paradoxical:

> If `value={value}` tells the input what value it has, but the user is supposed to type into the input, who actually owns the value—the browser input or React?

That question gets us directly into:

**DOM event → event object → `e.target.value` → callback prop → parent state update → re-render → new `value` prop → input changes on screen.**

That's one of the core React data-flow loops, and it's worth tracing one keystroke—say changing `"Zh"` to `"Zhe"`—all the way around before we touch the final conditional hint.



## Lesson 2 - `input` tag & React component & controlled input & one-day data flow

Absolutely. This is the part of `UserBar.jsx` where React's one-way data flow becomes concrete.

The key lines are:

```jsx
value={value}
onChange={(e) => onChange(e.target.value)}
```



Let's understand them independently first, then trace one keystroke through the whole system.

### 1. First: `{...}` inside JSX means "evaluate JavaScript"

We've seen JSX that looks almost like HTML:

```jsx
<input
  id="current-user"
  type="text"
  placeholder="your name"
/>
```

Those values are literal strings.

But then we encounter:

```jsx
value={value}
```

The `{}` have a special role inside JSX:

> Leave JSX markup temporarily, evaluate the JavaScript expression inside, and use its resulting value here.

For example:

```js
const name = "Zhe"
```

then:

```jsx
<span>{name}</span>
```

means conceptually:

```html
<span>Zhe</span>
```

Likewise:

```jsx
<input value={value} />
```

means:

> Set this input's displayed value according to whatever the JavaScript variable `value` currently contains.

This distinction will appear everywhere in React:

```jsx
<div className="hello">       // literal string
<div className={className}>   // JavaScript variable

<input disabled={isDisabled}> // JavaScript expression

<span>{client.client_name}</span>
```

So JSX is not "HTML with weird syntax." It's markup-like syntax with JavaScript expressions embedded inside it.

---

### 2. Where did this particular `value` come from?

Go back to the component signature:

```jsx
export default function UserBar({ value, onChange }) {
```



`value` is a prop.

So conceptually the parent does:

```jsx
<UserBar value={something} ... />
```

React invokes `UserBar` with that value.

Then `UserBar` says:

```jsx
<input value={value} />
```

Therefore:

```text
Parent
  │
  │ value
  ↓
UserBar
  │
  │ value
  ↓
<input>
```

Data flows **downward**.

If:

```js
value = "Zhe"
```

the input displays:

```text
┌─────────────────┐
│ Zhe             │
└─────────────────┘
```

So far, straightforward.

---

### 3. But this creates an interesting problem

Suppose the user clicks the box and presses:

```text
a
```

What should happen?

Our normal browser intuition says:

> The input owns its contents. I type `a`, so the browser changes the input.

But React has been explicitly told:

```jsx
value={value}
```

meaning:

> The authoritative value for this input is this React value.

That's what makes this a **controlled input**.

Conceptually:

```text
UNCONTROLLED-ish mental model

DOM input
    │
    └── owns its current text


CONTROLLED REACT INPUT

React value
    │
    ↓
DOM input displays that value
```

The React-side value is the source of truth.

And therefore we need a mechanism for user typing to get back into React's data.

That's:

```jsx
onChange={...}
```

---

### 4. `onChange` here is an event handler

This line:

```jsx
onChange={(e) => onChange(e.target.value)}
```

is confusing at first because **two different things are both called `onChange`**.

Let's rename them mentally.

The left-hand one:

```jsx
onChange={...}
```

belongs to the `<input>`.

It means roughly:

> React, when this input changes, call this event handler.

The right-hand one:

```js
onChange(...)
```

is the `onChange` **prop passed into `UserBar`**.

So if we temporarily renamed the prop:

```js
function UserBar({
    value,
    onUserNameChange
}) {
```

then the line could be written:

```jsx
<input
    value={value}
    onChange={(e) => onUserNameChange(e.target.value)}
/>
```

That's conceptually identical, but much easier to see.

So:

```text
LEFT onChange
─────────────
React input event prop


RIGHT onChange
──────────────
function supplied by UserBar's parent
```

The identical names are conventional but not required.

---

### 5. Now the arrow function

We already understand:

```js
(e) => ...
```

So:

```js
(e) => onChange(e.target.value)
```

is an ordinary JavaScript function.

Approximately:

```js
function (e) {
    return onChange(e.target.value)
}
```

React receives this function as the input's event handler.

Notice something important:

```jsx
onChange={(e) => ...}
```

passes the **function itself**.

We are **not calling it during render**.

Compare:

```jsx
onChange={(e) => onChange(e.target.value)}
```

with something like:

```jsx
onChange={someFunction()}
```

The latter calls `someFunction()` immediately while evaluating the JSX and gives `onChange` whatever that function returned.

The actual code creates a callback:

```text
render
  ↓
create function
  ↓
give function to React
  ↓
...
user types later
  ↓
React invokes function
```

This connects directly to our earlier discussion about functions being values.

---

### 6. What is `e`?

This is the next important piece.

When the input changes, React calls your handler and supplies information about the event:

```js
(e) => ...
```

The variable name `e` is just short for:

```js
event
```

It could have been written:

```jsx
onChange={(event) => onChange(event.target.value)}
```

Same thing.

So when the user types something:

```text
User changes input
       ↓
browser event occurs
       ↓
React event handling
       ↓
your callback gets called
       ↓
e = information about that event
```

The event contains, among other things, a reference to the element involved.

That's what:

```js
e.target
```

is accessing here.

In this case, the target is the `<input>`.

---

### 7. Therefore `e.target.value`

Suppose the input currently displays:

```text
Zh
```

and the user types:

```text
e
```

The input's current text associated with the change is:

```text
Zhe
```

So conceptually:

```js
e.target
```

refers to the input, and:

```js
e.target.value
```

is:

```js
"Zhe"
```

Therefore this:

```js
(e) => onChange(e.target.value)
```

becomes conceptually:

```js
onChange("Zhe")
```

And now we've crossed an important boundary:

```text
DOM/browser event
      ↓
event object
      ↓
e.target.value
      ↓
ordinary JavaScript string
      ↓
"Zhe"
```

The component extracts the useful application data from the UI event.

---

### 8. But `UserBar` doesn't update any state!

This is a very important observation.

Look at the entire file.

There is no:

```js
useState(...)
```

inside `UserBar`.

There isn't even an import from React.

So how does typing `"Zhe"` actually change anything?

`UserBar` says:

```js
onChange("Zhe")
```

but **the function came from its parent**.

Remember:

```js
function UserBar({ value, onChange })
```

So `UserBar` doesn't own the user-name state.

It receives:

```text
current value
+
function for requesting a change
```

from above.

Conceptually:

```text
                PARENT
                   │
        ┌──────────┴──────────┐
        │                     │
        ↓                     ↓
      value                onChange
        │                     │
        └──────────┬──────────┘
                   ↓
                UserBar
```

When the user interacts, the callback allows information to travel conceptually upward:

```text
                PARENT
                   ↑
                   │ onChange("Zhe")
                   │
                UserBar
                   ↑
                   │
              user typing
```

This is one of the central React patterns:

> **Data flows down through props; events/callbacks communicate requested changes upward.**

---

### 9. Now connect this to `useCurrentUser.js`

We shouldn't claim the exact parent wiring until we inspect it, but we already know what `useCurrentUser()` provides:

```js
return [currentUser.trim(), currentUser, setCurrentUser]
```

So somewhere above `UserBar`, there is enough machinery available to wire a current-user value and setter into this component. The hook itself stores the live React state and synchronizes changes to `localStorage`.

That means the architecture we've studied is deliberately separated:

```text
useCurrentUser
────────────────────
owns/manages user state
persistence mechanics


UserBar
────────────────────
displays supplied value
reports user edits
```

`UserBar` doesn't know:

```text
localStorage
STORAGE_KEY
useState
useCallback
```

It doesn't need to.

That's good component isolation.

---

### 10. Let's trace one keystroke

Let's use a conceptual parent state:

```text
currentUser = "Zh"
```

The parent supplies:

```jsx
<UserBar
    value={currentUser}
    onChange={setCurrentUser}
/>
```

Again, we'll verify the actual wiring when we reach the parent; this is the conceptual wiring supported by the interfaces we've studied.

Now React renders `UserBar`.

#### Step 1 — component receives props

```text
value = "Zh"
onChange = setCurrentUser
```

#### Step 2 — JSX is evaluated

```jsx
<input
    value={value}
    onChange={(e) => onChange(e.target.value)}
/>
```

So the input displays:

```text
┌───────────────┐
│ Zh            │
└───────────────┘
```

#### Step 3 — user types `e`

An input-change event occurs.

React calls:

```js
(e) => onChange(e.target.value)
```

#### Step 4 — extract the new input value

```js
e.target.value
```

is:

```text
"Zhe"
```

So:

```js
onChange(e.target.value)
```

effectively becomes:

```js
setCurrentUser("Zhe")
```

#### Step 5 — remember what that setter does

From our previous file:

```js
const setCurrentUser = useCallback((value) => {
    setState(value)

    try {
        if (value) localStorage.setItem(STORAGE_KEY, value)
        else localStorage.removeItem(STORAGE_KEY)
    } catch {
        ...
    }
}, [])
```

So:

```text
setCurrentUser("Zhe")
          │
          ├── setState("Zhe")
          │
          └── localStorage.setItem(..., "Zhe")
```

#### Step 6 — state update causes a re-render

The React state is now:

```text
currentUser = "Zhe"
```

The relevant parent/component tree renders again.

Now `UserBar` receives:

```text
value = "Zhe"
```

#### Step 7 — JSX executes again

```jsx
<input value={value} ... />
```

now means:

```jsx
<input value="Zhe" ... />
```

and the UI displays:

```text
┌───────────────┐
│ Zhe           │
└───────────────┘
```

That completes the loop.

---

### 11. This loop is worth burning into your mental model

```text
                    React state
                    currentUser
                         │
                         │ props
                         ↓
                     UserBar
                         │
                         │ value={value}
                         ↓
                    <input>
                         │
                         │ user types
                         ↓
                  change event
                         │
                         ↓
                  e.target.value
                         │
                         │ "Zhe"
                         ↓
                    onChange(...)
                         │
                         ↓
                 setCurrentUser
                         │
                         ↓
                     setState
                         │
                         ↓
                    RE-RENDER
                         │
                         └───────────────┐
                                         │
                                         ↓
                                  new React state
```

Or compressed:

```text
state
  ↓
props
  ↓
input
  ↓
user event
  ↓
callback
  ↓
state update
  ↓
re-render
  ↓
input
```

That's controlled-component data flow.

---

### 12. Why not just let the browser own the input?

This is a good "what if we removed it?" question.

Suppose we removed:

```jsx
value={value}
```

and simply had:

```jsx
<input
    onChange={(e) => onChange(e.target.value)}
/>
```

The browser could manage the visible input value itself while React listens to changes.

That can work; React supports uncontrolled approaches too.

So `value={value}` isn't some universal requirement that every text input must have.

The generated code deliberately chooses a **controlled input** because the current user already exists as React/application state.

That gives a single source of truth:

```text
currentUser state
       │
       ├── determines input display
       ├── can be persisted
       ├── can be trimmed for modified_by
       └── can be used elsewhere in the UI
```

Rather than potentially having:

```text
DOM thinks value = X

React thinks value = Y
```

The controlled pattern says:

> React owns the authoritative value; the input is a UI projection of that value.

That's why state and rendering are so tightly connected in React.

---

### 13. The other input properties are comparatively simple

```jsx
id="current-user"
type="text"
placeholder="your name"
...
autoComplete="off"
```



These are configuration for the underlying input.

`type="text"` says it's a text input.

`placeholder="your name"` provides hint text when empty.

`id="current-user"` connects to the label we studied:

```jsx
<label htmlFor="current-user">
```

And:

```jsx
autoComplete="off"
```

asks the browser not to provide normal autocomplete behavior here.

These aren't the conceptual heart of the component.

The important pair remains:

```jsx
value={value}
onChange={(e) => onChange(e.target.value)}
```

---

### 14. Now the final piece of `UserBar`

After the input:

```jsx
<span className="hint">
  {value.trim()
    ? 'Sent as modified_by on every change.'
    : 'Enter a name to enable changes.'}
</span>
```



This combines two things we've already learned:

```text
{ ... }
    → evaluate JavaScript inside JSX

condition ? A : B
    → ternary operator
```

So this is actually an excellent demonstration of **declarative rendering based on data**.

Let's unpack it.

The JavaScript expression is:

```js
value.trim()
    ? 'Sent as modified_by on every change.'
    : 'Enter a name to enable changes.'
```

Suppose:

```js
value = "Zhe"
```

Then:

```js
value.trim()
```

gives:

```text
"Zhe"
```

A non-empty string is truthy.

Therefore the ternary produces:

```text
"Sent as modified_by on every change."
```

So React describes:

```jsx
<span className="hint">
    Sent as modified_by on every change.
</span>
```

---

### 15. What if the input is empty?

```js
value = ""
```

Then:

```js
value.trim()
```

produces:

```text
""
```

Empty string is falsy.

Therefore the ternary selects:

```text
"Enter a name to enable changes."
```

So the UI becomes conceptually:

```text
Acting as  [ your name              ]

Enter a name to enable changes.
```

Once the user types:

```text
Z
```

state changes.

React re-renders.

Now:

```js
value.trim()
```

is `"Z"`, which is truthy.

Without you manually finding the `<span>` and changing its text, the component simply returns a different description:

```text
Acting as  [ Z                      ]

Sent as modified_by on every change.
```

This is the declarative React model in perhaps its cleanest form:

```text
value = ""
     ↓
render
     ↓
"Enter a name..."


value = "Zhe"
     ↓
render
     ↓
"Sent as modified_by..."
```

You don't write:

```text
IF user enters first character:
    find hint DOM element
    replace its text
```

You write:

```jsx
{value.trim() ? messageA : messageB}
```

and let React reconcile the resulting UI.

---

### 16. There's a nice connection to the hook's return values

Remember `useCurrentUser()` deliberately returned both:

```js
currentUser.trim()
```

and:

```js
currentUser
```

The distinction now makes more sense.

The raw value matters for the input:

```text
"  Zhe  "
```

because that's literally what the user typed.

But when deciding whether there is a meaningful identity:

```js
value.trim()
```

turns:

```text
"      "
```

into:

```text
""
```

which is falsy.

So typing five spaces doesn't make the UI say:

```text
Sent as modified_by...
```

The component treats whitespace-only input as semantically empty.

And elsewhere the hook exposes the trimmed version for the actual `modified_by` value.

That's a small but coherent design:

```text
RAW USER INPUT
"  Zhe  "
     │
     ├──────────────→ input display
     │
     └── trim()
           ↓
         "Zhe"
           ↓
     semantic identity
```

---

### 17. `UserBar` has no state of its own—and that's significant

Now that we've understood the whole component, look at what it **doesn't** contain:

```text
no useState
no useEffect
no useCallback
no fetch
no localStorage
no FastAPI
no Promise
```

Its contract is tiny:

```text
INPUTS
────────────────────
value: string
onChange: function


OUTPUT
────────────────────
UI description
```

Conceptually:

```text
UserBar(value, onChange)
          ↓
       rendered UI
```

And interaction goes:

```text
user edits input
      ↓
UserBar extracts new string
      ↓
calls supplied onChange(string)
```

It doesn't care what `onChange` actually does.

The parent could supply a function that:

```text
updates React state
```

or:

```text
writes somewhere else
```

or:

```text
logs the value
```

`UserBar` doesn't know.

That is **component composition through a callback contract**.

---

### 18. So our complete mental model of `UserBar.jsx` is now

```text
              Parent
                │
        ┌───────┴────────┐
        │                │
      value           onChange
        │                │
        ↓                │
    ┌───────────────────────┐
    │       UserBar         │
    │                       │
    │ label                 │
    │                       │
    │ input                 │
    │ value={value}         │
    │                       │
    │ conditional hint      │
    └───────────┬───────────┘
                │
                │ user types
                ↓
             event
                │
                ↓
        e.target.value
                │
                ↓
        onChange(newValue)
                │
                ↓
              Parent
                │
         state changes
                │
                ↓
           re-render
                │
                └────────────→ new value prop
```

And if I had to reduce this entire file to **three concepts worth carrying forward**, they would be:

```text
1. COMPONENT
   A JS function that receives props and returns UI description.

2. CONTROLLED INPUT
   value comes down from React;
   user edits are reported through a callback.

3. ONE-WAY DATA FLOW
   data ↓ through props
   events ↑ through callbacks
   state changes → re-render
```

Everything else—`className`, `htmlFor`, ternaries, template expressions—is supporting syntax.

The next file, `ClientRow.jsx`, should build directly on this. `UserBar` gave us **one controlled input and one callback**; a client row should introduce a richer local interaction where a particular client can presumably be edited or deleted. That's where props and callbacks should start feeling less like React syntax and more like an actual component architecture.




## Code Appendix

```js
export default function UserBar({ value, onChange }) {
  return (
    <div className="user-bar">
      <label htmlFor="current-user">Acting as</label>
      <input
        id="current-user"
        type="text"
        placeholder="your name"
        value={value}
        onChange={(e) => onChange(e.target.value)}
        autoComplete="off"
      />
      <span className="hint">
        {value.trim()
          ? 'Sent as modified_by on every change.'
          : 'Enter a name to enable changes.'}
      </span>
    </div>
  )
}
```
