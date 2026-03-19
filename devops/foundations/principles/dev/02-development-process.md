# Development Rule — Notebook → Notes → Script

## Purpose

This rule exists to **reduce cognitive load, prevent refactor paralysis, and preserve momentum under pressure**.

It formalizes a workflow where *understanding is prepaid* before implementation, allowing fast, mechanical translation from exploration to production code.

This is not a style preference — it is an **operating principle**.

## Core Insight

> **Never translate a notebook directly into a script.**
>
> First translate *intent* into notes.

The moment intent is externalized, implementation becomes trivial.

## The Three-Stage Pipeline

### Stage 1 — Notebook (Exploration Space)

**Role:** Discover reality

Characteristics:

* Messy
* Iterative
* Re-runnable cells
* Inline checks, prints, ad-hoc logic

Rules:

* Do not optimize structure
* Do not future-proof
* Do not worry about file layout

The notebook exists to answer:

> *“What is actually going on?”*

---

### Stage 2 — Structured Notes (Intent Freezing)

**Role:** Lock understanding without code

This is the critical, non-optional stage.

Write **only**:

* **Inputs** — what the pipeline consumes
* **Outputs** — what the pipeline guarantees
* **Transform steps** — ordered, named, conceptual
* **Assumptions / invariants** — what must be true

Rules:

* No implementation detail
* No pandas / Spark / syntax
* Bullet points only

Example:

```text
Inputs:
- Silver SO dataframe
- SO contract mapping

Outputs:
- Gold SO table
- Date dimension table

Transform steps:
1. Normalize price fields
2. Apply SO mapping
3. Enrich date fields

Assumptions:
- Required columns exist
- Mapping contract is complete
```

Once this exists, **you no longer need to re-read the notebook**.

---

### Stage 3 — Script (Mechanical Transcription)

**Role:** Deterministic system artifact

At this stage:

* You are not "figuring things out"
* You are not exploring
* You are implementing a known contract

Rules:

* Orchestrator functions reflect the transform steps 1:1
* Helper functions are single-responsibility
* No hidden state
* No notebook-only assumptions

The script is a *translation*, not a discovery.

## Why This Works

### 1. Cognitive Load Is Shifted Earlier

Understanding happens once, deliberately — not repeatedly under stress.

### 2. Fear Disappears

You never face a blank file. You are filling in a known shape.

### 3. Refactors Become Cheap

If logic changes, update the notes first. Code follows.

### 4. Energy Is Preserved

Low-energy days can still produce clean code because thinking is already done.

## Mandatory Rule

> **If you feel resistance moving code out of a notebook, you skipped Stage 2.**

Do not push through.
Go back and write the notes.

## When to Use This Rule

* Production pipelines
* High-stakes demos
* Work done under time pressure
* Any system you expect to touch again

## When *Not* to Use This Rule

* One-off analysis
* Throwaway exploration
* Pure learning notebooks

## Mental Model

* Notebook = *Thinking*
* Notes = *Understanding*
* Script = *Execution*

Do not collapse these layers.

## Status

**Non-negotiable principle.**

This rule exists to protect future-you.
