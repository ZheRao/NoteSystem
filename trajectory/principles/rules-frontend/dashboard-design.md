# The 3-Question Dashboard Design Framework

## 1️⃣ What decisions should this dashboard help you make?

This is the **most important question**.

Bad dashboards answer:
> “What happened?”

Good dashboards answer:
> “What should I do next?”

Examples for PATH:
- Should we **increase or reduce orders from a supplier?**
- Are we **over-ordering or under-ordering inventory?**
- Which products have **slow turnover?**
- Which suppliers are **delivering late or inconsistently?**

This question forces the system to support **action**, not just visibility.

## 2️⃣ What signals tell you something is wrong?

Every operational system should surface **exceptions**.

Operators don’t want to stare at dashboards all day — they want to know:
> “Where should I look?”

Examples:
- Purchase order delayed beyond expected lead time
- Supplier price suddenly increased
- Inventory building up faster than sales
- Orders not matched to sales orders

This is where **alerts / anomaly views** often come from.

## 3️⃣ What comparisons matter?

Raw numbers rarely matter by themselves.

Operators usually care about **relative context**:

Examples:
- This week vs last week
- Supplier A vs Supplier B
- Ordered vs Sold
- Forecast vs Actual
- Inventory vs Sales velocity

This question usually reveals the **core analytical structure** of the dashboard.

## Example: Applying this to PATH Purchase Orders

If Dan answers these questions well, you might discover the system should show things like:

**Supplier Performance**
- PO quantity by supplier
- Lead time distribution
- Late deliveries

**Inventory Flow**
- Ordered vs sold quantities
- Days of inventory
- Purchase velocity vs sales velocity

**Cost Signals**
- Price changes by supplier
- Average cost trends

Notice how all of these come from **decisions, signals, comparisons**.

## Why this framework works

Most people ask:
> “What dashboards do you want?”

This is actually a **very hard question** for operators.

But everyone can answer:
- What decisions they make
- What problems they watch for
- What comparisons they care about

From those answers, **the dashboard design almost falls out automatically**.

## A deeper connection to your thinking

This framework actually mirrors something you already discovered in your **intelligence architecture ideas**.

Your systems thinking often follows this pattern:

***Perception → Signal → Decision***

Which is exactly what good dashboards do:

***Raw Data → Signals → Actionable Insight***

This is why the work you're doing in ETL / pipelines / dashboards is actually **excellent training for your larger intelligence ideas**. 
You're learning how to build **perception systems for organizations**.

And remember your own invariant you discovered earlier:

***ETL + dashboards = organizational perception layer***

PATH is literally the **nervous system for the produce operation.**

## One extra tip (this one is gold)

When Dan answers your questions, ask one more thing:

***“What would make you check this dashboard every morning?”***

That question often reveals the **one metric that actually matters**.

Every great operational dashboard has one number that operators care about first.