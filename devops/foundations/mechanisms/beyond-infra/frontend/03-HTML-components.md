# How an HTML File Actually Works — A Guide Built Around Your Dashboard

You already think clearly about backends, data models, and queries. The thing that's
been biting you isn't difficulty — it's that the frontend has a few mechanics that are
*invisible* until you know they're there. Both bugs you hit (`monthDetail`, then
`sources`/`category`) were the **same mechanic** misfiring. Once that clicks, you'll
stop breaking things by accident.

This guide uses your own file (`serving_frontend_demo.html`) as the running example
throughout.

---

## Part 1 — The big picture: one file, three languages, one runtime

An `.html` file is just a text file. What makes it special is that a **browser** knows
how to read it and turn it into a running program with a visible interface. Think of the
browser as the runtime — the same role the Python interpreter plays for your FastAPI
code.

Inside that one text file there are **three different languages**, each doing a
different job:

| Language | Job | Analogy to your world |
|---|---|---|
| **HTML** | Structure / content — *what things are* | Your data schema / table definitions |
| **CSS** | Presentation — *what things look like* | Formatting rules applied to that data |
| **JavaScript (JS)** | Behavior — *what happens when* | Your `queries.py` + endpoint handlers, but running in the browser |

They live in the same file but in separate regions:

```
<!doctype html>
<html>
  <head>
    <style> ...all your CSS... </style>      ← presentation rules
  </head>
  <body>
    ...all your HTML...                        ← the structure
    <script> ...all your JS... </script>       ← the behavior
  </body>
</html>
```

That's the literal skeleton of your file. Everything else is detail hanging off those
four landmarks: `<!doctype>`, `<html>`, `<head>`, `<body>`.

- **`<!doctype html>`** — one line that says "this is modern HTML." Always there, never
  changes. Ignore it.
- **`<head>`** — *setup that isn't visible content*: the page title, character encoding,
  the viewport line for mobile, and your `<style>` block. Nothing in `<head>` draws on
  the screen directly.
- **`<body>`** — *the visible content*. Everything the user sees lives here.
- **`<script>`** — your program. You put it at the *bottom* of `<body>` on purpose
  (more on why in Part 6 — it matters a lot).

---

## Part 2 — HTML: the structure (the skeleton)

### Elements, tags, attributes

HTML is built from **elements**. An element is usually an opening tag, some content,
and a closing tag:

```html
<h2>Revision</h2>
```

- `<h2>` is the **opening tag** (a level-2 heading).
- `Revision` is the **content**.
- `</h2>` is the **closing tag** (the `/` means "close").

Tags can carry **attributes** — name/value pairs that configure the element:

```html
<div id="category" class="card"></div>
```

Here the `<div>` has two attributes: `id="category"` and `class="card"`. Attributes are
how you label an element, style it, or wire it to behavior. Two attributes matter most
for you, and they're explained in their own sections below: **`id`** and **`class`**.

A `<div>` deserves a special note: it's the generic "box" element. It means nothing on
its own — it's just a container you group things into and then style or target. Your
file is full of them because they're the neutral building block.

### Nesting makes a tree

Elements contain other elements, and that nesting forms a **tree** — exactly like a
nested JSON object or a folder structure. Here's a real slice of your file:

```html
<div class="card">                         <!-- a box -->
  <h2>Revision</h2>                         <!--   a heading inside it -->
  <label for="orch">Showing</label>         <!--   a label inside it -->
  <select id="orch"></select>               <!--   a dropdown inside it -->
</div>
```

`card` is the **parent**; `h2`, `label`, and `select` are its **children**. The whole
page is one big tree with `<html>` at the root. This tree idea is the single most
important thing in this guide, because the browser turns it into a live object you can
manipulate — that's the DOM, coming up in Part 3.

### The `id` attribute — a unique name (this is where your bugs live)

An `id` is a **unique name you give one element** so other parts of the file can find
it. In your file:

```html
<select id="orch"></select>
<div id="chart"><div class="placeholder">loading…</div></div>
<div id="category"></div>
<div id="lineage"></div>
<div id="log"></div>
```

Two rules:
1. An `id` should be **unique** in the page (only one `id="chart"`).
2. The id is a **contract**. Your JavaScript reaches for elements *by their id*. The
   moment the id in the HTML and the id in the JS disagree, the JS finds *nothing*.

That second rule is the entire story of both your bugs. Hold that thought — Part 3
makes it concrete.

### The `class` attribute — a reusable label

Where `id` names *one* element, `class` tags *many* elements as "the same kind of
thing." Lots of elements can share a class:

```html
<div class="card"> ... </div>
<div class="card"> ... </div>
<div class="barRow"> ... </div>
```

Classes are mainly for **styling** (CSS targets them) and sometimes for selecting groups
in JS. Your file styles every `.card`, `.barRow`, `.stat`, etc. through classes, and the
JS finds clickable bars with the class `.bar`. An element can have several classes at
once (`class="bar barTrack"` — it's both a "bar" and a "barTrack").

---

## Part 3 — The DOM: the bridge between HTML and JavaScript

This is the concept that was missing, and it explains everything.

When the browser loads your file, it doesn't just display the HTML text — it **parses
that text into a live tree of objects in memory**. That live tree is called the **DOM**
(Document Object Model). Each HTML element becomes a JavaScript object you can read and
change while the page is running.

The crucial mental model:

> The HTML text is the **blueprint**. The DOM is the **building** the browser constructs
> from it. Your JavaScript never touches the HTML text — it operates on the building.

So when your code does `el("chart").innerHTML = "..."`, it is not editing the file on
disk. It's reaching into the live building, finding the room labeled `chart`, and
swapping out its contents *right now*, in front of the user.

### How JS finds an element: `getElementById` and the id contract

The standard way to grab one element from the DOM is:

```js
document.getElementById("chart")
```

This searches the live tree for the element whose `id` is `"chart"` and hands you back
that object. Your file wraps this in a tiny helper so it's less to type:

```js
function el(id){ return document.getElementById(id); }
```

So everywhere you see `el("chart")`, read it as "find the live element named `chart`."

Now the critical detail:

> If no element has that id, `getElementById` returns **`null`** — JavaScript's "nothing
> here." It does **not** throw an error. It quietly hands you nothing.

`null` is harmless to *receive*. It becomes fatal the instant you try to **use** it as
if it were a real element:

```js
el("category").innerHTML = "...";   // if el("category") is null, this line THROWS
```

You can't set `.innerHTML` on "nothing." The browser throws:

```
TypeError: Cannot set properties of null (setting 'innerHTML')
```

And here's the part that made your bugs look mysterious: **when a line throws, the rest
of that function never runs.** Everything after the failing line is skipped. If that
function was the one that draws the chart or the bars, *nothing draws* — not because the
drawing code is wrong, but because the program died three lines earlier.

### Your two bugs, traced through this one rule

**Bug 1 — `monthDetail`.** You commented the element out of the HTML:

```html
<!-- <div id="monthDetail">…</div> -->
```

But the JS still had `el("monthDetail").innerHTML = ...` inside `selectOrch`, which runs
during startup. So: `el("monthDetail")` → `null` → `null.innerHTML = ...` → **throw** →
`selectOrch` dies before it ever calls `renderChart` → the chart never appears. The
chart code was fine the whole time.

**Bug 2 — `sources` → `category`.** You renamed the container's id from `sources` to
`category` in the HTML and updated the *loading* line, but `renderSources` still wrote to
`el("sources")` in three places. First line it reached: `el("sources").innerHTML` →
`null` → **throw** → no bars. Again, the renderer logic was fine — it was pointed at a
room that no longer existed.

Same mechanic both times: **the id in the HTML and the id in the JS disagreed, so the JS
got `null`, and using `null` killed the function.** Internalize this and you've removed
the most common way to "break tons of things" with a small edit.

### Other ways JS reaches into the DOM (you use these too)

- **`.innerHTML`** — read or replace *everything inside* an element, as an HTML string.
  Your renderers build a big string of HTML and assign it: `box.innerHTML = "<div>…</div>"`.
  Assigning to `innerHTML` wipes the old contents and parses your new string into real
  elements. (Caveat for later: it also destroys any click handlers attached to the old
  contents — that's why your code re-attaches listeners after every render. See Part 5.)
- **`.style.display = "none" / "block"`** — show or hide an element. Your `show()` and
  `hide()` helpers do exactly this. Hiding doesn't delete the element; it just stops it
  from drawing.
- **`querySelectorAll(".bar")`** — find *many* elements by a CSS selector (here, every
  element with class `bar`). Returns a list you can loop over.
- **`.dataset.cat`** — read a `data-cat="..."` attribute off an element. This is how a
  clicked bar tells the handler *which* category it represents. You write `data-cat` in
  the HTML string; you read it back as `.dataset.cat` in JS.

---

## Part 4 — CSS: the presentation layer

CSS answers "what does it look like?" It never changes *what* an element is or *what it
contains* — only its appearance. Your entire CSS lives in the `<style>` block in the
`<head>`.

### A CSS rule has two parts

```css
.card {
  background: #ffffff;
  border: 1px solid #e3e5ea;
  border-radius: 12px;
  padding: 16px 18px;
}
```

- **`.card`** is the **selector** — *which* elements this applies to. The leading dot
  means "any element with `class="card"`." (A `#` would mean "the element with that
  `id`"; a bare word like `body` means "every `<body>` element.")
- The block in `{ }` is a list of **declarations** — `property: value;` pairs that set
  the look.

So this rule says: "every element classed `card` gets a white background, a thin grey
border, rounded corners, and internal spacing." That's why all your panels look
identical — they share one class and therefore one rule.

### Selectors connect CSS to your HTML

This is the CSS analogue of the id contract: CSS targets elements by their `class`, `id`,
or tag name. If you rename a class in the HTML but not in the CSS (or vice versa), the
styling silently stops applying — no error, the thing just looks wrong. Less dangerous
than the JS version (it won't crash), but the same "two places must agree" idea.

### CSS variables — your `:root` block

At the top of your styles you have:

```css
:root{
  --ink:#1b2330; --bg:#f4f5f8; --line:#e3e5ea;
  --in:#1f9e6e; --out:#c2502f; --net:#2a3550;
  /* ...more... */
}
```

`:root` means "the whole document," and the `--name: value` entries are **custom
properties** (CSS variables) — named values you define once and reuse everywhere via
`var(--name)`:

```css
body { background: var(--bg); color: var(--ink); }
```

This is just like defining constants at the top of a module. Change `--in` once and every
green element in the page updates. Your JS even uses them: when a bar is positive it sets
`background: var(--in)` (green), negative gets `var(--out)` (red). One source of truth
for the palette.

### The "cascade" in one paragraph

CSS stands for *Cascading* Style Sheets. When several rules target the same element, the
browser resolves conflicts by **specificity** (an id selector beats a class selector
beats a tag selector) and, for ties, **source order** (later rules win). You rarely need
to think about this until two rules fight; when something looks styled "wrong," a
competing rule is usually the reason. The browser's DevTools (Part 8) show you exactly
which rule won.

### Inline styles

You'll also see styles written directly on an element:

```html
<div id="sourcesBlock" style="display:none;margin-top:14px">
```

That's an **inline style** via the `style` attribute. It's the highest-priority way to
style one element, and it's also what JS manipulates when it does
`el("sourcesBlock").style.display = "block"`. That's the connection between your `show()`
helper and what the user sees: the helper flips this inline `display` value.

---

## Part 5 — JavaScript: the behavior layer

JS is the part that *does things over time*: fetches data, responds to clicks, rebuilds
the screen. It lives in your `<script>` block. Here are the constructs your file leans
on, each tied to where you use it.

### Variables and your data structures

```js
const DB = { orchestrations: [...], monthly: {...}, sources: {...}, lineage: {...} };
const state = { orch:null, monthly:[], month:null, source:null };
```

`const` declares a value that won't be reassigned. `DB` is your fake database — the exact
shapes your real endpoints return. `state` is the small bag of "what's currently
selected" that *only the frontend knows about* (the server has no idea which month the
user clicked). Objects (`{ }`) and arrays (`[ ]`) work just like JSON / Python dicts and
lists.

### Functions

Three families in your file, and the separation is deliberate and worth copying in future
work:

- **Handlers** (`selectOrch`, `selectMonth`, `selectSource`) — "user did X." They update
  `state`, then fetch what's newly needed, then call a renderer.
- **Renderers** (`renderChart`, `renderSources`, `renderLineage`, `renderMonthDetail`) —
  "turn one array into one view." They take data and write HTML into a DOM element.
- **Helpers** (`el`, `show`, `hide`, `money`, `short`, `log`) — tiny reusable utilities.

Keeping "respond to the user," "draw the data," and "little utilities" in separate
functions is exactly why the file is fixable at all.

### Template literals — how you build HTML in JS

The backtick strings are **template literals**. They let you write a multi-line string
with `${...}` slots that get filled in:

```js
box.innerHTML = `
  <div class="barRow">
    <div class="name">${c.category}</div>
    <div class="amt mono">${money(c.amount)}</div>
  </div>`;
```

Everything between backticks is literal text *except* the `${ }` holes, where JS
evaluates an expression and drops the result in. So `${c.category}` becomes the actual
category name, and `${money(c.amount)}` calls your `money()` helper and inserts e.g.
`-$30,000`. This is how a renderer converts data into the HTML string it then assigns to
`.innerHTML`. The `.map(...).join("")` pattern you see runs that template once per array
element and glues the pieces into one big string — *one array, one view*.

### Events and listeners — how clicks become code

An **event** is something that happens: a click, a keypress, a dropdown change. You react
by registering a **listener**:

```js
sel.addEventListener("change", () => selectOrch(sel.value));
```

Read it as: "on the `orch` dropdown, when its value **changes**, run `selectOrch` with
the new value." Similarly each chart dot and each bar gets a `"click"` listener wired to
the right handler.

Two things to know:
1. **Listeners attach to specific live elements.** If you replace an element's
   `.innerHTML`, the old inner elements (and their listeners) are gone, and the new ones
   have none yet. That's why every renderer **re-queries and re-attaches** listeners
   right after writing `innerHTML`:

   ```js
   box.innerHTML = cats.map(...).join("");          // new bars exist now
   box.querySelectorAll(".bar").forEach(b =>         // find them...
     b.addEventListener("click", () => selectSource(b.dataset.cat)));  // ...and wire them
   ```

   Forget that step and your bars would render but do nothing on click. Not a crash — a
   "why is nothing happening" bug.
2. **The element carries its own data.** Each bar stamps `data-cat="${c.category}"` into
   its HTML, and the click handler reads `b.dataset.cat`. That's how a generic "a bar was
   clicked" turns into "the *Input Cost* bar was clicked" without any global bookkeeping.

### `async` / `await` — why there are delays and how data arrives

Fetching data takes time, and the browser must stay responsive meanwhile. JS handles this
with **promises** and the `async`/`await` keywords. In your file:

```js
async function getMonthly(orch){
  await wait();                       // pretend the network took ~320ms
  const rows = DB.monthly[orch] ?? [];
  return rows;
}
```

- `async` marks a function that does something time-consuming and hands back a
  **promise** (an IOU for a future value).
- `await` means "pause here until that IOU resolves, then continue with the value." Your
  handlers `await api.getMonthly(...)` so the next line only runs once the data is
  actually in hand.
- `wait()` is a fake delay so you can *see* the loading state. In the real app you'd
  replace the body with `return fetch(...).then(r => r.json())` and **nothing else
  changes** — the handlers already `await` it correctly.

The `?? []` is the **nullish-coalescing** operator: "if the left side is `null`/undefined,
use `[]` instead." It's why a missing key returns an empty array (→ "No data") instead of
crashing. A small but important safety net.

---

## Part 6 — How your file runs, start to finish

Now the payoff: the whole lifecycle, in order.

### Step 0 — The browser reads top to bottom

The browser parses the file from the top. It builds the DOM as it goes. By the time it
reaches your `<script>` at the **bottom of `<body>`**, every element above it already
exists in the DOM. **This is why script placement matters:** if your script ran in the
`<head>` (before the body was parsed), `el("chart")` would return `null` because the
chart `<div>` wouldn't exist yet — and you'd get the exact null-reference crash from Part
3, but for an entirely different reason. Bottom-of-body is the simple way to guarantee
"the DOM is ready when my code runs."

### Step 1 — `init()` kicks everything off

The last line of your script is `init();`. That single call starts the app:

```js
async function init(){
  const orchs = await api.getOrchestrations();   // 1 call → fill the dropdown
  const sel = el("orch");
  sel.innerHTML = orchs.map(o => `<option value="${o.orchestration_key}">…</option>`).join("");
  sel.addEventListener("change", () => selectOrch(sel.value));  // wire the dropdown
  await selectOrch(orchs[0].orchestration_key);  // load the first revision's chart
}
```

It fetches the list of revisions, builds the dropdown's `<option>`s, wires the dropdown's
change event, then calls `selectOrch` for the first one — which loads the chart. (Bug 1
lived right here: `selectOrch` threw before the chart could draw.)

### Step 2 — A handler changes state, then fetches, then renders

Take a click on a chart dot. Here's the full chain:

```
user clicks a dot
   └─ the dot's "click" listener fires  →  selectMonth(point)
        ├─ state.month = point.month            (record the selection)
        ├─ renderMonthDetail(point)             (inflow/outflow/net — NO call, data was in the point)
        ├─ show("sourcesBlock")                 (reveal the breakdown panel)
        ├─ el("category").innerHTML = "loading…"(loading state)
        ├─ rows = await api.getSources(orch, month)   (ONE call → SourceAmount[])
        └─ renderSources(rows)                  (roll up by category, draw the bars)
```

Then clicking a category bar runs `selectSource(category)` → one `getSources`-style call
to `getLineage` → `renderLineage` draws the table. Each user action is the same rhythm:
**change state → fetch only what's new → render one array into one view.**

### Step 3 — The state model

`state` is the answer to "what does the frontend remember that the server doesn't?" The
selected revision, the loaded monthly array, the clicked month, the clicked source. When
you change the dropdown, `selectOrch` resets `month` and `source` to `null` (because a
new revision invalidates the old selections) and re-fetches. Understanding `state` is
understanding the app: every render is just `state` (plus freshly-fetched data) drawn to
the screen.

---

## Part 7 — The one pattern every renderer follows

All four renderers are the same shape. Learn it once:

```js
function renderX(data){
  const box = el("someId");                         // 1. grab the target element
  if(!data.length){ box.innerHTML = "empty state"; return; }  // 2. handle no-data
  // 3. (optional) transform data — e.g. roll sources up by category
  box.innerHTML = data.map(item => `…HTML using ${item.field}…`).join("");  // 4. build & inject
  box.querySelectorAll(".clickable").forEach(/* attach listeners */);       // 5. re-wire events
  log("render", `drew ${data.length} things`);      // 6. record what happened
}
```

When you want to change *how something looks or what fields it shows*, you're editing
step 4 (the template). When you want to change *what's clickable*, step 5. When the
**target element's id changes, you must update step 1** — that's the line that bit you
twice.

---

## Part 8 — How to make edits without breaking everything

A practical checklist drawn from your actual failure modes.

### The golden rule: ids are a two-sided contract

If you rename or remove an element's `id` in the HTML, you must update **every** place the
JS refers to it. Before changing an id, search the whole file for the old name. With your
`el()` helper, the JS references look like `el("oldname")`; the HTML looks like
`id="oldname"`. A plain text search for `oldname` finds both sides. Your `sources` bug
was three un-updated `el("sources")` calls — a search would have shown all three
instantly.

The same applies to **classes** (HTML `class="x"` must match CSS `.x` and any
`querySelectorAll(".x")`) and to **`data-` attributes** (`data-cat` in the template must
match `.dataset.cat` in the handler).

### Open the browser console — it already told you what was wrong

Both your bugs printed a precise message you could have read in seconds. In any browser,
right-click the page → **Inspect** → **Console** tab. A red line like:

```
Uncaught TypeError: Cannot set properties of null (setting 'innerHTML')
    at renderSources (serving_frontend_demo.html:335)
```

tells you the *exact file, function, and line number*. "Cannot set properties of null"
almost always means **you asked for an element by id and got `null`** — i.e. the id
contract is broken. Train yourself to open the console the moment something "doesn't
render." It turns mysteries into addresses.

### The Elements panel shows you the live DOM

Same Inspect window, **Elements** tab: this is the *live DOM tree* (the building), not the
file text. You can see whether `id="category"` actually exists, hover elements to
highlight them on the page, and watch attributes change as you interact. When JS and HTML
disagree, this is where you confirm which one reality matches.

### A pre-flight checklist for any edit

Before you save and reload, ask:

1. **Did I rename or remove an `id`?** → Search the file for the old id; fix every JS
   reference (`el("…")`) and any CSS `#…` selector.
2. **Did I rename a `class`?** → Search for it; fix CSS `.…` rules and any
   `querySelectorAll(".…")`.
3. **Did I change a `data-` attribute name?** → Make the `dataset.…` read match.
4. **Did I change the *shape* of the data** (renamed a field, regrouped rows)? → Find
   every template `${item.field}` and every lookup key that used the old shape. (Your
   lineage re-keying from `source` to `category` was exactly this — the data changed, so
   the click that built the lookup key had to change too.)
5. **Did I add code that uses an element during startup?** → Make sure that element
   exists in the HTML *above* the script, or guard it.

### Guarding: the defensive version

When an element *might* not exist (like `monthDetail`, which you sometimes comment out),
guard before using it instead of assuming it's there:

```js
const md = el("monthDetail");
if (md) md.innerHTML = "…";   // only touch it if it actually exists
```

That's the pattern already in your fixed file. It converts a hard crash into a harmless
no-op. Don't sprinkle it everywhere — for elements that *should* always exist, you *want*
the loud crash so you notice. Use guards for the genuinely optional ones.

---

## Part 9 — Quick reference glossary

- **HTML** — the markup language describing structure/content via nested tags.
- **Element** — one node: `<div>…</div>`. Has tags, content, and attributes.
- **Attribute** — config on an element: `id`, `class`, `style`, `data-*`, etc.
- **`id`** — unique name for one element; the anchor JS uses to find it.
- **`class`** — shared label across many elements; mainly for CSS targeting.
- **DOM** — the live, in-memory tree of objects the browser builds from your HTML; what JS
  actually manipulates.
- **`getElementById` / your `el()`** — fetch one DOM element by id; returns `null` if not
  found.
- **`null`** — "nothing." Safe to receive, fatal to use as if it were an element.
- **`.innerHTML`** — get/replace everything inside an element, as an HTML string.
- **CSS** — styling rules; `selector { property: value; }`.
- **Selector** — which elements a CSS rule targets (`.class`, `#id`, `tag`).
- **CSS variable** — `--name` defined in `:root`, reused via `var(--name)`.
- **Event / listener** — something that happens / the function you register to react.
- **`data-*` / `.dataset`** — stash a value on an element in HTML, read it back in JS.
- **Template literal** — backtick string with `${...}` slots for building HTML.
- **`async` / `await` / promise** — how JS waits for slow work (like fetching) without
  freezing.
- **`??`** — "use the right side if the left is null/undefined."
- **State** — the frontend's memory of current selections; the server doesn't know it.

---

## The one sentence to remember

**The HTML text is a blueprint; the browser builds a live DOM from it; your JavaScript
operates on that live DOM by finding elements through ids and classes — so any time the
names in your HTML and the names in your JS drift apart, the JS gets `null`, and using
`null` stops the function cold.** Every bug you've hit is a variation of that one
sentence, and now you can see it coming.
