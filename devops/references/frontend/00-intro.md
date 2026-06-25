# Frontend syntax crash course

A from-scratch syntax reference for `serving_frontend_demo.html`. No prior frontend
knowledge assumed. Every rule below is tied to a real line from your file, so you can
read this with the code open beside it.

How to use this: skim section 1 once, then keep section 4 (JavaScript) open while you
read the code. Section 6 decodes a few of the densest real lines character by character —
that's the payoff once the pieces click.

---

## 1. One file, three languages

A web page is built from three languages that do three different jobs. Your file packs
all three into one `.html` file:

| Language   | Job                          | Where it lives in your file        |
|------------|------------------------------|------------------------------------|
| HTML       | structure (what exists)      | between `<body>` and `</body>`      |
| CSS        | appearance (how it looks)    | inside the `<style>` block          |
| JavaScript | behaviour (what it does)     | inside the `<script>` block         |

The browser reads the file top to bottom. It builds the HTML structure, applies the CSS
styling, then runs the JavaScript, which can reach back in and change everything.

The "tags" that switch languages:

```html
<style> ... CSS goes here ... </style>
<script> ... JavaScript goes here ... </script>
```

---

## 2. HTML syntax

### 2.1 Tags and elements

HTML is made of **tags** wrapped in angle brackets. Most come in pairs — an opening tag
and a closing tag (closing tags have a `/`). The opening tag, the closing tag, and
everything between them is called an **element**.

```html
<h2>Revision</h2>
```

`<h2>` opens, `</h2>` closes, `Revision` is the content. This whole thing is an "h2
element" (a level-2 heading).

Some tags are **self-closing** because they have no inner content — they end with `/>`
or just don't take a closing tag:

```html
<meta charset="utf-8">
<input>
```

### 2.2 Attributes

Tags can carry **attributes** — extra settings written as `name="value"` inside the
opening tag.

```html
<select id="orch"></select>
```

Here `id="orch"` is an attribute. `id` is the attribute name, `"orch"` is its value.
This element is empty (nothing between `<select>` and `</select>`) — JavaScript fills it
in later.

The two attributes you'll see constantly:

- `id="orch"` — a **unique** name for one element. JavaScript finds elements by id.
- `class="card"` — a **shared** label many elements can have. CSS styles by class.

```html
<div class="card">
  <h2>Revision</h2>
</div>
```

`<div>` is a generic box/container — the most common HTML tag. It groups other elements.
Notice the indentation: the `<h2>` sits *inside* the `<div>`. HTML elements nest inside
each other like boxes inside boxes.

### 2.3 The tags that appear in your file

| Tag         | What it is                                              |
|-------------|--------------------------------------------------------|
| `<div>`     | generic container box                                  |
| `<p>`       | a paragraph of text                                    |
| `<h1> <h2>` | headings (h1 biggest)                                  |
| `<span>`    | inline container (for a word or two inside a line)     |
| `<select>` + `<option>` | a dropdown menu and its choices            |
| `<label>`   | a text label attached to a form control                |
| `<button>`  | a clickable button                                     |
| `<table> <thead> <tbody> <tr> <th> <td>` | a data table and its rows/cells |
| `<svg> <circle> <line> <polyline> <text>` | vector graphics (your chart) |

`data-*` attributes (e.g. `data-i="0"`, `data-src="revenue"`) are custom attributes you
invent to stash a value on an element so JavaScript can read it back later when the
element is clicked. More on this in 5.5.

---

## 3. CSS syntax

CSS only describes appearance. It never *does* anything.

### 3.1 The rule shape

```css
select { padding: 6px 8px; border-radius: 8px; }
```

Read this as three parts:

- **selector** (`select`) — which elements this applies to (here: all `<select>` tags).
- **property** (`padding`) — what aspect to change.
- **value** (`6px 8px`) — what to set it to.

Each `property: value;` pair is a **declaration**, ends with a semicolon, and lives
inside `{ }`.

### 3.2 Selectors — choosing what to style

```css
.card   { ... }   /* a dot means: elements with class="card"          */
#log    { ... }   /* a hash means: the element with id="log"           */
th, td  { ... }   /* a comma means: apply to BOTH <th> and <td>        */
.tag.call { ... } /* no space, stacked: elements with BOTH classes     */
.row .t { ... }   /* a space means: a .t INSIDE a .row (descendant)    */
```

The `/* ... */` is a **CSS comment** — ignored by the browser, just a note.

### 3.3 Variables (custom properties)

The `:root { ... }` block at the top defines reusable values, like constants:

```css
:root {
  --ink: #1b2330;
  --accent: #3b4cca;
}
```

Anything starting with `--` is a variable. You use it elsewhere with `var(...)`:

```css
body { color: var(--ink); }
```

This is why colours stay consistent — change `--accent` once and everything using
`var(--accent)` updates. The `#3b4cca` values are **hex colour codes** (red/green/blue in
hexadecimal).

### 3.4 Units and values you'll see

- `16px` — pixels (absolute size).
- `1fr` — "one fraction" of available space (used in grid layouts).
- `60ch` — width of 60 characters.
- `100vh` — 100% of the viewport (screen) height.
- `#fff` — shorthand hex colour (white).

### 3.5 Inline styles

Sometimes a `style` attribute appears directly on an HTML element instead of in the
`<style>` block:

```html
<div id="sourcesBlock" style="display:none">
```

`display:none` hides the element completely. JavaScript flips this between `none` and
`block` to show/hide sections (see 5.4). Same CSS, just written in place.

---

## 4. JavaScript syntax — the main event

This is where the logic lives, and where you'll spend most of your effort. Everything
below appears in your file.

### 4.1 Statements, comments, semicolons

Code runs top to bottom, one statement per line (roughly). Comments are notes the
computer ignores:

```javascript
// this is a single-line comment
/* this is a
   multi-line comment */
```

Semicolons `;` end statements. JavaScript is forgiving about them but your file uses them.

### 4.2 Variables: `const` and `let`

A variable is a named box holding a value.

```javascript
const DB = { ... };      // const = a name that won't be reassigned
let clock = 0;           // let  = a name you intend to change later
```

Your file uses `const` for almost everything (the data, the functions, the API client)
and `let` only for `clock`, which counts up (`clock++` means "add 1 to clock"). Rule of
thumb: use `const` unless you know you'll reassign it.

### 4.3 The basic data types

```javascript
"2026-01"        // string  — text, in quotes (single or double both fine)
120000           // number  — no quotes, no commas
true  / false    // boolean — yes/no
null             // "deliberately empty"
undefined        // "no value was ever set"
```

### 4.4 Objects — `{ }`

An **object** is a bundle of named values. The names are called **keys** (or
"properties"), each paired with a value. This is the shape of every row your serving
layer returns:

```javascript
{ month: "2026-01", inflow: 120000, outflow: -30000, net: 90000 }
```

Four keys (`month`, `inflow`, `outflow`, `net`), each with a value. You read a value with
**dot notation**:

```javascript
point.month     // → "2026-01"
point.net       // → 90000
```

So when the code says `point.inflow`, it means "the inflow value out of that one month
object." Your `state` is also an object:

```javascript
const state = { orch: null, monthly: [], month: null, source: null };
```

It starts with empty/null values and gets filled in as the user clicks.

### 4.5 Arrays — `[ ]`

An **array** is an ordered list. Your serving layer returns arrays of objects — a list of
month-objects, a list of source-objects, etc.

```javascript
[
  { month: "2026-01", net: 90000 },
  { month: "2026-02", net: -25000 },
  { month: "2026-03", net: 265000 },
]
```

- `data.length` → how many items (here 3).
- `data[0]` → the first item (counting starts at **0**, so `data[0]` is item 1).
- `data[+c.dataset.i]` → the item at position `i` (the `+` turns text "0" into number 0).

Arrays of objects are *the* core data shape of this whole app. The diagram from earlier —
MonthlyPoint[], SourceAmount[], LineageRow[] — those `[]` mean "array of."

### 4.6 Template literals — backticks and `${ }`

Normal strings use `"quotes"`. **Template literals** use backticks `` ` `` and can embed
live values with `${ ... }`. This is how the renderers build HTML:

```javascript
`GET /forecast/${short(orch)}/monthly …`
```

Everything is literal text *except* `${short(orch)}`, which runs that code and drops the
result into the string. If `short(orch)` returns `"3-pre-harvest"`, the whole string
becomes `GET /forecast/3-pre-harvest/monthly …`. (If you've used Python f-strings, this is
the same idea: `${x}` here ≈ `{x}` there.)

Backtick strings can also span multiple lines, which is why the HTML templates in the
renderers can be written across many lines.

### 4.7 Functions

A **function** is a reusable block of code with a name. You **define** it once and
**call** it (run it) many times.

```javascript
function money(n) {                       // define: takes one input called n
  const s = n < 0 ? "-" : "";
  return s + "$" + Math.abs(n).toLocaleString();
}

money(90000)    // call it → returns "$90,000"
```

- `n` is a **parameter** — a placeholder for whatever value you pass in.
- `return` hands a value back to whoever called the function.
- A function that doesn't `return` just *does* something (like updating the page).

### 4.8 Arrow functions — `=>`

The same idea written shorter. These two are equivalent:

```javascript
function show(id) { el(id).style.display = "block"; }   // classic
const show = (id) => { el(id).style.display = "block"; }; // arrow
```

The `=>` ("fat arrow") separates the inputs from the body. You'll see arrows most often as
**throwaway functions passed to other functions** — especially for clicks and `.map()`:

```javascript
c.addEventListener("click", () => selectMonth(p));
```

Read `() => selectMonth(p)` as: "a little function that takes no arguments and, when run,
calls `selectMonth(p)`." Here it means "when this dot is clicked, run selectMonth on point
p." When an arrow has no `{ }` braces, it automatically returns whatever the single
expression evaluates to — handy for `.map()`.

### 4.9 Operators and shorthands

```javascript
=        assign:  state.month = point.month
===      equal?   data.length === 1         (strict equality — use this one)
!        not:     if (!data.length)         ("if length is zero/falsy")
< > <= >= comparisons
+        add numbers, OR glue strings together
&&  ||   and / or
```

Three compact ones worth memorising because they're everywhere in your file:

**Ternary `? :`** — a one-line if/else that produces a value:

```javascript
fill="${d.net >= 0 ? 'var(--in)' : 'var(--out)'}"
```

Reads as: "if `d.net >= 0` use the green variable, otherwise the red one." Form is
`condition ? valueIfTrue : valueIfFalse`.

**Nullish coalescing `??`** — "use the left side, but if it's null/undefined use the right":

```javascript
const rows = DB.monthly[orch] ?? [];
```

"Look up the months for this orchestration; if there are none, fall back to an empty
array." This is what prevents crashes on missing data.

**Increment `++`**:

```javascript
clock++;     // same as clock = clock + 1
```

### 4.10 Array methods: `.map()`, `.join()`, `.forEach()`

These are the workhorses that turn arrays into views.

**`.map()`** — transform every item in an array into something else, producing a *new*
array of the same length. The renderers use it to turn each data row into a string of HTML:

```javascript
rows.map(r => `<div class="name">${r.source}</div>`)
```

Read: "for each row `r` in `rows`, produce this HTML string." If `rows` has 3 items, you
get an array of 3 HTML strings. The `r` is just the name you chose for "the current item";
it could be anything.

`.map()` can also hand you the **index** (position) as a second argument:

```javascript
data.map((d, i) => `${xs[i]},${y(d.net)}`)
```

Here `d` is the current item and `i` is its position (0, 1, 2 …). This is how the chart
knows where to place each dot horizontally.

**`.join("")`** — collapse an array of strings into one string, with nothing between them:

```javascript
rows.map(r => `<div>...</div>`).join("")
```

`.map()` gives you `["<div>...</div>", "<div>...</div>"]`; `.join("")` glues them into one
big HTML string ready to drop into the page. **This map→join pair is the single most
repeated pattern in your file.**

**`.forEach()`** — run some code for each item, but *don't* build a new array. Used for
side effects like attaching click handlers:

```javascript
el("chart").querySelectorAll(".pt").forEach(c => {
  c.addEventListener("click", () => selectMonth(p));
});
```

"For each dot `c`, attach a click handler to it."

**`Math.min` / `Math.max` / `Math.abs`** — number helpers. `Math.abs(-30000)` → `30000`
(absolute value). `Math.max(...vals)` → the biggest number in the list (the `...` spreads
the array out into separate arguments).

### 4.11 `async` / `await` and Promises

Fetching data over the network takes time. JavaScript doesn't freeze the page while
waiting — it lets the wait happen "in the background." The two keywords:

```javascript
async function selectOrch(orch) {        // async = "this function does waiting inside"
  state.monthly = await api.getMonthly(orch);   // await = "pause here until it's ready"
  renderChart(state.monthly);            // this line waits for the line above to finish
}
```

- `async` marks a function as one that contains waiting.
- `await` pauses *that function* (not the whole browser) until the slow thing finishes,
  then continues with the result.

A **Promise** is the "I'll get back to you" object that `await` waits on. You mostly don't
write Promises directly — you `await` functions that return them. The one place your file
creates one is the fake delay:

```javascript
const wait = (ms = 320) => new Promise(r => setTimeout(r, ms));
```

That just means "a Promise that resolves after `ms` milliseconds" — the artificial pause
so you can see the loading state. In the real app, `await fetch(...)` replaces this, and
the wait becomes a real network round trip.

---

## 5. The DOM — how JavaScript touches the page

The **DOM** (Document Object Model) is the live, in-memory version of your HTML that
JavaScript can read and rewrite. `document` is the entry point. This section is the bridge
between "I have an array" and "something appears on screen."

### 5.1 Finding elements

```javascript
document.getElementById("chart")     // the one element with id="chart"
el("chart")                          // your file's shortcut for the same thing
```

Your file defines `function el(id){ return document.getElementById(id); }` just to type
less. So everywhere you see `el("sources")`, read "the element with id=sources."

```javascript
el("chart").querySelectorAll(".pt")  // ALL elements with class="pt" inside #chart
```

`querySelectorAll` uses CSS selectors (`.pt` = class pt) and returns a list you can
`.forEach()` over.

### 5.2 Changing content with `.innerHTML`

This is the big one. `.innerHTML` is the HTML *inside* an element. **Assigning to it
replaces that content entirely:**

```javascript
el("chart").innerHTML = `<div class="placeholder">loading…</div>`;
```

The chart container's contents become a loading message. Later:

```javascript
el("chart").innerHTML = `<svg> ... the whole chart ... </svg>`;
```

…and the loading message is replaced by the chart. This single mechanism — build an HTML
string from an array, assign it to `.innerHTML` — is how every view in your app draws
itself.

### 5.3 Reading values and data attributes

```javascript
sel.value             // the currently selected <option>'s value (which revision)
c.dataset.i           // reads the data-i="..." attribute off element c
b.dataset.src         // reads the data-src="..." attribute off element b
```

`dataset.src` corresponds to the HTML attribute `data-src`. You wrote the value onto the
element when you built it, and read it back when it's clicked — that's how a clicked bar
"remembers" which source it is.

### 5.4 Changing style / showing + hiding

```javascript
el(id).style.display = "block";   // show
el(id).style.display = "none";    // hide
```

Your `show(id)` and `hide(id)` helpers wrap these. Setting `display:none` removes an
element from view; `block` brings it back. That's how the sources and lineage cards stay
hidden until there's something to put in them.

### 5.5 Events — responding to the user

An **event** is something the user does (click, key press, changing a dropdown). You
**listen** for it with `addEventListener(eventName, functionToRun)`:

```javascript
c.addEventListener("click", () => selectMonth(p));
sel.addEventListener("change", () => selectOrch(sel.value));
c.addEventListener("keydown", e => {
  if (e.key === "Enter" || e.key === " ") { e.preventDefault(); selectMonth(p); }
});
```

- First argument: the event name (`"click"`, `"change"`, `"keydown"`).
- Second argument: a function to run when it happens (usually an arrow function).
- That function can receive an **event object** (`e`) describing what happened —
  `e.key` is which key was pressed; `e.preventDefault()` stops the browser's default
  reaction.

This is the loop that drives everything: render creates elements → attach listeners →
user acts → a handler changes state and fetches → render again.

### 5.6 Building elements the other way

Most of your file builds HTML as strings. One helper builds an element step by step
instead, in `log()`:

```javascript
const row = document.createElement("div");   // make a fresh <div>
row.className = "row";                        // give it class="row"
row.innerHTML = `...`;                        // put content inside
box.appendChild(row);                         // attach it to the page
```

`createElement` makes a node, `appendChild` glues it onto an existing element. This is the
"add one row to the bottom of the log" pattern, as opposed to "replace everything" with
`.innerHTML =`.

---

## 6. Decoding real lines from your file

Now the syntax above, applied to actual dense lines. Read each piece left to right.

### Line: filling the dropdown

```javascript
sel.innerHTML = orchs.map(o => `<option value="${o.orchestration_key}">${o.client_name} — ${o.revision_name}</option>`).join("");
```

- `orchs` — the array of revision objects from the server.
- `.map(o => ...)` — for each one (call it `o`)…
- `` `<option value="${o.orchestration_key}">${o.client_name} — ${o.revision_name}</option>` `` —
  build an `<option>` HTML string, splicing in three values from `o`.
- `.join("")` — glue all the option strings into one.
- `sel.innerHTML = ...` — drop them inside the `<select>`, creating the dropdown choices.

One array → one block of `<option>` HTML → into the page. The same shape you'll see again
and again.

### Line: one clickable chart dot (inside a `.map`)

```javascript
data.map((d, i) => `
  <circle class="pt" data-i="${i}" cx="${xs[i]}" cy="${y(d.net)}" r="6"
     fill="${d.net >= 0 ? 'var(--in)' : 'var(--out)'}"></circle>`)
```

- `(d, i) =>` — current month object `d`, its position `i`.
- `data-i="${i}"` — stash the index on the circle so a click can find this month later.
- `cx="${xs[i]}"` — horizontal position, precomputed in the `xs` array.
- `cy="${y(d.net)}"` — vertical position, from the `y()` function that maps a dollar value
  to a pixel height.
- `fill="${d.net >= 0 ? 'var(--in)' : 'var(--out)'}"` — green if net is positive, red if
  negative (ternary).

### Line: the empty-state guard

```javascript
if (!data.length) { el("chart").innerHTML = `<div class="placeholder">No data…</div>`; return; }
```

- `data.length` — number of items.
- `!data.length` — true when length is `0` (`!` flips it; `0` counts as "falsy").
- So: "if the array is empty, show a 'no data' message and `return` early" — stop the
  function before it tries to draw a chart from nothing.

### Line: the data lookup with a fallback

```javascript
const rows = DB.sources[`${orch}|${month}`] ?? [];
```

- `` `${orch}|${month}` `` — build a lookup key like `"c0001_2026_3-pre-harvest|2026-01"`.
- `DB.sources[ ... ]` — look that key up in the sources object.
- `?? []` — if nothing's there, use an empty array so the next line doesn't crash.

In the real app, this whole line becomes `await fetch(...)` — but the *shape* it produces
(an array of rows) is identical, which is the entire point the demo is making.

---

## 7. One-page cheat sheet

| You see…             | It means…                                              |
|----------------------|--------------------------------------------------------|
| `<tag>...</tag>`     | an HTML element (structure)                            |
| `id="x"` / `class="x"` | unique name / shared label                          |
| `.x { p: v; }`       | CSS: style elements with class x                       |
| `var(--name)`        | use a CSS variable                                      |
| `const` / `let`      | declare a variable (won't change / will change)        |
| `{ key: value }`     | object (named fields) — read with `obj.key`            |
| `[ a, b, c ]`        | array (ordered list) — count with `.length`            |
| `` `text ${x}` ``    | template literal: text with live values spliced in     |
| `(a) => ...`         | arrow function (a small inline function)               |
| `cond ? a : b`       | ternary: if cond then a else b                         |
| `x ?? y`             | use x, or y if x is null/undefined                     |
| `arr.map(x => ...)`  | transform every item → new array                       |
| `arr.join("")`       | merge array of strings into one string                 |
| `arr.forEach(x =>)`  | do something per item (no new array)                   |
| `async` / `await`    | this function waits / pause until ready                |
| `el("x").innerHTML = ...` | replace what's inside element x                    |
| `addEventListener("click", fn)` | run fn when clicked                         |
| `e.dataset.src`      | read a `data-src` attribute off an element             |

---

### Suggested learning order

1. Objects and arrays (4.4–4.5) — your data is nothing but these.
2. Template literals and `.map().join()` (4.6, 4.10) — how data becomes HTML.
3. `.innerHTML` and events (5.2, 5.5) — how HTML reaches the page and reacts.
4. `async`/`await` (4.11) — how data arrives.
5. Re-read the demo's region 4 (handlers) and 5 (renderers) — they'll now read plainly.

Once these feel comfortable, the natural next step is React, which automates the
map→join→innerHTML→addEventListener cycle you now understand by hand.