# Weekly Sales Analytics Dashboard

## Purpose

This dashboard is designed to **demonstrate senior-level analytical thinking**, not just visual creation. It showcases the ability to:

* Translate ambiguous business questions into structured analytical layers
* Balance executive readability with analytical depth
* Use interactivity to replace visual clutter
* Separate *trend detection* from *diagnostic analysis*

The design intentionally emphasizes **decision flow**, **progressive disclosure**, and **analytical rigor**.

### TL;DR

**Sales Overview**  
Top-down trend monitoring across multiple weeks, with the ability to investigate trends for a specific crop or location.

**Sales Detail**  
Decomposition of a selected week’s Year-over-Year sales across crops, locations, customer types, and individual customers.


## Core Business Questions Addressed

1. **Are we up or down vs last year, week by week?**
2. **Which locations and crops explain the change?**
3. **What is the sales mix by customer type, and who drives it?**
4. **How do these dimensions interact (location ↔ crop ↔ customer)?**

Each question is answered at the *appropriate analytical layer*, avoiding cognitive overload.

## Dashboard Architecture (Two-Level Analytical System)

### Level 1 — Weekly Sales Overview (Trend Detection)

**Primary Visual**

* Clustered bar chart
* X-axis: Week
* Y-axis: Sales
* Legend: Year (Year-over-Year comparison)

**Purpose**

* Immediate YoY trend recognition
* Fast executive scanning
* Establishes the analytical anchor for the dashboard

**Design Rationale**

* Clustered bars are optimal for YoY magnitude comparison
* No additional dimensions on canvas to preserve clarity

---

### Tooltip (Contextual Explanation, Not Comparison) - **disabled**

> Tooltip shows crop and location decomposition  
> Sales Detail already exists for decomposition  
> → Redundancy + extra cognitive load


Triggered on hover for a specific **week + year**.

**Tooltip Contents**

* Horizontal bar chart: Top sales locations (single-year breakdown)
* Horizontal bar chart: Top sales crops (single-year breakdown)

**Key Design Decision**

* Tooltips always focus on **one year only**
* Year-over-Year comparison is handled exclusively by the main visual

This avoids mixed mental models and preserves analytical clarity.

---

### Level 2 — Weekly Sales Detail (Diagnostic Analysis)

Activated by selecting a specific week from the overview.

## Section 1 — Location & Crop Diagnostics

### Visuals

1. **Stacked Horizontal Bar — Sales by Location**

   * Axis: Sales
   * Stacks: Year (YoY comparison)

2. **Stacked Horizontal Bar — Sales by Crop**

   * Same structure as location chart for symmetry

### Questions Answered

* Which locations grew or declined this week?
* Which crops are driving performance?
* Is performance concentrated or diversified?

### Design Principles

* Horizontal orientation improves label readability
* Stacked bars enable part-to-whole and YoY comparison simultaneously
* Visual symmetry reduces cognitive friction

## Section 2 — Sales Mix by Customer Type

### Visuals

1. **Donut Chart — Sales Mix (%)**

   * Customer Types:

     * Retail / Direct
     * Contract-price Wholesale
     * Open Market

2. **Stacked Horizontal Bar — Sales by Customer**

   * Breaks down contribution within each customer type

### Questions Answered

* What is the customer-type composition of sales?
* Which customers drive each mix category?

### Rationale

* Donut chart communicates high-level composition
* Bar chart restores magnitude and granularity
* Combined, they balance abstraction with specificity

## Interactivity & Analytical Power

All visuals are fully interactive:

* Selecting a **week** filters all detail visuals
* Selecting a **location** filters crops, customer mix, and customers
* Selecting a **crop** filters locations and customers

This enables hypothesis-driven exploration such as:

* “Is this crop only strong in one region?”
* “Is wholesale growth driven by one customer?”
* “Are declines broad-based or isolated?”

Interactivity replaces redundant visuals and elevates the dashboard from reporting to analysis.

## Analytical Design Principles Demonstrated

* **Separation of Concerns**: Trend vs diagnosis vs composition
* **Progressive Disclosure**: Overview → drill-down → interaction
* **Cognitive Load Management**: Minimal canvas, rich interaction
* **Executive Usability**: 3-second comprehension without hovering
* **Analytical Storytelling**: Visuals aligned to business questions

## What This Design Demonstrates Professionally

Beyond system building, this dashboard highlights:

* Analytical problem framing
* Business-oriented visualization design
* Executive communication skills
* Multi-dimensional data modeling intuition
* Advanced Power BI interaction patterns

This design reflects **analytics leadership capability**, not just technical execution.

## Intended Audience

* CFO / Finance Leadership
* Operations & Production Planning
* Sales Strategy & Commercial Teams

Designed to support both **high-level decisions** and **deep analytical inquiry** without redesign.
