# PATH Pipeline Design Revelation

## Core Insight

**Pipelines scale cleanly only when dataset-specific logic is separated from layer-general logic.**

* **Layers (silver, gold, etc.) define *what kind* of transformation happens**
* **Datasets (SO, PO, etc.) define *what shape* the data has and *how* it should be processed**

Mixing these two dimensions leads to coupling, branching logic, and inevitable refactors (as seen in the QBO system).

## The Anti-Pattern (What Breaks at Scale)

```python
transform(dataset="so")
transform(dataset="po")
```

This design forces:

* `if dataset == ...` branching inside transforms
* Shared assumptions across unrelated datasets
* Ever-growing functions that know too much
* High blast radius when adding or modifying datasets

This pattern *appears* flexible early on, but scales poorly and accumulates architectural debt.

## The Correct Pattern (What Scales Cleanly)

**Dataset-specific pipelines composed from layer-general helpers.**

```python
extract_so()  -> transform_so()  -> load_so()
extract_po()  -> transform_po()  -> load_po()
```

* Each dataset owns its own pipeline entrypoint
* Each transform function is a small orchestrator
* Shared logic lives in helper modules, not in branching code

## Repo Structure (Reference Architecture)

```
path/
  io/
    extract_so.py      # extract_so()
    extract_po.py      # extract_po()

  silver/
    so.py              # transform_so()
    po.py              # transform_po()
    _helpers.py        # general helpers (schema coercion, parsing, cleaning)

  gold/
    fact_sales.py      # combines standardized outputs (SO + PO)
    _helpers.py

  pipelines/
    so_pipeline.py     # run_so()
    po_pipeline.py     # run_po()
    full_refresh.py    # run_all()

  utils/
    global_helpers.py
```

---

## Design Rules (Non-Negotiable)

1. **No dataset branching inside layer transforms**
   ❌ `if dataset == "so"` inside `silver/`

2. **Helpers must be dataset-agnostic**
   Functions like `_convert_price_column()` or `_coerce_schema()` must make *no assumptions* about SO vs PO.

3. **Dataset-specific logic lives in dataset-specific files**

   * `transform_so()` knows SO contracts, columns, and business rules
   * `transform_po()` knows PO contracts, columns, and business rules

4. **Pipelines orchestrate, helpers execute**
   Pipelines decide *what runs*. Helpers decide *how to do one thing well*.

## Why This Prevents Future Refactors

* Adding a new dataset does **not** modify existing pipelines
* Failures are isolated to one dataset
* Tests are smaller and more targeted
* Refactors improve helpers without touching orchestration

This design turns pipeline growth from *exponential complexity* into *linear composition*.


## Helper Placement Rule (Escalation Rule)

A helper function should be placed according to **where it is *proven* to be reusable**, not where it *might* be reusable.

**Escalation path:**

1. **Start local (dataset-specific)**
   Implement the function inside the pipeline-specific script, e.g.:

   * `silver/so.py`

2. **Promote to layer helper when reused within the same layer**
   If the function is needed by another dataset in the *same layer* (e.g. `silver/po.py`), then:

   * Generalize it
   * Move it to `silver/_helpers.py`

3. **Promote to global helper only when reused across layers**
   If the function is needed outside the layer (e.g. from `gold/po.py`), then:

   * Further generalize it
   * Move it to `utils/<domain>_helpers.py`

**Rule of thumb:**

> *Helpers earn their generality through reuse — never by prediction.*


## Mental Model

> **Layers define transformation intent. Datasets define transformation shape.**

Once these are separated, the system remains readable, extensible, and durable — even under rapid iteration.


## PATH-Specific Takeaway

This structure directly addresses the architectural failure mode observed in the QBO system and establishes PATH as a **long-lived, multi-dataset intelligence pipeline** rather than a one-off transformation script.

This is a permanent design upgrade, not a refactor.
