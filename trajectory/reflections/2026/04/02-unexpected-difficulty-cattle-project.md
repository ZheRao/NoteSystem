# Reflection — Percentage-Based Allocation & Stage 1 QA Complexity

## 1. Hypothesized Percentage Weirdness (V3 Budget Allocation - downstream system)

V3 introduces a fundamental shift:
> Budget allocation is derived from monthly actual totals (post-allocation) rather than predefined inventory percentages.

At first glance, this seems intuitive — use reality (actuals) to drive allocation.

However, this introduces several non-obvious behaviors:

### 1.1 Sign-driven distortion

Actual totals are **signed financial values** (Net Income level), not neutral quantities.

This means:
- totals can be **positive or negative**
- CC and F totals can have:
    - same sign → stable percentages
    - opposite signs → **invalid or unintuitive percentages**
    - near-zero sum → **unstable / undefined percentages**

Example failure modes:
- negative percentages
- percentages > 100%
- division by zero

These are mathematically valid but economically meaningless in a budget allocation context.

### 1.2 Structural skew from upstream routing

The prior system has two stages:
1. Class-based deterministic routing
2. Percentage-based split (inventory-driven)

This creates a subtle effect:
- positive contributions may be routed out in stage 1
- remaining pool becomes **heavily skewed (e.g., strongly negative)**
- percentages are then derived from this distorted pool

Example:
- `-608k` as grand total, `-3M` as the total after class reroute → `+2.4M` got routed out of stage 1 → splitting `-3M` among two operation modes in stage 2

Result:
> The allocation basis is not a neutral representation of activity, but a residual distribution shaped by prior logic

### 1.3 Mismatch between math and business meaning

There is a fundamental tension:

- mathematically:
```
percentage = CC_total / (CC_total + F_total)
```
business expectation:
> percentages should represent a clean distribution of operations (0%–100%)

When actual totals include:
- reversals
- corrections
- mixed-sign contributions

The math remains valid, but the **interpretation breaks down**.

### 1.4 Key insight

The problem is not computing percentages —
it is defining what those percentages are supposed to mean.

## 2. Why QA Was Unexpectedly Hard (prior upstream system)

This system is not a simple transformation. It is a **multi-stage allocation engine with state propagation**.

QA difficulty came from the need to validate:

### 2.1 Multiple transformation stages

Each transaction flows through:
1. Class reroute (deterministic)
2. Algorithmic split (percentage-based)
3. Offset generation (net-zero enforcement)

Each stage modifies both:
- values
- structure

### 2.2 Multiple orthogonal dimensions

For a single `(location, month)` slice, QA required validating across:

- Transformation type
    - class reroute
    - algorithm split

- Record type
    - actual split
    - offset

- Metadata correctness
    - pillar label
    - location label
    - record type label
    - classification source
    - memo modifications

- Temporal logic
    - fallback to previous month when percentage missing

This creates a combinatorial surface of checks.

### 2.3 Invariant-based validation (not just correctness)

QA is not:
> “does this number look right?”

It is:
- totals must reconcile exactly
- offsets must cancel precisely
- splits must preserve value
- fallback must select the correct historical period

This requires **strict invariants**, not visual inspection.

### 2.4 Hidden coupling between stages

Errors or design effects in early stages (e.g., class routing) can:
- distort later-stage distributions
- create unexpected downstream behavior
- still pass reconciliation checks

This makes debugging non-local and cognitively expensive.

## 3. Why This Is Hard

This work is difficult because it combines:
```
accounting semantics
+ multi-stage transformations
+ signed numerical behavior
+ temporal fallback logic
+ business interpretation requirements
```
Your brain is forced to simultaneously track:
- value flow
- structural changes
- metadata propagation
- invariants
- and downstream meaning

This is **state tracking across multiple layers**, which is inherently heavy.

## 4. Why This Is Necessary

Despite the difficulty, this level of rigor is required because:

### 4.1 Financial outputs must be trusted
- small inconsistencies → confusion
- repeated inconsistencies → loss of trust
- loss of trust → system abandonment

### 4.2 Errors are often silent
- the math runs
- outputs look clean
- but interpretation is wrong

This is more dangerous than visible failure.

### 4.3 Policy is embedded in logic

Allocation rules are not just computation — they are:
> encoded business decisions

If not explicitly validated, the system:
- makes decisions implicitly
- without stakeholder agreement

5. Why This Is Often Overlooked

Most people assume:
```
“just split by percentage”
```
They underestimate:
- the importance of defining the **source of the percentage**
- the impact of **signed financial data**
- the complexity introduced by **multi-stage transformations**
- the need for **strict invariants**
- the gap between **mathematical correctness and business meaning**

As a result:
- systems are built quickly
- appear correct
- but degrade trust over time

## 6. Key Takeaways
- The hardest part is not the computation, but the **definition of meaning**
- QA must be **invariant-driven**, not intuition-driven
- Intermediate correctness does not guarantee **final interpretability**
- Multi-stage systems require **dimension isolation** during validation
- **Trust > speed** in financial systems

## Final Reflection

> The multiplication took minutes.  
> Defining what should be multiplied — and proving it behaves correctly — took weeks.

And that is the real work.