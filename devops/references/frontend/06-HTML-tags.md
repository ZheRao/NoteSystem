# HTML Elements: A Structural Reference

*Written for someone who thinks in schemas, trees, and invariants.*

---

## Part 0 — The one mental model

**HTML is a serialization format for a tree.**

That single sentence resolves most confusion. You already know serialization formats: JSON serializes objects, Avro serializes records, Parquet serializes columnar tables. HTML serializes a *document tree*. The angle brackets are delimiters in the wire format, nothing more.

The pipeline the browser runs is one you'd recognize:

```
bytes → tokenizer → tree construction → DOM → (+ CSSOM) → render tree → layout → paint
```

The **DOM** is the actual data structure. The HTML text is only its on-disk representation. When you write HTML you are not "writing a page" — you are *declaring a tree*, and the browser materializes it.

Second reframe, equally important:

| Layer | Language | Question it answers |
|---|---|---|
| Structure | HTML | *What is this thing?* |
| Presentation | CSS | *What does it look like?* |
| Behavior | JavaScript | *What does it do when poked?* |

A tag name is a **type annotation**, not a styling instruction. `<p>Hello</p>` and `<div>Hello</div>` can be made pixel-identical with three lines of CSS. They are not the same thing, for the same reason `TIMESTAMP` and `VARCHAR(32)` are not the same thing even when they hold identical-looking strings. The type carries meaning that downstream consumers depend on: screen readers, search crawlers, reader mode, browser autofill, your CSS selectors, your `querySelector` calls, automated tests.

In that analogy, `<div>` and `<span>` are `BLOB`. They store anything and tell you nothing.

---

## Part 1 — Anatomy of an element

```html
<a href="/docs" class="nav-link" data-section="7">Read the docs</a>
```

| Piece | Value here |
|---|---|
| Start tag | `<a href="/docs" class="nav-link" data-section="7">` |
| Tag name | `a` |
| Attributes | `href`, `class`, `data-section` |
| Content | `Read the docs` |
| End tag | `</a>` |

### Tag vs. element

These are used sloppily in conversation, but they are distinct and the distinction matters:

- A **tag** is a delimiter in the text. `<a>` and `</a>` are two tags.
- An **element** is the node in the tree: the start tag, its attributes, its content, and its end tag, as one object.

You style elements. You query elements. You never manipulate a tag at runtime — by the time JavaScript runs, tags no longer exist; only nodes do.

### Void elements

Some elements are defined as having no content. They take no closing tag, and writing one is a parse error:

```
area  base  br  col  embed  hr  img  input  link  meta  source  track  wbr
```

```html
<input type="email" name="user_email">    <!-- correct -->
<input type="email" name="user_email" />  <!-- tolerated; the slash is ignored -->
<input type="email"></input>              <!-- invalid -->
```

This is not a style preference. The parser's content model for `input` is *nothing*, so there is no place for children to go.

### Attributes

Three shapes:

```html
<input disabled>                 <!-- boolean: presence is truth -->
<input type=text>                <!-- unquoted value (legal, fragile) -->
<input type="text" value="3">    <!-- quoted value (do this) -->
```

**Invariant: attribute values are always strings.** `value="3"` is the two-character... one-character string `"3"`, never the number 3. Every numeric attribute you read from the DOM needs an explicit cast. This bites people coming from typed backends constantly.

**Boolean attributes** are true if present and false if absent. `disabled="false"` is *true*. The only way to make it false is to remove the attribute.

### Global vs. specific attributes

**Global** — legal on every element: `id`, `class`, `style`, `title`, `hidden`, `lang`, `dir`, `tabindex`, `role`, `aria-*`, `data-*`.

**Element-specific** — `href` means something on `<a>` and nothing on `<div>`. The browser silently ignores attributes an element doesn't define. No error, no warning. This is a recurring source of "why isn't it working."

`data-*` is the sanctioned escape hatch for your own metadata:

```html
<li data-farm-id="4471" data-crop="canola">Field 7</li>
```

Readable from JS as `el.dataset.farmId` and `el.dataset.crop` (kebab-case → camelCase, automatically).

---

## Part 2 — The invariants

These are the rules you can rely on. Internalize these and most of HTML becomes derivable.

**1. Exactly one parent.**
Every node except the root document has exactly one parent. It is a tree, not a graph, not a DAG.

**2. Nesting is strict; overlap is impossible.**
```html
<p><em>valid nesting</em></p>       <!-- fine -->
<p><em>overlap</p></em>             <!-- meaningless; parser will repair it -->
```

**3. Content models constrain children.**
Each element declares what may go inside it. `<ul>` accepts `<li>` and essentially nothing else. `<p>` accepts only phrasing content. `<select>` accepts only `<option>` and `<optgroup>`.

**4. The parser never fails. It repairs.**
This is the single most important and most dangerous property of HTML. There is no such thing as a parse error that stops the page. Invalid markup is silently coerced into *some* valid tree — just not necessarily the one you wrote.

```html
<p>Before <div>Inside</div> After</p>
```

You wrote a `div` inside a `p`. The parser knows that's illegal, so it closes the `p` first:

```
p
├─ "Before "
div
└─ "Inside"
"After"
p (empty — the stray </p> creates one)
```

Your CSS rule `p div { ... }` now matches nothing, and you will spend twenty minutes in DevTools before you notice the tree doesn't match your source. **Compare the Elements panel to your source whenever behavior is inexplicable.**

**5. Tag name determines three things at once:** default CSS `display`, semantic meaning, and implicit ARIA role. You can override the first. You cannot override the second or third except deliberately, with `role`.

**6. `id` is unique per document.** Duplicates don't throw; `getElementById` just returns the first and every reference-based association silently targets the wrong node.

**7. Whitespace collapses.** Any run of spaces, tabs, and newlines in normal flow renders as a single space. Leading and trailing whitespace in a block is dropped. `<pre>` and `white-space: pre` opt out.

**8. Case-insensitive tag and attribute names; case-sensitive values.** `<DIV CLASS="Foo">` parses as `div` with class `Foo`. Write lowercase.

---

## Part 3 — Content categories

The spec groups elements into overlapping categories. You need two of them day-to-day.

**Flow content** — nearly everything that can appear in the body. `div`, `p`, `ul`, `table`, `form`, `section`, and also all phrasing content.

**Phrasing content** — text and things that mark up text inline. `span`, `a`, `code`, `em`, `strong`, `img`, `input`, `label`, `button`, and raw text itself.

The relationship is containment: **all phrasing content is flow content; not all flow content is phrasing content.** Think of it as a subtype.

The practical consequence is a rule you will hit within your first week:

> `<p>` accepts **only** phrasing content.

So `<p><span>ok</span></p>` is legal and `<p><div>no</div></p>` is not. Same for `<label>`, `<button>`, `<h1>`–`<h6>`, `<a>` in most practical use — all phrasing-only or phrasing-preferring.

Other categories, briefly: **metadata** (`title`, `meta`, `link`, `style`, `script` — belong in `<head>`), **embedded** (`img`, `video`, `canvas`, `iframe`), **interactive** (`a[href]`, `button`, `input`, `select`, `textarea`, `details`), **sectioning** (`section`, `article`, `nav`, `aside`), **heading** (`h1`–`h6`).

### Block vs. inline is CSS, not HTML

"Block-level" and "inline" are *default CSS `display` values*, not HTML categories — though they correlate strongly with flow vs. phrasing, which is why people conflate them.

| | Default display | Occupies | Width/height respected |
|---|---|---|---|
| `div`, `p`, `ul`, `li`, `form` | `block` | Full line | Yes |
| `span`, `a`, `code`, `label` | `inline` | Only its content | No (use `inline-block`) |

`display: block` on a `<span>` changes its *layout* and nothing else. It remains phrasing content, it remains illegal to put a `<div>` in it inside a `<p>`, and it still reports no semantic role. **Changing display never changes the content model.**

---

## Part 4 — The elements

### 4.1 Generic containers: `div` and `span`

```html
<div class="dashboard-panel">
  <p>Yield forecast updated <span class="timestamp">3 minutes ago</span>.</p>
</div>
```

| | `div` | `span` |
|---|---|---|
| Category | Flow | Phrasing |
| Default display | `block` | `inline` |
| Semantics | None | None |
| ARIA role | `generic` | `generic` |
| Can contain | Flow content | Phrasing content |

They are the same element in every respect except content category and default display. Both exist for exactly one purpose: to attach styling or scripting hooks to a region when **no semantic element describes it**.

The discipline: reach for a semantic element first (`nav`, `header`, `main`, `section`, `article`, `aside`, `footer`, `figure`). Fall back to `div` when nothing fits. "Div soup" — a page that is 200 nested `div`s — parses fine, renders fine, and is invisible to every consumer that isn't a browser with CSS enabled.

`span` is for marking a *fragment of text* inside a sentence. If you want `<span>` around a block, you want `<div>`.

---

### 4.2 `p` — paragraph

```html
<p>Forecast confidence dropped after the July rainfall revision.</p>
```

- **Content model: phrasing only.** No `div`, no `ul`, no `table`, no nested `p`.
- Cannot nest. `<p>a<p>b</p></p>` produces two siblings.
- The closing tag is technically optional — the parser auto-closes a `p` when it meets another block-level element. Always write it anyway; relying on implicit closing makes the real tree diverge from your source.
- Default margin above and below comes from the UA stylesheet, not from the element's nature.

**Semantic point:** `<p>` means "this is a paragraph of prose," not "this is a block of text with space around it." Use `<div>` for the latter.

---

### 4.3 `code` — computer code

```html
<p>Call <code>df.groupby("farm_id")</code> before aggregating.</p>
```

- Phrasing content. Inline. Monospace by default (UA stylesheet only — the semantics don't imply the font).
- Marks a **fragment** of code inside running text.

For a multi-line block you need `<pre>` as well, because `<code>` alone doesn't preserve whitespace:

```html
<pre><code>def forecast(acres, yield_per_acre):
    return acres * yield_per_acre
</code></pre>
```

`<pre>` supplies the whitespace preservation; `<code>` supplies the semantics. Neither alone does the job. The content inside must still be entity-escaped — `&lt;`, `&gt;`, `&amp;` — because `<pre>` is not a raw-text element; only `<script>`, `<style>`, `<textarea>`, and `<title>` are.

Related phrasing elements in the same family: `<kbd>` (user input), `<samp>` (program output), `<var>` (variable name), `<pre>` (preformatted block).

---

### 4.4 `a` — anchor

```html
<a href="/reports/2026-q3" rel="noopener" target="_blank">Q3 report</a>
```

| Attribute | Purpose |
|---|---|
| `href` | Destination. URL, `#fragment`, `mailto:`, `tel:` |
| `target` | Browsing context. `_blank` opens a new tab |
| `rel` | Relationship. `noopener`, `noreferrer`, `nofollow` |
| `download` | Download rather than navigate |

**Invariants:**

- **`href` is what makes it a link.** `<a>` without `href` is a placeholder: no role of `link`, not focusable, not keyboard-activatable, no pointer cursor.
- **No interactive content inside an `<a>`.** No nested `<a>`, no `<button>`, no `<input>`, no `<select>`. The activation target would be ambiguous.
- `<a>` may wrap block content in modern HTML (`<a href="..."><div>…</div></a>` is legal) as long as nothing inside is interactive. This is the one phrasing element with that exemption.
- `target="_blank"` without `rel="noopener"` gives the opened page a handle to your `window` object. Modern browsers imply `noopener`, but write it — you don't control which browser.

**Link vs. button** — this is a real decision, not a cosmetic one:

| Use `<a href>` | Use `<button>` |
|---|---|
| Navigates somewhere | Performs an action |
| Has a meaningful URL | Has no URL |
| Should be right-clickable, bookmarkable, middle-clickable | Shouldn't be |
| Activated by Enter | Activated by Enter **and** Space |

Styling a `<div>` to look like a button gives you neither. It is not focusable, not keyboard-operable, announces as nothing.

---

### 4.5 Lists: `ul` and `li`

```html
<ul>
  <li>Canola</li>
  <li>Durum wheat</li>
  <li>Lentils
    <ul>
      <li>Red</li>
      <li>Green</li>
    </ul>
  </li>
</ul>
```

**Invariants:**

- `<ul>` (and `<ol>`) may contain **only** `<li>` as element children (plus `<script>` and `<template>`). Not `<div>`, not text, not `<p>`.
- `<li>` must have `<ul>`, `<ol>`, or `<menu>` as its parent.
- `<li>` **can** contain flow content — paragraphs, divs, tables, whatever.

**The nesting rule that trips everyone:** a nested list goes *inside* an `<li>`, never between two `<li>`s. Look at "Lentils" above — the inner `<ul>` is a child of that `<li>`. Putting it between list items is invalid, and the parser's repair will reparent it somewhere you didn't intend.

**`ul` vs `ol`:** `<ul>` when order is incidental, `<ol>` when order is meaningful (steps, rankings). This is semantics again — a screen reader announces "list, 3 items," and `<ol>` conveys sequence. `<ol>` takes `start`, `reversed`, and `type`; `<li>` inside it takes `value`.

**`<dl>`/`<dt>`/`<dd>`** — description lists, for name/value pairs. Useful for metadata tables that aren't really tables. Structure is flat: `<dl>` contains `<dt>` and `<dd>` as siblings, optionally wrapped in a `<div>` (the one place a `div` is explicitly blessed as a list child).

---

### 4.6 Forms

Forms are a self-contained subsystem with their own state model, submission protocol, and validation engine. Worth learning as a unit.

#### `form`

```html
<form action="/api/forecast" method="post">
  <!-- controls -->
</form>
```

| Attribute | Meaning |
|---|---|
| `action` | Where the data goes. Omitted → current URL |
| `method` | `get` (data in query string) or `post` (data in body) |
| `enctype` | `application/x-www-form-urlencoded` (default), `multipart/form-data` (required for file uploads), `text/plain` |
| `novalidate` | Skip built-in constraint validation |

**The submission invariant, which is the thing to memorize:**

> On submit, the browser collects every **non-disabled** control that has a **`name`** attribute and is **associated with this form**, and sends `name=value` pairs.

Three conditions, all required. A control with no `name` is not submitted — it's display-only from the server's point of view. A `disabled` control is not submitted (use `readonly` if you want it sent but uneditable).

"Associated with this form" normally means "is a descendant of it," but the `form="some-form-id"` attribute lets a control live anywhere in the document and still belong to a form. This is the escape hatch when your layout won't let you nest.

#### `input`

The polymorphic one. A single element whose `type` attribute switches it into roughly two dozen different widgets with different value semantics.

```html
<input type="text"     name="farm_name">
<input type="email"    name="contact"     required>
<input type="number"   name="acres"       min="0" step="0.1">
<input type="date"     name="seeded_on">
<input type="checkbox" name="irrigated"   value="yes">
<input type="radio"    name="tillage"     value="no_till">
<input type="file"     name="yield_csv"   accept=".csv">
<input type="hidden"   name="client_id"   value="4471">
<input type="range" type="color" type="password" type="search" type="tel" type="url">
```

Common attributes: `name`, `value`, `placeholder`, `required`, `disabled`, `readonly`, `min`, `max`, `step`, `pattern`, `maxlength`, `autocomplete`, `checked` (checkbox/radio only).

**Things that surprise people:**

- **Checkboxes submit nothing when unchecked.** Not `false`, not empty string — the key is simply absent from the payload. Server-side you check for key presence, not value. The classic workaround is a hidden input with the same `name` immediately before it.
- **Radio grouping is by `name`, not by DOM position.** All radios sharing a `name` are mutually exclusive, wherever they sit in the tree.
- `type="number"` still gives you a string from `.value`. Everything does.
- An unrecognized `type` falls back to `text` silently. That's why new input types are safe to use.
- `placeholder` is not a label. It disappears on focus, fails contrast guidelines, and is not read as the accessible name.

#### `label`

```html
<!-- Explicit association: for → id -->
<label for="acres-input">Seeded acres</label>
<input id="acres-input" type="number" name="acres">

<!-- Implicit association: wrapping -->
<label>
  Seeded acres
  <input type="number" name="acres">
</label>
```

`<label>` does two concrete things:

1. **Extends the click target.** Clicking the label focuses or toggles the control. Large practical difference for checkboxes and radios.
2. **Supplies the accessible name.** A screen reader announces "Seeded acres, number, edit." Without a label it announces "edit" — nothing.

**Invariants:**

- `for` must match the control's `id` exactly. Case-sensitive. A typo fails silently — no error, just a dead label.
- One label per control for the primary name (multiple labels are legal but get concatenated, which is rarely what you want).
- A label may contain at most one labelable control.
- Phrasing content only.

Prefer the explicit `for`/`id` form. It survives refactors that move the input, and it works when your CSS layout needs the label and input as siblings.

#### `select` and `option`

```html
<select name="crop" id="crop-select">
  <option value="">Choose a crop…</option>
  <option value="canola" selected>Canola</option>
  <option value="wheat">Durum wheat</option>
  <optgroup label="Pulses">
    <option value="lentil">Lentils</option>
    <option value="pea">Field peas</option>
  </optgroup>
</select>
```

**Invariants:**

- `<option>` must be a child of `<select>`, `<optgroup>`, or `<datalist>`. Nowhere else.
- `<optgroup>` must be a child of `<select>` and may contain only `<option>`. **Groups do not nest.**
- `<option>` contains **plain text only**. No `<span>`, no `<strong>`, no markup at all. If you need styled options you need a custom component, and then you owe the keyboard and ARIA behavior yourself.

**`value` vs. text content:** the `value` attribute is what gets submitted; the text content is what the user sees. If `value` is omitted, the text content is submitted instead:

```html
<option value="no_till">No-till (conservation)</option>  <!-- submits "no_till" -->
<option>No-till</option>                                  <!-- submits "No-till" -->
```

Always set `value` explicitly. Otherwise a copy edit to the label silently changes your API payload.

Other attributes: `selected` (boolean, the default choice), `multiple` on `<select>` (multi-select; submits repeated `name=value` pairs), `size` (visible row count), `disabled`.

#### `button`

```html
<button type="submit">Run forecast</button>
<button type="button" onclick="…">Add row</button>
<button type="reset">Clear</button>
```

**The trap, and it is a good one:**

> Inside a `<form>`, `<button>` defaults to `type="submit"`.

A button with no `type` attribute submits the form. If you wired a click handler and the page reloads and your state vanishes, this is why. **Always write `type` explicitly.** There is no cost and it removes an entire class of bug.

| `type` | Behavior |
|---|---|
| `submit` | Submits the form. **Default.** |
| `button` | Does nothing on its own |
| `reset` | Restores all controls to their initial values |

Other properties:

- `<button>` accepts phrasing content, so you can put an icon and a `<span>` inside it. `<input type="submit">` is a void element and can only carry a text `value` — prefer `<button>`.
- If a `<button type="submit">` has a `name` and `value`, those are submitted too. That's how you tell which of several submit buttons was pressed.
- **Implicit submission:** pressing Enter in a text input submits the form via the first submit button, whether or not you wanted that.
- Focusable and keyboard-activatable for free, by Enter *and* Space.

---

## Part 5 — How elements relate: four mechanisms

Relationships in HTML are not all tree edges. There are four distinct kinds, and mixing them up causes confusion.

### 1. Containment — the tree itself

Structural parent/child, enforced by the content model.

```
ul → li          (li must be a child of ul/ol/menu)
select → option  (option must be a child of select/optgroup/datalist)
table → tbody → tr → td
```

This drives CSS descendant selectors, `closest()`, event bubbling, and inheritance of inherited properties.

### 2. Reference — `id`-based pointers

An attribute on one element names the `id` of another. The two need no tree relationship at all.

```html
<label for="x">…</label>              <input id="x">
<div aria-labelledby="hdr">…</div>    <h2 id="hdr">…</h2>
<input form="main-form">              <form id="main-form">
<input list="crops">                  <datalist id="crops">
```

These are the "foreign keys" of the document. Like foreign keys with no constraint enforcement: a dangling reference produces no error, just a missing relationship.

### 3. Implicit ownership — form association

Form controls are owned by their nearest ancestor `<form>`, without any attribute. The `form` attribute overrides this. Radio groups are a second implicit relationship, keyed on shared `name` within the same form.

### 4. Cascade and query targets

The tree is the surface CSS selectors and `querySelectorAll` run against. `ul > li`, `form input[required]`, `p + p` — every one of these is a query over the tree you declared. Which is the practical reason structure matters even when you don't care about semantics: **your structure is your query API.**

---

## Part 6 — The accessibility tree

Alongside the DOM, the browser builds a second tree exposing each element's **role**, **name**, **state**, and **value** to assistive technology. Most of it is derived automatically from tag names:

| Element | Implicit role | Accessible name from |
|---|---|---|
| `<a href>` | `link` | Text content |
| `<button>` | `button` | Text content |
| `<input type="text">` | `textbox` | Associated `<label>` |
| `<select>` | `combobox` | Associated `<label>` |
| `<ul>` | `list` | — |
| `<li>` | `listitem` | — |
| `<p>` | `paragraph` | — |
| `<div>`, `<span>` | `generic` | — |

This is the concrete payoff for semantic markup. `<button>` gives you role, focusability, Enter/Space activation, and a disabled state for free. `<div onclick>` gives you a rectangle. Reproducing what `<button>` does requires `role="button"`, `tabindex="0"`, keydown handlers for Enter and Space, `aria-disabled` management, and focus styles — and you will get one of them wrong.

The rule: **use the native element; use `role` only to describe something that has no native equivalent.**

---

## Part 7 — The traps, collected

| Trap | What actually happens |
|---|---|
| `<div>` inside `<p>` | Parser closes the `p` early; your tree ≠ your source |
| `<button>` with no `type` in a form | Submits the form and reloads the page |
| `<label for>` typo | Silently no association; no error anywhere |
| `disabled` on a control you need submitted | Omitted from the payload; use `readonly` |
| Unchecked checkbox | Key absent from payload, not `false` |
| `<option>` with no `value` | Submits its visible text; label edits break the API |
| Markup inside `<option>` | Ignored or stripped |
| Nested `<ul>` placed between `<li>`s | Invalid; reparented unpredictably |
| `disabled="false"` | Still disabled — boolean attributes are presence-based |
| `.value` treated as a number | Always a string |
| Duplicate `id` | First match wins; references point at the wrong node |
| `placeholder` used as a label | No accessible name; announced as unlabeled |
| `<div>` styled as a button | Not focusable, not keyboard-operable, no role |
| `display: block` on a `<span>` to hold a `<div>` | Layout changes; content model does not |
| `target="_blank"` without `rel="noopener"` | Opened page gets a handle on your window |

---

## Part 8 — What changes in React

Since this feeds directly into React work, the deltas:

| HTML | JSX | Why |
|---|---|---|
| `class="x"` | `className="x"` | `class` is a reserved word |
| `for="x"` | `htmlFor="x"` | `for` is a reserved word |
| `<input>` | `<input />` | JSX requires every element closed |
| `onclick="…"` | `onClick={…}` | camelCase, and a function not a string |
| `style="color:red"` | `style={{ color: 'red' }}` | Object, camelCased properties |
| `tabindex` | `tabIndex` | camelCase throughout |

Structural differences:

- **Controlled inputs.** `<input value={x} onChange={e => setX(e.target.value)} />`. Passing `value` without `onChange` freezes the field — React re-renders it back to `x` after every keystroke. Use `defaultValue` for an uncontrolled input.
- **`checked` / `onChange`** is the same pattern for checkboxes and radios.
- **`<select value={…}>`** — React puts `value` on the `select`, not `selected` on the `option`. Don't use `selected`.
- **Form submission** — `<form onSubmit={e => { e.preventDefault(); … }}>`. Without `preventDefault()` the browser does a full navigation and your SPA state is gone. Note the corollary: the trap in Part 7 about implicit button types is *worse* in React, because the page reload looks like a crash.
- **Lists need `key`** on the element produced by each iteration, and the key belongs on the outermost element returned by the map callback.
- **Fragments** — `<>…</>` when you need to return siblings without introducing a `div`. This is the escape hatch that keeps div soup down.

The underlying HTML content models are unchanged. React will happily render `<p><div/></p>` into the DOM, and the browser will repair it exactly as before.

---

## Part 9 — Quick reference

| Element | Category | Display | Content model | Key attributes |
|---|---|---|---|---|
| `div` | Flow | block | Flow | Global only |
| `span` | Phrasing | inline | Phrasing | Global only |
| `p` | Flow | block | **Phrasing only** | Global only |
| `code` | Phrasing | inline | Phrasing | Global only |
| `a` | Phrasing, interactive | inline | Flow, **no interactive** | `href`, `target`, `rel`, `download` |
| `ul` | Flow | block | **`li` only** | Global only |
| `li` | — | list-item | Flow | `value` (in `ol`) |
| `form` | Flow | block | Flow, **no nested form** | `action`, `method`, `enctype`, `novalidate` |
| `label` | Phrasing, interactive | inline | Phrasing, ≤1 control | `for` |
| `input` | Phrasing, interactive | inline-block | **Void** | `type`, `name`, `value`, `required`, `disabled` |
| `select` | Phrasing, interactive | inline-block | **`option`/`optgroup` only** | `name`, `multiple`, `size`, `required` |
| `option` | — | block | **Text only** | `value`, `selected`, `disabled` |
| `button` | Phrasing, interactive | inline-block | Phrasing, **no interactive** | `type`, `name`, `value`, `disabled` |

---

## Part 10 — A worked example

Everything above, in one small form:

```html
<form action="/api/fields" method="post">
  <h2 id="form-heading">Add a field</h2>

  <div class="field-row">
    <label for="field-name">Field name</label>
    <input id="field-name" name="field_name" type="text" required
           autocomplete="off" maxlength="80">
  </div>

  <div class="field-row">
    <label for="acres">Seeded acres</label>
    <input id="acres" name="acres" type="number" min="0" step="0.1" required>
  </div>

  <div class="field-row">
    <label for="crop">Crop</label>
    <select id="crop" name="crop" required>
      <option value="">Choose a crop…</option>
      <option value="canola">Canola</option>
      <option value="durum">Durum wheat</option>
      <optgroup label="Pulses">
        <option value="lentil">Lentils</option>
        <option value="pea">Field peas</option>
      </optgroup>
    </select>
  </div>

  <fieldset>
    <legend>Tillage practice</legend>
    <label><input type="radio" name="tillage" value="no_till" checked> No-till</label>
    <label><input type="radio" name="tillage" value="min_till"> Minimum till</label>
    <label><input type="radio" name="tillage" value="conventional"> Conventional</label>
  </fieldset>

  <p>
    Acreage is validated against the parcel registry. See
    <a href="/docs/parcels">the parcel documentation</a> for how
    <code>field_name</code> is matched.
  </p>

  <input type="hidden" name="client_id" value="4471">

  <button type="submit">Save field</button>
  <button type="button" class="secondary">Cancel</button>
</form>
```

Trace the mechanisms:

- **Containment** — `form` owns every control; `select` owns its `option`s; `optgroup` owns the pulse options.
- **Reference** — each `label[for]` points at an `id`. The radios use the implicit wrapping form instead, which is reasonable for short labels.
- **Implicit ownership** — nothing carries a `form` attribute, so every control is owned by the ancestor `form`. The three radios share `name="tillage"`, which is what makes them mutually exclusive.
- **Submission payload** — `field_name`, `acres`, `crop`, `tillage`, `client_id`. Note `client_id` ships via a hidden input, and the Cancel button contributes nothing because it has no `name` and `type="button"`.
- **The `div`s** are pure layout hooks with no semantics — the correct use of `div`. `fieldset`/`legend` is used instead where a semantic grouping exists, because it gives the radio group an accessible group name for free.
- **`type` is explicit on both buttons.** Cancel would otherwise submit the form.