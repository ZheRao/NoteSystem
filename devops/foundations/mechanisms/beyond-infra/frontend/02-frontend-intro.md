# Webpage and Serving Layer communication

You're right that this is the crux, and I think the fastest way to make it finally click is to *show* you — a real, runnable page where you can watch the data flow and see exactly which array shape feeds which visual, and when a click does or doesn't make a call. Let me explain the patterns clearly and then build you a working demo to study.

Let me check the design guidance before I build the page.The single most useful correction up front: **a chart is filled by one API call that returns the whole array, never one call per point.** Twelve calls to draw a twelve-point line would be twelve network round trips for one picture. The query returns all the points at once, and the chart library loops the array for you. Let me give you the full pattern set, then a working page where you can watch every one of these happen.

## The patterns that decide your query shapes

**The unit of design is a "view," not a data point.** A view is a region of the screen that fills together — the line chart is one view, the breakdown panel is another. Each view is backed by exactly one endpoint that returns a complete, ready-to-render collection. So your endpoint count ≈ your view count, and each returns an array.

**A chart eats an array of objects, one object per mark.** Every charting library takes `data={array}` and maps each element to a line vertex or a bar. That's *why* `MonthlyPoint[]` is the right shape: each element is one point on the line. The query's job is to produce that array, pre-aggregated, at the granularity the visual draws. You never ship 997 raw rows to sum in the browser. This gives you a direct visual→shape dictionary: line/area → `[{x, y…}]`; bars → `[{category, value}]`; table → `[{col…}]`; single KPI → one object; dropdown → `[{id, label}]`.

**The frontend holds the selections; the server stays stateless.** "Which month is selected" lives in frontend state. When a selection changes, the frontend fires one fetch with that selection as a parameter and re-renders. The server just answers parameterized questions — it has no idea what "selected" means.

**Include-in-parent vs fetch-on-demand — this resolves your exact example.** You asked: clicking a month to show its inflow/outflow/net — how does that talk to the server? Trick answer: it often *doesn't*. Those three numbers are already inside the `MonthlyPoint` the user clicked, so showing them needs **no call** — the frontend already has the data in hand. A call is only needed for genuinely *new* data: the breakdown **by source**, which the monthly payload doesn't contain. That's the whole design lever — fold small always-wanted detail into the parent payload (instant, call-free drill); leave larger or sometimes-wanted detail out and fetch it lazily on the click.

**Fetch lazily on interaction (avoid both extremes).** Too chatty = many tiny calls (the 12-call mistake, or fetch-a-list-then-one-call-per-item). Too chunky = one giant upfront payload with all levels, most of which the user never opens. The sweet spot, which your drill-down naturally is: load the chart now; load the breakdown when a month is clicked; load lineage when a source is clicked.

**Three render states per fetch: loading, data, empty/error.** This lightly shapes the API. An empty-but-valid result is `200 []` (the UI shows "no data"); a nonexistent resource is `404`. That's the deliberate split in your code: `/monthly` 404s for an unknown orchestration, but `/sources` returns `[]` for a month with no streams.

**Cache by parameters.** Data-fetching libraries cache each response under a key built from the endpoint + its params, so re-clicking March serves from cache instead of re-calling. This is a quiet reason your fixed-query-plus-runtime-values design matters: stable parameterized endpoints make clean cache keys.

**Parallel for independent data, waterfall only when dependent.** The client picker and revision picker don't depend on each other → fetch together. Level 1 genuinely can't load before the user picks a month → that's a fine, user-driven waterfall. Avoid *accidentally* serializing calls that could run in parallel.

Now watch all of it run. Let me build the page.I spotted a stray typo in the CSS that I need to fix before this goes out.Let me validate the JavaScript parses cleanly and the data is internally consistent.JS parses fine; my consistency check just had a scoping issue with `eval`. Let me fix that and re-run the data validation.Validated: the JavaScript parses cleanly and the demo's data is internally consistent — every lineage allocation sums back to its stream value, and each month's source totals equal that month's net. Here's the page.Open it in any browser — it's one self-contained file, no server needed. Watch the **wire log** on the right as you interact; it narrates every pattern from the explanation as it happens:

On load, you'll see one `CALL` to `/orchestrations`, then one `CALL` to `/monthly` that returns the whole series, then a `RENDER` noting the chart drew all points from that single array — proving "one view, one call, not one-per-point." Click a point on the line and you'll see two distinct entries: a `NO CALL` (the month's inflow/outflow/net came straight from the clicked point) immediately followed by exactly one `CALL` to `/sources` for the genuinely new breakdown data. Click a bar and one `CALL` to `/lineage` fills the table with the many-to-many fan-out. Change the revision dropdown and you'll see a `STATE` change trigger a fresh `/monthly` call — selection lives in the frontend.

The thing that makes this directly useful to you: the `api` object in the code mirrors your FastAPI endpoints one-to-one, and each method has a comment showing the real `fetch(...)` line it would become. The mock data is shaped exactly like what your real `queries.py` returns. So to point this at your actual backend, you'd replace each method body with its `fetch` call and change nothing else — the renderers already consume the real array shapes.

Read the renderers (`renderChart`, `renderSources`, `renderLineage`) alongside the models in your `schema.py` and you'll see the loop close: each renderer takes one array, and the array's fields are exactly the model's fields. That's the whole relationship you were missing — the visual decides the array, the array is the model, the model is the query.

When you want to take the real step, the natural next move is wiring one actual React (or plain-fetch) page to your running FastAPI, starting with the chart against `/monthly`. Want me to build that against the real endpoints next?


# Deeper Machanics of Rendering and Communication

Now we're at the real mechanics — and the reason this feels hard is that the browser works on a fundamentally different model than Python or Power BI. Let me build that model up, because once it's in place, "where do clicks go," "call vs no-call," and "why React" all answer themselves.

## The core shift: the screen is a live tree, and your data lives somewhere else

In Python you call `plt.plot(df)` and get back a finished image — you hand over the whole frame, the library loops it internally, done. In Power BI a visual is *bound* to a dataset, and the engine silently redraws it whenever a filter changes. Both hide the loop and the redraw from you.

The browser hides nothing. Two separate things exist:

The **DOM** is the page as a live tree of element objects (`<svg>`, `<circle>`, `<div>`…). The browser paints this tree to pixels and keeps it on screen. It's persistent and mutable — you can change a node at any moment and the browser repaints.

Your **data** lives separately, in ordinary JavaScript variables in memory (in the demo, `state.monthly`). 

The crucial fact: **the DOM is a *projection* of your data, not the same thing as your data.** Changing the array in memory does **not** change the screen by itself. Something has to *re-project* the array into DOM elements. That projection is a function you write (or a framework writes): `render(data) → DOM`. This one idea is the whole answer to "how do the values get allocated to the right places" — they don't, automatically; a render function reads the array and places each element.

```
JS memory (source of truth)          the DOM (a projection)            screen
  state.monthly = [ {…}, {…}, {…} ] ──render(data)──► <svg><circle/>…  ──paint──► chart
        ▲                                                                            │
        │   setState(newData) / re-render   ◄──── click handler ◄──── user clicks ───┘
        └────────────── re-run render with new data → DOM rewritten → screen updates
```

Data flows one way into the DOM; user events flow back out; a change re-runs the projection. "Dynamic" is nothing more than *re-running render when the data changes.*

## "Each point has its own coordinates" — that's the render loop, and it uses *all* the data at once

Let me correct one thing directly, because it's the source of the confusion. The points are **not** independently fetched or independently cached. The whole array is in one variable, and the chart's render function loops over *all* of it in one pass, computing each point's coordinates from a **scale function** that maps data-space to pixel-space. From your demo's `renderChart`:

```js
const y = v => H-pad - (v-lo)/((hi-lo)||1)*(H-2*pad);   // a scale: a net value → a pixel y
const dots = data.map((d,i) => `<circle cx="${xs[i]}" cy="${y(d.net)}" .../>`).join("");
```

`xs[i]` and `y(d.net)` are exactly what Power BI's axes do internally — turn a value into a position. The reason each circle "has its own coordinates" is just that the loop computes a position *per datum*; they're all produced together from the single array, in one render. Nothing is independent or piecemeal — it's one array, one loop, one projection.

## Where you enable a user action: attach a listener to the rendered element

Interactivity is added *after* an element is rendered, by attaching an **event listener** to that specific element. The browser then calls your function whenever the user clicks that node. From the demo:

```js
el("chart").querySelectorAll(".pt").forEach(c => {
  const p = data[+c.dataset.i];                 // the datum THIS circle represents
  c.addEventListener("click", () => selectMonth(p));   // ← action enabled here
});
```

The key is the closure: each circle's handler captures `p`, the specific point it stands for. So "where do I enable clicking a monthly point?" → on each rendered point, with a handler that remembers which point it is. That handler is your doorway from "the user did something" back into your code.

## Where you decide call-vs-no-call: *inside* the handler

The handler is just plain code, and the branch you asked about lives there. Some handlers render from data already in memory; some fetch then render; some do both. Your `selectMonth` does both, on purpose:

```js
async function selectMonth(p){
  renderMonthDetail(p);                          // NO CALL — p already holds inflow/outflow/net
  const rows = await api.getSources(orch, p.month);  // CALL — the breakdown is new data
  renderSources(rows);                           // render the fetched array
}
```

The decision rule is exactly what we covered last turn, now located precisely: **inside the handler, ask "is the data I need already in a variable I hold?" If yes → render it (no call). If no → fetch, then render.** That's the entire logic of an interactive view, and it's ordinary `if`/sequence code — nothing framework-specific.

## So where does "caching" come in?

Two different things, worth separating. The array sitting in `state.monthly` is **in-memory state** — that's why clicking around the points you already have is instant and call-free; the data is right there in a variable. That's distinct from **HTTP caching** (last turn's topic), which is about not re-calling the *server* for a request you already made. Here, the in-memory array is the source of truth, and the DOM is a disposable projection you regenerate from it whenever something changes.

## How React fits — and why HTML "already did everything"

You're right that plain HTML/JS did everything. It always can — React ultimately compiles down to the same DOM operations (`createElement`, listeners, etc.). React isn't more *powerful*; it's more *maintainable*, and it brings back the thing you miss from Power BI: **automatic re-projection.**

Look at what you had to do manually in the vanilla demo: build DOM strings, inject `innerHTML`, re-attach listeners every render, and remember to re-run the right render when the right piece of state changes. For three views that's manageable; for a real product (dozens of views, forms, shared selections) that manual bookkeeping is where bugs live — stale handlers, a panel you forgot to update, rebuilding the whole tree when one number changed.

React replaces "imperatively update the DOM" with "**declare the UI as a function of state**." You write what the UI should look like for any given state; when state changes, React re-runs your function and **patches only the DOM that actually differs**. Same data→projection model — but you stop writing the projection plumbing by hand. Here's your `selectMonth` flow in React:

```jsx
function Dashboard({ orch }) {
  const [monthly, setMonthly]   = useState([]);
  const [selected, setSelected] = useState(null);   // the clicked point — in memory
  const [sources, setSources]   = useState(null);

  useEffect(() => { api.getMonthly(orch).then(setMonthly); }, [orch]);  // one call → chart

  async function onPointClick(p) {
    setSelected(p);                                  // NO CALL — re-renders the trio from memory
    setSources(await api.getSources(orch, p.month)); // CALL — re-renders the bars
  }

  return (
    <>
      <LineChart data={monthly} onPointClick={onPointClick} />
      {selected && <MonthTrio point={selected} />}   {/* from memory, no call */}
      {sources  && <SourceBars data={sources} />}    {/* from the fetched array */}
    </>
  );
}

function LineChart({ data, onPointClick }) {
  return (
    <svg>
      {data.map((p, i) => (
        <circle key={p.month} cx={x(i)} cy={y(p.net)} onClick={() => onPointClick(p)} />
      ))}
    </svg>
  );
}
```

Notice what changed and what didn't. The **decision logic is identical** — `onPointClick` still renders the trio from memory (no call) and fetches sources (call); that's your app, not the framework. What disappeared is the plumbing: `data.map` *is* the render loop, `onClick` replaces `addEventListener`, and `setSelected`/`setSources` replace every manual `innerHTML` rewrite — calling a setter changes state, and React automatically re-runs the projection and updates the screen. That's the Power BI feeling — "data changed, view updated" — restored, plus reusable components (`<LineChart>`, `<MonthTrio>`) you can compose.

So the honest summary: for a one-page demo, vanilla is fine and clearer for learning. React earns its place the moment you have real interactivity and shared state, because it removes the manual data↔DOM synchronization that becomes the bug factory — it is the declarative binding layer the raw browser lacks.

If you'd like to *feel* the difference, I can rebuild that same demo as a single runnable React file (CDN-loaded, no build step) so you can put the two side by side — same behavior, and you'll see exactly which lines vanish. Want me to?



# Appendix code

## `demo.HTML`

```HTML
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Serving layer ↔ frontend: how data shapes drive visuals</title>
<!--
  WHAT THIS IS
  A learning instrument, not the real product. It runs entirely in this one
  file (open it in any browser, no server needed). The `api` object below
  IMITATES your FastAPI endpoints: each method returns the exact array shape
  your real queries.py returns, after a small fake delay so you can see the
  loading state. Everywhere a real fetch would go, a comment shows the real URL.

  WHAT TO WATCH (right-hand "wire log")
    * one VIEW = one CALL returning a whole ARRAY (the line chart is 1 call, not 12)
    * clicking a month to see inflow/outflow/net makes NO call (already in hand)
    * clicking a month to see the SOURCE breakdown makes exactly ONE call
    * loading / empty states
    * changing the revision re-fetches the chart (selection lives in the frontend)
-->
<style>
  :root{
    --ink:#1b2330; --ink-soft:#5b6472; --line:#e3e5ea; --bg:#f4f5f8;
    --card:#ffffff; --accent:#3b4cca;          /* API / calls */
    --in:#1f9e6e; --out:#c2502f; --net:#2a3550; /* inflow / outflow / net */
    --tag-call:#3b4cca; --tag-nocall:#6b7280; --tag-render:#1f9e6e; --tag-state:#9a6b1f;
  }
  *{box-sizing:border-box}
  body{
    margin:0; background:var(--bg); color:var(--ink);
    font-family:system-ui,-apple-system,Segoe UI,Roboto,sans-serif; line-height:1.5;
  }
  .mono{font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace}
  header{padding:22px 28px 6px}
  header h1{margin:0;font-size:20px;letter-spacing:-.01em}
  header p{margin:4px 0 0;color:var(--ink-soft);font-size:13px;max-width:60ch}
  .wrap{display:grid;grid-template-columns:1fr 380px;gap:18px;padding:18px 28px 40px}
  @media(max-width:880px){.wrap{grid-template-columns:1fr}}
  .card{background:var(--card);border:1px solid var(--line);border-radius:12px;padding:16px 18px;margin-bottom:16px}
  .card h2{margin:0 0 2px;font-size:14px;letter-spacing:.01em}
  .endpoint{font-size:11.5px;color:var(--accent);margin:0 0 12px}
  .endpoint b{color:var(--ink-soft);font-weight:600}
  label{font-size:12px;color:var(--ink-soft)}
  select{font:inherit;padding:6px 8px;border:1px solid var(--line);border-radius:8px;background:#fff;margin-left:8px}
  .hint{font-size:11.5px;color:var(--ink-soft);margin:10px 0 0}
  .placeholder{color:var(--ink-soft);font-size:13px;font-style:italic;padding:8px 0}
  /* clickable marks */
  .pt{cursor:pointer}
  .pt:focus-visible{outline:2px solid var(--accent);outline-offset:2px}
  .bar{cursor:pointer}
  .bar:focus-visible{outline:2px solid var(--accent);outline-offset:2px}
  .barRow{display:grid;grid-template-columns:88px 1fr 92px;align-items:center;gap:10px;margin:7px 0}
  .barRow .name{font-size:12.5px}
  .barTrack{height:22px;background:#f0f1f4;border-radius:5px;position:relative;overflow:hidden}
  .barFill{height:100%;border-radius:5px}
  .barRow .amt{text-align:right;font-size:12.5px}
  /* the clicked-month trio: shown with NO call */
  .trio{display:flex;gap:10px;margin:4px 0 2px;flex-wrap:wrap}
  .stat{flex:1;min-width:96px;border:1px solid var(--line);border-radius:9px;padding:8px 10px}
  .stat .k{font-size:10.5px;color:var(--ink-soft);text-transform:uppercase;letter-spacing:.04em}
  .stat .v{font-size:16px;font-weight:600;margin-top:2px}
  .nocall{display:inline-block;font-size:10.5px;color:var(--tag-nocall);border:1px dashed #c7cad1;border-radius:999px;padding:1px 8px;margin-top:8px}
  table{width:100%;border-collapse:collapse;font-size:12px}
  th,td{text-align:left;padding:6px 8px;border-bottom:1px solid var(--line)}
  th{color:var(--ink-soft);font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:.03em}
  td.num{text-align:right;font-variant-numeric:tabular-nums}
  /* the signature element: the wire log */
  .console{background:#11151d;border-radius:12px;padding:14px 14px 16px;position:sticky;top:18px;height:calc(100vh - 60px);display:flex;flex-direction:column}
  .console h2{color:#cfd3da;font-size:12px;letter-spacing:.06em;text-transform:uppercase;margin:0 0 10px}
  #log{overflow:auto;flex:1;font-size:12px}
  .row{display:flex;gap:8px;padding:3px 0;border-bottom:1px solid #1c222d}
  .row .t{color:#5d6675;min-width:34px}
  .tag{min-width:64px;font-weight:700;font-size:10.5px;align-self:center}
  .tag.call{color:#8a98ff}
  .tag.nocall{color:#9aa3b0}
  .tag.render{color:#5fd6a6}
  .tag.state{color:#e2b259}
  .msg{color:#cdd2db}
  .msg em{color:#8a98ff;font-style:normal}
  .legend{font-size:11px;color:#7b8492;margin-top:10px;line-height:1.7}
  .legend b{color:#cdd2db;font-weight:600}
</style>
</head>
<body>
<header>
  <h1>How data shapes drive the visuals</h1>
  <p>The left is the screen a user sees. The right is the wire — every call (and every
     deliberate non-call) is logged so you can see the patterns happen.</p>
</header>

<div class="wrap">
  <div>
    <!-- TOP OF THE CHAIN: a picker. /orchestrations returns an array too. -->
    <div class="card">
      <h2>Revision</h2>
      <p class="endpoint mono">GET /orchestrations <b>→ Orchestration[]</b></p>
      <label for="orch">Showing</label>
      <select id="orch"></select>
      <p class="hint">Changing this is a frontend state change → it re-fetches the chart below (one call).</p>
    </div>

    <!-- VIEW 1: the line chart. ONE call returns the whole series. -->
    <div class="card">
      <h2>Monthly net cash flow — the chart</h2>
      <p class="endpoint mono">GET /forecast/{orch}/monthly <b>→ MonthlyPoint[]</b> &nbsp;(1 call · whole series)</p>
      <div id="chart"><div class="placeholder">loading…</div></div>
      <p class="hint">Each dot is one element of the array. Click a dot to drill in.</p>
    </div>

    <!-- VIEW 2: clicking a month. Two different mechanisms on purpose. -->
    <div class="card">
      <h2>Selected month</h2>
      <p class="endpoint mono">inflow / outflow / net come from the clicked point — <b>no call</b></p>
      <div id="monthDetail"><div class="placeholder">Click a point on the chart above.</div></div>
      <div id="sourcesBlock" style="display:none;margin-top:14px">
        <p class="endpoint mono">GET /forecast/{orch}/sources?month= <b>→ SourceAmount[]</b> &nbsp;(1 call · all sources)</p>
        <div id="sources"></div>
      </div>
    </div>

    <!-- VIEW 3: clicking a source → lineage. -->
    <div class="card" id="lineageCard" style="display:none">
      <h2>Lineage</h2>
      <p class="endpoint mono">GET /forecast/{orch}/lineage?month=&source= <b>→ LineageRow[]</b> &nbsp;(1 call · all rows)</p>
      <div id="lineage"></div>
    </div>
  </div>

  <!-- THE SIGNATURE ELEMENT: the wire log -->
  <div class="console">
    <h2>Wire log</h2>
    <div id="log"></div>
    <div class="legend">
      <b>CALL</b> a round trip to the API &nbsp; <b>NO&nbsp;CALL</b> data already in hand<br>
      <b>STATE</b> a frontend selection changed &nbsp; <b>RENDER</b> a view drew an array
    </div>
  </div>
</div>

<script>
/* ============================================================================
   1. MOCK DATABASE — the same fixture data your loader put in SQLite.
      In the real app this lives in the store and queries.py shapes it.
   ========================================================================== */
const DB = {
  orchestrations: [
    {orchestration_key:"c0001_2026_3-pre-harvest", revision_name:"3-pre-harvest", client_name:"North Field Co"},
    {orchestration_key:"c0002_2026_1-baseline",    revision_name:"1-baseline",    client_name:"Prairie Acres"},
  ],
  // level 0: one object per month  (this is MonthlyPoint[])
  monthly: {
    "c0001_2026_3-pre-harvest":[
      {month:"2026-01",month_num:1,inflow:120000,outflow:-30000,net:90000},
      {month:"2026-02",month_num:2,inflow:0,     outflow:-25000,net:-25000},
      {month:"2026-03",month_num:3,inflow:280000,outflow:-15000,net:265000},
    ],
    "c0002_2026_1-baseline":[
      {month:"2026-01",month_num:1,inflow:60000,outflow:0,     net:60000},
      {month:"2026-02",month_num:2,inflow:0,    outflow:-12000,net:-12000},
    ],
  },
  // level 1: keyed by "orch|month"  (each value is SourceAmount[])
  sources: {
    "c0001_2026_3-pre-harvest|2026-01":[{source:"fertilizer",amount:-30000},{source:"revenue",amount:120000}],
    "c0001_2026_3-pre-harvest|2026-02":[{source:"fertilizer",amount:-25000}],
    "c0001_2026_3-pre-harvest|2026-03":[{source:"seed",amount:-15000},{source:"revenue",amount:280000}],
    "c0002_2026_1-baseline|2026-01":[{source:"revenue",amount:60000}],
    "c0002_2026_1-baseline|2026-02":[{source:"chemical",amount:-12000}],
  },
  // level 2: keyed by "orch|month|source"  (each value is LineageRow[])
  lineage: {
    "c0001_2026_3-pre-harvest|2026-01|fertilizer":[
      {output_name:"Input Costs",crop_sys:"wheat", input_output_key:"iok_001",revision_name:"3-pre-harvest",stream_value:-30000,perc:0.5,allocated_value:-15000},
      {output_name:"Input Costs",crop_sys:"canola",input_output_key:"iok_002",revision_name:"3-pre-harvest",stream_value:-30000,perc:0.5,allocated_value:-15000},
    ],
    "c0001_2026_3-pre-harvest|2026-01|revenue":[
      {output_name:"Wheat Revenue",crop_sys:"wheat",input_output_key:"iok_001",revision_name:"3-pre-harvest",stream_value:120000,perc:0.6,allocated_value:72000},
      {output_name:"Wheat Revenue",crop_sys:"wheat",input_output_key:"iok_003",revision_name:"3-pre-harvest",stream_value:120000,perc:0.4,allocated_value:48000},
    ],
    "c0001_2026_3-pre-harvest|2026-03|revenue":[
      {output_name:"Wheat Revenue", crop_sys:"wheat", input_output_key:"iok_001",revision_name:"3-pre-harvest",stream_value:200000,perc:0.6,allocated_value:120000},
      {output_name:"Wheat Revenue", crop_sys:"wheat", input_output_key:"iok_003",revision_name:"3-pre-harvest",stream_value:200000,perc:0.4,allocated_value:80000},
      {output_name:"Canola Revenue",crop_sys:"canola",input_output_key:"iok_002",revision_name:"3-pre-harvest",stream_value:80000, perc:1.0,allocated_value:80000},
    ],
    "c0001_2026_3-pre-harvest|2026-03|seed":[
      {output_name:"Input Costs",crop_sys:"wheat", input_output_key:"iok_001",revision_name:"3-pre-harvest",stream_value:-15000,perc:0.5,allocated_value:-7500},
      {output_name:"Input Costs",crop_sys:"canola",input_output_key:"iok_002",revision_name:"3-pre-harvest",stream_value:-15000,perc:0.5,allocated_value:-7500},
    ],
    "c0001_2026_3-pre-harvest|2026-02|fertilizer":[
      {output_name:"Input Costs",crop_sys:"wheat", input_output_key:"iok_001",revision_name:"3-pre-harvest",stream_value:-25000,perc:0.5,allocated_value:-12500},
      {output_name:"Input Costs",crop_sys:"canola",input_output_key:"iok_002",revision_name:"3-pre-harvest",stream_value:-25000,perc:0.5,allocated_value:-12500},
    ],
    "c0002_2026_1-baseline|2026-01|revenue":[
      {output_name:"Canola Revenue",crop_sys:"canola",input_output_key:"iok_002",revision_name:"1-baseline",stream_value:60000,perc:1.0,allocated_value:60000},
    ],
    "c0002_2026_1-baseline|2026-02|chemical":[
      {output_name:"Input Costs",crop_sys:"wheat", input_output_key:"iok_001",revision_name:"1-baseline",stream_value:-12000,perc:0.5,allocated_value:-6000},
      {output_name:"Input Costs",crop_sys:"canola",input_output_key:"iok_002",revision_name:"1-baseline",stream_value:-12000,perc:0.5,allocated_value:-6000},
    ],
  },
};

/* ============================================================================
   2. THE API CLIENT — imitates your FastAPI endpoints.
      Each method = one endpoint = one round trip = returns one whole array.
      Replace the body of each with a real `fetch(...)` and nothing else changes.
   ========================================================================== */
const wait = (ms=320)=>new Promise(r=>setTimeout(r,ms));

const api = {
  async getOrchestrations(){
    log("call", `GET /orchestrations …`);
    await wait(180);
    // REAL APP: return fetch('/orchestrations').then(r=>r.json())
    const rows = DB.orchestrations;
    log("call", `← /orchestrations returned <em>${rows.length}</em> rows (1 call)`);
    return rows;
  },
  async getMonthly(orch){
    log("call", `GET /forecast/${short(orch)}/monthly …`);
    await wait();
    // REAL APP: return fetch(`/forecast/${orch}/monthly`).then(r=>r.json())
    const rows = DB.monthly[orch] ?? [];
    log("call", `← /monthly returned <em>${rows.length}</em> points in <em>one</em> call`);
    return rows;
  },
  async getSources(orch, month){
    log("call", `GET /forecast/${short(orch)}/sources?month=${month} …`);
    await wait();
    // REAL APP: return fetch(`/forecast/${orch}/sources?month=${month}`).then(r=>r.json())
    const rows = DB.sources[`${orch}|${month}`] ?? [];
    log("call", `← /sources returned <em>${rows.length}</em> rows in one call`);
    return rows;
  },
  async getLineage(orch, month, source){
    log("call", `GET /forecast/${short(orch)}/lineage?month=${month}&source=${source} …`);
    await wait();
    // REAL APP: return fetch(`/forecast/${orch}/lineage?month=${month}&source=${source}`).then(r=>r.json())
    const rows = DB.lineage[`${orch}|${month}|${source}`] ?? [];
    log("call", `← /lineage returned <em>${rows.length}</em> rows in one call`);
    return rows;
  },
};

/* ============================================================================
   3. FRONTEND STATE — the selections the server knows nothing about.
   ========================================================================== */
const state = { orch:null, monthly:[], month:null, source:null };

/* ============================================================================
   4. EVENT HANDLERS — each user action: change state, then fetch what's new.
   ========================================================================== */
async function selectOrch(orch){
  state.orch = orch; state.month = null; state.source = null;
  log("state", `selectedOrch = ${short(orch)}  (reset month & source)`);
  hide("sourcesBlock"); hide("lineageCard");
  el("monthDetail").innerHTML = `<div class="placeholder">Click a point on the chart above.</div>`;
  el("chart").innerHTML = `<div class="placeholder">loading…</div>`;   // loading state
  state.monthly = await api.getMonthly(orch);                          // ONE call fills the chart
  renderChart(state.monthly);
}

async function selectMonth(point){
  state.month = point.month; state.source = null;
  log("state", `selectedMonth = ${point.month}`);

  // (a) inflow / outflow / net: ALREADY in the clicked point → NO call.
  renderMonthDetail(point);
  log("nocall", `showed ${point.month} inflow/outflow/net — data was already in MonthlyPoint`);

  // (b) the source breakdown is NEW data → exactly ONE call.
  show("sourcesBlock"); hide("lineageCard");
  el("sources").innerHTML = `<div class="placeholder">loading…</div>`;  // loading state
  const rows = await api.getSources(state.orch, point.month);
  renderSources(rows);
}

async function selectSource(source){
  state.source = source;
  log("state", `selectedSource = ${source}`);
  show("lineageCard");
  el("lineage").innerHTML = `<div class="placeholder">loading…</div>`;  // loading state
  const rows = await api.getLineage(state.orch, state.month, source);
  renderLineage(rows);
}

/* ============================================================================
   5. RENDERERS — each turns ONE array into ONE view.
   ========================================================================== */
function renderChart(data){
  if(!data.length){ el("chart").innerHTML = `<div class="placeholder">No data for this revision.</div>`; return; }
  const W=560,H=240,pad=34;
  const xs = data.map((_,i)=> data.length===1 ? W/2 : pad + i*(W-2*pad)/(data.length-1));
  const vals = data.map(d=>d.net);
  const lo = Math.min(0,...vals), hi = Math.max(0,...vals);
  const y = v => H-pad - (v-lo)/((hi-lo)||1)*(H-2*pad);
  const zero = y(0);
  const line = data.map((d,i)=>`${xs[i]},${y(d.net)}`).join(" ");
  const dots = data.map((d,i)=>`
     <circle class="pt" tabindex="0" role="button"
        aria-label="${d.month}, net ${money(d.net)}"
        data-i="${i}" cx="${xs[i]}" cy="${y(d.net)}" r="6"
        fill="${d.net>=0?'var(--in)':'var(--out)'}" stroke="#fff" stroke-width="2"></circle>
     <text x="${xs[i]}" y="${H-12}" text-anchor="middle" font-size="11" fill="var(--ink-soft)">${d.month.slice(5)}</text>
  `).join("");
  el("chart").innerHTML = `
    <svg viewBox="0 0 ${W} ${H}" width="100%" role="img" aria-label="Monthly net cash flow line chart">
      <line x1="${pad}" y1="${zero}" x2="${W-pad}" y2="${zero}" stroke="var(--line)" stroke-dasharray="4 4"/>
      <polyline points="${line}" fill="none" stroke="var(--net)" stroke-width="2"/>
      ${dots}
    </svg>`;
  // wire up the clicks (each dot is one array element)
  el("chart").querySelectorAll(".pt").forEach(c=>{
    const p = data[+c.dataset.i];
    c.addEventListener("click", ()=>selectMonth(p));
    c.addEventListener("keydown", e=>{ if(e.key==="Enter"||e.key===" "){e.preventDefault();selectMonth(p);} });
  });
  log("render", `chart drew <em>${data.length}</em> points from the array (no per-point calls)`);
}

function renderMonthDetail(p){
  el("monthDetail").innerHTML = `
    <div class="trio">
      <div class="stat"><div class="k">Inflow</div><div class="v" style="color:var(--in)">${money(p.inflow)}</div></div>
      <div class="stat"><div class="k">Outflow</div><div class="v" style="color:var(--out)">${money(p.outflow)}</div></div>
      <div class="stat"><div class="k">Net</div><div class="v">${money(p.net)}</div></div>
    </div>
    <span class="nocall">these three came from the clicked point — no API call</span>`;
}

function renderSources(rows){
  if(!rows.length){ el("sources").innerHTML = `<div class="placeholder">No sources for this month.</div>`; return; }
  const max = Math.max(...rows.map(r=>Math.abs(r.amount)),1);
  el("sources").innerHTML = rows.map(r=>`
    <div class="barRow">
      <div class="name">${r.source}</div>
      <button class="bar barTrack" data-src="${r.source}" aria-label="${r.source}, ${money(r.amount)}">
        <div class="barFill" style="width:${Math.abs(r.amount)/max*100}%;background:${r.amount>=0?'var(--in)':'var(--out)'}"></div>
      </button>
      <div class="amt mono">${money(r.amount)}</div>
    </div>`).join("");
  el("sources").querySelectorAll(".bar").forEach(b=>
    b.addEventListener("click", ()=>selectSource(b.dataset.src)));
  log("render", `breakdown drew <em>${rows.length}</em> bars from the array`);
}

function renderLineage(rows){
  if(!rows.length){ el("lineage").innerHTML = `<div class="placeholder">No lineage rows.</div>`; return; }
  el("lineage").innerHTML = `
    <table>
      <thead><tr><th>Output</th><th>Crop</th><th>Input key</th><th>Revision</th>
        <th class="num">Stream</th><th class="num">%</th><th class="num">Allocated</th></tr></thead>
      <tbody>${rows.map(r=>`<tr>
        <td>${r.output_name}</td><td>${r.crop_sys}</td><td class="mono">${r.input_output_key}</td>
        <td>${r.revision_name}</td><td class="num mono">${money(r.stream_value)}</td>
        <td class="num mono">${(r.perc*100).toFixed(0)}%</td><td class="num mono">${money(r.allocated_value)}</td>
      </tr>`).join("")}</tbody>
    </table>`;
  log("render", `lineage drew <em>${rows.length}</em> rows (the many-to-many fan-out)`);
}

/* ============================================================================
   6. HELPERS + INIT
   ========================================================================== */
function el(id){return document.getElementById(id);}
function show(id){el(id).style.display="block";}
function hide(id){el(id).style.display="none";}
function short(orch){return orch.split("_").slice(-1)[0];}
function money(n){const s=n<0?"-":"";return s+"$"+Math.abs(n).toLocaleString();}

let clock=0;
function log(kind,msg){
  clock++;
  const labels={call:"CALL",nocall:"NO CALL",render:"RENDER",state:"STATE"};
  const row=document.createElement("div");
  row.className="row";
  row.innerHTML=`<span class="t mono">${String(clock).padStart(2,"0")}</span>
                 <span class="tag ${kind} mono">${labels[kind]}</span>
                 <span class="msg">${msg}</span>`;
  const box=el("log"); box.appendChild(row); box.scrollTop=box.scrollHeight;
}

async function init(){
  const orchs = await api.getOrchestrations();         // top of the chain: also an array
  const sel = el("orch");
  sel.innerHTML = orchs.map(o=>`<option value="${o.orchestration_key}">${o.client_name} — ${o.revision_name}</option>`).join("");
  sel.addEventListener("change", ()=>selectOrch(sel.value));
  await selectOrch(orchs[0].orchestration_key);        // load the first revision's chart
}
init();
</script>
</body>
</html>
```