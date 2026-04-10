
# Reflection — Separating Planned vs. Unplanned Financial Variance

## Context
A financial tracking system was designed with the following structure:

- `balance` → total assets  
- `debt` → liabilities  
- `buffer` → emergency reserve  
- `illiquid_fund` → locked capital (e.g., GIC)  
- `planned_fund` → earmarked for known future expenses  
- `future_fund` → derived residual  

> `future_fund = balance - debt - buffer - illiquid_fund - planned_fund`

Additionally:
- `monthly_delta` tracks change in `future_fund` over time



## Key Insight

Planned expenditures are **pre-accounted** through `planned_fund`.

Therefore:

> When a planned expense is executed:
> - `balance ↓`
> - `planned_fund ↓`
> - `future_fund ≈ unchanged`

This creates a critical property:

> **Planned spending produces no signal in `future_fund`.**



# System Property

This design separates financial changes into two categories:

## 1. Planned Variance (Expected, Silent)
- Already deducted via `planned_fund`
- Execution does not affect `future_fund`
- No action required

## 2. Unplanned Variance (Unexpected, Visible)
- Appears as deviation in:
  - `future_fund`
  - `monthly_delta`
- Always requires explanation



## Monitoring Invariant

> `future_fund` should increase steadily over time (based on income - baseline spending)

Therefore:

- Stable or increasing → system behaving normally  
- Sudden decrease → **unexpected outflow or misclassification**  
- Deviation from expected `monthly_delta` → **investigate**



## System Insight

This system effectively:

> **filters out known noise and amplifies unknown signals**

- Planned expenses → removed from monitoring layer  
- Unexpected changes → immediately visible  

This is analogous to:
- removing known variance in data pipelines  
- isolating anomalies for investigation  



## Design Principle

> **Account for expected changes upfront so that only unexpected changes generate signals**

This avoids:
- false alarms from planned activity  
- cognitive overload from interpreting normal fluctuations  



## Practical Outcome

- No need to mentally track multiple categories during spending  
- No ambiguity about whether a purchase is “safe”  
- No drift from hidden or accumulated small expenses  

All decisions reduce to:

> **Does this affect `future_fund`?**



## Extension Opportunities

- Add `expected_monthly_delta` and compare with actual  
- Define thresholds for anomaly detection  
- Tag deviations with reasons (log-based audit trail)  
- Track rolling average of `monthly_delta` for trend validation  



## Meta Insight

This marks a transition from:

> “Tracking money manually”

to:

> **Operating a system where deviations from expected behavior are automatically surfaced**



## Closing Thought

> A well-designed system does not just track state —  
> it makes **unexpected changes impossible to ignore**.