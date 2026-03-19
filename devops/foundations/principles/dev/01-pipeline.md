# Gold Standard — Data Pipeline Flow

## Purpose

This document defines the **gold standard for building data pipelines**.

It encodes a repeatable, legible, and low-cognitive-load structure that scales from small operational scripts to production-grade systems.

This is not about tools.
This is about **responsibility boundaries**.

## Core Principle

> **A pipeline is a sequence of responsibility boundaries, not a sequence of files.**

Each stage exists to *eliminate a specific class of uncertainty*.

## The Canonical Pipeline Stages

### Stage 1 — Prep (Intent + Configuration)

**Responsibility:** Declare *what* will run and *where*, not *how*.

Allowed:

* Imports
* Dates / runtime context
* Path resolution
* Config loading

Forbidden:

* Business logic
* Data mutation
* Schema assumptions

Example:

```python
from path_adapter import csvReadConfig, process_raw_table, read_configs, gold_transformation

read_cfg = csvReadConfig()
today = dt.date.today()

path_config = read_configs(config_type="io", name="paths.json")
raw_path = path_config["win"]["raw_from_email"]
silver_root = path_config["database"]["silver_root"]
gold_root = path_config["win"]["gold_root"]
```

Mental model:

> *"If this breaks, the environment is wrong — not the data."*

---

### Stage 2 — Silver (Normalization Boundary)

**Responsibility:** Convert raw, chaotic inputs into a **canonical, trusted shape**.

Silver guarantees:

* Column names are standardized
* Required fields exist
* Types are reasonable (strings, numerics, dates)
* Dataset identity is explicit (SO, PO, etc.)

Silver does **not**:

* Apply business meaning
* Enrich semantics
* Join dimensions

Example call:

```python
process_raw_table(
    dataset="SO",
    cfg=read_cfg,
    today=today,
    raw_path=raw_path,
    silver_root=silver_root
)
```

Typical output logs:

```text
Loaded SO contract configs
SO columns standardized
SO Silver table saved with 4607 rows
```

Mental model:

> *"Anything downstream may trust the schema."*

---

### Stage 3 — Gold (Business Semantics)

**Responsibility:** Apply meaning, rules, and interpretation.

Gold operations include:

* Contract-based mappings
* Business rule enforcement
* Derived fields
* Dimensional table creation

Gold guarantees:

* Outputs are decision-ready
* Row counts are explainable
* Transformations are deterministic

Example call:

```python
gold_transformation(
    dataset="SO",
    silver_root=silver_root,
    gold_root=gold_root,
    today=today
)
```

Typical output logs:

```text
Read SO Silver table with 4607 rows
Applied SO contract mappings
Created date tables with 535 rows
Gold SO table written with 4607 rows
```

Mental model:

> *"Gold answers questions."*

## Conservation Laws (Non-Negotiable)

### 1. Row Count Conservation

Row count changes must be:

* intentional
* logged
* explainable

Unexpected drops or explosions are failures.

---

### 2. Directional Dependency

* Prep → Silver → Gold
* Never reverse
* Never skip

Gold must never fix Silver problems.

---

### 3. Determinism

Given the same inputs and configs:

* Outputs must be identical
* Logs must tell the full story

## Logging as a First-Class Output

Each stage must log:

* What it read
* What it assumed
* What it wrote

Logs are **part of the artifact**, not debugging noise.

## What This Structure Prevents

* Notebook sprawl
* Hidden business logic
* Fragile demos
* Refactor paralysis
* "I don’t remember how this works" syndrome

## When This Standard Is Mandatory

* Production pipelines
* Finance-facing data
* Demos
* Any system expected to live beyond a week
## When This Is Overkill

* One-off exploration
* Disposable analysis

## Status

**Gold Standard.**

Deviations must be intentional and documented.
