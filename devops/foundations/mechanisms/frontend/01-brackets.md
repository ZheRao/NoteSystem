# Brackets

The reason this keeps feeling slippery is that the same bracket means different things depending on **where the parser is standing** when it hits it. Once you learn the three "positions," most of the confusion goes away, so that's the frame for this whole note.

## The core mental model: three positions

The parser is always in one of three modes:

1. **Statement position**: the start of a line or block, where JS expects "do something" (`if`, `const`, `return`, a function body).
2. **Expression position**: anywhere JS expects a *value* (right of `=`, inside a function call, after `return`, after `=>`).
3. **JSX position**: inside `<Tag> ... </Tag>`, where the parser is reading markup, not JavaScript.

Almost every bracket question comes down to "which position am I in?"

Layer note: nearly everything below is the **JavaScript language** layer. JSX is a syntax extension compiled by your build tool (Vite/Babel) for **React**. None of it is a browser API.


## `()` Parentheses: "call it" or "group it"

**1. Calling a function** (JS)
```js
doThing()            // call with no arguments
setCount(count + 1)  // call with one argument
```
`doThing` alone is the function *as a value*. `doThing()` *runs* it. This is the source of a classic React bug:
```jsx
<button onClick={handleClick}>    // ✅ pass the function; React calls it later
<button onClick={handleClick()}>  // ❌ calls it NOW, during render, and passes the result
<button onClick={() => handleClick(id)}>  // ✅ wrap it when you need arguments
```

**2. Declaring parameters** (JS)
```js
function add(a, b) { ... }
const add = (a, b) => a + b
const double = x => x * 2   // one param: parens optional
```

**3. Grouping / controlling evaluation** (JS)
```js
(a + b) * c
```

**4. Wrapping multi-line JSX** (JS + JSX)
```jsx
return (
  <div>
    <p>Hi</p>
  </div>
)
```
Why it's there: JS auto-inserts a semicolon after a bare `return` followed by a newline, so without the parens you'd return `undefined`. The parens keep the parser in expression mode across lines.

**5. Returning an object from an arrow function** (JS). Covered under `{}` below, because that's where the problem comes from.


## `{}` Curly braces: the overloaded one

This is where position matters most.

**1. Block, in statement position** (JS)
```js
if (ok) { doA(); doB(); }
function f() { ... }
const f = () => { ... }   // block body: you MUST write return
```

**2. Object literal, in expression position** (JS)
```js
const task = { id: 1, title: "Buy grain" }
```

**3. Destructuring, on the left of `=` or in parameters** (JS)
```js
const { title, done } = task          // pull properties out
function TaskItem({ task, onToggle }) // destructure props in the parameter list
```
Removing the braces in that second line would make `task` the *entire props object*, not the `task` prop. That's a common silent bug.

**4. JSX escape hatch: "switch back to JavaScript here"** (JSX)
```jsx
<p>{task.title}</p>
<p>{done ? "Done" : "Pending"}</p>
<input value={text} onChange={e => setText(e.target.value)} />
```
Inside JSX, `{}` means "evaluate this JS *expression* and drop the value here." It must be an expression, so you can't put an `if` statement or a `for` loop inside. That's why you see ternaries, `&&`, and `.map()` in JSX instead.

**5. Double braces `{{ }}`** (JSX + JS)
```jsx
<div style={{ color: "red", padding: 8 }}>
```
This isn't special syntax. The outer `{}` is the JSX escape and the inner `{}` is an object literal. Two different meanings stacked together.

**6. Template literal slot** (JS)
```js
`Task ${task.id} is due`
```

**7. Named imports/exports** (JS modules)
```js
import { useState, useEffect } from "react"   // named
import React from "react"                      // default: no braces
```

### The trap: arrow function returning an object
```js
const make = () => { id: 1 }     // ❌ returns undefined
const make = () => ({ id: 1 })   // ✅ returns the object
```
After `=>`, a `{` is read as a **block** (statement position), so `id:` becomes a label and nothing is returned. Wrapping it in `()` forces expression position, so the `{` becomes an object. You'll hit this constantly with state updates:
```js
setTask(prev => ({ ...prev, done: true }))
```


## `[]` Square brackets: arrays and "look up by computed key"

**1. Array literal** (JS)
```js
const tasks = [a, b, c]
const next = [...tasks, newTask]   // copy + append (immutable update)
```

**2. Access by index or computed key** (JS)
```js
tasks[0]
task["title"]     // same as task.title
task[fieldName]   // key comes from a variable: dot notation can't do this
```

**3. Array destructuring** (JS)
```js
const [count, setCount] = useState(0)
```
This is just JS destructuring by *position*. `useState` returns a 2-element array, and you name the slots whatever you want. That's why it's `[]` here and not `{}`: arrays destructure by position, objects by name.

**4. Computed property names inside an object literal** (JS)
```js
setForm(prev => ({ ...prev, [e.target.name]: e.target.value }))
```
Without the brackets you'd literally create a key called `"e.target.name"`.

**5. Dependency arrays** (React convention, plain JS array)
```js
useEffect(() => { ... }, [roomId])
```


## `<>` Angle brackets

```jsx
<TaskItem task={t} />    // JSX element (capitalized = your component)
<div>...</div>            // lowercase = HTML element
<>...</>                  // Fragment: groups siblings without adding a DOM node
a < b                     // plain comparison in normal JS
```
(In TypeScript you'll also see `useState<Task[]>([])` for generics, but that's a separate topic when you get there.)


## One-glance cheat sheet

| You see | Statement position | Expression position | Inside JSX |
|---|---|---|---|
| `()` | `if (...)`, params | call `f()`, grouping, `=> ({...})` | wrap multi-line JSX |
| `{}` | block (needs `return`) | object literal; destructuring on left of `=` | "evaluate JS expression here" |
| `[]` | — | array, index/computed key, array destructuring | — |
| `<>` | — | comparison | elements, fragments |

**Debugging questions to ask yourself:**
- Did I mean to *call* this, or *pass* it? (`f` vs `f()`)
- Is this `{` a block or an object? (Check what's directly before it. After `=>` or at line start, it's a block.)
- Am I destructuring by name (`{}`) or by position (`[]`)?
- Inside JSX, is what's in my `{}` an expression, not a statement?

Tier-wise for your notes: the `=>` block-vs-object trap, `f` vs `f()` in handlers, and props destructuring are **tier 1**, the ones that cause silent bugs. Computed keys and fragments are **tier 2**. Generics are **tier 3**, for when you adopt TypeScript.