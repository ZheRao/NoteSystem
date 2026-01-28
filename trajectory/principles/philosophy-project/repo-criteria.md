# GitHub Creation Guidebook — Signal, Truth, and Trajectory

> **Purpose**: This document defines *what* to create on GitHub, *why* it matters, and *how* to judge whether a project deserves to exist — so that every public artifact compounds signal, truth, and long‑term trajectory.

This is not a popularity playbook. It is a **signal‑integrity framework**.

## 0. First Principle (Non‑Negotiable)

> **A GitHub repo is not a code container. It is a compressed transmission of judgment under constraints.**

If a stranger reads your repo and cannot infer:

* how you think,
* what you value,
* what you *don’t* do,
* how you reason under pressure,

then the repo failed — regardless of stars, completeness, or cleverness.

## 1. The Three Axes of Legitimate Signal

Every repo must clearly project *at least one* of the following axes. Exceptional repos project two. Rare ones project all three.

### Axis A — **Reality Survivability**

> Can this thinking survive messy, adversarial, real‑world conditions?

Signals include:

* explicit failure modes
* idempotency and restart safety
* externalized config as contracts
* atomicity, invariants, and boundaries
* time, cost, and operational trade‑offs

This axis answers:

> “Has this person *actually shipped systems* that were allowed to break?”

---

### Axis B — **Mechanistic Understanding**

> Does this person understand systems from first principles, not APIs?

Signals include:

* from‑scratch implementations
* minimal but complete abstractions
* symmetry (encoder/decoder, forward/backward)
* removal of convenience layers
* explicit math → code mapping

This axis answers:

> “Does this person understand *why* things work?”

---

### Axis C — **Architectural Taste & Foresight**

> Does this person see the invisible structure before others do?

Signals include:

* hierarchy and decomposition
* restraint (what is *not* built)
* naming clarity
* future extensibility without speculative bloat
* clear separation of concerns

This axis answers:

> “Would I trust this person to design something meant to last?”

## 2. The Repo Worthiness Test (Create or Kill)

Before starting *any* repo, it must pass **all** of the following:

### Test 1 — Essence Test

> Can the core idea be stated in **one sentence**?

If not, the idea is still fog.

### Test 2 — Compression Test

> Does this repo compress *years* of lived experience into an artifact?

If it merely expands effort into files, do not build it.

### Test 3 — Generalization Test

> If domain specifics were stripped away, would the insight remain valuable?

If the value collapses without context, it belongs in private work, not GitHub.

### Test 4 — Scarcity Test

> Is this something *few people can honestly write*?

If many people could plausibly produce it, signal dilution is likely.

## 3. Canonical Repo Categories (Approved Set)

You should only create repos that fall into one of these categories.

### Category 1 — **Generalized Industrial Systems**

**Purpose**: Surface private, real‑world system experience without exposing proprietary details.

**Characteristics**:

* domain‑agnostic contracts
* config‑driven orchestration
* explicit data‑layer separation (e.g., Bronze/Silver/Gold)
* operational invariants documented

**Primary Artifact**: README + architecture docs

**Core Question Answered**:

> “Can this person build systems that don’t collapse under reality?”

### Category 2 — **From‑Scratch Cognitive Primitives**

**Purpose**: Demonstrate mechanistic understanding of intelligence and learning.

**Examples**:

* autograd engine
* tiny CNN
* tiny RNN

**Rules**:

* minimal code
* no frameworks unless essential
* correctness > performance
* explanation > cleverness

**Primary Artifact**: explanation.md + code

**Core Question Answered**:

> “Does this person understand learning at the root?”

### Category 3 — **Design‑First Repos (Code Is Secondary)**

**Purpose**: Demonstrate architectural foresight and system taste.

**Characteristics**:

* docs‑first structure
* diagrams and invariants
* failure‑mode analysis
* "why not X" sections

**Primary Artifact**: `/docs`

**Core Question Answered**:

> “Can this person reason before implementing?”

### Category 4 — **Executable Essays / Post‑Mortems**

**Purpose**: Externalize lived judgment without leaking code.

**Examples**:

* system rebuild lessons
* trade‑off analyses
* constraint‑driven decisions

**Primary Artifact**: narrative + minimal reproducible examples

**Core Question Answered**:

> “Has this person learned from reality?”

## 4. What You Must Explicitly Document (Always)

Every repo must include **at least two** of the following sections:

* **Invariants** — what must never break
* **Failure Modes** — how this can fail and how it degrades
* **Non‑Goals** — what is intentionally excluded
* **Trade‑offs** — what you sacrificed and why
* **Evolution Path** — how this grows *without* breaking identity

Absence of these is interpreted as absence of experience.

## 5. The Non‑Intervention Rule (Critical)

> **Do not over‑teach. Do not over‑justify. Do not rescue the reader.**

Your role is to:

* expose structure
* state invariants
* reveal trade‑offs

Not to convince.

Those who can see, will see.

## 6. Temporal Discipline (Anti‑Noise Rule)

* Fewer repos > many repos
* One flagship repo per *conceptual phase*
* Repos should feel *inevitable*, not reactive

If you feel urgency, pause. Signal compounds slowly.

## 7. Identity Alignment Check (Final Gate)

Before publishing, ask:

> “If this repo were the *only thing* someone knew about me — would it misrepresent who I am becoming?”

If yes, do not publish.

GitHub is not a diary. It is a **public memory of identity‑level abstractions**.

## 8. North Star

> **Your goal is not to be seen.
> Your goal is to make the right people feel: “This person sees deeper than most.”**

That recognition propagates quietly — but permanently.

*This document is alive. It should evolve only when your internal architecture evolves.*
