# Frontend State & Async Request Patterns

Oct 19, 2026 · @zhe

## How to use this doc

Every pattern here exists to keep one promise: **the screen is always a correct picture of a small, explicit state model, and every change to that model is a named, guarded transition.** The patterns come from the GrowLytics Client list exercise (React + Vite, fake async service).

Each pattern has the same parts: the problem it solves, the mechanism, the code shape, and how it fails. Mechanisms covered elsewhere in their own notes (`<dialog>`, `<form>`, functions, `useEffect`) are only referenced here.

**Four layers** — every line belongs to one:

| Layer | Examples in this project |
| --- | --- |
| JavaScript language | closures, `async`/`await`, spread, `?.` / `??` |
| Browser / web API | `showModal()`, `preventDefault()`, `setTimeout`, popover |
| React | `useState`, `useRef`, `useEffect`, batching |
| My application abstraction | `workflow`, `runWrite`, `requestLock`, generations |

**Three tiers** — how deeply to know each pattern: (1) must reproduce mechanically, (2) understand purpose and failure modes, (3) look up when needed. Each section marks its core pieces.

## 1. State modeling

Model the UI as a few independent pieces of state, each with a closed set of values, and write down which combinations are legal. Anything not in the table is a bug.

**1.1 Workflow as a finite set of modes** (tier 1). One string answers "what is the user doing": `'view' | 'add' | 'edit' | 'delete'`. Dialogs open *because of* this value; they never hold their own truth.

**1.2 The invariants table** (tier 1). List every legal combination of related state:

| `workflow` | `draft` | `clientId` | `error` |
| --- | --- | --- | --- |
| `'view'` | `null` | `null` | `null` |
| `'add'` | `{ name: '' }` | `null` | any |
| `'edit'` | `{ name: <current> }` | record id | any |
| `'delete'` | `null` | record id | any |

Every exit must land on the `'view'` row. If an exit forgets one column (e.g. leaves `clientId` set), the table makes the bug visible before it ships.

**1.3 `null` as a meaningful value** (tier 1). `draft === null` means "no draft exists", not "forgot to initialise". Keep the meaning and make the render tolerate it: `draft?.name ?? ''`. Use `?.` only where `null` is legal; where it would mean a bug (e.g. `clients`), let it crash early.

**1.4 The draft holds only editable fields** (tier 2). Identity (`clientId`) lives outside the draft because the user cannot change it. Mirror the record's field names (`name`, later `memo`) so building and saving a draft is one-to-one.

**1.5 Derive, don't store** (tier 1). Anything computable from existing state is computed during render, never stored and synced:

```js
const selectedClient = clients.find(c => c.id === clientId)
const isPending      = request === 'pending'
const isSystemBusy   = request === 'pending' || isLoading || error !== null
const canStartRequest = request === 'idle' && !isSystemBusy
const canEnterAction  = workflow === 'view' && canStartRequest
```

A stored copy can drift from its source; a derived value cannot.

**1.6 Updates are immutable** (tier 1). Setters replace, never merge: `setDraft(prev => ({ ...prev, name }))`. Arrays get new arrays: add `[...prev, x]`, edit `prev.map(...)`, delete `prev.filter(...)`. React re-renders only on a new reference.

## 2. Controlled transitions

State only changes through named functions, each of which moves the model from one legal row of the invariants table to another.

**2.1 Named enter / exit / reset functions** (tier 1). One function per transition, setting every column of the target row:

```js
const enterEdit = (client) => {
  if (!canEnterAction) return          // entry guard
  setClientId(client.id)
  setDraft({ name: client.name })
  setWorkflow('edit')
}

const resetWorkflow = () => {          // back to the 'view' row
  setClientId(null)
  setDraft(null)
  setWorkflow('view')
  setError(null)
}
```

**2.2 Guards go on transitions that start something** (tier 1). `enterAdd`, `enterEdit`, `submitModify` check `canEnterAction` / `canStartRequest` first and return early. The guard reads the render snapshot, which is exactly the state the user saw when they clicked.

**2.3 One exit, many triggers** (tier 1). Esc, Cancel, a successful save and an abort all converge on `resetWorkflow`. Both exit functions for add and edit landed on the same row, so they became one.

**2.4 Exits must be idempotent and side-effect free** (tier 2). With dialogs an exit can run twice (direct call, then the dialog's `close` event). Setting identical values twice is harmless because React skips the re-render. Adding a side effect (e.g. `refresh()`) breaks that, so keep `resetWorkflow` a pure state reset and trigger side effects explicitly where they belong.

**2.5 A close event cannot be vetoed** (tier 2). `onClose` runs *after* the dialog closed. A guard there only stops state from following the screen, which desyncs them. Rule: guard the actions that *initiate* a close (disable Cancel, block Esc); always let close handlers bring state in line.

**2.6 Setters schedule, they don't assign** (tier 1). After `setClientId(5)`, the variable `clientId` in the same handler is still the old snapshot. All setters in one handler are batched into one render; only the final value is ever shown. Use the value you already have (`client.id`) or read new state in the next render.

## 3. Modal dialogs driven by state

React state decides whether a dialog is open; an effect makes the DOM follow. Full mechanism: the standalone `<dialog>` note.

**3.1 The sync effect** (tier 1). Every dialog component uses the same shape, with guards so it is safe to run in any order:

```js
useEffect(() => {
  const dialog = dialogRef.current
  if (open) { if (!dialog.open) dialog.showModal() }
  else      { if (dialog.open) dialog.close() }
}, [open])
```

**3.2 The parent translates state into props** (tier 2). App passes `open={workflow === 'edit'}` and `title=...`; the dialog never sees `workflow`. Dialogs stay reusable and App remains the only owner of the state machine.

**3.3 Always mounted** (tier 2). Render every dialog unconditionally. Conditional rendering works for opening, but unmounting skips the `close` event, and a loading ternary that swaps the tree tears down dialogs mid-display.

**3.4 Stacking** (tier 2). Each `showModal()` goes on top of the previous one; only the top dialog is interactive and Esc closes only the top one. An error dialog over an edit dialog needs no extra code.

**3.5 Blocking Escape while busy** (tier 2). `onCancel` + `preventDefault()` is not absolute: browsers let a second Esc through without user interaction in between. Block the key at the document instead, only while busy:

```js
useEffect(() => {
  if (!blockEsc) return
  const block = (e) => { if (e.key === 'Escape') e.preventDefault() }
  document.addEventListener('keydown', block, true)   // capture: sees it first
  return () => document.removeEventListener('keydown', block, true)
}, [blockEsc])
```

Document level matters because disabling the focused button drops focus to `<body>`, outside any dialog's own handler. Scope `blockEsc` to pending, loading or a non-dismissable error, not every error.

**3.6 Destructive confirmations** (tier 2). Cancel first with `autoFocus`, the destructive button labelled with the action ("Delete", not "Confirm"), so Enter never deletes by accident.

## 4. Request lifecycle and the write template

Every write goes through one function, `runWrite`, which owns the request state, the lock, the generation check and the success/failure routing. Callers only validate their own input and hand over the request.

&#91;embedded content: request lifecycle · 3 states\]

Only a write that succeeds *and* refreshes reaches `succeeded`; every failure and every abort goes straight back to `idle`, with `error` set when the user must act.

**4.1 Request as its own state** (tier 1). `request: 'idle' | 'pending' | 'succeeded'`, independent of `workflow`. Spinner open = `'pending'`; success dialog open = `'succeeded'`; its self-close returns to `'idle'`. Failure returns to `'idle'` and sets `error`.

**4.2 Pass a thunk, not a promise** (tier 1). `runWrite(() => fakeDeleteClient(id))` hands over a function that *would* start the request. `runWrite(fakeDeleteClient(id))` would start it immediately, before any guard could refuse.

**4.3 Callers validate, the template runs** (tier 2). Validation is a pure check that returns an error object or `null`; the caller sets the error and stops. `runWrite` stays generic and knows nothing about forms.

```js
const submitModify = async (event) => {
  event.preventDefault()                         // ALWAYS first (see 12)
  if (!canStartRequest || requestLock.current) return
  const validationError = validateClientName(draft.name)
  if (validationError) { setError(validationError); return }
  if (workflow === 'add')  await runWrite(() => fakeCreateClient(draft.name))
  if (workflow === 'edit') await runWrite(() => fakeEditClient(clientId, draft.name))
}
```

**4.4 The template** (tier 1 — this is the core of sections 5–8):

```js
const runWrite = async (action) => {
  const myGeneration = ++requestGeneration.current   // tag this request (6)
  setIsWaitingLong(false); setObservationRound(0)     // long-wait timer (7)
  setRequest('pending')
  try {
    requestLock.current = true                         // lock inside try (5)
    await action()
    if (myGeneration !== requestGeneration.current) {  // stale success (6)
      setNotice({ title: 'Previous request completed', message: '...' })
      return
    }
    const refreshed = await refresh()                  // server is truth (8)
    if (myGeneration !== requestGeneration.current) return   // re-check after every await
    if (refreshed) requestSucceed()                    // 'succeeded' + resetWorkflow
    else setRequest('idle')                            // refresh set its own error
  } catch (err) {
    if (myGeneration !== requestGeneration.current) return   // stale failure: silent
    requestFailed({ type: 'retryable', title: 'Request failed', message: err.message })
  } finally {
    if (myGeneration === requestGeneration.current) requestLock.current = false
  }
}
```

**4.5 Snapshot after await** (tier 2). Code after `await` runs seconds later but still sees the render snapshot from the click. That is correct for "which operation did I start"; if you need *current* state after an await, capture it before or read a ref.

## 5. Duplicate submission lock

A `useRef(false)` flag that is set the instant a write starts and cleared when it is finished, so a second click or Enter press cannot start a second request.

**5.1 Why a ref, not state** (tier 1). State changes only become visible on the *next* render. Two clicks inside the same render both read the old `request === 'idle'` snapshot and both pass the guard. A ref's `.current` changes immediately and every handler reads the same live box.

```js
const requestLock = useRef(false)
// guard in every write entry point:
if (!canStartRequest || requestLock.current) return
```

The state guard (`canStartRequest`) and the ref guard do different jobs: state drives what the UI shows and disables; the ref closes the gap between a click and the next render.

**5.2 Acquire inside `try`, release in `finally`** (tier 1). `finally` only protects code inside the `try`. If anything throws between taking the lock and entering the `try` (a typo was enough), the lock stays held forever and every later write silently returns at the guard.

**5.3 Release only if this request still owns it** (tier 2). After an abort a newer request may hold the lock. The late `finally` of the old request must not release it: `if (myGeneration === requestGeneration.current) requestLock.current = false`.

**5.4 Abort releases explicitly** (tier 2). Abort bumps the generation, so the old request's `finally` will no longer release; `abortRequest` therefore clears the lock itself.

**5.5 Same idea for reads** (tier 2). `refreshLock` prevents overlapping refreshes (e.g. StrictMode running the mount effect twice). Set it inside `try`, clear it in `finally`.

Failure mode to recognise: "after one operation, buttons do nothing and there is no error" almost always means a lock that was never released.

## 6. Stale and aborted results

A promise cannot be cancelled; it will settle eventually no matter what the user did meanwhile. Tag each request with a number and, when it settles, act only if it is still the latest.

**6.1 The generation counter** (tier 1). A ref holding an integer. Starting a request takes a ticket; abort invalidates all outstanding tickets by moving the counter on.

```js
const requestGeneration = useRef(0)

// start
const myGeneration = ++requestGeneration.current
// after it settles
if (myGeneration !== requestGeneration.current) { /* stale */ }
```

It is a ref for the same reason as the lock: the late callback must see the *live* counter, not the number captured in an old render.

**6.2 Re-check after every `await`** (tier 1). Each `await` is a boundary where the user can abort. A check after the write does not protect the code after the refresh; `runWrite` checks once per await.

**6.3 Stale success vs stale failure** (tier 2). Decide per outcome, not just "ignore":

| Outcome after abort | Effect on server data | Behaviour |
| --- | --- | --- |
| Late success | the write happened; the list on screen is now out of date | non-blocking notice: "refresh when convenient" |
| Late failure | nothing changed | silent `return` |

The stale branch touches no workflow, request or error state, so it can never re-enable or block anything the user is currently doing.

**6.4 The abort transition** (tier 2). One named function, never inline in JSX:

```js
const abortRequest = () => {
  requestGeneration.current++     // every outstanding result is now stale
  requestLock.current = false     // the old finally will no longer release it
  setRequest('idle')
  resetWorkflow()
  refresh()                       // the write may or may not have landed
}
```

**6.5 Abort is not cancel** (tier 2). Stopping waiting does not stop the server. Say so to the user before they click (helper text under the buttons), and report a late success after.

**6.6 Invariant to keep in mind** (tier 3). Today every stale result belongs to an aborted request because the lock allows only one write at a time. If concurrency rules ever change, "stale" may also mean "superseded", and the notice wording would need to change.

## 7. Long-wait observation

After 5 seconds pending, offer the user a choice instead of an endless spinner: keep waiting, or stop waiting and refresh.

**7.1 Two pieces of state** (tier 1). `isWaitingLong` (show the choice) and `observationRound` (a counter whose only job is to restart the timer).

**7.2 The timer effect** (tier 1). Setup starts a timeout while pending; cleanup cancels it the moment the request settles or a new round begins:

```js
useEffect(() => {
  if (request !== 'pending' || isLoading) return
  const timer = setTimeout(() => setIsWaitingLong(true), 5000)
  return () => clearTimeout(timer)
}, [request, observationRound, isLoading])
```

**7.3 Keep Waiting = start a new round** (tier 2). `setIsWaitingLong(false)` hides the choice and `setObservationRound(r => r + 1)` changes a dependency, so the effect cleans up and starts a fresh 5-second timer. A dependency used purely as a restart trigger is a legitimate effect pattern.

**7.4 Reset per request** (tier 2). `runWrite` sets `isWaitingLong` to `false` and the round to `0` before going pending, so a new request never inherits an old request's choice.

**7.5 Stop Waiting = abort** (tier 2). It calls `abortRequest` (section 6.4). Helper text under the buttons, always visible, states that stopping does not cancel the server operation.

## 8. Refresh after write

After a successful write, fetch the list from the server and replace local state with it. The server is the source of truth; local state mirrors it.

**8.1 Replace, don't patch** (tier 2). Patching local state per operation (`add` pushes, `delete` filters) duplicates server logic in the client and drifts as soon as anyone else changes data. A refresh needs no per-operation code and survives swapping the fake service for the real one.

**8.2 One trigger per path** (tier 1). Refresh exactly once on: mount (initial load), successful write (inside `runWrite`), abort. Not inside `resetWorkflow` and not on Cancel: a side effect inside an exit runs twice with the double close and stacks overlapping requests.

**8.3 Loading is its own state** (tier 2). `isLoading` set in `refresh`, cleared in `finally` so a failure can never leave it stuck. It feeds `isSystemBusy`, which blocks new actions.

```js
const refresh = async (initial = false) => {
  if (refreshLock.current) return
  setIsLoading(true)
  try {
    refreshLock.current = true
    setClients(await fakeGetClients())
    return true
  } catch {
    setError(initial ? loadFailedError : writeOkRefreshFailedError)
    return false
  } finally {
    setIsLoading(false)
    refreshLock.current = false
  }
}
```

**8.4 Refresh failure is a different error from write failure** (tier 2). After a successful write, a failed refresh means "saved, but the screen is stale". Its message must stop the user from resubmitting the same change. It reports through `error`, not `requestFailed`, and `runWrite` reads the returned boolean to decide between `requestSucceed()` and plain `'idle'`.

**8.5 Loading never swaps the tree** (tier 2). Vary only the list area (placeholder vs `ClientList`); keep every dialog mounted outside the loading condition.

**8.6 Out-of-order reads** (tier 3). Two overlapping refreshes can resolve in either order, and the last to resolve wins. The refresh lock prevents overlap; a read generation counter (same as 6.1) is the fallback if overlap becomes legitimate.

## 9. Error model

`error` is `null` or one fixed-shape object, `{ type, title, message }`. The type decides how the error can be left and what is preserved.

| Type | Example | Dismissable | On dismiss |
| --- | --- | --- | --- |
| `validation` | whitespace-only name | yes (Close, Esc) | clear the bad field, keep the dialog open |
| `retryable` | write rejected by the server | yes (Close, Esc) | keep the draft; pressing Save again is the retry |
| `non-retryable` | list cannot be loaded | no | only path forward is reloading the page |

**9.1 Data in state, behaviour in handlers** (tier 1). The error object holds no functions. One `dismissError` handler branches on `error?.type`; it uses `?.` because it may run a second time after `error` is cleared.

**9.2 An error blocks everything** (tier 1). `error !== null` is part of `isSystemBusy`, so no new action or request can start while one is showing. That is why non-error messages (the late-success notice) must not use this state.

**9.3 Retry for free** (tier 2). A retryable failure leaves `workflow` and `draft` untouched. The edit dialog is still open underneath, so after Close the user is back in the form with their input intact.

**9.4 Non-dismissable means non-closable** (tier 2). Removing `onClose` does not stop a dialog from closing; it only stops state from following. Hide the Close button, block Esc, and offer the recovery action (Reload) as the only control.

**9.5 Validation is a pure check** (tier 1). `validateClientName(name)` returns an error object or `null` and never touches state. The caller decides to `setError`. Mixing the two let an inverted `if` queue an error that a later `setError(null)` in the same batch silently erased.

## 10. Feedback surfaces

Each kind of message gets a surface whose blocking behaviour matches what the user must do about it.

| Surface | Driven by | Blocks the user? | Closes |
| --- | --- | --- | --- |
| Pending spinner | `request === 'pending'` / `isLoading` | yes (modal, Esc blocked) | when the state leaves pending |
| Success dialog | `request === 'succeeded'` | briefly (modal) | itself, after 1–2 s |
| Error dialog | `error !== null` | yes (modal) | by the user, per error type |
| Notice banner | `notice !== null` | no (popover) | itself, after 5 s |

**10.1 Composition with `children`** (tier 2). `PendingDialog` renders `{children}` in a neutral slot. App passes the Keep/Stop buttons and helper text in; the spinner itself knows nothing about them, and removing them is deleting lines in App.

**10.2 Self-closing timer with the latest callback** (tier 3). The success dialog and banner start a timer when opened and clear it on close. The callback is read through a ref so that App re-rendering during the countdown (e.g. a refresh landing) does not restart the timer:

```js
const onDoneRef = useRef(onDone)
useEffect(() => { onDoneRef.current = onDone })
useEffect(() => {
  if (!open) return
  const t = setTimeout(() => onDoneRef.current?.(), duration)
  return () => clearTimeout(t)
}, [open, duration])
```

**10.3 Success as a request state** (tier 2). `'succeeded'` sits between `'pending'` and `'idle'`, so the success dialog needs no extra boolean and no new request can start while it shows. Known gap: by the time it renders, `workflow` is already `'view'`, so it cannot say which operation succeeded unless the operation is stored with the request outcome.

**10.4 Non-blocking notices live in the top layer** (tier 2). A banner rendered normally sits behind an open modal's backdrop. `popover="manual"` + `showPopover()` puts it above modals without stealing focus or blocking anything. Its own `notice` state stays out of every guard. A fresh object per `setNotice` restarts the timer, because the effect compares references.

**10.5 Important information is not a tooltip** (tier 2). Hover never happens on touch and is skipped by fast clickers. Consequences of an action go in always-visible helper text.

## 11. Fake service for testing every branch

A module that behaves like a real API — private data, delayed responses, real failures — so every success, failure, hang and abort path can be exercised before a backend exists. Swapping in the real service later changes only this file.

**11.1 A module is a tiny server** (tier 2). Top-level variables (`db`, `nextId`, `callCount`) are created once at import and shared by every function. App cannot touch `db`; it only calls exported endpoints.

**11.2 One `simulate` helper** (tier 1). Delay, maybe fail, otherwise run the endpoint's work — after the delay:

```js
function simulate(work, errorMessage) {
  callCount++
  const shouldFail = false            // toggle by hand while testing
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (shouldFail) return reject(new Error(errorMessage))
      try { resolve(work()) } catch (err) { reject(err) }
    }, 1000)
  })
}

export function fakeCreateClient(name) {
  return simulate(() => {             // `return` passes the promise up
    const record = { id: nextId, name, modified_by: `Test User ${nextId}` }
    nextId++
    db = [...db, record]
    return { ...record }              // resolves with a copy
  }, 'Client cannot be created')
}
```

**11.3 Return the promise** (tier 1). The inner `return` fills the promise later; the outer `return simulate(...)` hands the promise to `runWrite` now. Missing the outer one makes `await action()` resolve instantly with `undefined` while the work happens a second later.

**11.4 Copies, not references** (tier 2). Resolve with `[...db]` and `{ ...record }`, and replace `db` immutably. App's state and the fake database must never be the same object, as with real JSON over a network.

**11.5 Build a record once** (tier 2). Store and return the same constructed object, so what the server keeps and what it reports cannot diverge.

**11.6 Branch test matrix** (tier 2). For each of add, edit, delete: write ok + refresh ok; write fails → retry; write ok + refresh fails; slow write → Keep Waiting; hang → Stop Waiting → late success notice. A shared `callCount` with strict alternation pairs every successful write with a failed refresh, so set `shouldFail` deliberately per scenario.

## 12. Pitfall index

Every bug hit in this project, the symptom it showed, and the pattern that prevents it. Detailed write-ups live in the mistake log.

| Symptom | Cause | Prevented by |
| --- | --- | --- |
| Page reloads, all state lost | submit handler returned before `preventDefault()`, or a button with no `type` inside a form | `preventDefault()` as line one; explicit `type` on every button |
| Buttons do nothing after one operation | lock taken outside `try`, or `finally` never released it | 5.2, 5.3 |
| Spinner reappears and never leaves | brace-less `if` swallowed the next block | 4.4; every guard ends in `return` |
| Success shown before the write happened | endpoint did not `return` its promise | 11.3 |
| `clients` became `undefined`, crash in ClientList | updater arrow with `{ }` and no `return` | expression-body arrows for updaters |
| Field silently dropped from state | setter replaced the object instead of merging | 1.6 |
| Old value logged right after setting it | setters schedule, they don't assign | 2.6 |
| Error queued but never shown | a later setter in the same batch overwrote it | 9.5 |
| Dialog closed but state still says open | close handler guarded or missing | 2.5 |
| Esc closes a "frozen" dialog on the second press | browser anti-trap rule; focus fell to `<body>` | 3.5 |
| App frozen with no visible error | loading ternary unmounted the error dialog; `isLoading` not cleared on failure | 8.3, 8.5 |
| Several refreshes per action | side effect inside an exit that runs twice | 2.4, 8.2 |
| Handler does nothing | wrapper returned a function instead of calling it | functions note |
| Payload nested twice (`{ id: { id } }`) | wrapped a result that was already an object | name by what you confirmed, not what you expect |
