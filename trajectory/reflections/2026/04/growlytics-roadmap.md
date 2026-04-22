# Growlytics Platform Roadmap

**Guiding Principles**
- Build data foundation first to ensure accuracy and trust
- Deliver incremental value early (forecast as first product)
- Evolve toward a scalable multi-service platform
- Separate clearly:
    - Data System (ETL)
    - Serving System (API)
    - Interface Layer (BI → Web App)
- Establish a scalable data and access layer to enable reliable integration between data systems and user-facing applications
    - Ensure all outputs are accessible through structured systems (not file-based workflows), enabling future web and client-facing experiences

## Two Delivery Strategies

**Option A — Forecast Product First (Faster Visibility)**

**Option B — Platform Foundation First (Stronger System)**

## Infrastructure Consideration (Important)
---

To support a client-facing platform, data must be accessible through structured systems rather than file-based workflows.

While tools like Excel are useful for initial exploration, they do not provide:
- reliable integration with applications
- real-time or controlled data access
- scalable multi-user support

As a result, part of the roadmap focuses on establishing a **data and serving layer** that allows:
- consistent access to data
- integration with BI tools and web applications
- a foundation for future product capabilities

## OPTION A — Forecast Product First
---

### Phase 1 — Forecast Data Foundation (2 months)

**Data System (Backend ETL)**
- Define standardized forecast data model:
    - cash flow structure
    - inflow / outflow categories
    - cost & revenue drivers
- Build ingestion pipelines:
    - source data (e.g., QBO)
    - Excel/manual uploads
    - this is the entry point from raw data into the system
- Normalize and store:
    - transactional data
    - forecast input tables

**Milestone**

✅ **Reliable, reproducible forecast dataset from `bronze` storage layer to `silver`**

→ **Business Outcome**
- Forecast inputs are centralized and standardized, eliminating fragmented Excel sources
- Data becomes consistently usable across analyses without manual reconciliation
- Foundation established for repeatable forecasting across clients

---
### Phase 2 — Forecast Logic & Modeling (3 months)

**Data System**
- Understand and implement core logic:
    - cash balance roll-forward
    - forecast projections
    - risk indicators (e.g. lowest cash point)
- Establish reusable logic layer (separate from UI)

**Milestone**

✅ **Forecast outputs generated consistently from raw inputs to `gold` storage layer**

→ **Business Outcome**
- Forecast calculations become consistent, traceable, and no longer dependent on manual Excel formulas
- Stakeholders can trust outputs due to transparent and reproducible logic
- New scenarios (e.g., cost or revenue changes) can be evaluated without rebuilding models

---
### Phase 3 — Initial Interface (Power BI) (1 month) — Month 6 Mark

**Interface Layer**
- Build initial dashboard:
    - cash flow forecast
    - KPI summary (cash position, risk)
    - cost & revenue drivers
    - key events

**Milestone**

✅ **Internal stakeholders can view and interact with forecast outputs**

→ **Business Outcome**
- Stakeholders gain immediate visibility into cash flow forecasts and key drivers
- Decision-making shifts from static spreadsheets to interactive exploration
- Early validation of forecast usefulness before investing in full product development

---
### Phase 3.5 → Phase 4 — Infrastructure Foundation (2-3 months)

**Serving System (Infrastructure Foundation)**

This phase establishes the required foundation for transitioning from internal analysis tools to a client-facing application.

This includes defining:
- where each stage of data resides (raw, processed, and modeled layers)
- how the server layer interacts with the underlying data systems
- where the server layer is hosted and executed

These decisions form the foundation for how frontend applications can reliably communicate with backend systems, 
and are essential for enabling a responsive, secure, and scalable user experience.

**Milestone**

✅ **Determine platform infrastructure, documenting feasibility, trade off, and estimated cost breakdown**

→ **Business Outcome**
- Clear architecture defined for how data, logic, and applications connect
- Reduces risk of rework when transitioning to a client-facing product
- Establishes foundation for scalable, multi-client deployment
---
### Phase 4 — Serving Layer Introduction (3 months) — Year 1 Mark

**Serving System**

- Introduce lightweight API layer
- Enable:
    - structured querying
    - controlled data access
    - future frontend compatibility

**Milestone**

✅ **Data becomes application-ready (not just BI-ready)**

→ **Business Outcome**
- Data becomes accessible through structured systems rather than manual exports
- Enables integration with applications, automation, and future services
- Reduces reliance on BI tools as the only access point

---
### Phase 5 — Frontend Application (Client-facing) (4 months) — Month 16 Mark

**Interface Layer**
- Develop web-based dashboard
- Introduce:
    - navigation
    - interactivity
    - user-specific views
- Critical components before launch:
    - clients' data isolation
    - clients' authentication workflow (may use third party - `Clark`)

**Milestone**

✅ **Client-facing product experience ready — Forecast Dashboard**  
✅ **Full platform experience begins to emerge (with more system integration)**

→ **Business Outcome**
- Clients can directly access and interact with their forecast data
- Eliminates manual report delivery and improves client experience
- Creates foundation for expanding into additional services beyond forecasting

---
### Summary of Option A
- Early visibility achieved within first 5–6 months (Power BI)
- Client-facing product delivered in ~12–16 months
- Reduces future development effort through reusable data and logic layers
- May require refinement as system expands

## OPTION B — Platform Foundation First
---

### Phase 1 — Unified Data System (4-6 months)

**Data System**
- Design unified schema:
    - transactions
    - dimensions (location, crop, etc.)
- Standardize ingestion pipelines across sources
- Normalize all data into structured layers

**Milestone**

✅ **Establish centralized data/knowledge hub that enables all possible downstream analysis, even cross-source consolidated analysis, with a single source of truth**  
✅ **Any analysis can be done based on this data pool**

→ **Business Outcome**
- All operational and financial data becomes centralized and consistent
- Eliminates discrepancies across reports and departments
- Enables cross-source and cross-client analysis

---
### Phase 2 — Business Logic Layer (6 months) — Year 1 Mark

**Data System**
- Build reusable logic layer:
    - metrics
    - aggregations
    - benchmarking
- Separate business logic from reporting tools

**Milestone**

✅ **Build `assembly lines` from Excel logic, massively expand consistency, traceability, and scalability**  
✅ **New clients = new input data + reused/new logic**  
✅ **Different year = rotating system parameters (e.g., `fiscal_year=2025` → `fiscal_year=2026`)**

→ **Business Outcome**
- Core business calculations become standardized and reusable
- New analyses and models can be created without rebuilding logic from scratch
- System becomes scalable to new clients with significantly reduced setup effort

---
### Phase 3-5 — similar to Option A and Forecast only (~ 11 months)
- Phase 3 — Initial Interface (Power BI) (1 month)
- Phase 3.5 — Infrastructure (2-3 months)
- Phase 4 — Serving Layer Introduction (3 months)
- Phase 5 — Frontend Application (Client-facing) (4 months)

**Milestone**

✅ **Client-facing product experience ready — Forecast Dashboard**  
✅ **Full platform experience begins to emerge (expansion much faster than Option A)**

→ **Business Outcome**
- Faster expansion into new services due to strong underlying system
- Reduced long-term maintenance and operational overhead
- Ability to deliver consistent, scalable client-facing products

---
### Summary of Option B
- Slower visible output (~ 2 years - Forecast model (1st client-facing system))
- Major Excel pain relieved after Year 1
- Enable taking on more clients without degrading quality
- Reduces machinery significantly and enables consolidated analysis

## Important Notes
- Data standardization is the **critical path** for all features
- Initial phases focus on **forecast to validate the model**
- Architecture will evolve as additional services are introduced
- Timeline may adjust based on:
    - data quality
    - integration complexity
    - scope refinement