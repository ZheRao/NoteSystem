# High-level React Process Cycle

I actually think this is the perfect place to start learning React—not from syntax, but from **how data flows through the application**. Looking at your code, I can also tell Claude intentionally wrote it in a very "React-ish" way: the UI is almost entirely driven by **state**. Once you understand that idea, about 80% of the framework suddenly makes sense.

Let's ignore React syntax almost completely. Imagine this is just a program that happens to draw a webpage.

---

## Step 0. The page first opens

The browser downloads your React application.

The first thing that happens is this function runs:

```jsx
export default function ForecastExplorer() {
```

Think of this as

> "Draw the entire page."

Everything inside this function decides what should currently be visible.

---

## Step 1. React creates some memory

Near the top are lots of `useState()` calls.

```jsx
const [flow, setFlow] = useState([]);
const [mn, setMn] = useState(null);
const [selRow, setSelRow] = useState(null);
const [mb, setMb] = useState(null);
```

Forget the syntax.

Just imagine React secretly creates memory boxes.

```
flow
──────────────
[]

mn
──────────────
null

selRow
──────────────
null

mb
──────────────
null
```

At the very beginning they are all empty.

Nothing has been loaded yet.

---

## Step 2. React draws using the current memory

Immediately afterwards React reaches

```jsx
<AreaChart
    data={flow}
>
```

But remember

```
flow = []
```

So effectively it's doing

```
draw chart with:

[]
```

There is nothing to draw.

So you either see nothing or an empty chart.

---

## Step 3. React notices "I should load data"

Now React reaches this:

```jsx
useEffect(() => {
    if (!orchKey) return;

    api.cashflow(orchKey)
        .then(setFlow);
}, [orchKey]);
```

Forget the syntax.

Read it like English.

> Whenever `orchKey` changes,
> go ask the backend for cashflow.

Notice this **does not happen immediately**.

Instead React says

> I'll remember to do this after the screen has been drawn.

That's what `useEffect` means.

---

## Step 4. Browser sends HTTP request

The effect executes

```jsx
api.cashflow(orchKey)
```

which expands into

```jsx
getJSON(
    `/forecast/${orchKey}/cashflow`
)
```

which expands into

```jsx
fetch(...)
```

So the browser sends

```
GET

/forecast/ABC/cashflow
```

to FastAPI.

Nothing freezes.

The browser just keeps waiting.

---

## Step 5. FastAPI runs

On the server

```python
@app.get("/forecast/{orchestration_key}/cashflow")
```

runs.

Eventually it executes

```python
queries.net_cash_flow(...)
```

which returns something like

```json
[
    {
        "month":"Jan",
        "net":120,
        "inflow":300,
        "outflow":-180
    },
    {
        "month":"Feb",
        "net":75,
        "inflow":250,
        "outflow":-175
    }
]
```

This JSON travels back across the network.

---

## Step 6. React receives JSON

Remember this line?

```jsx
.then(setFlow)
```

That means

> when the JSON arrives,
> put it into the `flow` memory box.

So

before

```
flow

[]
```

after

```
flow

[
  Jan...
  Feb...
  Mar...
]
```

That's literally all it does.

---

## Step 7. Here is React's magic

The moment

```
flow
```

changes,

React immediately says

> Oh!

> My memory changed!

> I need to redraw the page.

It automatically calls

```jsx
ForecastExplorer()
```

again.

Not because you told it to.

Because the state changed.

---

## Step 8. Second render

Now React reaches

```jsx
<AreaChart
    data={flow}
>
```

But this time

```
flow
```

contains

```
Jan
Feb
Mar
...
```

instead of

```
[]
```

So now Recharts receives

```
AreaChart

data:

Jan
Feb
Mar
...
```

instead of

```
AreaChart

data:

[]
```

Recharts now has something to draw.

It creates the line chart.

---

At this point the chart appears.

---

## Step 9. User clicks July

Suppose the user clicks

```
July
```

This line executes

```jsx
onClick={(e) =>
    pickMonth(
        e.activePayload[0].payload.month_num,
        e.activePayload[0].payload
    )
}
```

Let's inspect what

```
payload
```

actually is.

It is simply

```json
{
    "month_num":7,
    "month":"Jul",
    "net":100,
    "inflow":300,
    "outflow":-200
}
```

Exactly one row from

```
flow
```

Nothing more.

---

## Step 10. pickMonth()

Now

```jsx
pickMonth(...)
```

runs

```jsx
setMn(7)

setSelRow({
    month:"Jul",
    net:100,
    inflow:300,
    outflow:-200
})
```

So the memory becomes

```
mn

7
```

and

```
selRow

{
    month:"Jul",
    ...
}
```

Again—

React notices

```
State changed.
```

---

## Step 11. React redraws again

It calls

```jsx
ForecastExplorer()
```

again.

Now this code

```jsx
{mn && (
    ...
)}
```

becomes

```
if (mn != null)
```

which is

```
if (7)
```

True.

So Level 1 panel appears.

It already has

```
selRow
```

so it immediately displays

```
Inflow

300

Outflow

-200

Net

100
```

without any API call.

That's why the comment says

```
these three came from the chart point
```

---

## Step 12. Another useEffect wakes up

Changing

```
mn
```

also triggered

```jsx
useEffect(() => {

    api.month(...)

}, [mn])
```

React notices

> Oh,

> mn changed.

Time to load month details.

So the browser sends

```
GET

/month/7
```

---

## Step 13. FastAPI returns categories

FastAPI returns

```json
{
    categories:

    [
        {
            category:"Revenue",
            amount:500
        },
        {
            category:"Fuel",
            amount:-200
        }
    ]
}
```

---

## Step 14. setMb()

Then

```jsx
.then(setMb)
```

runs.

Now

```
mb

=
{
    categories...
}
```

---

## Step 15. React redraws AGAIN

React notices

```
mb changed.
```

Calls

```jsx
ForecastExplorer()
```

again.

Now

```jsx
const cats =
    mb
    ? mb.categories
    : [];
```

becomes

```
cats

=

[
Revenue
Fuel
...
]
```

instead of

```
[]
```

Now this loop

```jsx
cats.map(...)
```

has something to iterate over.

So React creates

```
Revenue ███████████

Fuel ████

Chemical █████
```

The category bars appear.

---

## Notice the pattern

This is the entire philosophy of React.

It almost never says

```
Draw this.

Erase that.

Move this.

Replace that.
```

Instead it always follows

```
State

↓

Render

↓

User changes state

↓

Render again

↓

Network changes state

↓

Render again

↓

User clicks

↓

Render again
```

The browser isn't being manually instructed to update individual pieces.

Instead it's constantly asking:

> "Given the current state, what should the page look like?"

---

## I think this is the most important realization

Coming from Python, you naturally think procedurally:

```
draw chart

wait

click

draw panel

wait

draw categories
```

React **doesn't think procedurally**.

It thinks declaratively.

The page is more like a mathematical function:

```
UI = f(state)
```

Every time the state changes:

```
state₀
↓

UI₀


state₁
↓

UI₁


state₂
↓

UI₂
```

Nothing on the page explicitly says

> "insert this `<div>` now."

Instead, React throws away the old description of the UI, computes a new one from the latest state, and efficiently updates only the parts of the real DOM that actually changed. Once this mental model clicks, `useState`, `useEffect`, API calls, and rendering all start to feel like different pieces of the same pipeline rather than unrelated React features. 

# Detail: React State Change + Re-rendering

Yes—your mental model is now very close.

`ForecastExplorer()` is re-run to produce a new description of the UI whenever React decides the component needs to render again. The function contains conditional descriptions such as:

```jsx
{!mn && <EmptyMessage />}
{mn && <MonthPanel />}
{mn && cat && <SourcesPanel />}
```

That is similar to a collection of `if` statements or a `match`/`switch`, though React is not literally executing one branch and permanently drawing it. It re-evaluates the complete description and reconciles it against what is already on screen.

The three key questions are:

1. How does React remember state when the function starts over?
2. What triggers another render?
3. How does React know whether a `useEffect` should run?

---

## 1. How does `useState` persist when the function runs again?

Consider:

```jsx
const [mn, setMn] = useState(null);
```

It may look as though every function call executes:

```js
mn = null;
```

But that is not what happens.

`useState(null)` means approximately:

> React, give me the state value assigned to this hook position. If this component has never rendered before, initialize it to `null`.

The state is **not stored inside the local JavaScript variable `mn`**.

React stores the state outside the function, in React’s internal representation of that particular component instance.

A simplified picture is:

```text
React's internal memory for ForecastExplorer
──────────────────────────────────────────────
hook position 1 → orchestrations = [...]
hook position 2 → orchKey = "client-2026-3-pre-harvest"
hook position 3 → flow = [...]
hook position 4 → mn = 7
hook position 5 → selRow = {...}
hook position 6 → mb = {...}
...
```

Each time React calls:

```jsx
ForecastExplorer()
```

the local variables are recreated, but the calls to `useState` retrieve values from that internal memory.

So on the first render:

```jsx
const [mn, setMn] = useState(null);
```

returns:

```js
mn === null
```

Later:

```jsx
setMn(7);
```

updates React’s internal state slot.

On the next render, the same source line:

```jsx
const [mn, setMn] = useState(null);
```

returns:

```js
mn === 7
```

The `null` argument is only the **initial value**. It is not reapplied on every render.

A rough conceptual implementation would look like:

```js
function useState(initialValue) {
    const existingState = ReactInternalMemory[currentHookPosition];

    if (existingState already exists) {
        return existingState;
    }

    ReactInternalMemory[currentHookPosition] = initialValue;
    return initialValue;
}
```

The real implementation is considerably more sophisticated, but that is the correct mental model.

---

## 2. Why hook order matters

React identifies state largely by the order in which hooks are called.

Your component has:

```jsx
const [orchestrations, setOrchestrations] = useState([]);
const [orchKey, setOrchKey] = useState(null);
const [flow, setFlow] = useState([]);
const [mn, setMn] = useState(null);
const [selRow, setSelRow] = useState(null);
const [mb, setMb] = useState(null);
```



React conceptually associates them with positions:

```text
useState call 1 → orchestrations
useState call 2 → orchKey
useState call 3 → flow
useState call 4 → mn
useState call 5 → selRow
useState call 6 → mb
```

That is why hooks must not be conditionally called like this:

```jsx
if (something) {
    const [mn, setMn] = useState(null);
}
```

Suppose `something` is true on one render and false on another. All later hook positions would shift, and React could no longer reliably tell which state belongs to which hook.

So hooks are called unconditionally at the top level, while conditional logic is used later for rendering.

---

## 3. What exactly triggers another render?

The most common trigger in your code is a state setter:

```jsx
setMn(7);
setFlow(rows);
setMb(response);
setCat("Revenue");
```

Calling a setter tells React:

> The state associated with this component has changed. Schedule another render.

For example:

```jsx
setMn(7);
```

does not directly call:

```jsx
ForecastExplorer();
```

in the ordinary procedural sense.

Instead, it asks React to schedule an update. React may batch several updates together before rendering.

Your `pickMonth` function does several updates:

```jsx
function pickMonth(m, point) {
    if (!m) return;

    setMn(m);
    setSelRow(point ?? null);
    setCat(null);
    setOok(null);
    setChosenRev(null);
    setLin(null);
}
```



React normally batches these into one subsequent render rather than rendering six separate times.

Conceptually:

```text
User clicks July
      ↓
pickMonth runs
      ↓
queue:
    mn = 7
    selRow = July object
    cat = null
    ook = null
    chosenRev = null
    lin = null
      ↓
React performs one new render
```

Other common causes of rendering include:

* a parent component rendering and passing new props;
* a subscribed context value changing;
* an external store subscription notifying React;
* certain framework-level updates.

But in this component, state setters are the main trigger.

---

## 4. Does assigning a normal variable trigger rendering?

No.

This will not trigger React:

```jsx
mn = 7;
```

Nor would this be valid if `mn` was declared with `const`.

Even if you had:

```jsx
let number = 1;
number = 2;
```

React would not care. It does not observe arbitrary JavaScript assignments.

React only knows about state changes made through mechanisms such as:

```jsx
setMn(7);
```

That setter is React’s notification mechanism.

---

## 5. How does React know whether a `useEffect` should run?

Consider:

```jsx
useEffect(() => {
    if (!orchKey || !mn) {
        setMb(null);
        return;
    }

    api.month(orchKey, mn)
        .then(setMb)
        .catch((e) => {
            setMb(null);
            setErr(e.message);
        });
}, [orchKey, mn]);
```



The last argument:

```jsx
[orchKey, mn]
```

is the **dependency array**.

It tells React:

> After a render, run this effect when either `orchKey` or `mn` has changed since the previous render.

React remembers the previous dependency values.

Suppose the renders look like this:

```text
Render 1:
orchKey = null
mn      = null

dependencies = [null, null]
```

After the first render, the effect runs because it is the initial mount. It sees:

```jsx
if (!orchKey || !mn)
```

and exits.

Later, orchestration data loads:

```text
Render 2:
orchKey = "farm-2026-pre-harvest"
mn      = null

dependencies changed:
[null, null]
        ↓
["farm-2026-pre-harvest", null]
```

The effect runs again, but still exits because `mn` is null.

Then the user clicks July:

```text
Render 3:
orchKey = "farm-2026-pre-harvest"
mn      = 7

dependencies changed:
["farm-2026-pre-harvest", null]
                ↓
["farm-2026-pre-harvest", 7]
```

Now the effect runs and calls:

```jsx
api.month(orchKey, mn);
```

If some unrelated state changes afterward:

```jsx
setErr(null);
```

and the next render still has:

```text
orchKey = "farm-2026-pre-harvest"
mn      = 7
```

then the dependency values have not changed, so this particular effect does not run again.

---

## 6. The effect is checked after rendering

The sequence is important.

It is approximately:

```text
1. State changes
2. React calls ForecastExplorer()
3. ForecastExplorer returns JSX
4. React updates the browser DOM
5. React checks effects
6. Effects whose dependencies changed are executed
```

So `useEffect` is not usually part of producing the immediate render result.

It handles work that should happen **because a render committed with particular state values**.

That is why network requests belong there.

For the month-selection flow:

```text
User clicks July
      ↓
setMn(7)
setSelRow(July row)
      ↓
React renders immediately using mn=7 and selRow=July
      ↓
The month panel can show inflow/outflow/net
      ↓
After that render, React sees [orchKey, mn] changed
      ↓
useEffect fires API request for category details
      ↓
JSON returns
      ↓
setMb(JSON)
      ↓
React renders again
      ↓
category rows appear
```

This explains why the three summary numbers can appear before the category response arrives.

They come from `selRow`, while the category rows depend on `mb`.

---

## 7. The four dependency-array patterns

These are useful to distinguish.

### No dependency array

```jsx
useEffect(() => {
    console.log("effect");
});
```

This runs after every committed render.

Conceptually:

```text
render → effect
render → effect
render → effect
```

This is often dangerous if the effect updates state, because it can create a loop.

---

### Empty dependency array

```jsx
useEffect(() => {
    loadInitialData();
}, []);
```

This means:

> Run after the component is first mounted, but not because ordinary state values later changed.

Your orchestration loader uses this pattern:

```jsx
useEffect(() => {
    api.orchestrations()
        .then((rows) => {
            setOrchestrations(rows);
            const def =
                rows.find((r) =>
                    r.orchestration_key.endsWith("3-pre-harvest")
                ) ?? rows[0];

            if (def) {
                setOrchKey(def.orchestration_key);
            }
        })
        .catch((e) => setErr(e.message));
}, []);
```



This is the initial page-loading effect.

---

### One dependency

```jsx
useEffect(() => {
    loadCashflow(orchKey);
}, [orchKey]);
```

Runs initially and whenever `orchKey` changes.

Your cash-flow loading effect follows this pattern:

```jsx
useEffect(() => {
    if (!orchKey) return;

    setMn(null);
    setSelRow(null);
    setMb(null);
    setCat(null);
    setSrcs([]);
    setOok(null);
    setChosenRev(null);
    setLin(null);

    api.cashflow(orchKey)
        .then(setFlow)
        .catch((e) => {
            setFlow([]);
            setErr(e.message);
        });
}, [orchKey]);
```



---

### Multiple dependencies

```jsx
useEffect(() => {
    loadMonth(orchKey, mn);
}, [orchKey, mn]);
```

Runs when either one changes.

---

## 8. Is `useEffect` the state-transition mechanism?

Partly, but I would refine that wording.

There are two kinds of state transitions in this component.

### Direct transitions caused by user events

The user clicks something:

```jsx
onClick={() => pickCat(cc.category)}
```

Then:

```jsx
function pickCat(c) {
    setCat(c);
    setOok(null);
    setChosenRev(null);
    setLin(null);
}
```

This is a direct transition:

```text
User event → setter → new state
```

No effect is needed to change `cat`.

### Indirect transitions caused by side effects

After `cat` changes, this runs:

```jsx
useEffect(() => {
    if (!orchKey || !mn || !cat) {
        setSrcs([]);
        return;
    }

    api.sources(orchKey, mn, cat)
        .then(setSrcs)
        .catch(...);
}, [orchKey, mn, cat]);
```



That creates:

```text
State change
    ↓
Effect notices dependency change
    ↓
API request
    ↓
JSON response
    ↓
another setter
    ↓
another state change
```

So `useEffect` is best thought of as:

> A mechanism for synchronizing the React component with something outside the render itself.

That “something outside” may be:

* a server;
* a timer;
* browser storage;
* DOM event subscriptions;
* external libraries;
* sockets;
* analytics;
* document title.

In your app, its main job is synchronizing selection state with the FastAPI server.

---

## 9. The complete lifecycle of `mn`

Here is the exact story.

Initially, React internally stores:

```text
mn = null
```

The function runs:

```jsx
const [mn, setMn] = useState(null);
```

React returns the stored value:

```text
mn = null
```

The JSX reaches:

```jsx
{!mn && <EmptyPanel />}
```

which renders the prompt.

The user clicks a chart point.

Recharts calls:

```jsx
pickMonth(7, julyPoint);
```

That calls:

```jsx
setMn(7);
```

React updates its internal state:

```text
mn: null → 7
```

React schedules a render.

The function runs again.

This time:

```jsx
const [mn, setMn] = useState(null);
```

returns the stored value `7`, not the initial value `null`.

The JSX reaches:

```jsx
{mn && <MonthPanel />}
```

and renders the month panel.

After committing that render, React compares the month effect’s dependencies:

```text
previous: [orchKey, null]
current:  [orchKey, 7]
```

They differ, so React runs the effect.

The effect calls:

```jsx
api.month(orchKey, 7)
```

When the response arrives:

```jsx
setMb(json);
```

updates another internal state slot and schedules another render.

---

## 10. A useful Python analogy

Imagine React maintained a state dictionary outside your function:

```python
react_state = {
    "mn": None,
    "flow": [],
    "mb": None,
}
```

Your component function would conceptually be:

```python
def ForecastExplorer():
    mn = react_state["mn"]
    flow = react_state["flow"]
    mb = react_state["mb"]

    if mn is None:
        page = draw_empty_message()
    else:
        page = draw_month_panel(mn)

    if mb is not None:
        page += draw_categories(mb["categories"])

    return page
```

And a setter might behave like:

```python
def set_mn(new_value):
    react_state["mn"] = new_value
    schedule_render()
```

Then effects would be roughly:

```python
if previous_dependencies != current_dependencies:
    run_effect()
```

React is not implemented this simply, but this analogy accurately captures the relationship among persistent state, renders, setters, and dependency arrays.

---

## 11. One correction to “the function runs whenever state changes”

That is a good beginner model, but technically:

> React schedules a render when a state update is requested.

It may not produce a meaningful DOM update if the new value is effectively unchanged.

For example:

```jsx
setMn(7);
```

when `mn` is already `7` may be skipped or rendered without causing visible changes, depending on context.

React can also batch updates, interrupt work, or render more than once in development mode to detect unsafe behavior.

But for understanding your application, this is sufficient:

```text
setter called
    ↓
React stores new state
    ↓
React invokes component again
    ↓
component returns UI description for new state
```

---

## The central model

You can now think of your component as three coordinated systems:

```text
Persistent React memory
useState
    ↓
Current values:
flow, mn, selRow, mb, cat, srcs, ...
```

```text
UI description
ForecastExplorer(state)
    ↓
JSX describing what should currently exist
```

```text
Side-effect synchronization
useEffect(dependencies)
    ↓
When selected state values change:
call API, receive data, update more state
```

Together:

```text
event or API result
        ↓
state setter
        ↓
persistent React state changes
        ↓
component function runs again
        ↓
new JSX is calculated
        ↓
browser UI is updated
        ↓
changed effect dependencies trigger side effects
        ↓
API result may update state again
```

That is the core engine of this entire React file.
