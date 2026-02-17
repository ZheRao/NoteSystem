# FinanceConsolidated – Actual vs. Actual Power BI System

## 1. Purpose

This document describes the **Actual vs. Actual** financial comparison framework implemented in Power BI.

The system allows users to:

* Select **any two fiscal years**
* Automatically compare **Actuals vs. Actuals** across the entire report
* Maintain **bounded, reusable logic** in the semantic model
* Avoid hard‑coded year flags (e.g., `CurrentFY` switches)

The design emphasizes:

* Semantic correctness
* Minimal report‑level logic
* Predictable fiscal‑year behavior (Nov–Oct fiscal calendar)

## 2. Core Design Principles

### 2.1 Two‑Year Selection Model

* Users select **two fiscal years** via slicer
* Semantic model derives:

  * `SelectedYearMax` → comparison "current"
  * `SelectedYearMin` → comparison "prior"
* All YoY logic flows from these two values

### 2.2 Bounded Measures

* **Time boundaries live in the semantic model**
* Reports consume measures without re‑implementing date logic
* Prevents duplicated logic and report drift

### 2.3 Fiscal‑First Calendar

* Fiscal year starts **November 1**
* Month ordering is handled explicitly
* Calendar year conversion is deterministic

## 3. Semantic Model – Date Table Measures

### 3.1 Selected Fiscal Years

```DAX
SelectedYearMax = MAX(DateTable[Fiscal Year])
SelectedYearMin = MIN(DateTable[Fiscal Year])
```

These measures establish the comparison window and are reused throughout the model.

---

### 3.2 EndDateSelected (Fiscal‑Aware)

Determines the **effective reporting end date** based on user month selection and fiscal rules.

```DAX
EndDateSelected =
VAR _maxFY = [SelectedYearMax]
VAR _max_month_num = MAX(DateTable[Month Number])
VAR _calendar_year = IF(
    _max_month_num <= 2,
    _maxFY - 1,
    _maxFY
)
VAR _max_month_normal = LOOKUPVALUE(
    DateTable[Month Number (Normal)],
    DateTable[Month Number],
    _max_month_num
)
RETURN
EOMONTH(DATE(_calendar_year, _max_month_normal, 1), 0)
```

**Why this exists:**

* Fiscal months Nov–Dec belong to the *previous* calendar year
* Ensures correct FYTD alignment when slicing months

---

### 3.3 CurrentFY (Reference Only)

```DAX
CurrentFY = LOOKUPVALUE(
    DateTable[Fiscal Year],
    DateTable[Date],
    TODAY()
)
```

Used strictly for reference or UI logic — not for calculations.

## 4. Report Layer – ProfitTypeSummary (Actual Overview)

### 4.1 Amount by Selected Years

**Max Year (Current)**

```DAX
AmountMaxYear =
VAR _maxFY = [SelectedYearMax]
RETURN
CALCULATE(
    SUM(ProfitTypeSummary[AmountDisplay]),      // AmountDisplay shows all amounts as positive
    ProfitTypeSummary[FiscalYear] = _maxFY,
    ProfitTypeSummary[DataType] = "Actual"
)
```

**Min Year (Prior)**

```DAX
AmountMinYear =
VAR _minFY = [SelectedYearMin]
RETURN
CALCULATE(
    SUM(ProfitTypeSummary[AmountDisplay]),
    ProfitTypeSummary[FiscalYear] = _minFY,
    ProfitTypeSummary[DataType] = "Actual"
)
```

---

### 4.2 YoY Percentage Change

```DAX
%ΔYoY =
VAR _current = [AmountMaxYear]
VAR _past = [AmountMinYear]
RETURN
DIVIDE(_current - _past, ABS(_past), 0)
```

* Normalized by absolute prior value
* Prevents sign inversion for negative bases

---

### 4.3 Total Sales (Row‑Context Safe)

```DAX
TotalSales =
VAR _maxFY = [SelectedYearMax]
RETURN
CALCULATE(
    SUM(ProfitTypeSummary[AmountDisplay]),
    REMOVEFILTERS(ProfitTypeSummary[ProfitType]),
    REMOVEFILTERS(ProfitTypeSummary[ProfitTypeOrder]),
    ProfitTypeSummary[ProfitType] = "Sales Revenue",
    ProfitTypeSummary[FiscalYear] = _maxFY,
    ProfitTypeSummary[DataType] = "Actual"
)
```

Ensures denominator consistency regardless of row context.

---

### 4.4 % of Sales

```DAX
% of Sales =
VAR _maxFY = [SelectedYearMax]
VAR _total = CALCULATE(
    SUM(ProfitTypeSummary[AmountDisplay]),
    ProfitTypeSummary[FiscalYear] = _maxFY,
    ProfitTypeSummary[DataType] = "Actual"
)
VAR _sales = [TotalSales]
RETURN
_total / _sales
```

---

### 4.5 Conditional Formatting – Summary YoY

```DAX
Summary%Δ Color =
VAR v = [%ΔYoY]
VAR pt = SELECTEDVALUE(ProfitTypeSummary[ProfitType])
VAR threshold = 0.5
VAR sign =
    SWITCH(
        TRUE(),
        pt IN {
            "Direct Operating Expenses",
            "Operating Overheads",
            "Cost of Goods Sold",
            "Other Expense"
        }, -1,
        1
    )
VAR adj = v * sign
RETURN
IF(
    ISBLANK(v), BLANK(),
    SWITCH(
        TRUE(),
        adj > threshold * 3, "#3FD41D",
        adj > threshold * 2, "#65E647",
        adj > threshold, "#9EF38B",
        adj > 0, "#C6F2BC",
        adj < -threshold * 3, "#EE084E",
        adj < -threshold * 2, "#F22C69",
        adj < -threshold, "#FA5F8F",
        adj < 0, "#FEBED2",
        "#E6E6E6"
    )
)
```

**Key idea:** expense increases are bad → sign‑corrected before color logic.

---

### 4.6 Per‑Unit Summary

```DAX
SummaryPerUnit =
VAR _maxFY = [SelectedYearMax]
RETURN
CALCULATE(
    DIVIDE(
        SUM(ProfitTypeSummary[AmountDisplay]),
        SUM(Unit_PowerBI[Unit]),
        0
    ),
    ProfitTypeSummary[FiscalYear] = _maxFY,
    ProfitTypeSummary[DataType] = "Actual"
)
```

## 5. Transaction‑Level Measures (Actuals)

### 5.1 FYTD

```DAX
AmountFYTD =
VAR _end = [EndDateSelected]
VAR _start = DATE(
    YEAR(_end) - IF(MONTH(_end) < 11, 1, 0),
    11,
    1
)
RETURN
CALCULATE(
    SUM(Transactions[AmountCAD]),                   // AmountCAD preserves positive for revenue and negative for expense
    DATESBETWEEN(DateTable[Date], _start, _end)
)
```

---

### 5.2 FYTD – Last Month

```DAX
EndDateMonthPrior = 
VAR _currentEnd = [EndDateSelected]
RETURN
EOMONTH(_currentEnd,-1)
```

```DAX
AmountFYTDLastMonth =
VAR _end = [EndDateMonthPrior]
VAR _start = DATE(
    YEAR(_end) - IF(MONTH(_end) < 11, 1, 0),
    11,
    1
)
RETURN
CALCULATE(
    SUM(Transactions[AmountCAD]),
    DATESBETWEEN(DateTable[Date], _start, _end)
)
```

---

### 5.3 Prior‑Year‑To‑Date

```DAX
AmountPYTD =
VAR _end = DATE(
    [SelectedYearMin],
    MONTH([EndDateSelected]),
    DAY([EndDateSelected])
)
VAR _start = DATE(
    YEAR(_end) - IF(MONTH(_end) < 11, 1, 0),
    11,
    1
)
RETURN
CALCULATE(
    SUM(Transactions[AmountCAD]),
    DATESBETWEEN(DateTable[Date], _start, _end)
)
```

---

### 5.4 Per‑Unit Metrics

```DAX
AmountPerUnitMax =
VAR _maxFY = [SelectedYearMax]
RETURN
CALCULATE(
    DIVIDE(
        SUM(Transactions[AmountDisplay]),
        MAX(Unit_PowerBI[Unit]),
        0
    ),
    DateTable[Fiscal Year] = _maxFY
)
```

```DAX
AmountPerUnit =
DIVIDE(
    SUM(Transactions[AmountCAD]),
    MAX(Unit_PowerBI[Unit]),
    0
)
```

---

### 5.5 Month‑Over‑Month %Δ

```DAX
Amount%Δ =
VAR _curr = [AmountFYTD]
VAR _prev = [AmountFYTDLastMonth]
VAR pt = SELECTEDVALUE(Account[ProfitType])
VAR sign =
    SWITCH(
        TRUE(),
        pt IN {
            "Direct Operating Expenses",
            "Operating Overheads",
            "Cost of Goods Sold",
            "Other Expense"
        }, -1,
        1
    )
RETURN
DIVIDE((_curr - _prev) * sign, ABS(_prev), 0)
```

---

### 5.6 Conditional Formatting – Transaction Level

```DAX
%Δ Color =
VAR v = [Amount%Δ]
VAR pt = SELECTEDVALUE(Account[ProfitType])
VAR threshold = 0.5
VAR sign =
    SWITCH(
        TRUE(),
        pt IN {
            "Direct Operating Expenses",
            "Operating Overheads",
            "Cost of Goods Sold",
            "Other Expense"
        }, -1,
        1
    )
VAR adj = v * sign
RETURN
IF(
    ISBLANK(v), BLANK(),
    SWITCH(
        TRUE(),
        adj > threshold * 3, "#3FD41D",
        adj > threshold * 2, "#65E647",
        adj > threshold, "#9EF38B",
        adj > 0, "#C6F2BC",
        adj < -threshold * 3, "#EE084E",
        adj < -threshold * 2, "#F22C69",
        adj < -threshold, "#FA5F8F",
        adj < 0, "#FEBED2",
        "#E6E6E6"
    )
)
```

## 6. Result

* Single semantic model supports **all Actual vs. Actual comparisons**
* No hard‑coded fiscal logic in visuals
* Fiscal calendar behaves consistently across reports
* Model is extensible to **Budget vs Actual** or **Multi‑Year** comparisons

*This system forms the foundation of a scalable, perception‑first finance nerve system.*

## 7. Budget Overview Consolidated – Budget vs. Actual Framework

### 7.1 Purpose

The **Budget Overview Consolidated** report extends the finance semantic model to support **dynamic Budget vs. Actual comparisons** without relying on fiscal flags or hard‑coded year logic.

The system allows users to:

* Select **any fiscal year for Budget**
* Select **any fiscal year for Actuals**
* Automatically compute variance across the report
* Calibrate forward budgets (e.g., compare **2026 Budget vs. 2025 Actual**)
* Maintain a clean, flag‑free semantic model

This design reinforces the platform’s architectural direction: **bounded logic lives in the semantic layer — not in report visuals.**

---

### 7.2 Core Design Principles

#### Independent Year Selection

Unlike the Actual vs. Actual model (which derives min/max years), Budget comparisons require **orthogonal selection paths**:

* Budget year is derived from the **Budget table**
* Actual year continues to flow from the **Date semantic model** (`SelectedYearMax`)

This separation prevents cross-filter ambiguity and keeps both measures deterministic.

#### No Gold Flags

The system intentionally removes constructs such as:

```
current_fy
is_current_year
budget_switch
```

Hard-coded governance flags create rigidity and force model rewrites each fiscal cycle. Their removal ensures:

* Forward compatibility
* Cleaner DAX
* Reduced model entropy

---

### 7.3 Semantic Model – Root Measure

```DAX
SelectedYearBudget = MAX(Budget[FiscalYear])
```

**Role:**

* Establishes the active budget year
* Mirrors the behavior of `SelectedYearMax`
* Enables symmetric measure construction

## 7.4 ProfitTypeSummary – Budget vs. Actual Measures

This layer powers the **executive summary view**, where accounting types aggregate into profit categories.

---

### Total Actual

```DAX
TotalActual =
VAR _year = [SelectedYearMax]
RETURN
CALCULATE(
    SUM(ProfitTypeSummary[AmountDisplay]),
    ProfitTypeSummary[FiscalYear] = _year,
    ProfitTypeSummary[DataType] = "Actual"
)
```

---

### Total Budget

```DAX
TotalBudget =
VAR _year = [SelectedYearBudget]
RETURN
CALCULATE(
    SUM(ProfitTypeSummary[AmountDisplay]),
    ProfitTypeSummary[FiscalYear] = _year,
    ProfitTypeSummary[DataType] = "Budget"
)
```

---

### Budget vs. Actual %Δ

```DAX
TotalBudgetActual%Δ =
VAR _actual = [TotalActual]
VAR _budget = [TotalBudget]
RETURN DIVIDE(_actual-_budget, ABS(_budget), 0)
```

**Normalization by absolute budget** prevents sign distortion when expense categories carry negative bases.

---

### Per‑Unit Metrics

#### Actual Per Unit

```DAX
TotalActualPerUnit =
VAR _year = [SelectedYearMax]
RETURN
CALCULATE(
    DIVIDE(
        SUM(ProfitTypeSummary[AmountDisplay]),
        SUM(Unit_PowerBI[Unit]),
        0
    ),
    ProfitTypeSummary[FiscalYear] = _year,
    ProfitTypeSummary[DataType] = "Actual"
)
```

---

#### Budget Per Unit

```DAX
TotalBudgetPerUnit =
VAR _year = [SelectedYearBudget]
RETURN
CALCULATE(
    DIVIDE(
        SUM(ProfitTypeSummary[AmountDisplay]),
        SUM(Unit_PowerBI[Unit]),
        0
    ),
    ProfitTypeSummary[FiscalYear] = _year,
    ProfitTypeSummary[DataType] = "Budget"
)
```

**Architectural Note:**

Per‑unit symmetry ensures that operational scale changes do not masquerade as financial variance.

## 7.5 Budget Table – Transaction-Level Aggregations

The **Budget table** stores monthly totals across individual accounts for both Budget and Actual data types, enabling FYTD-style rollups.

---

### Budget FYTD

```DAX
BudgetFYTD =
VAR _year = [SelectedYearBudget]
RETURN
CALCULATE(
    SUM(Budget[AmountCAD]),
    Budget[DataType] = "Budget",
    Budget[FiscalYear] = _year
)
```

---

### Actual FYTD

```DAX
ActualFYTD =
VAR _year = [SelectedYearMax]
RETURN
CALCULATE(
    SUM(Budget[AmountCAD]),
    Budget[DataType] = "Actual",
    Budget[FiscalYear]=_year
)
```

---

### Base Measures

```DAX
Actual =
CALCULATE(
    SUM(Budget[AmountCAD]),
    Budget[DataType] = "Actual",
)
```

```DAX
Budget =
CALCULATE(
    SUM(Budget[AmountCAD]),
    Budget[DataType] = "Budget"
)
```

These base measures support reusable slicing patterns across visuals.

---

### FYTD Budget vs. Actual %Δ

```DAX
BudgetActual%Δ =
VAR _actual = [ActualFYTD]
VAR _budget = [BudgetFYTD]
RETURN DIVIDE(_actual-_budget, ABS(_budget), 0)
```

## 7.6 Architectural Outcome

The Budget consolidation introduces a **dual-year financial perception system** capable of supporting:

* Forward planning
* Budget calibration
* Executive variance analysis
* Scenario comparisons

Most importantly, it achieves this **without introducing semantic fragility.**

### Structural Gains

* Eliminates fiscal switches
* Preserves measure symmetry
* Keeps time logic bounded
* Enables future extensions (Forecast vs Actual, Multi-scenario planning)

## 7.7 Position in the Finance Nerve System

With this addition, the semantic model now supports:

* Actual vs Actual
* Budget vs Actual

from the same architectural foundation.

This continues the evolution toward a **perception-first financial nerve system** — where leadership observes reality through stable abstractions rather than brittle report logic.

The model is now materially closer to supporting true organizational intelligence.

