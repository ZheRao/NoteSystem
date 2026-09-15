# Problem space
Yes. **This is exactly how I'd use CS 349 for you:** not “take another entire course,” but extract a **Strata UI engineering checklist** so you don't accidentally design only the happy path.

Waterloo's current CS 349 is particularly appropriate because it explicitly focuses on UI architecture, event-driven/asynchronous behavior, components, feedback, testing, and accessibility—not merely making interfaces pretty. ([Computer Science Computing Facility][1])

And for Strata v1, I think you can get **80–90% of what you need from perhaps 10–15 hours of targeted study**, then learn deeper material when the product forces you to.

### Strata UI reference sheet

I'd organize your thinking around **eight questions**.

| Area                                | What you need to understand for Strata               | V1 depth    |
| ----------------------------------- | ---------------------------------------------------- | ----------- |
| **1. UI state**                     | What states can every screen/action be in?           | 🔴 Deep     |
| **2. Feedback**                     | How does the user know what the system is doing/did? | 🔴 Deep     |
| **3. Forms & validation**           | How do users safely modify forecast inputs?          | 🔴 Deep     |
| **4. Async operations**             | What happens between request and response?           | 🔴 Deep     |
| **5. Error/recovery**               | What happens when something fails?                   | 🔴 Deep     |
| **6. Component/state architecture** | Where should React state live?                       | 🟠 Moderate |
| **7. Accessibility**                | Keyboard, labels, focus, semantic controls           | 🟠 Moderate |
| **8. Layout/visual design**         | Hierarchy, consistency, responsive layout            | 🟡 Basic    |

The first five are where I think your current mental model has the biggest missing piece.

---

## 1. Start thinking of UI as a **state machine**

This is probably the single most useful CS 349-ish concept for you.

Right now you might think:

```text
User edits input
      ↓
POST /inputs
      ↓
database updated
```

Backend Zhe says:

> Great. Correct. Transaction committed. 😎

UI Zhe needs to ask:

> **What does the human see during every stage of that process?**

Consider editing an input:

```text
             ┌───────────┐
             │   CLEAN   │
             └─────┬─────┘
                   │ edit
                   ↓
             ┌───────────┐
             │   DIRTY   │
             └─────┬─────┘
                   │ Save
                   ↓
             ┌───────────┐
             │  SAVING   │
             └─────┬─────┘
              success│ failure
             ┌───────┴───────┐
             ↓               ↓
          SAVED            ERROR
                             │
                           retry
```

But Strata may eventually have:

```text
LOADING
READY
DIRTY
VALIDATING
SAVING
SAVED
INVALID
CONFLICT
ERROR
STALE
DISABLED
EMPTY
```

That's an enormous conceptual upgrade.

Don't think:

> “I need a Save button.”

Think:

> **“What states can this interaction occupy, and what should the UI render for each one?”**

This maps nicely onto React's own recommended mental model: model UI changes as **state changes**, rather than issuing imperative commands like “show this message” or “disable that button.” ([React][2])

#### Strata milestone

For every important screen, you should be able to draw:

```text
states → events → transitions → visible feedback
```

If you can't, the interaction isn't fully designed yet.

---

## 2. Feedback: never make the user wonder **“did that work?”**

This may be the biggest practical improvement you can make to Strata v1.

Suppose Gerry changes:

```text
Canola seed cost
$71.00 → $74.50
```

and clicks **Save**.

Bad:

```text
[Save]
```

Nothing visibly happens.

Now Gerry wonders:

> Did I click it?
> Did it save?
> Should I click again?
> Is the forecast recalculating?

Good:

```text
[ Saving... ]

       ↓

✓ Saved
```

Failure:

```text
⚠ Couldn't save changes.

Your edits have not been lost.

[Retry]
```

Validation:

```text
Seed cost
[-20]

⚠ Cost must be zero or greater.
```

Conflict eventually:

```text
⚠ This configuration changed after you opened it.

Your version:       $74.50
Current version:    $76.00

[Review changes]
```

This is **not cosmetic polish**.

It's exposing system state to the human.

And I suspect you'll actually enjoy thinking about UI this way because it connects directly to your backend thinking:

```text
DATABASE STATE
      ↓
API STATE
      ↓
APPLICATION STATE
      ↓
UI STATE
      ↓
USER'S MENTAL MODEL
```

The UI's job is partly to keep that last model synchronized with reality.

---

## 3. Forms and validation

This is extremely important for Strata because much of the product is effectively **structured financial/agricultural input management**.

For every input ask:

**What constitutes valid data?**

And distinguish:

```text
client-side validation
        +
server-side validation
```

Client-side validation gives immediate feedback.

Server-side validation protects system correctness.

MDN explicitly recommends both; client-side validation alone isn't trustworthy. ([MDN Web Docs][3])

For example:

```text
Acres
[ -500 ]

Acres must be greater than zero.
```

But don't merely think about datatype constraints.

You have at least three levels:

```text
Syntactic
"Is this a number?"

Domain
"Can acres be negative?"

Business
"Is this input permitted in this forecast phase?"
```

That third category is particularly important for Strata.

#### Strata milestone

For each editable field you should eventually know:

```text
type
required/optional
allowed range
format
domain rules
business rules
error message
server validation
```

---

## 4. Async behavior

This deserves disproportionate attention because React + FastAPI means your interface is inherently asynchronous.

Imagine:

```text
React
  │
  │ POST /forecast/run
  ↓
FastAPI
  │
  │ computation
  ↓
database
  │
  ↓
response
  │
  ↓
React
```

Between request and response, **time exists**.

Your UI has to represent it.

Questions:

> Can the user click Run Forecast twice?

> Can they edit inputs while calculation is happening?

> What happens if they navigate away?

> What if the request takes 100 ms?

> 2 seconds?

> 30 seconds?

> What if the connection disappears?

> What if the server returns 500?

> What if the first request finishes *after* a newer request?

That last one is particularly interesting:

```text
request A ──────────────────────→ response A
request B ───────→ response B
```

If B represents newer state but A returns afterward...

**should response A overwrite B?**

Congratulations: frontend work has found another way to drag you back toward concurrency. 😂

Waterloo explicitly includes asynchronous events/event-driven architecture in CS 349's learning objectives, so this isn't an incidental frontend detail. ([Computer Science Computing Facility][1])

---

## 5. Design the unhappy paths

This is where I think Strata v1 needs deliberate attention.

For every API-driven interaction, ask:

```text
What if it succeeds?
What if it's slow?
What if it returns nothing?
What if validation fails?
What if authorization fails?
What if the server fails?
What if the network fails?
What if the underlying data changed?
```

You don't necessarily need sophisticated handling for all of these in v1.

But **you should know what happens**.

For example:

```text
GET /clients

loading:
    skeleton/spinner

success + clients:
    client list

success + []:
    "No clients yet"
    [Create client]

401/403:
    access message

500:
    "Couldn't load clients"
    [Retry]
```

Notice that **empty isn't an error**.

That's another state.

---

## 6. Component architecture + state ownership

Here I'd spend a couple of hours with the official [React Managing State guide](https://react.dev/learn/managing-state?utm_source=chatgpt.com).

You should understand:

**local state** — belongs to one component.

**shared state** — multiple components need it, so ownership moves upward.

**server state** — authoritative information came from your backend.

**derived state** — calculate it rather than duplicating it unnecessarily.

React specifically warns that redundant/duplicate state is a common source of bugs. ([React][2])

For Strata, I'd want you able to answer something like:

```text
Who owns selected_client_id?

App?
ClientSelector?
ForecastPage?

Who owns unsaved forecast edits?

Who owns loading state?

Who owns authenticated user?

Who owns the currently committed revision?
```

You don't need Redux architecture astronautics for v1.

You **do** need intentional state ownership.

---

## 7. Accessibility — learn the cheap 80%

Don't go down the entire WCAG rabbit hole right now.

For Strata v1, I'd establish several non-negotiables:

**Use semantic HTML.** A button should actually be `<button>`, not a clickable `<div>`.

**Every input gets a real label.**

**Keyboard navigation works.**

**Visible focus exists.**

**Errors identify the problematic field and explain what happened.**

**Don't communicate meaning solely through color.**

Forms are one of the trickier accessibility surfaces, and labels, keyboard order, focus indication and understandable errors are foundational practices. ([Web.dev][4])

A fantastic five-minute test is simply:

> **Put your mouse away and use Strata.**

Can you Tab through everything?

Can you tell what's selected?

Can you activate buttons?

Can you edit/save?

If not, fix that before worrying about sophisticated accessibility.

[web.dev — Form accessibility](https://web.dev/learn/forms/accessibility?authuser=2&utm_source=chatgpt.com)

---

## 8. Layout and visual hierarchy

I'd deliberately keep your study here shallow initially.

You don't need to become a UI designer.

Learn enough to reason about:

```text
visual hierarchy
spacing
alignment
grouping
consistency
information density
responsive behavior
```

For Strata especially, **information architecture beats prettiness**.

Users should immediately understand:

```text
CLIENT
   ↓
FORECAST YEAR
   ↓
INPUT SOURCE
   ↓
REVISION / PHASE
   ↓
DATA
```

If that conceptual hierarchy is obvious on screen, you're already accomplishing something important.

---

## Your Strata v1 UI Definition of Done

This is the part I'd actually save somewhere.

Before calling an important Strata workflow “done,” you should be able to answer:

1. **What states can this screen/action occupy?**
2. **What does the user see while data loads?**
3. **What does an empty result look like?**
4. **How does the user know they have unsaved changes?**
5. **What happens while Save is executing?**
6. **How does the user know Save succeeded?**
7. **What happens if Save fails? Are their edits preserved?**
8. **What validation occurs before submission, and what is enforced by the backend?**
9. **What happens if underlying data changed since the user loaded it?**
10. **Can an action accidentally execute twice?**
11. **Where does each piece of React state live, and why?**
12. **Can the workflow be completed with the keyboard?**
13. **Are inputs properly labeled and errors understandable?**
14. **Does navigation clearly communicate client/year/source/revision context?**
15. **Can I write a test for the important happy AND unhappy paths?**

If you can confidently answer those fifteen questions for your major workflows, I would consider your first iteration **very respectable UI engineering**.

---

### What I would study *right now*

Don't take CS 349.

Do perhaps **one focused weekend**:

**2–3 hours:** Read the [official Waterloo CS 349 material/course framing](https://student.cs.uwaterloo.ca/~cs349/1261/?utm_source=chatgpt.com) selectively, especially event architecture, UI architecture/components, asynchronous interaction and feedback. Waterloo's historical detailed syllabus also explicitly covers MVC-style architecture, component trees, undo/layout/multithreading, accessibility and interaction algorithms. ([Cheriton School of Computer Science][5])

**2–3 hours:** Work through [React's Managing State chapter](https://react.dev/learn/managing-state?utm_source=chatgpt.com). This is probably your highest-ROI technical reading because you're actually using React.

**1–2 hours:** Read [web.dev's form accessibility guide](https://web.dev/learn/forms/accessibility?authuser=2&utm_source=chatgpt.com) and learn semantic inputs, labels, focus and keyboard navigation.

Then stop reading.

Take **one actual Strata workflow**, probably:

> **load forecast inputs → edit → validate → save**

and draw its state machine:

```text
LOADING
   ↓
READY
   ↓ edit
DIRTY
   ↓ save
SAVING
  ↙   ↓    ↘
ERROR SAVED CONFLICT
```

Then implement those states.

That exercise will teach you substantially more than another ten hours of generic UI lectures because every concept suddenly has consequences in **your system**.

And I think this gives you exactly the boundary you were looking for: **you don't need frontend/UI to become another deep specialization. You need enough UI engineering knowledge that the excellent machinery you're building underneath Strata isn't exposed through an interface that lies about, obscures, or mishandles the state of that machinery.**

[1]: https://student.cs.uwaterloo.ca/~cs349/1261/?utm_source=chatgpt.com "CS349 Winter 2026 : About"
[2]: https://react.dev/learn/managing-state?utm_source=chatgpt.com "Managing State – React"
[3]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/Guides/Understanding_WCAG/Understandable?utm_source=chatgpt.com "Understandable - Accessibility | MDN"
[4]: https://web.dev/learn/forms/accessibility?authuser=2&utm_source=chatgpt.com "Accessibility  |  web.dev"
[5]: https://cs.uwaterloo.ca/current/courses/course_descriptions/cDescr/newCDescr/CS349?utm_source=chatgpt.com "CS 349 | SCS | UW | Cheriton School of Computer Science | University of Waterloo"


# Quick study for frontend

Yes — I misunderstood you. You were asking: **“I can't enroll in CS 349; what publicly accessible structured course can replace it?”** not whether CS 349 itself is a good idea.

I searched specifically for alternatives, and I think there is a much cleaner answer.

## My first choice for you: MIT 6.813/6.831 — User Interface Design & Implementation

This is remarkably close to the role you wanted CS 349 to play, and the course materials are publicly available through MIT.

[MIT 6.831 User Interface Design & Implementation](https://ocw.mit.edu/courses/6-831-user-interface-design-and-implementation-spring-2011/?utm_source=chatgpt.com)

The course explicitly combines three things:

**design → implementation → evaluation.**

Its curriculum includes usability, learnability, efficiency, errors/user control, **UI software architecture**, layout, input/output, prototyping, accessibility, heuristic evaluation and user testing. ([MIT OpenCourseWare][1])

That's much closer to your actual missing knowledge than another “learn React” course.

For example, you've been asking:

> What should happen when saving fails?
> How should loading be represented?
> How should users recover from errors?
> How should UI state correspond to system state?
> How do I architect the interaction rather than merely render components?

Those are **UI engineering questions**, not React syntax questions.

And MIT actually has programming assignments plus a project sequence that goes:

```text
analysis
   ↓
design
   ↓
paper prototype
   ↓
computer prototype
   ↓
implementation
   ↓
user testing
```

([MIT OpenCourseWare][2])

That's an excellent mental framework for Strata.

### One limitation

It's an older course. The implementation technology therefore isn't what I would want you copying into a 2026 React/TypeScript application.

So I would use MIT for:

**“How should I think about user interfaces?”**

not:

**“How exactly should I implement modern Strata?”**

---

## Then use one modern React course for implementation

This is where Brian Holt's **Complete Intro to React v9** becomes useful.

[Complete Intro to React v9](https://react-v9.holt.courses/?utm_source=chatgpt.com)

Its sequence is coherent:

```text
components
JSX
hooks
effects
user input
context
      ↓
routing
      ↓
TanStack Query
      ↓
error boundaries
forms
      ↓
testing
      ↓
React 19
```

It specifically includes effects, user inputs, TanStack Query, error boundaries and testing—all things that matter when Strata starts orchestrating FastAPI requests. ([Master.dev][3])

So I'd pair the two:

```text
MIT 6.813
"What is a well-designed interactive system?"
            +
Complete Intro to React
"How do I implement one using modern React?"
            ↓
          STRATA
```

That combination makes considerably more sense to me than trying to assemble your education from random documentation.

---

## There's one other very interesting option: MIT 6.4500

MIT now has a newer course called **Design for the Web: Languages and User Interfaces**.

[MIT 6.4500 Design for the Web](https://designftw.mit.edu/?utm_source=chatgpt.com)

This one is tantalizingly close to what you want because it combines **HTML + CSS + JavaScript + reactive UI + UI design principles**. It teaches learnability, efficiency, safety, prototyping and user testing alongside actual web technologies. ([Design for the Web][4])

The catch is that it uses **Vue rather than React** and isn't packaged as conveniently as a self-contained OCW course.

So I wouldn't make it your primary implementation course, but its conceptual material is very relevant.

---

## What about Full Stack Open?

This remains an excellent resource, but I've changed my mind about making it your primary recommendation.

[University of Helsinki Full Stack Open](https://fullstackopen.com/en/about/?utm_source=chatgpt.com)

It gives you exactly the integrated picture you were asking about:

```text
React
 ↕
REST
 ↕
backend
 ↕
database

+ TypeScript
+ testing
+ validation
```

The University of Helsinki describes its main focus as React single-page applications communicating with REST APIs. ([University of Helsinki][5])

And its TypeScript section is particularly relevant: React with types, validating unknown inputs, narrowing, schemas and type-safe API interaction. ([MOOC.fi][6])

The problem is **scope**.

The full course is enormous, and a lot of it teaches Node/Express, MongoDB, GraphQL, etc. You already have Python/FastAPI/database work underway. You don't need to rebuild Strata's backend in JavaScript just to understand the frontend.

So I'd raid Full Stack Open selectively later rather than adopt the whole thing.

---

# So I'd actually modify your curriculum like this

I don't think you need **another semester-long course before you're allowed to touch the frontend**.

I'd give this phase perhaps **30–50 focused hours**, intertwined with Strata.

Start with selected MIT 6.813 material, particularly:

**Usability → Learnability → Efficiency → Errors/User Control → UI Software Architecture → Prototyping → Input/Output → Accessibility → Heuristic Evaluation.**

Don't worry about completing every assignment or every piece of HCI research material.

Then do **Complete Intro to React v9** fairly thoroughly.

And throughout the React course, keep your existing client CRUD endpoints open beside you.

When you learn:

```text
component
```

build `<ClientList>`.

When you learn:

```text
user input
```

build `CreateClientForm`.

When you learn:

```text
effects / query
```

call `GET /clients`.

When you learn:

```text
mutation
```

call `POST /clients`.

When you learn:

```text
error handling
```

intentionally make FastAPI reject something.

When you learn:

```text
testing
```

test the client workflow.

So you get:

```text
MIT UI CONCEPT
      ↓
REACT IMPLEMENTATION CONCEPT
      ↓
TINY STRATA IMPLEMENTATION
      ↓
NEXT CONCEPT
```

rather than:

```text
Take giant frontend curriculum
       ↓
3 months pass
       ↓
finally resume Strata
```

And **then**, once the simple client CRUD UI works, I'd selectively use the TypeScript and React portions of Full Stack Open to strengthen whatever feels shaky.

---

The important realization from your last few messages is that I **would promote UI/application architecture somewhat higher in your learning priorities than we originally discussed**.

Not because I think you're becoming a frontend engineer.

It's because **Strata has reached the point where this knowledge is on its critical path.**

Your engine exists. Your database/input architecture is emerging. Your endpoints are nearly there. The next engineering problem isn't “learn enough JSX to put some buttons on screen.”

It's:

> **How does a human safely interact with this stateful, asynchronous system through an interface?**

That's a legitimate body of computer science/software engineering knowledge, and given how you learn, I agree with you: acquiring a structured conceptual model first is likely *more efficient* than spending the next two months repeatedly encountering concepts you don't know exist and asking me or Claude to patch each hole individually.

[1]: https://ocw.mit.edu/courses/6-831-user-interface-design-and-implementation-spring-2011/?utm_source=chatgpt.com "User Interface Design and Implementation | Electrical Engineering and Computer Science | MIT OpenCourseWare"
[2]: https://ocw.mit.edu/courses/6-831-user-interface-design-and-implementation-spring-2011/pages/assignments/?utm_source=chatgpt.com "Assignments | User Interface Design and Implementation | Electrical Engineering and Computer Science | MIT OpenCourseWare"
[3]: https://frontendmasters.com/courses/complete-react-v9/introduction/?utm_source=chatgpt.com "Introduction - Complete Intro to React, v9 | Master.dev"
[4]: https://designftw.mit.edu/?utm_source=chatgpt.com "6.4500 Design for the Web: Languages and User Interfaces"
[5]: https://www.helsinki.fi/en/admissions-and-education/open-university/multidisciplinary-themed-modules/full-stack?utm_source=chatgpt.com "Full Stack | University of Helsinki"
[6]: https://courses.mooc.fi/org/uh-cs/courses/full-stack-open-typescript?utm_source=chatgpt.com "Full Stack Open: Typescript - MOOC.fi courses"


# Editted study material for concentrated gap

## gap

- Frontend interaction reliability
  - UI state: what states can every screen/action be in?
    - states:
      - LOADING
      - READY
      - DIRTY
      - VALIDATING
      - SAVING
      - SAVED
      - INVALID
      - CONFLICT
      - ERROR
      - STALE
      - DISABLED
      - EMPTY
  - Feedback: how does the user know what the system is doing/did?
    - key point: never make the user wonder "did that work?"
    - key interfaces
      - saved
      - failed → retry
      - validation → suggest how to modify, e.g., `cost must not be negative`
  - Async operations: what happens between request and response?
    - key questions
      - Can the user click Run Forecast twice?
      - Can they edit inputs while calculation is happening?
      - What happens if they navigate away?
      - What if the request takes 100 ms?
      - 2 seconds?
      - 30 seconds?
      - What if the connection disappears?
      - What if the server returns 500?
      - What if the first request finishes after a newer request?
  - Error/recovery: what happens when something fails?
    - key questions
      - What if it succeeds?
      - What if it's slow?
      - What if it returns nothing?
      - What if validation fails?
      - What if authorization fails?
      - What if the server fails?
      - What if the network fails?
      - What if the underlying data changed?

Yes. Now that we've identified the gap precisely, I would **not take another broad frontend course after React v9**. I'd build a small, targeted second stage around the parts React v9 doesn't really teach: **server-state synchronization, explicit interaction state modeling, failure/recovery, and user-facing reliability.**

I checked the current Full Stack Open curriculum, and there are a few sections that fit extremely well.

### My post–React v9 curriculum for Strata

| Order | Resource                | Study                                      | Main gap it fills                               | Time |
| ----- | ----------------------- | ------------------------------------------ | ----------------------------------------------- | ---: |
| 1     | Full Stack Open Part 2c | **Getting data from server**               | Async/browser/API lifecycle                     | 1–2h |
| 2     | Full Stack Open Part 2d | **Altering data in server**                | Mutations, API → UI synchronization, errors     | 2–3h |
| 3     | Full Stack Open Part 6c | **React Query**                            | Server state, stale data, loading/error/refetch | 2–3h |
| 4     | State machines tutorial | **States + events + transitions + guards** | Explicit UI state modeling                      |  ~1h |
| 5     | Apply to Strata         | Draw interaction models                    | Loading/saving/error/conflict/recovery          | 2–3h |

So this is another **~8–12 hours**, not another 40-hour curriculum.

And I'd actually do it **after you attempt the simple Strata client frontend**, because then all of these concepts will attach to problems you've physically encountered.

## 1. Full Stack Open 2c — absolutely do this

[Full Stack Open — Getting data from server](https://fullstackopen.com/en/part2/getting_data_from_server/?utm_source=chatgpt.com)

This section explicitly teaches the browser's asynchronous execution model, promises, effects, API fetching, state updates, and rerendering. ([Full Stack Open][1])

The important model for you is:

```text
React renders
     ↓
effect starts GET /clients
     ↓
request is pending
     │
     │ JavaScript DOES NOT sit there waiting
     ↓
response eventually arrives
     ↓
callback executes
     ↓
setClients(...)
     ↓
React rerenders
```

You probably won't need much time here after React v9.

Your graduation criterion is simply:

> **Can I trace the exact lifecycle from mounting `<ClientPage>` through `GET /clients` to the user seeing clients?**

Including the period when the request hasn't finished yet.

## 2. Full Stack Open 2d — probably the highest-value section

[Full Stack Open — Altering data in server](https://fullstackopen.com/en/part2/altering_data_in_server/?utm_source=chatgpt.com)

This one I'd study carefully.

It covers changing remote data, PUT/POST-style operations, promises, error handling, updating React state based on server responses, and separating backend communication into its own service module. ([Full Stack Open][2])

This directly maps to:

```text
<CreateClientForm>
       ↓
local draft state
       ↓
POST /clients
       ↓
FastAPI
       ↓
SQLite transaction
       ↓
response
       ↓
update client state
       ↓
<ClientList> rerenders
```

And this is where I'd start asking your reliability questions:

```text
What if POST is pending?

What if it succeeds?

What if FastAPI returns 400?

What if FastAPI returns 409?

What if FastAPI returns 500?

What if the network disappears?
```

Full Stack Open won't completely answer your UX policy. **That's okay.**

Its purpose is to make you understand the machinery on which that policy operates.

## 3. Full Stack Open Part 6c — this is the one I especially want you to see

The current 2026 version of Full Stack Open Part 6 has been updated: it now teaches Zustand, `useReducer`/Context, and **React Query for server-state management**. ([Full Stack Open][3])

[Full Stack Open — Part 6 Advanced State Management](https://fullstackopen.com/en/part6/?utm_source=chatgpt.com)

Don't take all of Part 6.

You specifically care about **6c React Query / server state**.

Why?

Because after your first CRUD implementation, you're likely to have something like:

```text
FastAPI / SQLite
      ↓
AUTHORITATIVE CLIENTS

       versus

React
      ↓
CURRENTLY KNOWN CLIENTS
```

And those aren't necessarily identical.

The frontend's knowledge may be:

```text
loading
fresh
stale
refetching
errored
```

Meanwhile mutations may be:

```text
idle
pending
success
error
```

That's exactly the conceptual territory in your checklist.

React Query exists largely because **remote asynchronous state has fundamentally different properties from ordinary local UI state**. Full Stack Open specifically introduces it as a tool for simplifying server-state management. ([Full Stack Open][3])

Even if you decide **not to use React Query in Strata yet**, learn the conceptual model.

That's the important part.

## 4. Then spend ONE hour learning state machines

This is the major thing neither React v9 nor Full Stack Open gives you systematically.

And there's a nearly perfect free resource:

[Introduction to State Machines Using XState — free Egghead course](https://egghead.io/courses/introduction-to-state-machines-using-xstate?utm_source=chatgpt.com)

It's **58 minutes total**.

The course covers enumerating states, transitions, guards, hierarchical states and parallel states. ([Egghead][4])

I would **not learn XState because I think Strata needs XState**.

Ignore the library.

You're stealing its conceptual machinery.

The most important early lesson is literally about avoiding **Boolean explosion** by explicitly enumerating states. ([Egghead][4])

Because suppose you start writing:

```text
isLoading
isSaving
isDirty
isError
isValid
isStale
```

Now what does this mean?

```text
isLoading = true
isSaving = true
isError = true
isValid = false
isStale = true
```

Is that possible?

Should it be possible?

**Nobody knows. Welcome to frontend hell.** 😂

State machines force you to think:

```text
READY
  │
  │ EDIT
  ↓
DIRTY
  │
  │ SAVE
  ↓
SAVING
 ↙    ↘
ERROR  READY
```

And statecharts introduce something even more useful for the observation we made earlier: **hierarchical and parallel state**. That's how you can reason about an interaction being, say, `DIRTY` and `STALE` simultaneously without producing one gigantic flat enum. ([XState for React Developers][5])

That's extremely relevant to Strata.

Again:

**learn the model, don't adopt the framework yet.**

## 5. Then STOP STUDYING and design one Strata workflow

At this point I would not add another tutorial.

Take:

### Edit Client

and actually write:

```text
DATA
────────────
LOADING
EMPTY
PRESENT
ERROR


EDITING
────────────
CLEAN
DIRTY


VALIDATION
────────────
VALID
INVALID


SAVE
────────────
IDLE
SAVING
ERROR
SUCCESS
```

Then define transitions:

```text
LOAD
  ↓
LOADING
  ↓
PRESENT

USER_EDITS
  ↓
DIRTY

SAVE
  ↓
VALIDATING
   ↙     ↘
INVALID   SAVING
            ↓
       ┌────┴────┐
     ERROR     SUCCESS
```

Now ask:

> What does the user see in every state?

That's where your **feedback layer** enters.

```text
SAVING
→ button disabled
→ "Saving..."

SUCCESS
→ "Saved"

INVALID
→ field-level explanation

ERROR
→ edits remain
→ explain failure
→ Retry

EMPTY
→ "No clients yet"
→ Create Client
```

And now you've connected:

```text
React
   ↓
implementation primitives

Full Stack Open
   ↓
async API interaction

React Query concepts
   ↓
server-state lifecycle

State machines
   ↓
interaction modeling

Your checklist
   ↓
product reliability policy

STRATA
```

**That's the coherent picture you've been looking for.**

---

There is one important thing I **wouldn't study yet: conflict resolution**.

You should understand that this situation exists:

```text
Browser loads revision 5
        ↓
Someone changes server
        ↓
Server becomes revision 6
        ↓
Browser submits changes based on revision 5
        ↓
CONFLICT
```

But *solving* that properly starts pulling you into optimistic concurrency control, version numbers/ETags, API semantics, merge policies, etc.

That's actually more connected to the database/concurrency work you've already been doing than to React itself.

For the first client CRUD iteration, just know:

> **STALE and CONFLICT are states my architecture may eventually need to represent.**

Don't engineer them yet unless Strata's actual multi-user requirements demand them.

---

So I think you've now got a very clean learning sequence:

```text
NOW
│
├── Complete Intro to React v9
│       ↓
│   learn React vocabulary
│
├── BUILD Strata Client CRUD v0
│       ↓
│   discover actual gaps
│
├── Full Stack Open 2c
│   async fetching
│
├── Full Stack Open 2d
│   mutations + errors
│
├── Full Stack Open 6c
│   server state
│
├── ~1h state-machine course
│   explicit interaction modeling
│
└── REDESIGN Client CRUD v1
        ↓
    loading
    empty
    ready
    dirty
    validating
    saving
    saved
    invalid
    error
    retry
```

At that point, I would consider the conceptual gap you identified **sufficiently closed for Strata v1**.

Not “you know frontend.”

Rather, something much more useful:

**you know enough about frontend application behavior to recognize the important engineering decisions instead of accidentally letting React's implementation details make those decisions for you.**

[1]: https://fullstackopen.com/en/part2/getting_data_from_server/?utm_source=chatgpt.com "Fullstack part2 |"
[2]: https://fullstackopen.com/en/part2/altering_data_in_server/?utm_source=chatgpt.com "Fullstack part2 |"
[3]: https://fullstackopen.com/en/part6/?utm_source=chatgpt.com "Fullstack part6"
[4]: https://egghead.io/courses/introduction-to-state-machines-using-xstate?utm_source=chatgpt.com "Introduction to State Machines Using XState | egghead.io"
[5]: https://www.xstateforreactdevelopers.com/1-getting-started/1-intro-to-state/2-compound-states.mdx?utm_source=chatgpt.com "XState for React Developers"
