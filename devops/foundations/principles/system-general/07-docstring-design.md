# Docstring Design Reflection

## Context

During the cattle split project, docstrings were written to improve interpretability and communicate intent. While they successfully captured purpose and edge-case handling, they revealed an important gap: lack of strict structure and enforceable contracts.

This reflection formalizes how docstrings should be designed going forward, especially for long-term systems like `data-platform`.



## Core Principle

> Docstrings are not explanations. They are **contracts + invariants + mental models**.

A strong docstring allows someone to:

* Use the function correctly without reading implementation
* Understand system guarantees
* Identify failure conditions early



## Standard Structure (Target Format)

Every non-trivial function should follow this structure:

### 1. Purpose

A short, high-level description of *what this function achieves* in the system.

* Avoid implementation details
* Focus on intent and role

Example:

```
Purpose:
    Create synthetic split and offset transactions to redistribute original transactions
    across operation modes while preserving net-zero totals.
```



### 2. Inputs

Explicitly define inputs and their expectations.

Include:

* Type
* Required columns (for DataFrames)
* Constraints

Example:

```
Inputs:
    df: pd.DataFrame
        Required Columns:
            - Location: str
            - AmountDisplay: float
            - year_month_key: int
    perc_map: pd.DataFrame
        Required Columns:
            - Location: str
            - inventory_year_month_key: int
            - perc: float
```



### 3. Outputs

Define what is returned.

Example:

```
Outputs:
    DataFrame containing only synthetic split and offset transactions.
```



### 4. Invariants (CRITICAL)

These are **system laws** that must always hold.

Examples:

```
Invariants:
    - Sum(AmountDisplay) == 0
    - All inventory_year_month_key values exist in perc_map
    - Output does not include original transactions
```

These should map directly to validation logic in code.



### 5. Side Effects

Document mutations or external effects.

Examples:

```
Side Effects:
    - Modifies Memo field to include classification metadata
    - Prints warning if join failures occur
```



### 6. Failure Modes

Explicitly define when and why the function fails.

Examples:

```
Failure:
    - Raises ValueError if net sum != 0
    - Raises ValueError if inventory mapping is incomplete
```



## Key Design Rules

### Rule 1 — Prefer Abstraction Over Implementation

Bad:

```
Creates three copies of the dataframe
```

Good:

```
Generates synthetic transactions for each operation mode and corresponding offsets
```

Reason:

* Implementation changes
* Abstraction remains stable



### Rule 2 — Always Define Data Contracts

If a function depends on specific columns, they must be listed.

Otherwise:

* Hidden coupling forms
* Bugs become silent



### Rule 3 — Invariants Must Be Visible

If your code contains checks like:

```
if abs(total) > 0.001: raise ValueError
```

Then your docstring must include:

```
- Sum(...) == 0
```



### Rule 4 — Separate "What" from "How"

Docstrings should answer:

* What does this function guarantee?

NOT:

* How does it do it?



### Rule 5 — Optimize for Scanning

Engineers rarely read docstrings fully.

Structure enables:

* Quick validation
* Fast debugging
* Safe reuse



## Anti-Patterns Observed

### 1. Narrative-style docstrings

Problem:

* Hard to scan
* Key guarantees buried in text



### 2. Missing required columns

Problem:

* Implicit dependencies
* Breaks pipeline modularity



### 3. Mixing implementation with intent

Problem:

* Becomes outdated quickly



## Forward Application

### Immediate (Cattle Project)

* Accept current state due to delivery pressure
* Avoid over-refactoring under time constraints



### Next Phase (`data-platform`)

Apply strict docstring contracts from the beginning:

1. Every transformation function defines required schema
2. Every transformation encodes invariants
3. Docstrings align with validation logic
4. Avoid narrative style completely



## Long-Term Evolution

Docstrings should evolve into:

* Schema validation (pydantic / checks)
* Automated invariant testing
* Contract-driven pipelines

End goal:

> Code + Docstrings + Validation = Single Source of Truth



## Final Insight

> A system without explicit contracts relies on memory.
> A system with explicit contracts becomes composable, testable, and scalable.

Docstrings are the first step toward that transition.
