# JavaScript for Python Developers — A Comprehensive Tutorial

*Assumes solid Python experience, zero JavaScript. Written as a build-your-mental-model reference, not a cheat sheet. Every JS concept is anchored to its Python equivalent so you can transfer what you already know and only pay attention to what's actually different.*

---

## How to use this document

Read it top to bottom once. Each part has:

- **The mental model** — what must be true regardless of syntax.
- **Python → JavaScript** — the direct translation.
- **The gotchas** — where JS surprises Python people.
- **Practice** — attempt before peeking. Solutions are collected at the very end (Part 16).

To actually *run* the examples, you have two easy options:

1. **Browser console** — open any web page, press F12, click the "Console" tab, paste code, hit Enter. This is the fastest scratchpad. It's the JS equivalent of the Python REPL.
2. **Node.js** — you already have this via your Vite/npm setup. Save code to `scratch.js` and run `node scratch.js`. This is the JS equivalent of `python scratch.py`.

Throughout, `//` starts a comment (JS has no `#` comments). `/* ... */` is a block comment.

---

## Table of contents

- Part 0 — Orientation: how JS is fundamentally not Python
- Part 1 — Variables and data types
- Part 2 — Operators and expressions
- Part 3 — Control flow
- Part 4 — Functions (the big one)
- Part 5 — Arrays
- Part 6 — Objects
- Part 7 — Destructuring
- Part 8 — Spread and rest
- Part 9 — Classes and prototypes
- Part 10 — Modules (the Vite connection)
- Part 11 — Asynchronous JavaScript
- Part 12 — Error handling
- Part 13 — Useful built-ins (Map, Set, Date, regex, JSON)
- Part 14 — The DOM, briefly
- Part 15 — Python → JavaScript translation cheat sheet
- Part 16 — Practice solutions

---

## Part 0 — Orientation: how JS is fundamentally not Python

Before any syntax, absorb five structural facts. These explain *why* JS is the way it is, and most beginner confusion dissolves once you hold them.

**1. JavaScript runs in two very different homes.**
Python almost always runs on a server or your machine via the `python` interpreter. JavaScript runs in *the browser* (every browser ships a JS engine) and *also* on servers/your machine via **Node.js**. Same language, two environments with different available functions. `document.querySelector(...)` exists in the browser but not Node. `require('fs')` (file system) exists in Node but not the browser. The core language — variables, functions, arrays — is identical in both. For GrowLytics: your React frontend runs in the browser; Vite and npm run in Node.

**2. JavaScript is single-threaded and event-driven.**
Python you mostly write top-to-bottom, blocking on each line. JS is built around *not blocking*. When you fetch data from your FastAPI backend, JS doesn't sit and wait — it registers "when this arrives, run this function" and moves on. This is why asynchronous programming (Part 11) is not an advanced topic in JS; it's day-one essential. It's the single biggest mental shift for a backend Python developer.

**3. JavaScript is dynamically typed like Python — but coerces aggressively.**
Python raises `TypeError` if you do `"3" + 5`. JavaScript happily returns `"35"`. This "helpfulness" is the source of most JS jokes and bugs. You'll learn to defend against it with strict equality (`===`) and explicit conversions.

**4. JavaScript's "objects" are its dicts, and they're everywhere.**
A Python `dict` and a Python object (class instance) are separate things. In JS they're the *same underlying thing*: a plain object `{}` is simultaneously your dictionary, your struct, your namespace, and (via prototypes) your class instance. Once this clicks, huge swaths of JS make sense.

**5. There's old JS and modern JS.**
JS from ~2015 onward (called "ES6+" or "ES2015+") added `let`/`const`, arrow functions, classes, modules, destructuring, promises — everything that makes JS pleasant. You will write *only* modern JS. But old tutorials and old code use `var`, `function` everywhere, and callback pyramids. This document teaches modern JS and flags the old style only so you can recognize it in the wild.

Hold those five. Now the syntax.

---

## Part 1 — Variables and data types

### 1.1 Declaring variables: `const`, `let`, (and the deprecated `var`)

In Python you just write `x = 5`. In JS you must *declare* a variable with a keyword the first time:

```javascript
const cropCount = 12;   // cannot be reassigned
let revenue = 50000;    // can be reassigned
revenue = 60000;        // fine

cropCount = 13;         // TypeError: Assignment to constant variable
```

- **`const`** — a binding that can't be reassigned. This is your default. Reach for `const` first, always.
- **`let`** — a binding that can be reassigned. Use only when you genuinely need to reassign (loop counters, accumulators, values that change).
- **`var`** — the old way. It has confusing scoping rules (function-scoped, not block-scoped, and "hoisted" in surprising ways). **Never write `var` in new code.** You only need to recognize it.

There's no Python equivalent of this distinction — Python names are all reassignable. The `const`/`let` split is JS letting you signal intent and catch accidental reassignment.

> **Important nuance:** `const` prevents *reassigning the variable*, not *mutating the value*. This trips up everyone.
>
> ```javascript
> const crops = ["wheat", "canola"];
> crops.push("lentils");   // FINE — we mutated the array, didn't reassign
> console.log(crops);      // ["wheat", "canola", "lentils"]
> crops = ["barley"];      // ERROR — reassignment
> ```
>
> Same as Python: `x = (1, 2)` gives an immutable tuple, but `x = [1, 2]` with a "final" label would still let you `x.append(3)`. `const` is about the *name*, not the *object*.

### 1.2 The `console.log` = `print`

```javascript
console.log("hello");              // hello
console.log("crops:", crops);      // crops: [ 'wheat', 'canola', 'lentils' ]
console.log(1, 2, 3);              // 1 2 3  (accepts multiple args like print)
```

`console.log` is your `print`. There's also `console.error`, `console.warn`, `console.table` (surprisingly nice for arrays of objects), and `console.dir`.

### 1.3 Semicolons

JS statements *can* end with `;`. The language will insert them automatically in most cases ("Automatic Semicolon Insertion"), so they're technically optional — but leave them in. It avoids a small set of nasty edge cases and matches what nearly all professional codebases and formatters (Prettier) do. Just end statements with `;`.

### 1.4 The data types

JS has a small set of **primitive** types plus **objects**. This is the equivalent of Python's built-in types.

| Category | JavaScript | Python analog |
|---|---|---|
| Number | `number` (one type for int + float) | `int` and `float` (JS merges them) |
| Big integers | `bigint` (rare) | Python's arbitrary-precision `int` |
| Text | `string` | `str` |
| Boolean | `boolean` (`true`/`false`, lowercase) | `bool` (`True`/`False`) |
| "No value, intentionally" | `null` | `None` |
| "No value, not yet set" | `undefined` | *(no direct analog)* |
| Unique keys | `symbol` (rare) | *(no direct analog)* |
| Everything else | `object` (includes arrays, functions, dicts, dates...) | `dict`, `list`, objects, functions... |

Two things jump out for a Python person:

- **JS has ONE number type.** No separate int/float. `5` and `5.0` are the same. Division always produces a float-style result: `7 / 2` is `3.5`, not `3`. (For integer division you use `Math.floor(7 / 2)`.) The tradeoff: numbers are IEEE-754 doubles, so `0.1 + 0.2` is `0.30000000000000004` — exactly like Python floats. Integers are exact only up to about 9 quadrillion (`Number.MAX_SAFE_INTEGER`).
- **JS has TWO "nothing" values**, `null` and `undefined`, where Python has one (`None`). This is genuinely annoying and covered in 1.7.

### 1.5 `typeof` = `type()`

```javascript
typeof 42          // "number"
typeof "wheat"     // "string"
typeof true        // "boolean"
typeof undefined   // "undefined"
typeof null        // "object"   ← famous historical bug, just memorize it
typeof {}          // "object"
typeof []          // "object"   ← arrays report as "object"! see below
typeof function(){}// "function"
```

`typeof` returns a *string*. Two footguns: `typeof null` is `"object"` (a 25-year-old bug that can't be fixed without breaking the web), and arrays report as `"object"`. To check for an array specifically, use `Array.isArray(x)` — this is the reliable equivalent of Python's `isinstance(x, list)`.

### 1.6 Numbers

```javascript
const a = 10;
const b = 3;

a + b       // 13
a - b       // 7
a * b       // 30
a / b       // 3.3333333333333335   (always float-style)
a % b       // 1     (modulo, like Python)
a ** b      // 1000  (exponent, like Python's **)

Math.floor(a / b)   // 3    (integer division)
Math.round(3.6)     // 4
Math.ceil(3.1)      // 4
Math.max(1, 5, 2)   // 5    (note: takes args, not a list — see spread in Part 8)
Math.min(1, 5, 2)   // 1
Math.abs(-7)        // 7
Math.sqrt(16)       // 4
Math.random()       // random float in [0, 1)
```

Converting to/from numbers:

```javascript
Number("42")        // 42       (like int()/float())
Number("42.5")      // 42.5
Number("abc")       // NaN      (Not-a-Number — see below)
parseInt("42px")    // 42       (parses leading number, ignores rest)
parseFloat("3.14m") // 3.14
(255).toString(16)  // "ff"     (to hex string)
(3.14159).toFixed(2)// "3.14"   (returns a STRING, rounded)
```

**`NaN`** ("Not a Number") is JS's special "this math didn't work" value, like float `nan` in Python. Its infamous quirk: `NaN === NaN` is `false`. To test for it, use `Number.isNaN(x)`. It's contagious — any arithmetic with `NaN` yields `NaN`.

### 1.7 `null` vs `undefined` — the two nothings

This is a real conceptual difference from Python and worth nailing down.

- **`undefined`** means "this was never given a value." JS produces it automatically: an unset variable, a missing object property, a function with no `return`, a missing function argument. *You rarely assign it yourself.*
- **`null`** means "intentionally empty." *You* assign it to say "there is deliberately no value here." This is the true analog of Python's `None`.

```javascript
let x;                       // declared but unassigned
console.log(x);              // undefined

const farm = { name: "North Quarter" };
console.log(farm.acres);     // undefined  (property doesn't exist)

let selectedCrop = null;     // "no crop selected yet" — intentional
```

Practical rule: **treat both as "no value"** for logic purposes. JS gives you tools (`==`, `??`, optional chaining) that handle them together, covered in Part 2. When *you* need to represent "empty on purpose," use `null`.

### 1.8 Strings

Strings use single quotes, double quotes, or backticks. Single and double are identical (pick one and be consistent — most codebases use single or let Prettier decide). Backticks are special and the best.

```javascript
const a = 'wheat';
const b = "canola";

// Concatenation with +  (like Python)
const msg = 'Crop: ' + a;          // "Crop: wheat"

// Template literals (backticks) — like Python f-strings, use ${...}
const acres = 320;
const line = `${a} on ${acres} acres`;   // "wheat on 320 acres"

// Multi-line strings: backticks preserve newlines (like Python triple-quotes)
const report = `Farm report
  Crop: ${a}
  Acres: ${acres}`;
```

**Template literals (backticks with `${expression}`) are the JS f-string.** Any expression goes inside `${}`. You'll use these constantly, especially in React (JSX). Prefer them over `+` concatenation.

Common string methods (strings are immutable, like Python — methods return new strings):

```javascript
const s = "  Wheat Harvest  ";

s.length              // 17   (a PROPERTY, no parentheses — not len(s))
s.toUpperCase()       // "  WHEAT HARVEST  "
s.toLowerCase()       // "  wheat harvest  "
s.trim()              // "Wheat Harvest"  (like .strip())
s.trimStart()         // like .lstrip()
s.includes("Harvest") // true  (like "Harvest" in s)
s.startsWith("  W")   // true
s.endsWith("t  ")     // true
s.indexOf("Harvest")  // 8    (like .find(); -1 if not found)
s.replace("Wheat", "Barley")     // replaces FIRST occurrence
s.replaceAll("a", "@")           // replaces all
s.slice(2, 7)         // "Wheat"  (like s[2:7])
s.slice(-4)           // "st  "   (negative index like Python)
"a,b,c".split(",")    // ["a", "b", "c"]   (like .split())
["a","b"].join("-")   // "a-b"    (like "-".join(...) but method is on the array)
"wheat"[0]            // "w"      (indexing works)
"wheat".at(-1)        // "t"      (negative indexing needs .at())
"ab".repeat(3)        // "ababab" (like "ab" * 3)
"5".padStart(3, "0")  // "005"    (like zfill / rjust)
```

Two gotchas: `.length` is a property (no `()`), unlike Python's `len(s)` function. And `"wheat"[-1]` gives `undefined`, not `"t"` — for negative indexing you must use `.at(-1)`.

### 1.9 Booleans and truthiness

`true` and `false` are lowercase (Python capitalizes them). Every value is either "truthy" or "falsy" when used in a condition, just like Python — but the falsy set is different.

**The falsy values in JS (memorize this list — there are exactly these):**

```
false, 0, -0, 0n (bigint zero), "" (empty string), null, undefined, NaN
```

Everything else is truthy. Critically — and unlike Python — **empty array `[]` and empty object `{}` are TRUTHY in JS.**

```javascript
if ([])  console.log("runs!");   // runs — [] is truthy
if ({})  console.log("runs!");   // runs — {} is truthy
if ("")  console.log("nope");    // does not run
if (0)   console.log("nope");    // does not run
```

In Python, `if []:` is false. In JS it's true. This bites Python people constantly. To check for an empty array in JS you must check `arr.length === 0`.

### Practice — Part 1

1. Declare a `const` for a farm name and a `let` for its current revenue. Reassign the revenue. Then try reassigning the farm name and note the error.
2. Predict the output of each, then verify: `7 / 2`, `Math.floor(7 / 2)`, `"7" / 2`, `"7" + 2`, `typeof NaN`, `typeof []`, `Array.isArray([])`.
3. Given `const acres = 240; const crop = "canola";`, build the string `"canola: 240 acres"` two ways — with `+` and with a template literal.
4. Write a condition that prints `"empty"` only when an array `const items = []` has no elements. (Remember `[]` is truthy.)
5. Explain in one sentence each: when does JS give you `undefined` vs when would you use `null`?

---

## Part 2 — Operators and expressions

### 2.1 Arithmetic and assignment

Standard set, all familiar from Python: `+ - * / % **`. Compound assignment works too: `+= -= *= /= %= **=`.

JS also has `++` and `--` (increment/decrement), which Python lacks:

```javascript
let count = 5;
count++;        // count is now 6  (post-increment)
count--;        // back to 5
++count;        // 6 again (pre-increment: increments then returns)
```

Prefer `count += 1` for clarity unless you have a reason. `++`/`--` inside larger expressions cause subtle bugs.

### 2.2 Comparison — and why you must use `===`, never `==`

This is one of the most important sections in the document.

JS has **two** equality operators:

- `===` (strict equality) — compares value *and* type, no conversion. **This is what you want.**
- `==` (loose equality) — converts types before comparing. This is the source of endless bugs.

```javascript
5 === 5        // true
5 === "5"      // false  (number vs string — different types)
5 == "5"       // true   (== converts "5" to 5 first — surprising)

0 == false     // true   (== converts)
0 === false    // false  (different types)

"" == false    // true   (!!)
null == undefined   // true   (special case)
null === undefined  // false
```

The `==` conversion rules are a genuine minefield (there are famous truth tables). **The rule is simple: always use `===` and `!==`.** Forget `==` exists, with exactly one optional idiom noted below.

```javascript
5 !== "5"      // true   (strict not-equal)
```

**The one useful `==` idiom:** `value == null` is `true` for *both* `null` and `undefined` and nothing else. So `if (x == null)` is a concise "is this either nothing?" check. Some codebases allow this one exception. When in doubt, use `x === null || x === undefined`, or the `??` operator below.

Ordering comparisons are normal: `< > <= >=`. But beware they coerce too (`"10" < "9"` is `true` because it compares strings lexicographically). Compare numbers to numbers.

### 2.3 Logical operators — and their return values

Python: `and`, `or`, `not`. JavaScript: `&&`, `||`, `!`.

```javascript
true && false     // false   (AND)
true || false     // true    (OR)
!true             // false   (NOT)
```

Just like Python, `&&` and `||` **short-circuit and return one of the operands, not a strict boolean.** This is heavily exploited in JS (and React JSX).

```javascript
// || returns the first TRUTHY value (like Python's `or`)
const name = userInput || "default";   // if userInput is falsy, use "default"

// && returns the first FALSY value, else the last value (like Python's `and`)
const result = isLoggedIn && getUserData();   // only calls getUserData if logged in
```

In React you'll constantly see `{isLoading && <Spinner />}` — "if isLoading is truthy, render the spinner." That's this exact behavior.

**The `||` default trap:** `count || 10` returns `10` when `count` is `0`, because `0` is falsy. If `0` is a valid value you want to keep, `||` is wrong. Use `??` instead:

### 2.4 Nullish coalescing `??` — the "only null/undefined" default

`??` returns the right side *only* when the left is `null` or `undefined` (not merely falsy). This is the correct default operator most of the time.

```javascript
const a = 0 || 10;     // 10   ← probably a bug if 0 is valid
const b = 0 ?? 10;     // 0    ← correct: 0 is a real value, kept
const c = null ?? 10;  // 10
const d = undefined ?? 10;  // 10
const e = "" ?? "fallback";  // ""  (empty string is kept — it's not null/undefined)
```

Rule of thumb: use `??` for defaults unless you specifically want *all* falsy values (including `0` and `""`) to trigger the fallback, in which case use `||`.

### 2.5 Optional chaining `?.` — safe nested access

Python people reach for `.get()` on dicts or try/except to avoid `KeyError`/`AttributeError`. JS gives you `?.`: if the thing before it is `null` or `undefined`, the whole expression short-circuits to `undefined` instead of throwing.

```javascript
const farm = { owner: { name: "Zhe" } };

farm.owner.name        // "Zhe"
farm.manager.name      // TypeError: cannot read 'name' of undefined
farm.manager?.name     // undefined  (safe — no crash)
farm.owner?.name       // "Zhe"

// Works for method calls and array access too:
farm.getAcres?.()      // calls only if getAcres exists, else undefined
data?.crops?.[0]?.name // dig safely through nested structure
```

Combine `?.` with `??` for the classic safe-access-with-default:

```javascript
const managerName = farm.manager?.name ?? "unassigned";
```

This pair (`?.` and `??`) is one of the most useful modern-JS additions and appears everywhere in React data handling. Learn them well.

### 2.6 Ternary — JS's inline conditional

Python: `x if cond else y`. JavaScript: `cond ? x : y`.

```javascript
const status = revenue > 0 ? "profit" : "loss";

// Nesting is possible but gets ugly — use sparingly:
const label = score > 90 ? "A" : score > 80 ? "B" : "C";
```

The ternary is *huge* in React because JSX only allows expressions inside `{}`, not full `if` statements. You'll write `{isReady ? <Chart /> : <Loading />}` constantly.

### 2.7 A few more operators

```javascript
// Logical assignment (assign only under a condition):
let config = {};
config.mode ??= "default";   // set only if null/undefined
count ||= 1;                 // set if falsy
flag &&= false;              // set if truthy

// typeof (Part 1), instanceof (checks prototype chain — Part 9):
[] instanceof Array          // true
new Date() instanceof Date   // true

// Spread ... and rest ... — so important they get Part 8.
```

### Practice — Part 2

1. Predict then verify: `1 == "1"`, `1 === "1"`, `null == undefined`, `null === undefined`, `0 == false`, `"" == false`, `NaN === NaN`.
2. You have `let discount = 0;`. A colleague wrote `const applied = discount || 0.15;`. Why is this a bug, and how does `??` fix it?
3. Given `const data = { farm: { crops: null } };`, safely read `data.farm.crops.length` so it yields `undefined` instead of crashing, then default it to `0`.
4. Rewrite this as a ternary: `let msg; if (acres > 100) { msg = "large"; } else { msg = "small"; }`.
5. Explain what `isLoggedIn && "Welcome back"` evaluates to when `isLoggedIn` is `true`, and when it's `false`. (This is exactly the React conditional-render pattern.)

---

## Part 3 — Control flow

### 3.1 `if` / `else if` / `else`

Same idea as Python, but conditions go in `( )` and blocks in `{ }`. There's no `elif` — it's `else if`.

```javascript
if (revenue > 100000) {
  console.log("strong year");
} else if (revenue > 0) {
  console.log("modest profit");
} else {
  console.log("loss");
}
```

Braces are technically optional for single statements, but **always use braces**. Omitting them is a well-known source of bugs.

### 3.2 `switch`

Python got `match` recently; JS's older equivalent is `switch`. Note the mandatory `break` — without it, execution "falls through" to the next case (a classic bug, occasionally a deliberate feature).

```javascript
switch (cropType) {
  case "wheat":
  case "barley":                 // fall-through: both cases share code
    console.log("cereal grain");
    break;
  case "canola":
    console.log("oilseed");
    break;
  default:                       // like `else`
    console.log("unknown");
}
```

`switch` uses `===` comparison internally. Honestly, for most logic an object lookup or `if/else if` is cleaner; use `switch` when you have many discrete cases on one value.

### 3.3 Loops

JS has several loop forms. The distinctions matter — this is where Python's single elegant `for x in y` splits into multiple JS constructs.

**`for...of` — iterate over VALUES. This is your Python `for x in list`.**

```javascript
const crops = ["wheat", "canola", "lentils"];
for (const crop of crops) {
  console.log(crop);          // wheat, canola, lentils
}

// With index, use .entries() (like Python's enumerate):
for (const [i, crop] of crops.entries()) {
  console.log(i, crop);       // 0 wheat, 1 canola, 2 lentils
}

// Strings are iterable too:
for (const char of "hi") { console.log(char); }  // h, i
```

**`for...in` — iterate over KEYS. WARNING: this is a trap for Python people.**

`for...in` iterates over an object's *keys* (property names), not values. It looks like Python's `for x in list` but behaves completely differently — on an array it gives you *indices as strings*, and it includes inherited properties. **Rule: use `for...in` only for plain objects, and prefer `Object.keys()` even then. Never use `for...in` on arrays.**

```javascript
const farm = { name: "North", acres: 320 };
for (const key of Object.keys(farm)) {
  console.log(key, farm[key]);   // name North, acres 320
}
```

**Classic C-style `for` — explicit counter. Use when you need index control.**

```javascript
for (let i = 0; i < crops.length; i++) {
  console.log(i, crops[i]);
}
// Read as: start i=0; keep going while i < length; add 1 each pass.
```

Note `let i`, not `const` — it gets reassigned each iteration. This is the equivalent of Python's `for i in range(len(crops))`, but far more common in JS because there's no `range()`.

**`while` and `do...while`** — as expected:

```javascript
let n = 3;
while (n > 0) {
  console.log(n);
  n--;
}

do {
  console.log("runs at least once");
} while (false);
```

**`break` and `continue`** work exactly like Python.

### 3.4 There is no `range()` — the JS idioms

Python's `for i in range(5)` has no direct keyword. Options:

```javascript
// Classic for:
for (let i = 0; i < 5; i++) { console.log(i); }

// "Array from" trick (makes [0,1,2,3,4]):
Array.from({ length: 5 }, (_, i) => i).forEach(i => console.log(i));

// Or keys():
for (const i of Array(5).keys()) { console.log(i); }   // 0..4
```

The classic `for` is clearest for a simple count. The `Array.from` trick matters when you want an actual array of numbers to work with.

### 3.5 Iteration methods usually beat loops

Here's a mental-model shift that pays off enormously in React: in modern JS you often *don't write loops at all*. Instead you use array methods — `map`, `filter`, `reduce`, `forEach` — which are like Python's comprehensions and `map`/`filter`, but method-chained. These are so central they get their own big section (Part 5). A preview:

```javascript
// Python: [c.upper() for c in crops]
const upper = crops.map(c => c.toUpperCase());

// Python: [c for c in crops if len(c) > 5]
const long = crops.filter(c => c.length > 5);
```

React rendering is built on `.map()` — you'll turn an array of data into an array of UI elements. So while loops matter, the array methods in Part 5 are where you'll live.

### Practice — Part 3

1. Given `const crops = ["wheat", "canola", "lentils"]`, print each crop with its 1-based position (`"1. wheat"`, `"2. canola"`, ...) using `for...of` with `.entries()`.
2. Explain why `for (const x in ["a", "b", "c"])` prints `0 1 2` (strings!) instead of `a b c`, and what you should use instead.
3. Write a classic `for` loop that sums the numbers 1 through 100.
4. Write a `switch` that maps `"wheat"`/`"barley"` → `"cereal"`, `"canola"`/`"flax"` → `"oilseed"`, anything else → `"other"`. Watch your `break`s.
5. Produce the array `[0, 1, 2, 3, 4]` two different ways.

---

## Part 4 — Functions (the big one)

Functions are the heart of JS. This is a long part because JS has several ways to write functions, a scoping model you must understand, closures (which power React Hooks), and the notorious `this` keyword.

### 4.1 Three ways to define a function

**Function declaration** — the classic. Note: no `def`, no colon, braces required, `return` explicit.

```javascript
function netCashFlow(inflow, outflow) {
  return inflow - outflow;
}
netCashFlow(100, 30);   // 70
```

**Function expression** — a function assigned to a variable. The function itself can be anonymous.

```javascript
const netCashFlow = function (inflow, outflow) {
  return inflow - outflow;
};
```

**Arrow function** — the modern, concise form. You'll use these the most, especially in React.

```javascript
const netCashFlow = (inflow, outflow) => {
  return inflow - outflow;
};

// If the body is a single expression, drop the braces and `return` —
// the expression is returned implicitly:
const netCashFlow = (inflow, outflow) => inflow - outflow;

// One parameter: parentheses optional (but many teams keep them):
const double = x => x * 2;

// Zero parameters: empty parens required:
const now = () => Date.now();

// Returning an object literal: wrap in parentheses so { } isn't read as a block:
const makeCrop = name => ({ name, acres: 0 });
```

The arrow form's implicit return is the workhorse of array methods and React: `crops.map(c => c.toUpperCase())` reads cleanly precisely because `c => c.toUpperCase()` implicitly returns.

**When to use which:**
- **Arrow functions** for callbacks, array methods, and short helpers (the vast majority of what you write).
- **Named `function` declarations** for top-level, reusable, standalone functions where a name aids stack traces and hoisting is handy.
- **Function expressions** rarely by choice; you'll mostly see them in older code.

There is a real behavioral difference between arrow and the other two around `this` and hoisting — covered in 4.6 and 4.7.

### 4.2 Parameters: defaults, rest, and the missing-argument rule

**Default parameters** — just like Python:

```javascript
function greet(name = "farmer") {
  return `Hello, ${name}`;
}
greet();          // "Hello, farmer"
greet("Zhe");     // "Hello, Zhe"
```

**Missing arguments are `undefined`, not an error.** Unlike Python, JS does not enforce arity. Call a two-param function with zero args and the missing ones are simply `undefined`:

```javascript
function add(a, b) { return a + b; }
add(5);           // 5 + undefined = NaN   (no error!)
```

This is a frequent source of `NaN` and `undefined` bugs. Defaults protect you.

**Rest parameters** `...args` — like Python's `*args`. Collects remaining arguments into a real array:

```javascript
function total(...amounts) {
  return amounts.reduce((sum, x) => sum + x, 0);
}
total(10, 20, 30);   // 60
```

Note: JS has no `**kwargs`. The idiomatic equivalent is passing (and destructuring) an options object — covered in Parts 6–7.

There's also an old `arguments` object inside `function` (not arrow) bodies — an array-like of all args. Ignore it; use rest parameters `...args` instead.

### 4.3 Functions are first-class (same as Python)

You can pass functions as arguments, return them, and store them in variables/arrays/objects. This is identical to Python and underpins everything:

```javascript
const ops = {
  add: (a, b) => a + b,
  sub: (a, b) => a - b,
};
ops.add(2, 3);   // 5

function applyTwice(fn, x) {
  return fn(fn(x));
}
applyTwice(n => n + 1, 5);   // 7
```

A function passed to another function is called a **callback**. `crops.map(c => ...)` passes a callback. This is JS's bread and butter.

### 4.4 Scope: global, function, and block

Scope determines where a variable is visible. Python has function scope (and the `global`/`nonlocal` keywords). JS with `let`/`const` has **block scope** — a variable exists only inside the nearest `{ }`.

```javascript
function demo() {
  const outer = 1;
  if (true) {
    const inner = 2;
    console.log(outer);   // 1 — inner block can see outer
  }
  console.log(inner);     // ReferenceError — inner doesn't exist here
}
```

This is stricter and safer than Python's function scope. A `let`/`const` declared inside a loop or `if` block doesn't leak out. (The old `var` is function-scoped and *does* leak — another reason to avoid it.)

Inner functions can see outer variables (lexical scope), same as Python:

```javascript
function makeAdder(step) {
  return function (x) {
    return x + step;    // "step" is captured from the enclosing scope
  };
}
```

That capturing behavior is a **closure** — next.

### 4.5 Closures — the concept that powers React Hooks

A **closure** is a function that "remembers" the variables from the scope where it was created, even after that scope has finished executing. Python does this too, but in JS it's used far more aggressively and deliberately.

```javascript
function makeCounter() {
  let count = 0;                 // private state
  return function () {
    count += 1;                  // closes over `count`
    return count;
  };
}

const counter = makeCounter();
counter();   // 1
counter();   // 2
counter();   // 3   ← `count` persists between calls, but is unreachable from outside
```

`count` lives on because the returned function still references it. This gives you private, persistent state — a very common JS pattern (module-private variables, memoization, event handlers that remember configuration).

**Why you must understand this for React:** React's `useState` Hook is closures in action. When you write `const [value, setValue] = useState(0)`, the component function re-runs on every render, and closures are what let event handlers "capture" the value at a particular render. The most common React bug for beginners — a "stale closure" where a handler sees an old value — is *impossible to understand or fix without this concept.* So sit with the counter example until it's fully intuitive.

### 4.6 Hoisting — why some functions work before they're defined

**Hoisting** is JS moving declarations to the top of their scope before running code. It has two practical consequences:

```javascript
// This WORKS — function declarations are fully hoisted:
sayHi();
function sayHi() { console.log("hi"); }

// This FAILS — const/let are hoisted but NOT initialized
// (they sit in a "temporal dead zone" until the line runs):
sayBye();                          // ReferenceError
const sayBye = () => console.log("bye");
```

Practical rule: **define things before you use them** and you never have to think about hoisting. Function *declarations* are the one thing you can safely call before their definition, but relying on that hurts readability. Arrow functions assigned to `const` cannot be called before their line.

### 4.7 `this` — the keyword Python doesn't have

`this` is JS's most confusing feature for newcomers, and unlike Python's explicit `self`, its value depends on *how a function is called*, not where it's defined.

The short, practical version for a modern React developer:

1. **You will rarely write `this` yourself in modern React** (which uses function components, not classes). So don't over-invest here — but you must recognize it.
2. **In a regular `function`, `this` is set by the caller.** Called as `obj.method()`, `this` is `obj`. Called bare, `this` is `undefined` (in strict mode) or the global object. This dynamic binding is the source of "lost `this`" bugs.
3. **Arrow functions do NOT have their own `this`.** They inherit `this` from the surrounding scope where they were written (lexical `this`). This is usually what you want, and it's *the* reason arrows are preferred for callbacks.

```javascript
const farm = {
  name: "North Quarter",
  crops: ["wheat", "canola"],

  // Regular method: `this` is the farm when called as farm.listRegular()
  listRegular() {
    // BUG in old code: the callback here loses `this`
    this.crops.forEach(function (c) {
      // console.log(this.name, c);  // `this` is NOT farm here — undefined!
    });

    // FIX: arrow inherits `this` from listRegular's scope (the farm):
    this.crops.forEach(c => {
      console.log(this.name, c);     // "North Quarter wheat", ...
    });
  },
};
farm.listRegular();
```

Takeaway: **use arrow functions for callbacks so `this` behaves predictably**, and you'll mostly sidestep the whole `this` minefield. We revisit `this` briefly with classes in Part 9.

### Practice — Part 4

1. Write `netCashFlow(inflow, outflow)` three ways: as a function declaration, a function expression, and a one-line arrow with implicit return. Confirm all three return the same result.
2. Write an arrow function `makeCrop(name)` that returns the object `{ name, acres: 0 }`. (Watch the parentheses.)
3. Write `average(...nums)` using rest parameters and `.reduce()` that returns the mean, or `0` for no arguments.
4. Implement `makeAccumulator()` — it returns a function that keeps a running total across calls: `const acc = makeAccumulator(); acc(10); acc(5);` should return `10` then `15`. Explain which variable the closure captures.
5. Predict: does `greet()` throw or print? For `function greet() { console.log("hi"); }` called *before* its definition? What about the same as `const greet = () => console.log("hi")`?
6. In one sentence: why do arrow functions make `this` easier to reason about in callbacks?

---

## Part 5 — Arrays

Arrays are JS's list. But the *methods* — especially `map`, `filter`, `reduce` — are where you'll spend enormous amounts of time, because React rendering is built on transforming arrays. Treat this part as the most practically important after functions.

### 5.1 Basics

```javascript
const crops = ["wheat", "canola", "lentils"];

crops[0]          // "wheat"       (indexing, like Python)
crops.at(-1)      // "lentils"     (negative index needs .at())
crops[-1]         // undefined     (bracket negative index does NOT work)
crops.length      // 3             (a property, not len())
crops.length = 2  // truncates the array to ["wheat", "canola"] (!)

const empty = [];
const nums = [1, 2, 3, 4, 5];
const mixed = [1, "two", true, null, [6, 7]];   // heterogeneous, like Python lists
```

### 5.2 Mutating methods (change the array in place)

These modify the original — important to know because React prefers *not* mutating (Section 5.7).

```javascript
const a = [1, 2, 3];

a.push(4)       // add to end → returns new length; a = [1,2,3,4]  (like .append)
a.pop()         // remove & return last → 4; a = [1,2,3]
a.unshift(0)    // add to front → a = [0,1,2,3]
a.shift()       // remove & return first → 0; a = [1,2,3]
a.splice(1, 1)  // remove 1 element at index 1 → a = [1,3]  (like del a[1])
a.splice(1, 0, "x")  // insert "x" at index 1 without removing → a = [1,"x",3]
a.sort()        // sorts IN PLACE — see the gotcha below
a.reverse()     // reverses in place
```

**The `.sort()` gotcha:** by default `.sort()` converts everything to strings and sorts lexicographically. So `[10, 2, 1].sort()` gives `[1, 10, 2]`, not `[1, 2, 10]`. For numbers you must pass a comparator:

```javascript
[10, 2, 1].sort((a, b) => a - b);   // [1, 2, 10]   ascending
[10, 2, 1].sort((a, b) => b - a);   // [10, 2, 1]   descending
```

The comparator returns negative if `a` should come first, positive if `b` should, zero if equal. Memorize `(a, b) => a - b` for ascending numbers.

### 5.3 Non-mutating methods (return a new array/value)

These leave the original untouched — the ones React loves.

```javascript
const nums = [1, 2, 3, 4, 5];

nums.slice(1, 3)     // [2, 3]     (like nums[1:3]; original unchanged)
nums.slice(-2)       // [4, 5]
nums.concat([6, 7])  // [1..7]     (merge; or use spread — Part 8)
nums.includes(3)     // true       (like `3 in nums`)
nums.indexOf(3)      // 2          (-1 if absent)
nums.join(", ")      // "1, 2, 3, 4, 5"
[...nums].reverse()  // reverse a COPY (spread first to avoid mutating)
```

### 5.4 The big three: `map`, `filter`, `reduce`

These replace most loops. If you internalize these, you've learned the most transferable skill for React.

**`.map(fn)` — transform every element. Like Python's `[fn(x) for x in list]`.** Returns a *new* array of the same length.

```javascript
const nums = [1, 2, 3];
nums.map(n => n * 2);                    // [2, 4, 6]

const crops = ["wheat", "canola"];
crops.map(c => c.toUpperCase());         // ["WHEAT", "CANOLA"]

// The callback also receives index and the whole array:
crops.map((c, i) => `${i}: ${c}`);       // ["0: wheat", "1: canola"]

// map over objects — extremely common:
const records = [{ crop: "wheat", acres: 100 }, { crop: "canola", acres: 80 }];
records.map(r => r.crop);                // ["wheat", "canola"]
```

In React: `crops.map(c => <li key={c}>{c}</li>)` turns data into UI. This single pattern is ~80% of how lists get rendered.

**`.filter(fn)` — keep elements where the callback returns truthy. Like `[x for x in list if cond]`.**

```javascript
const nums = [1, 2, 3, 4, 5, 6];
nums.filter(n => n % 2 === 0);           // [2, 4, 6]

records.filter(r => r.acres > 90);       // [{ crop: "wheat", acres: 100 }]
```

**`.reduce(fn, initial)` — collapse an array to a single value. Like `functools.reduce`.** The workhorse for sums, groupings, and building objects.

```javascript
const nums = [1, 2, 3, 4];

// Sum: accumulator starts at 0, add each element
nums.reduce((acc, n) => acc + n, 0);     // 10

// Max acres across records:
records.reduce((max, r) => Math.max(max, r.acres), 0);   // 100

// Build an object (group/index) — the powerful pattern:
records.reduce((obj, r) => {
  obj[r.crop] = r.acres;
  return obj;
}, {});                                  // { wheat: 100, canola: 80 }
```

`reduce` reads: "start with `initial`; for each element, combine it with the running accumulator; return the final accumulator." The initial value is important — always provide it. Once `reduce` clicks, you can express almost any aggregation. This maps directly onto your ETL/aggregation instincts.

### 5.5 Other essential iteration methods

```javascript
const records = [{ crop: "wheat", acres: 100 }, { crop: "canola", acres: 80 }];

// forEach — run a side effect for each; returns nothing. Use when NOT building a value.
records.forEach(r => console.log(r.crop));

// find — first element matching; else undefined (like next(x for x in ... if))
records.find(r => r.crop === "canola");        // { crop: "canola", acres: 80 }
records.findIndex(r => r.crop === "canola");   // 1

// some — is ANY element true? (like Python any())
records.some(r => r.acres > 90);               // true

// every — are ALL elements true? (like Python all())
records.every(r => r.acres > 50);              // true

// flatMap — map then flatten one level
[[1, 2], [3]].flatMap(x => x);                 // [1, 2, 3]

// flat — flatten nested arrays
[1, [2, [3]]].flat();                          // [1, 2, [3]]  (one level)
[1, [2, [3]]].flat(Infinity);                  // [1, 2, 3]    (fully)
```

**`map` vs `forEach`:** use `map` when you want a new array back; use `forEach` for pure side effects (logging, mutating external state). A common beginner mistake is using `map` and ignoring its return — if you're not using the result, it should be `forEach`.

### 5.6 Method chaining — the pipeline style

Because non-mutating methods return arrays, you chain them into readable pipelines. This is your ETL transform-chain instinct, expressed inline:

```javascript
const records = [
  { crop: "wheat",   acres: 100, active: true },
  { crop: "canola",  acres: 80,  active: false },
  { crop: "lentils", acres: 60,  active: true },
];

const activeCropAcres = records
  .filter(r => r.active)          // keep active
  .map(r => r.acres)              // extract acres
  .reduce((sum, a) => sum + a, 0);// total them

console.log(activeCropAcres);     // 160
```

Read top-down: filter, then map, then reduce. Each step's output feeds the next. This is exactly the mental model of a materialized transform pipeline, just chained in memory.

### 5.7 Immutability — the React-critical habit

React decides whether to re-render by comparing whether data *references* changed, not deep contents. So in React you almost never mutate arrays; you create new ones. Learn the non-mutating equivalents now:

```javascript
const crops = ["wheat", "canola"];

// Add (instead of push):
const added = [...crops, "lentils"];          // new array, original intact

// Remove (instead of splice):
const removed = crops.filter(c => c !== "canola");

// Update an item (instead of crops[0] = ...):
const updated = crops.map(c => c === "wheat" ? "hard wheat" : c);

// Copy before sorting (sort mutates!):
const sorted = [...crops].sort();
```

`...crops` is the spread operator (Part 8). Building these reflexes now means React state updates will feel natural instead of mysterious.

### 5.8 Creating arrays programmatically

```javascript
Array.from({ length: 3 }, (_, i) => i);   // [0, 1, 2]
Array(3).fill(0);                         // [0, 0, 0]
Array.from("abc");                        // ["a", "b", "c"]
[...Array(3).keys()];                     // [0, 1, 2]
```

### Practice — Part 5

Use this data for several questions:
```javascript
const records = [
  { crop: "wheat",   acres: 100, revenue: 45000, active: true },
  { crop: "canola",  acres: 80,  revenue: 60000, active: true },
  { crop: "lentils", acres: 60,  revenue: 30000, active: false },
  { crop: "barley",  acres: 40,  revenue: 18000, active: true },
];
```

1. Using `.map()`, produce an array of just the crop names.
2. Using `.filter()`, get only the active records, then chain `.map()` to get their names. (One expression.)
3. Using `.reduce()`, compute total revenue across all records.
4. Using `.reduce()`, build the object `{ wheat: 100, canola: 80, lentils: 60, barley: 40 }` (crop → acres).
5. Find the record with the highest revenue (use `.reduce()`), then separately find the first record with `acres < 50` (use `.find()`).
6. Sort the records by revenue descending *without mutating* the original array. (Spread, then `.sort()`.)
7. Explain why `[10, 9, 100].sort()` gives `[10, 100, 9]` and how to fix it.
8. Given `const crops = ["wheat", "canola"]`, produce a *new* array with `"lentils"` added and `"wheat"` removed, without mutating `crops`.

---

## Part 6 — Objects

A JS object is your Python `dict` *and* your struct/class-instance rolled into one. This dual role is central. Most data flowing through a React app — API responses, component props, state — is objects (and arrays of objects).

### 6.1 Creating and accessing

```javascript
const farm = {
  name: "North Quarter",     // keys are strings (quotes optional if valid identifier)
  acres: 320,
  active: true,
  crops: ["wheat", "canola"],
};

// Two ways to access (this is the dict/attribute duality):
farm.name          // "North Quarter"   — dot notation (when key is a valid identifier)
farm["name"]       // "North Quarter"   — bracket notation (like Python dict[key])

// Bracket notation is required for dynamic keys and non-identifier keys:
const key = "acres";
farm[key]          // 320
farm["total-acres"]// dot won't work for hyphenated keys
```

Unlike Python dicts, JS object keys are always strings (or symbols) under the hood. `{ 1: "a" }` stores the key as `"1"`.

### 6.2 Adding, updating, deleting

```javascript
farm.owner = "Zhe";           // add or update
farm["region"] = "SW Sask";   // add via bracket
delete farm.active;           // remove a key (like del dict[key])

"name" in farm                // true  (key existence, like Python's `in`)
farm.hasOwnProperty("name")   // true  (own property, ignoring inherited)
farm.missing                  // undefined  (missing key → undefined, NOT KeyError)
```

Key difference from Python: accessing a missing key returns `undefined` rather than raising `KeyError`. This is why optional chaining (`?.`) and defaults (`??`) matter so much.

### 6.3 Shorthand, methods, and computed keys — modern conveniences

```javascript
const name = "wheat";
const acres = 100;

// Property shorthand: { name: name } → { name } when var name matches key
const crop = { name, acres };          // { name: "wheat", acres: 100 }

// Method shorthand:
const calc = {
  add(a, b) { return a + b; },         // same as add: function(a,b){...}
  scale: (x) => x * 2,                 // arrow as a value (no own `this`)
};
calc.add(2, 3);                        // 5

// Computed property names: use an expression as a key
const field = "revenue";
const record = { [field]: 45000 };     // { revenue: 45000 }
```

Property shorthand (`{ name, acres }`) is everywhere in React — passing props, building state objects. Get comfortable reading it.

### 6.4 Iterating over objects

Objects are not directly iterable with `for...of`. Convert to arrays first:

```javascript
const farm = { name: "North", acres: 320, active: true };

Object.keys(farm)     // ["name", "acres", "active"]        (like dict.keys())
Object.values(farm)   // ["North", 320, true]              (like dict.values())
Object.entries(farm)  // [["name","North"], ["acres",320], ["active",true]]  (like dict.items())

// Idiomatic iteration:
for (const [key, value] of Object.entries(farm)) {
  console.log(key, value);
}

// Transform an object (there's no dict-comprehension; use entries + map + fromEntries):
const doubled = Object.fromEntries(
  Object.entries({ a: 1, b: 2 }).map(([k, v]) => [k, v * 2])
);                    // { a: 2, b: 4 }
```

`Object.fromEntries` is the inverse of `Object.entries` — together they're your "dict comprehension" toolkit.

### 6.5 Copying and merging — and the shallow-copy trap

```javascript
const base = { name: "North", acres: 320 };

// Spread copy (shallow) — the modern idiom:
const copy = { ...base };                     // new object, same top-level values
const merged = { ...base, acres: 400 };       // override acres → { name:"North", acres:400 }
const combined = { ...base, ...{ owner: "Zhe" } };  // merge two objects

// Object.assign does the same (older style):
const copy2 = Object.assign({}, base);
```

**Shallow-copy trap:** spread copies only one level deep. Nested objects/arrays are shared by reference:

```javascript
const original = { name: "North", crops: ["wheat"] };
const shallow = { ...original };
shallow.crops.push("canola");     // MUTATES original.crops too — same array!
console.log(original.crops);      // ["wheat", "canola"]  ← surprise
```

For nested updates (common in React state), spread at each level you change:

```javascript
const updated = {
  ...original,
  crops: [...original.crops, "canola"],   // new array, original safe
};
```

For a full deep copy of plain data, `structuredClone(obj)` works in modern environments, or `JSON.parse(JSON.stringify(obj))` for JSON-safe data (loses functions, dates become strings).

### 6.6 Objects are reference types (equality gotcha)

Like Python, objects and arrays compare by *identity*, not contents:

```javascript
{ a: 1 } === { a: 1 }        // false — different objects
[1, 2] === [1, 2]            // false

const x = { a: 1 };
const y = x;
x === y                      // true  — same reference
y.a = 99;
console.log(x.a);            // 99    — same object, both names point to it
```

This is identical to Python's `is` vs `==` distinction — except JS's `===` behaves like Python's `is` for objects, and there is no built-in deep-equality `==`. To compare contents, compare fields explicitly or (for plain data) `JSON.stringify(a) === JSON.stringify(b)` as a quick hack.

### 6.7 JSON — parsing and stringifying

Your FastAPI backend speaks JSON; the frontend converts constantly.

```javascript
// Object → JSON string (like json.dumps):
const json = JSON.stringify({ crop: "wheat", acres: 100 });
// '{"crop":"wheat","acres":100}'

JSON.stringify(obj, null, 2);   // pretty-print with 2-space indent

// JSON string → object (like json.loads):
const obj = JSON.parse('{"crop":"wheat","acres":100}');
obj.crop;                       // "wheat"
```

JSON keys must be double-quoted strings; `undefined`, functions, and `NaN` don't survive stringify (they're dropped or become `null`). When you `fetch()` from your API (Part 11), you'll call `response.json()` which does the parse for you.

### Practice — Part 6

1. Create a `farm` object with `name`, `acres`, and a `crops` array. Add an `owner` key, update `acres`, then delete `owner`.
2. Given `const field = "revenue"` and `const amount = 45000`, build `{ revenue: 45000 }` using a computed key and property shorthand.
3. Loop over `{ wheat: 100, canola: 80 }` with `Object.entries` and print `"wheat: 100 acres"` for each.
4. Turn `{ a: 1, b: 2, c: 3 }` into `{ a: 10, b: 20, c: 30 }` using `Object.entries`, `.map`, and `Object.fromEntries`.
5. Given `const original = { name: "N", crops: ["wheat"] }`, produce a copy with `"canola"` added to crops such that `original.crops` stays `["wheat"]`. Then explain why a plain `{ ...original }` followed by `.crops.push()` would fail this requirement.
6. Convert `{ crop: "wheat", acres: 100 }` to a pretty-printed JSON string, then parse it back.

---

## Part 7 — Destructuring

Destructuring is unpacking values from arrays and objects into variables. Python has array-ish unpacking (`a, b = [1, 2]`); JS's version is more powerful and *pervasive in React*. Every React component you write will destructure props; every `useState` call destructures an array. Master this.

### 7.1 Array destructuring

```javascript
const coords = [50.28, -107.79];

const [lat, lng] = coords;        // lat = 50.28, lng = -107.79

// Skip elements with commas:
const [first, , third] = [1, 2, 3];   // first=1, third=3

// Rest into an array (like Python's *rest):
const [head, ...tail] = [1, 2, 3, 4]; // head=1, tail=[2,3,4]

// Defaults for missing values:
const [a, b, c = 0] = [1, 2];     // a=1, b=2, c=0

// Swap without a temp variable:
let x = 1, y = 2;
[x, y] = [y, x];                  // x=2, y=1
```

**The React connection:** `useState` returns a two-element array `[value, setterFunction]`, and you destructure it by position:

```javascript
const [count, setCount] = useState(0);   // count = 0, setCount = the updater
```

The names are yours to choose — position is what matters. This is why `useState` destructuring "just works" once you understand array destructuring.

### 7.2 Object destructuring

This is the big one for React (props are objects).

```javascript
const farm = { name: "North", acres: 320, owner: "Zhe" };

// Pull keys into same-named variables:
const { name, acres } = farm;         // name = "North", acres = 320

// Rename while destructuring (key:newName):
const { name: farmName } = farm;      // farmName = "North"  (no `name` variable created)

// Defaults for missing keys:
const { region = "unknown" } = farm;  // region = "unknown"  (farm has no region)

// Rename AND default together:
const { region: r = "SW Sask" } = farm;

// Rest into an object (collect the leftovers):
const { name: n, ...rest } = farm;    // n = "North", rest = { acres: 320, owner: "Zhe" }
```

### 7.3 Destructuring in function parameters — the `**kwargs` replacement

Since JS has no keyword arguments, the idiom is to accept a single options object and destructure it in the parameter list. This is *the* standard pattern for functions with many/optional arguments — and it's exactly how React components receive props.

```javascript
// Instead of positional args you must remember the order of:
function createForecast({ crop, acres, revenue = 0, active = true }) {
  return `${crop}: ${acres} acres, $${revenue}, ${active ? "active" : "inactive"}`;
}

// Call with a self-documenting object — order doesn't matter, extras ignored:
createForecast({ crop: "wheat", acres: 100, revenue: 45000 });
// "wheat: 100 acres, $45000, active"

// Provide a default for the whole object so calling with no args doesn't crash:
function greet({ name = "farmer" } = {}) {
  return `Hi ${name}`;
}
greet();                         // "Hi farmer"  — the `= {}` prevents a crash
```

**A React component is literally this pattern:**

```javascript
function CropCard({ crop, acres, active = false }) {
  return <div>{crop}: {acres} acres</div>;
}
// Used as: <CropCard crop="wheat" acres={100} />
// React collects the attributes into a props object and calls CropCard(props),
// which destructures crop, acres, active out of it.
```

Once you see that props are just a destructured object argument, React components stop looking magical.

### 7.4 Nested destructuring

You can destructure deeply, mixing arrays and objects. Powerful but readable only in moderation:

```javascript
const response = {
  farm: { name: "North", location: { lat: 50.28, lng: -107.79 } },
  crops: ["wheat", "canola"],
};

const {
  farm: { name, location: { lat } },
  crops: [firstCrop],
} = response;
// name = "North", lat = 50.28, firstCrop = "wheat"
// NOTE: `farm` and `location` themselves are NOT created as variables here —
// only the leaf names (name, lat) are.
```

Deep destructuring is common when pulling specific fields out of API responses. Don't over-nest; two levels is usually the readability limit.

### Practice — Part 7

1. Given `const point = [50.28, -107.79, 825]`, destructure into `lat`, `lng`, and `elevation`.
2. From `const [head, ...tail] = [1, 2, 3, 4, 5]`, what are `head` and `tail`?
3. Given `const farm = { name: "North", acres: 320 }`, destructure `name`, and destructure `acres` renamed to `size`, and pull `region` with a default of `"SW Sask"`.
4. Write `describe({ crop, acres = 0 })` that returns `"crop on acres acres"`, and make it callable as `describe()` without crashing.
5. Rewrite this React-style: given `function Card(props)` that uses `props.title` and `props.value`, rewrite the signature to destructure `title` and `value` directly, with `value` defaulting to `0`.
6. From `const data = { user: { name: "Zhe", roles: ["admin", "dev"] } }`, destructure the name and the first role in a single statement.

---

## Part 8 — Spread and rest (`...`)

The three-dot `...` operator does two opposite jobs depending on context. Both are indispensable in modern JS and React. You've already seen glimpses; here they are consolidated.

### 8.1 Spread — "unpack this into individual pieces"

Spread *expands* an iterable/object into its elements. Think Python's `*args`/`**kwargs` at the *call/construction* site.

```javascript
// Spread arrays into a new array (copy, merge, insert):
const a = [1, 2];
const b = [3, 4];
const merged = [...a, ...b];        // [1, 2, 3, 4]
const copy = [...a];                // [1, 2]   shallow copy
const inserted = [0, ...a, 3];      // [0, 1, 2, 3]

// Spread into function arguments (like Python f(*args)):
Math.max(...[5, 1, 9, 3]);          // 9   (Math.max wants separate args)

// Spread objects into a new object (copy, merge, override):
const base = { name: "North", acres: 320 };
const updated = { ...base, acres: 400 };        // { name: "North", acres: 400 }
const withOwner = { ...base, owner: "Zhe" };    // add a field

// Spread a string into characters:
[..."abc"];                         // ["a", "b", "c"]
```

**Spread is how you do immutable updates**, the core React state habit:

```javascript
// Array: add / remove / replace without mutation
const nextCrops = [...crops, "lentils"];              // add
const fewer = crops.filter(c => c !== "wheat");       // remove
const changed = crops.map(c => c === "wheat" ? "hard wheat" : c);  // replace

// Object: update a field without mutation (the useState pattern)
const nextFarm = { ...farm, acres: farm.acres + 10 };
```

Order matters in object spread: later keys win. `{ ...base, acres: 400 }` overrides; `{ acres: 400, ...base }` would let `base.acres` win.

### 8.2 Rest — "collect the remaining pieces into one"

Rest *gathers* multiple items into a single array or object. Same `...` syntax, but it appears in a *binding* position (function parameters or destructuring left-hand side). Think Python's `*args`/`**kwargs` at the *definition* site.

```javascript
// Rest in function parameters (like *args):
function sum(...nums) {
  return nums.reduce((a, n) => a + n, 0);
}
sum(1, 2, 3, 4);                    // 10

// Mixed: some named, rest collected:
function log(level, ...messages) {
  console.log(level, messages);    // messages is an array
}

// Rest in array destructuring:
const [first, ...others] = [1, 2, 3];   // first=1, others=[2,3]

// Rest in object destructuring (collect leftover keys):
const { name, ...meta } = { name: "North", acres: 320, owner: "Zhe" };
// name = "North", meta = { acres: 320, owner: "Zhe" }
```

### 8.3 Telling spread from rest

Same symbol, opposite jobs — here's the rule:

- **Spread** appears where *values are consumed*: inside `[...]`, `{...}`, or a function *call* `fn(...)`. It expands one thing into many.
- **Rest** appears where *names are bound*: in a function *parameter list* or the left side of a destructuring `=`. It collects many things into one.

```javascript
const combined = [...a, ...b];     // SPREAD (building an array)
const [x, ...ys] = combined;       // REST   (binding names)
fn(...args);                       // SPREAD (calling)
function fn(...args) {}            // REST   (defining)
```

If it's producing/passing values → spread. If it's capturing into a variable → rest.

### 8.4 A very common React pattern: prop forwarding

Rest + spread combine to "take some props, pass the rest along":

```javascript
function Button({ variant, ...rest }) {
  // pull out `variant` for our own use; forward everything else to the <button>
  return <button className={variant} {...rest} />;
}
// <Button variant="primary" onClick={fn} disabled />
// → onClick and disabled land on the actual <button> via {...rest}
```

You don't need to write this yet, but recognizing it now means it won't surprise you later.

### Practice — Part 8

1. Merge `const a = [1, 2]` and `const b = [3, 4]` into `[1, 2, 3, 4]` using spread. Then make a shallow copy of `a`.
2. Use spread to call `Math.min` on the array `[8, 3, 11, 5]`.
3. Given `const farm = { name: "North", acres: 320 }`, produce a new object with `acres` set to `500` and an added `owner: "Zhe"`, without mutating `farm`.
4. Write `function concatAll(...arrays)` that flattens any number of arrays into one, using rest and spread.
5. From `const { id, ...rest } = { id: 7, crop: "wheat", acres: 100 }`, what are `id` and `rest`?
6. For each, label spread or rest: `[...xs]`, `function f(...xs)`, `f(...xs)`, `const [a, ...b] = xs`, `{ ...obj }`.

---

## Part 9 — Classes and prototypes

JS has `class` syntax that looks a lot like Python's, but underneath it works via **prototypes**, not the class-based model Python uses. For modern React (function components + Hooks) you'll write classes rarely — but you must read them, and understanding prototypes explains a lot of JS behavior. Keep this part in perspective: understand it, don't obsess over it.

### 9.1 Class syntax (the familiar-looking part)

```javascript
class Farm {
  // The constructor — like Python's __init__. `this` is the instance (like self,
  // but implicit — you don't declare it as a parameter).
  constructor(name, acres) {
    this.name = name;         // instance properties
    this.acres = acres;
  }

  // A method (no `function` keyword, no comma between members):
  describe() {
    return `${this.name}: ${this.acres} acres`;
  }

  // Getter — accessed like a property, not called:
  get sizeClass() {
    return this.acres > 200 ? "large" : "small";
  }

  // Setter:
  set expand(extra) {
    this.acres += extra;
  }

  // Static method — belongs to the class, not instances (like @staticmethod):
  static fromRecord(record) {
    return new Farm(record.name, record.acres);
  }
}

const f = new Farm("North", 320);   // `new` is required (Python has no `new`)
f.describe();       // "North: 320 acres"
f.sizeClass;        // "large"   (getter — no parentheses)
f.expand = 50;      // setter — f.acres becomes 370
Farm.fromRecord({ name: "South", acres: 90 });   // static call on the class
```

Key differences from Python:
- `constructor`, not `__init__`.
- `this`, not `self`, and it's **implicit** — never listed as a parameter.
- `new` is **mandatory** to instantiate (forgetting it is a classic bug; strict mode throws).
- Methods are listed without `function` and without commas between them.

### 9.2 Inheritance

```javascript
class Farm {
  constructor(name) { this.name = name; }
  describe() { return `Farm ${this.name}`; }
}

class GrainFarm extends Farm {
  constructor(name, crops) {
    super(name);            // MUST call super() before using `this`
    this.crops = crops;
  }
  describe() {
    return `${super.describe()} grows ${this.crops.join(", ")}`;
  }
}

const g = new GrainFarm("North", ["wheat", "canola"]);
g.describe();   // "Farm North grows wheat, canola"
g instanceof GrainFarm;   // true
g instanceof Farm;        // true (inheritance chain)
```

`extends` and `super` mirror Python's inheritance and `super().__init__()`. The rule that `super()` must run before touching `this` in a subclass constructor is stricter than Python.

### 9.3 The prototype reality underneath

Here's what's actually happening, because it explains JS quirks. Every JS object has a hidden link to a **prototype** object. When you access `obj.foo` and `obj` doesn't have `foo`, JS walks up the prototype chain looking for it. Methods defined in a `class` actually live on the prototype, shared by all instances (not copied per instance).

```javascript
const arr = [1, 2, 3];
// arr doesn't literally contain a `map` function; arr's prototype (Array.prototype)
// does. That's why every array has .map, .filter, etc.
arr.map;                          // function (found via the prototype chain)
Object.getPrototypeOf(arr) === Array.prototype;   // true
```

This is why `typeof [].map` is `"function"` even though you never defined it, and why adding a method to `Array.prototype` (don't) would make it appear on every array. `class` is essentially syntactic sugar over this prototype mechanism. You don't need to manipulate prototypes directly in application code — just know the chain exists and that it's how method lookup works.

### 9.4 Why React moved away from classes (context, not a task)

Early React used classes (`class App extends React.Component`) with `this.state`, `this.setState`, and lifecycle methods. Because `this` binding in class methods is error-prone (event handlers routinely "lost" their `this`), modern React uses **function components with Hooks** instead — plain functions plus `useState`, `useEffect`, etc. You'll learn that style. So for GrowLytics you'll mostly write functions, not classes. Still, you'll encounter class components in older codebases and some libraries, so being able to read the above matters.

### Practice — Part 9

1. Write a `class Crop` with a constructor taking `name` and `acres`, a `describe()` method returning `"name (acres ac)"`, and a getter `isLarge` returning `true` when `acres > 50`. Instantiate it and call both.
2. Add a `static` method `Crop.fromArray([name, acres])` that constructs a `Crop` from a two-element array.
3. Write `class Cereal extends Crop` that adds a `yield` property and overrides `describe()` to append `", yield: <yield>"` using `super.describe()`.
4. What error occurs if you write `const c = Crop("wheat", 100)` (forgetting `new`)? Why?
5. In one sentence, explain where an array's `.map` method actually lives and how JS finds it when you call `arr.map(...)`.

---

## Part 10 — Modules (the Vite connection)

Modules are how you split code across files and share it — JS's equivalent of Python's `import`/`from ... import`. This is directly relevant to your Vite/React setup: every `.jsx` file is a module, and Vite's whole job revolves around resolving and bundling these `import` statements.

### 10.1 The modern system: ES Modules (ESM)

Modern JS uses `import`/`export`. There are two flavors of export: **named** and **default**.

**Named exports** — export multiple things by name (like exporting several functions from a Python module):

```javascript
// file: queries.js
export function getCropAcres(db, crop) { /* ... */ }
export function getNetCashFlow(db, month) { /* ... */ }
export const DEFAULT_REVISION = 1;

// You can also declare first, then export together:
function helper() {}
const CONSTANT = 42;
export { helper, CONSTANT };
```

```javascript
// file: api.js — import named exports by their exact names, in braces
import { getCropAcres, DEFAULT_REVISION } from "./queries.js";

// Rename on import (like Python's `import x as y`):
import { getNetCashFlow as netFlow } from "./queries.js";

// Import everything as a namespace object (like `import module`):
import * as queries from "./queries.js";
queries.getCropAcres(db, "wheat");
```

Named imports must match the exported names exactly (braces, exact spelling). This is like `from queries import get_crop_acres`.

**Default export** — one "main" export per file:

```javascript
// file: ForecastExplorer.jsx
export default function ForecastExplorer() { /* ... */ }

// or: export default SomethingAlreadyDefined;
```

```javascript
// Import a default with ANY name you choose, no braces:
import ForecastExplorer from "./ForecastExplorer.jsx";
import Explorer from "./ForecastExplorer.jsx";   // same thing, your choice of name
```

A file can have **one** default export and **many** named exports, and you can mix them:

```javascript
import ForecastExplorer, { DEFAULT_REVISION } from "./ForecastExplorer.jsx";
// default (any name) first, then named (braces)
```

**React convention:** a component file usually `export default`s its component and named-exports helpers/constants. `import App from "./App.jsx"` is a default import; `import { useState } from "react"` is a named import from the React package.

### 10.2 Import paths

```javascript
import x from "./local.js";        // relative path — a file you wrote (the ./ is required)
import y from "../utils/fmt.js";   // parent directory
import { useState } from "react";  // bare specifier — a package from node_modules
```

Relative imports start with `./` or `../` and point to your files. "Bare" specifiers (no `./`) refer to installed npm packages. Vite resolves both. In Vite projects you'll often see imports without file extensions (`"./App"`) — the bundler fills them in; both styles appear in the wild.

### 10.3 The old system: CommonJS (`require`) — recognize only

Node historically used a different system you'll see in older code and config files:

```javascript
// CommonJS — the OLD style (still common in Node config files):
const express = require("express");     // instead of import
module.exports = { getCropAcres };      // instead of export
```

You'll write ESM (`import`/`export`) in your Vite/React app. But `require`/`module.exports` shows up in tooling configs and older tutorials — now you can recognize it. Don't mix the two styles in one file.

### 10.4 How this connects to Vite

When you write `import ForecastExplorer from "./ForecastExplorer.jsx"`, the browser can't natively resolve that the way Node can, and it certainly can't understand JSX. Vite (in dev) serves modules on the fly and transforms JSX→JS; Vite (in build) bundles everything into optimized files. Every `import`/`export` in your source is a node in the dependency graph Vite walks. So the module system isn't just organization — it's the literal input to your build tool. Your existing mental model of "a build step that resolves and materializes a dependency graph" maps cleanly here.

### Practice — Part 10

1. In a file `mathUtils.js`, write and export (named) two functions: `sum(nums)` and `mean(nums)`. In another file, import both and use them.
2. Change `sum` to a default export and `mean` to remain named. Rewrite the import line to bring in both correctly.
3. Import React's `useState` and `useEffect` in a single line. Are these named or default imports? How can you tell?
4. Explain the difference between `import Thing from "./x.js"` and `import { Thing } from "./x.js"` — what must be true of `x.js` in each case?
5. You see `const fs = require("fs")` in a file. Which module system is this, and what's the ESM equivalent?

---

## Part 11 — Asynchronous JavaScript

This is the part that most changes how a backend Python developer thinks. In JS, "wait for something slow" (a network request, a timer, reading a file) is almost never blocking. Your React app fetching from your FastAPI backend is fundamentally async. Take your time here; it's the highest-leverage section for real frontend work.

### 11.1 The mental model: single thread, event loop, non-blocking

JS runs on **one thread**. If it blocked while waiting for a 200ms network response, the entire UI would freeze — no clicks, no scrolling. So instead, JS uses this model:

1. Run synchronous code top to bottom.
2. When something slow starts (a fetch, a timer), hand it off to the environment and **keep going** — don't wait.
3. When the slow thing finishes, its callback is queued.
4. The **event loop** runs queued callbacks once the main code is idle.

The consequence: code doesn't necessarily run in the order it's written. This is the core surprise.

```javascript
console.log("1");
setTimeout(() => console.log("2"), 0);   // "run this later", even with 0ms delay
console.log("3");
// Output: 1, 3, 2   ← NOT 1, 2, 3
```

`setTimeout(fn, 0)` doesn't run `fn` immediately — it schedules it to run *after* the current synchronous code finishes. This tiny example captures the whole model: synchronous first, queued callbacks after.

Python has `asyncio` and this will feel somewhat familiar if you've used it — but in JS async is the default fabric of the language, not an opt-in library.

### 11.2 Promises — a value that arrives later

A **Promise** is an object representing a value that isn't ready yet. It's in one of three states: *pending*, *fulfilled* (resolved with a value), or *rejected* (failed with an error). It's conceptually like a Python `Future`.

You mostly *consume* promises returned by APIs rather than creating them. Consuming with `.then()`/`.catch()`:

```javascript
fetch("/api/crops")                     // returns a Promise
  .then(response => response.json())    // .then runs when it resolves; returns a Promise
  .then(data => console.log(data))      // chain: each .then gets the previous result
  .catch(error => console.error(error))// .catch runs if ANY step rejects
  .finally(() => console.log("done"));  // always runs
```

Each `.then` receives the previous step's result and can return a new value (or another promise) to pass down the chain. `.catch` handles a rejection anywhere upstream. This chaining avoids the old "callback pyramid of doom." You should be able to *read* `.then` chains, but you'll mostly *write* the nicer `async/await` form below.

Creating a promise (occasionally needed, e.g. to wrap a timer):

```javascript
const wait = (ms) => new Promise(resolve => setTimeout(resolve, ms));
wait(1000).then(() => console.log("1 second later"));
```

### 11.3 `async` / `await` — write async code that reads like sync

This is the modern, preferred style and will feel natural coming from Python's `async`/`await` — the syntax is nearly identical.

- Mark a function `async` — it now always returns a promise.
- Inside it, `await` a promise to pause until it resolves, getting the value directly.

```javascript
async function loadCrops() {
  const response = await fetch("/api/crops");   // pause until response arrives
  const data = await response.json();           // pause until body is parsed
  return data;                                  // this becomes the resolved value
}

// loadCrops() itself returns a promise, so you await it (or .then it):
async function main() {
  const crops = await loadCrops();
  console.log(crops);
}
main();
```

This reads top-to-bottom like synchronous code, but each `await` yields control so the UI stays responsive. It's the same shape as Python's `async def` / `await`. Two rules:

- `await` only works **inside an `async` function** (or at the top level of a module in modern setups).
- An `async` function **always returns a promise**, even if you `return` a plain value. Callers must `await` it or `.then` it.

### 11.4 Error handling with `try/catch` (the clean part)

With `async/await`, errors from `await`ed promises are caught with ordinary `try/catch` — much nicer than `.catch()` chains and identical in spirit to Python:

```javascript
async function loadCrops() {
  try {
    const response = await fetch("/api/crops");
    if (!response.ok) {
      // fetch does NOT reject on HTTP 404/500 — you must check manually:
      throw new Error(`HTTP ${response.status}`);
    }
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Failed to load crops:", error.message);
    return [];                    // sensible fallback
  } finally {
    console.log("load attempt finished");
  }
}
```

**Critical `fetch` gotcha:** `fetch` only rejects on *network* failure (no connection, DNS error). It does **not** reject on HTTP error statuses like 404 or 500 — those resolve normally with `response.ok === false`. You must check `response.ok` yourself and throw. Python's `requests` doesn't raise on 404 either unless you call `raise_for_status()`; same idea, but easy to forget.

### 11.5 Running promises in parallel

Sequential `await`s run one after another. When operations are independent, run them concurrently with `Promise.all` (like `asyncio.gather`):

```javascript
// SLOW — sequential (each waits for the previous):
const crops = await fetchCrops();
const prices = await fetchPrices();      // only starts after crops finishes

// FAST — parallel (both start immediately, wait for both):
const [crops, prices] = await Promise.all([fetchCrops(), fetchPrices()]);
```

`Promise.all` rejects if *any* promise rejects. Related tools: `Promise.allSettled` (wait for all, capture successes and failures individually), `Promise.race` (first to settle wins), `Promise.any` (first to *succeed*).

### 11.6 A realistic GrowLytics-shaped example

Fetching forecast data from your FastAPI backend and handling states — the exact pattern you'll wrap in a React `useEffect` later:

```javascript
async function fetchForecast(month) {
  try {
    const res = await fetch(`/api/forecast?month=${month}`);
    if (!res.ok) throw new Error(`Server returned ${res.status}`);
    const data = await res.json();
    // transform the API shape into what the UI wants:
    return data.records.map(r => ({
      crop: r.crop,
      net: r.inflow - r.outflow,
    }));
  } catch (err) {
    console.error("forecast fetch failed:", err.message);
    return [];
  }
}

// Usage:
const forecast = await fetchForecast("2026-07");
```

When you reach React, this logic moves inside a `useEffect`, storing the result via `useState` — but the async core is exactly what you're learning here.

### 11.7 Common async pitfalls

```javascript
// 1) Forgetting await → you get a Promise, not the value:
const data = fetchForecast("2026-07");   // data is a PENDING PROMISE
console.log(data.length);                // undefined / error — it's not the array

// 2) await inside a plain forEach does NOT wait as you'd expect:
[1, 2, 3].forEach(async (x) => { await something(x); });   // fire-and-forget, unordered
// Use a for...of loop for sequential awaits:
for (const x of [1, 2, 3]) { await something(x); }
// Or Promise.all + map for parallel:
await Promise.all([1, 2, 3].map(x => something(x)));
```

The "forgot `await`" bug is the most common async mistake — if a value is mysteriously a `Promise { <pending> }`, you forgot to await it.

### Practice — Part 11

1. Predict the exact output order:
   ```javascript
   console.log("A");
   setTimeout(() => console.log("B"), 0);
   console.log("C");
   ```
   Explain why.
2. Write `wait(ms)` returning a promise that resolves after `ms` milliseconds (wrap `setTimeout`). Then use it with `.then()` to log `"done"` after 500ms.
3. Rewrite this `.then` chain using `async/await` with `try/catch`:
   ```javascript
   fetch("/api/crops").then(r => r.json()).then(d => console.log(d)).catch(e => console.error(e));
   ```
4. Why does the following log `undefined`, and how do you fix it?
   ```javascript
   async function getCount() { return 42; }
   console.log(getCount());
   ```
5. You need data from two independent endpoints. Show the parallel version with `Promise.all` and explain why it's faster than two sequential `await`s.
6. Explain the `fetch` + `response.ok` gotcha in one or two sentences: why isn't a 404 automatically an error?

---

## Part 12 — Error handling

You've seen `try/catch` in the async section; here's the full picture. It maps closely to Python's `try/except/finally` with a few JS specifics.

### 12.1 `try` / `catch` / `finally`

```javascript
try {
  const data = JSON.parse(userInput);   // may throw on invalid JSON
  process(data);
} catch (error) {
  // like Python's `except Exception as error:`
  console.error("Parsing failed:", error.message);
} finally {
  // always runs, error or not (cleanup) — like Python's finally
  console.log("done");
}
```

Differences from Python:
- One `catch` block, not multiple typed `except` clauses. To handle different error types you branch *inside* the catch (see 12.3).
- The caught binding is optional in modern JS: `catch { ... }` with no variable is legal if you don't need the error.

### 12.2 Throwing errors

`throw` is Python's `raise`. You can technically throw any value, but **always throw an `Error` object** (or a subclass) — they carry a message and a stack trace.

```javascript
function setAcres(n) {
  if (typeof n !== "number" || Number.isNaN(n)) {
    throw new Error("acres must be a number");   // like raise ValueError(...)
  }
  if (n < 0) {
    throw new RangeError("acres cannot be negative");
  }
  return n;
}
```

Built-in error types include `Error` (generic), `TypeError`, `RangeError`, `SyntaxError`. `JSON.parse` throws `SyntaxError`; calling a non-function throws `TypeError`; etc.

### 12.3 Inspecting and branching on errors

Since there's one `catch`, distinguish error types with `instanceof` or by checking properties:

```javascript
try {
  risky();
} catch (error) {
  if (error instanceof RangeError) {
    console.log("out of range:", error.message);
  } else if (error instanceof TypeError) {
    console.log("type problem:", error.message);
  } else {
    throw error;    // re-throw what you can't handle (like a bare `raise`)
  }
}
```

An `Error` object has `.message` (the text), `.name` (e.g. `"RangeError"`), and `.stack` (the traceback string).

### 12.4 Custom error types

Subclass `Error` for domain-specific errors — useful in a data platform where you want to distinguish, say, a validation failure from a fetch failure:

```javascript
class RevisionError extends Error {
  constructor(message, revision) {
    super(message);
    this.name = "RevisionError";   // set the name for clean logs
    this.revision = revision;      // attach domain data
  }
}

try {
  throw new RevisionError("output revision exceeds orchestration", 5);
} catch (e) {
  if (e instanceof RevisionError) {
    console.log(e.name, e.message, e.revision);  // RevisionError ... 5
  }
}
```

### 12.5 Errors in async code

As covered in Part 11: inside `async` functions, `await`ed rejections are caught by ordinary `try/catch`. For `.then()` chains, use `.catch()`. A rejected promise with no handler produces an "unhandled promise rejection" warning — always handle them.

```javascript
async function safeLoad() {
  try {
    return await loadData();
  } catch (err) {
    return { error: err.message };   // convert a throw into a value the UI can render
  }
}
```

Converting thrown errors into returned values (an `{ error }` shape or a fallback) is a common frontend pattern, because a React component generally wants *data to render*, not an exception to crash the render.

### Practice — Part 12

1. Write `parseConfig(text)` that returns the parsed object, or `null` if `JSON.parse` throws, logging the error message in the catch.
2. Write `divide(a, b)` that throws a `RangeError` when `b === 0` and a `TypeError` when either argument isn't a number; otherwise returns `a / b`. Then call it in a `try/catch` that prints a different message per error type.
3. Create a custom `ValidationError` class that stores a `field` property. Throw one and catch it, printing the field.
4. Explain the difference between `throw error` (re-throwing) inside a catch and simply not catching that case at all.
5. Rewrite `loadData().then(d => use(d)).catch(e => handle(e))` as an `async` function using `try/catch`.

---

## Part 13 — Useful built-ins (Map, Set, Date, regex, misc)

A grab-bag of standard tools you'll reach for. Lighter treatment — enough to know they exist and recognize them.

### 13.1 `Map` — a real dictionary with any key type

Plain objects work as dicts but have quirks: keys are coerced to strings, and they carry inherited properties. `Map` is a purpose-built dictionary (like Python's `dict`) with any-type keys and a clean API.

```javascript
const m = new Map();
m.set("wheat", 100);
m.set("canola", 80);
m.get("wheat");        // 100
m.has("canola");       // true
m.delete("canola");
m.size;                // 1  (a property, not .length)

// Any key type (objects, numbers), unlike plain objects:
const key = { id: 1 };
m.set(key, "value");

// Iterate (insertion order guaranteed):
for (const [k, v] of m) { console.log(k, v); }

// From/to arrays:
const m2 = new Map([["a", 1], ["b", 2]]);
Object.fromEntries(m2);        // { a: 1, b: 2 }
```

Use a plain object `{}` for simple string-keyed data (and JSON). Use `Map` when keys aren't strings, when you insert/delete frequently, or when you need reliable size/iteration. In React, plain objects and arrays dominate; `Map` shows up in logic-heavy code.

### 13.2 `Set` — unique values

Like Python's `set`:

```javascript
const s = new Set([1, 2, 2, 3, 3, 3]);
s.size;             // 3   (duplicates removed)
s.has(2);           // true
s.add(4);
s.delete(1);
[...s];             // [2, 3, 4]   (spread back to array)

// Deduplicate an array — the classic idiom:
const unique = [...new Set([1, 1, 2, 3, 3])];   // [1, 2, 3]
```

### 13.3 `Date` — timestamps (clunky, but standard)

```javascript
const now = new Date();               // current date/time
const d = new Date("2026-07-17");     // parse ISO string
now.getFullYear();                    // 2026
now.getMonth();                       // 0-based! July is 6, not 7 (classic trap)
now.getDate();                        // day of month
now.toISOString();                    // "2026-07-17T00:00:00.000Z"
Date.now();                           // milliseconds since epoch (a number)
```

The native `Date` API is famously awkward (0-based months, mutable, timezone pitfalls). For anything nontrivial, real projects use a library like `date-fns` or `Day.js`. Know `Date` exists; reach for a library when dates get serious.

### 13.4 Regular expressions

JS has regex literals with `/pattern/flags` syntax:

```javascript
const re = /\d+/g;                     // one or more digits, global flag
"a1b22c333".match(/\d+/g);            // ["1", "22", "333"]
"wheat-canola".replace(/-/g, " ");     // "wheat canola"
/^\d{4}-\d{2}$/.test("2026-07");       // true  (validate a YYYY-MM string)
"a,b;c".split(/[,;]/);                 // ["a", "b", "c"]
```

Flags: `g` (global/all matches), `i` (case-insensitive), `m` (multiline). The pattern language is nearly identical to Python's `re`. `.test()` returns a boolean; `.match()` returns matches; `.replace()` substitutes.

### 13.5 Numbers and formatting for display

```javascript
(1234.5).toFixed(2);                   // "1234.50"  (string, fixed decimals)
(1234567).toLocaleString("en-US");     // "1,234,567"  (thousands separators)

// Currency formatting — great for a finance UI:
new Intl.NumberFormat("en-CA", {
  style: "currency", currency: "CAD",
}).format(45000);                      // "$45,000.00"
```

`Intl.NumberFormat` is the proper way to format money and numbers for display — directly useful for GrowLytics' cash-flow figures.

### 13.6 Optional chaining + nullish, revisited in data code

You'll lean on these constantly when handling API data of uncertain shape:

```javascript
const revenue = response?.data?.records?.[0]?.revenue ?? 0;
// dig safely; default to 0 if any link is null/undefined
```

### Practice — Part 13

1. Deduplicate `[3, 1, 3, 2, 1, 2]` into `[3, 1, 2]` using a `Set` and spread.
2. Build a `Map` from crop → acres for wheat(100), canola(80), then iterate and print each entry. Convert it to a plain object.
3. Given a date string `"2026-07-17"`, create a `Date` and log its year, month (account for 0-based!), and day.
4. Write a regex test that validates a `"YYYY-MM"` month string (e.g. `"2026-07"` → true, `"2026-7"` → false).
5. Format `45000` and `1234567.5` as CAD currency strings using `Intl.NumberFormat`.
6. Safely read `config.display.decimals` defaulting to `2`, when `config` might be missing `display` entirely.

---

## Part 14 — The DOM, briefly

The DOM (Document Object Model) is the browser's live, tree-shaped representation of a web page, and the API for reading/changing it from JS. You've already touched this — you debugged a vanilla-JS frontend and learned the hard way that renaming an HTML `id` without updating every `el("...")` reference produces a null-reference crash that silently kills downstream rendering. That experience *is* the core DOM lesson. Here's the framing.

**Why this matters even though React hides it:** React exists precisely so you *don't* manually manipulate the DOM. Instead you describe what the UI should look like for a given state, and React efficiently updates the real DOM for you. But understanding the DOM explains what React is doing under the hood and why its rules exist.

### 14.1 The core API (vanilla JS)

```javascript
// Selecting elements (returns the element, or null if not found):
const el = document.getElementById("chart");
const btn = document.querySelector(".submit-btn");     // first CSS match
const items = document.querySelectorAll("li");         // all matches (array-like)

// Reading/changing content and attributes:
el.textContent = "Net cash flow: $45,000";   // set text (safe)
el.innerHTML = "<b>bold</b>";                 // set HTML (careful: injection risk)
el.classList.add("active");                   // toggle CSS classes
el.setAttribute("data-crop", "wheat");
el.style.color = "green";

// Creating and inserting:
const li = document.createElement("li");
li.textContent = "wheat";
document.querySelector("ul").appendChild(li);

// Events (the interactivity core):
btn.addEventListener("click", (event) => {
  console.log("clicked", event.target);
});
```

### 14.2 Why your `el("...")` bug happened

`getElementById("chart")` returns `null` when no element has that id. Reading a property off `null` (`el.textContent = ...` → `null.textContent`) throws `TypeError: Cannot set properties of null`. Because JS stops executing the current function at an uncaught throw, *every line after that never runs* — so all "downstream rendering" silently dies. Renaming the HTML `id` but not the JS selector recreates exactly this: the selector now finds nothing, returns `null`, and the first property access explodes. The DOM contract you identified — *ids in HTML and selectors in JS must stay in lockstep* — is precisely the invariant. Optional chaining (`el?.textContent`) prevents the crash but not the underlying "element missing" bug; the real fix is keeping the references synchronized.

**The React payoff:** this whole class of "manually find element by id, hope it exists, mutate it, keep everything in sync by hand" problem is what React eliminates. You'll declare the UI as a function of state, and React reconciles the DOM. You won't call `getElementById` in normal React code. So you don't need DOM mastery — just this understanding of what React is replacing and why.

### Practice — Part 14

1. In one sentence, explain why renaming an HTML element's `id` without updating the JS `getElementById` call crashes *all* subsequent rendering in that function.
2. Rewrite `const el = document.getElementById("total"); el.textContent = fmt(sum);` defensively so a missing element doesn't throw (but note this hides, not fixes, the real bug).
3. Conceptually: what does React let you stop doing, relative to the manual `querySelector` + `textContent` + `addEventListener` style?

---

## Part 15 — Python → JavaScript translation cheat sheet

A dense reference you can scan when you know what you want in Python and need the JS. Keep this near you for the first few weeks.

### Syntax and basics

| Python | JavaScript |
|---|---|
| `x = 5` | `const x = 5;` / `let x = 5;` |
| `# comment` | `// comment` or `/* block */` |
| `print(x)` | `console.log(x);` |
| `None` | `null` (intentional) / `undefined` (unset) |
| `True` / `False` | `true` / `false` |
| `and` / `or` / `not` | `&&` / `\|\|` / `!` |
| `x if c else y` | `c ? x : y` |
| `x == y` (value) | `x === y` (strict — use this) |
| `x is y` (identity) | `x === y` for objects |
| `len(x)` | `x.length` (arrays/strings) / `x.size` (Map/Set) |
| `type(x)` | `typeof x` / `Array.isArray(x)` |
| `int(s)` / `float(s)` | `Number(s)` / `parseInt(s)` / `parseFloat(s)` |
| `str(x)` | `String(x)` / `` `${x}` `` |
| `f"{a} and {b}"` | `` `${a} and ${b}` `` |
| `range(n)` | `for (let i=0; i<n; i++)` / `[...Array(n).keys()]` |
| `x ** 2` | `x ** 2` |
| `7 // 2` | `Math.floor(7 / 2)` |
| `raise ValueError("m")` | `throw new Error("m");` |
| `try/except/finally` | `try/catch/finally` |
| `with ... as` | *(no direct equivalent)* |

### Collections

| Python | JavaScript |
|---|---|
| `[1, 2, 3]` (list) | `[1, 2, 3]` (array) |
| `{"a": 1}` (dict) | `{ a: 1 }` (object) or `new Map()` |
| `{1, 2}` (set) | `new Set([1, 2])` |
| `lst.append(x)` | `arr.push(x)` |
| `lst.pop()` | `arr.pop()` |
| `x in lst` | `arr.includes(x)` |
| `key in dict` | `key in obj` / `obj.hasOwnProperty(key)` |
| `dict.get(k, default)` | `obj[k] ?? default` |
| `dict.keys()/values()/items()` | `Object.keys/values/entries(obj)` |
| `del dict[k]` | `delete obj[k];` |
| `lst[1:3]` | `arr.slice(1, 3)` |
| `lst[-1]` | `arr.at(-1)` |
| `lst[::-1]` | `[...arr].reverse()` |
| `sorted(lst)` | `[...arr].sort((a,b)=>a-b)` (numbers!) |
| `"-".join(lst)` | `arr.join("-")` |
| `s.split(",")` | `s.split(",")` |
| `a, b = b, a` | `[a, b] = [b, a];` |
| `a, *rest = lst` | `const [a, ...rest] = arr;` |

### Comprehensions → array methods

| Python | JavaScript |
|---|---|
| `[f(x) for x in xs]` | `xs.map(x => f(x))` |
| `[x for x in xs if c(x)]` | `xs.filter(x => c(x))` |
| `[f(x) for x in xs if c(x)]` | `xs.filter(x => c(x)).map(x => f(x))` |
| `sum(xs)` | `xs.reduce((a, x) => a + x, 0)` |
| `any(c(x) for x in xs)` | `xs.some(x => c(x))` |
| `all(c(x) for x in xs)` | `xs.every(x => c(x))` |
| `next(x for x in xs if c(x))` | `xs.find(x => c(x))` |
| `enumerate(xs)` | `xs.entries()` / `xs.map((x, i) => ...)` |
| `{k: v for ...}` | `Object.fromEntries(pairs.map(...))` |
| `functools.reduce(f, xs, init)` | `xs.reduce(f, init)` |

### Functions

| Python | JavaScript |
|---|---|
| `def f(a, b): return a+b` | `function f(a, b) { return a + b; }` |
| `lambda x: x*2` | `x => x * 2` |
| `def f(a, b=0):` | `function f(a, b = 0) {` |
| `def f(*args):` | `function f(...args) {` |
| `def f(**kwargs):` | `function f(opts) {}` + destructure |
| `f(*mylist)` | `f(...myArray)` |
| `self` | `this` (implicit; behaves differently) |
| `__init__` | `constructor` |
| `@staticmethod` | `static` |
| `super().__init__()` | `super()` |
| `class C(Base):` | `class C extends Base {` |

### Modules and async

| Python | JavaScript |
|---|---|
| `import module` | `import * as module from "./module.js"` |
| `from m import x` | `import { x } from "./m.js"` |
| `from m import x as y` | `import { x as y } from "./m.js"` |
| `import json` → `json.dumps/loads` | `JSON.stringify` / `JSON.parse` |
| `async def f():` | `async function f() {` |
| `await thing` | `await thing` |
| `asyncio.gather(a, b)` | `Promise.all([a, b])` |
| `requests.get(url).json()` | `(await fetch(url)).json()` |

---

## Part 16 — Practice solutions

Attempt the practice questions before reading these. Solutions are terse; the reasoning is in the relevant part.

### Part 1

```javascript
// 1
const farmName = "North Quarter";
let revenue = 50000;
revenue = 60000;              // fine
// farmName = "South";        // TypeError: Assignment to constant variable

// 2
7 / 2;                 // 3.5
Math.floor(7 / 2);     // 3
"7" / 2;               // 3.5   (/ coerces "7" to number)
"7" + 2;               // "72"  (+ with a string concatenates)
typeof NaN;            // "number"
typeof [];             // "object"
Array.isArray([]);     // true

// 3
"canola" + ": " + 240 + " acres";     // "canola: 240 acres"
`${crop}: ${acres} acres`;            // template literal version

// 4
if (items.length === 0) console.log("empty");

// 5
// undefined: JS auto-produces it for unset vars, missing props/args, no-return fns.
// null: you assign it to say "intentionally empty" (Python's None).
```

### Part 2

```javascript
// 1
1 == "1";        // true   (coerces)
1 === "1";       // false  (different types)
null == undefined;   // true  (special case)
null === undefined;  // false
0 == false;      // true   (coerces)
"" == false;     // true   (coerces)
NaN === NaN;     // false  (NaN never equals itself)

// 2  discount is 0 (a valid value), but 0 is falsy, so || replaces it with 0.15.
const applied = discount ?? 0.15;    // ?? keeps 0, only replaces null/undefined

// 3
const crops = data.farm?.crops?.length ?? 0;   // undefined-safe, defaults to 0

// 4
const msg = acres > 100 ? "large" : "small";

// 5  true  → "Welcome back"; false → false (short-circuits, renders nothing in JSX)
```

### Part 3

```javascript
// 1
for (const [i, crop] of crops.entries()) console.log(`${i + 1}. ${crop}`);

// 2  for...in iterates KEYS; array keys are indices ("0","1","2" as strings).
//    Use for...of for values.

// 3
let sum = 0;
for (let i = 1; i <= 100; i++) sum += i;   // 5050

// 4
switch (crop) {
  case "wheat":
  case "barley":  console.log("cereal");  break;
  case "canola":
  case "flax":    console.log("oilseed"); break;
  default:        console.log("other");
}

// 5
for (let i = 0; i < 5; i++) {}          // way one
[...Array(5).keys()];                   // way two → [0,1,2,3,4]
```

### Part 4

```javascript
// 1
function ncf1(i, o) { return i - o; }
const ncf2 = function (i, o) { return i - o; };
const ncf3 = (i, o) => i - o;

// 2
const makeCrop = name => ({ name, acres: 0 });

// 3
const average = (...nums) => nums.length ? nums.reduce((a, n) => a + n, 0) / nums.length : 0;

// 4
function makeAccumulator() {
  let total = 0;                 // captured by the closure
  return (n) => { total += n; return total; };
}
// acc(10) → 10, acc(5) → 15. The returned fn closes over `total`.

// 5  Declaration form prints "hi" even when called before definition (hoisted).
//    const arrow form throws ReferenceError (temporal dead zone).

// 6  Arrows have no own `this`; they inherit it from the surrounding scope,
//    so callbacks don't "lose" this.
```

### Part 5

```javascript
// 1
records.map(r => r.crop);
// 2
records.filter(r => r.active).map(r => r.crop);
// 3
records.reduce((sum, r) => sum + r.revenue, 0);
// 4
records.reduce((obj, r) => { obj[r.crop] = r.acres; return obj; }, {});
// 5
records.reduce((max, r) => r.revenue > max.revenue ? r : max);
records.find(r => r.acres < 50);
// 6
[...records].sort((a, b) => b.revenue - a.revenue);
// 7  Default sort stringifies: "10","100","9" sort lexicographically → 10,100,9.
[10, 9, 100].sort((a, b) => a - b);   // fix with numeric comparator
// 8
const crops = ["wheat", "canola"];
[...crops.filter(c => c !== "wheat"), "lentils"];   // ["canola","lentils"]
```

### Part 6

```javascript
// 1
const farm = { name: "North", acres: 320, crops: ["wheat"] };
farm.owner = "Zhe"; farm.acres = 400; delete farm.owner;
// 2
const record = { [field]: amount };          // { revenue: 45000 }
// 3
for (const [k, v] of Object.entries({ wheat: 100, canola: 80 }))
  console.log(`${k}: ${v} acres`);
// 4
Object.fromEntries(Object.entries({ a:1, b:2, c:3 }).map(([k, v]) => [k, v * 10]));
// 5
const copy = { ...original, crops: [...original.crops, "canola"] };
// Plain { ...original } shares the SAME crops array (shallow copy), so push mutates original.
// 6
JSON.parse(JSON.stringify({ crop: "wheat", acres: 100 }, null, 2));
```

### Part 7

```javascript
// 1
const [lat, lng, elevation] = point;
// 2  head = 1, tail = [2, 3, 4, 5]
// 3
const { name, acres: size, region = "SW Sask" } = farm;
// 4
function describe({ crop, acres = 0 } = {}) { return `${crop} on ${acres} acres`; }
// 5
function Card({ title, value = 0 }) { /* ... */ }
// 6
const { user: { name, roles: [firstRole] } } = data;   // name="Zhe", firstRole="admin"
```

### Part 8

```javascript
// 1
[...a, ...b];        // [1,2,3,4]
[...a];              // copy
// 2
Math.min(...[8, 3, 11, 5]);   // 3
// 3
({ ...farm, acres: 500, owner: "Zhe" });
// 4
function concatAll(...arrays) { return [].concat(...arrays); }  // or arrays.flat()
// 5  id = 7, rest = { crop: "wheat", acres: 100 }
// 6  spread, rest, spread, rest, spread
```

### Part 9

```javascript
// 1
class Crop {
  constructor(name, acres) { this.name = name; this.acres = acres; }
  describe() { return `${this.name} (${this.acres} ac)`; }
  get isLarge() { return this.acres > 50; }
}
const c = new Crop("wheat", 100);  // c.describe(), c.isLarge → true
// 2
static fromArray([name, acres]) { return new Crop(name, acres); }
// 3
class Cereal extends Crop {
  constructor(name, acres, yield_) { super(name, acres); this.yield = yield_; }
  describe() { return `${super.describe()}, yield: ${this.yield}`; }
}
// 4  TypeError: Class constructor Crop cannot be invoked without 'new'.
//    Classes must be instantiated with `new`.
// 5  .map lives on Array.prototype; JS walks arr's prototype chain to find it.
```

### Part 10

```javascript
// 1
export function sum(nums) { return nums.reduce((a, n) => a + n, 0); }
export function mean(nums) { return sum(nums) / nums.length; }
import { sum, mean } from "./mathUtils.js";
// 2
export default function sum(nums) { /* ... */ }
export function mean(nums) { /* ... */ }
import sum, { mean } from "./mathUtils.js";
// 3
import { useState, useEffect } from "react";   // named (braces + exact names)
// 4  default: x.js has `export default Thing` (import name is your choice).
//    named:   x.js has `export ... Thing` / `export { Thing }` (name must match).
// 5  CommonJS. ESM equivalent: import fs from "fs"; (or import * as fs from "fs";)
```

### Part 11

```javascript
// 1  A, C, B — synchronous logs first; setTimeout callback is queued for after.
// 2
const wait = ms => new Promise(res => setTimeout(res, ms));
wait(500).then(() => console.log("done"));
// 3
async function load() {
  try { const r = await fetch("/api/crops"); console.log(await r.json()); }
  catch (e) { console.error(e); }
}
// 4  async fns ALWAYS return a promise; getCount() is a Promise, not 42.
getCount().then(v => console.log(v));   // or: console.log(await getCount());
// 5
const [a, b] = await Promise.all([fetchA(), fetchB()]);
// Both requests start immediately and overlap, instead of the second waiting
// for the first to finish (sequential awaits).
// 6  fetch only rejects on network failure; HTTP 404/500 still "succeed" at the
//    network level, so you must check response.ok and throw yourself.
```

### Part 12

```javascript
// 1
function parseConfig(text) {
  try { return JSON.parse(text); }
  catch (e) { console.error(e.message); return null; }
}
// 2
function divide(a, b) {
  if (typeof a !== "number" || typeof b !== "number") throw new TypeError("numbers only");
  if (b === 0) throw new RangeError("divide by zero");
  return a / b;
}
try { divide(1, 0); }
catch (e) {
  if (e instanceof RangeError) console.log("range:", e.message);
  else if (e instanceof TypeError) console.log("type:", e.message);
}
// 3
class ValidationError extends Error {
  constructor(msg, field) { super(msg); this.name = "ValidationError"; this.field = field; }
}
try { throw new ValidationError("bad", "acres"); }
catch (e) { if (e instanceof ValidationError) console.log(e.field); }
// 4  `throw error` re-raises for a higher handler; not catching that case lets it
//    propagate immediately — similar outcome, but re-throwing documents intent and
//    lets you handle some cases while passing others up.
// 5
async function run() {
  try { use(await loadData()); } catch (e) { handle(e); }
}
```

### Part 13

```javascript
// 1
[...new Set([3, 1, 3, 2, 1, 2])];   // [3, 1, 2]
// 2
const m = new Map([["wheat", 100], ["canola", 80]]);
for (const [k, v] of m) console.log(k, v);
Object.fromEntries(m);              // { wheat: 100, canola: 80 }
// 3
const d = new Date("2026-07-17");
d.getFullYear();       // 2026
d.getMonth() + 1;      // 7  (getMonth is 0-based → +1)
d.getUTCDate();        // 17
// 4
/^\d{4}-\d{2}$/.test("2026-07");   // true;  "2026-7" → false
// 5
const cad = new Intl.NumberFormat("en-CA", { style: "currency", currency: "CAD" });
cad.format(45000);        // "$45,000.00"
cad.format(1234567.5);    // "$1,234,567.50"
// 6
const decimals = config?.display?.decimals ?? 2;
```

### Part 14

```javascript
// 1  getElementById returns null when the id is gone; the first property access on
//    null throws TypeError, which halts the function, so no later line runs.
// 2
const el = document.getElementById("total");
if (el) el.textContent = fmt(sum);      // or: el?.  — hides the missing-element bug
// 3  React lets you stop manually selecting elements, mutating textContent/attributes,
//    wiring addEventListener, and hand-syncing DOM with data — you declare UI from state.
```

---

## Where to go next

You now have the JavaScript foundation React assumes. When you move to React, these are the exact pieces that carry over, so notice them:

- **`.map()`** → rendering lists of components.
- **Destructuring** → reading props and `useState` returns.
- **Spread** → immutable state updates.
- **Arrow functions + closures** → event handlers and Hooks (and stale-closure bugs).
- **Ternary and `&&`** → conditional rendering in JSX.
- **`async/await` + `fetch`** → loading data in `useEffect`.
- **Modules** → every component file, wired together by Vite.

Suggested practice before React: rebuild a small slice of GrowLytics logic in plain JS — take an array of forecast records and, using only `map`/`filter`/`reduce`, compute monthly net cash flow and a per-crop breakdown. That single exercise touches arrays, objects, destructuring, and functions together, and it's the exact shape of the data transforms your React components will perform.


# React