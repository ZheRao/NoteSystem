
# User Input Ingestion Methods
**(Authoritative Reference & Enforcement Contract)**

**Purpose**

Some datasets used by the data platform are **human-maintained inputs** (e.g. account classifications, pay periods, mappings).  
These inputs **do not belong to Bronze/Silver/Gold**, because they have fundamentally different failure modes and ownership.

This document defines:
- the **approved ingestion mechanisms**
- the **rules teams must follow**
- the **boundary of responsibility** between data engineering and input providers

Failure to follow these rules will result in **pipeline failure by design**.

---

**Design Principle**

> Human inputs are **control-plane data**, not system-of-record data.

They must therefore:
- live outside Bronze/Silver/Gold
- be versioned and validated
- have explicit ownership and accountability

## Option 1 — SharePoint Drop (Current, Enforced)

Status  
✅ Designing

This is the **only supported input method today**.

---

**Overview**

Business teams (Finance, HR, etc.) drop files into **predefined SharePoint folders**.
Files are synced to Windows and ingested into WSL **only if they comply with the defined contract**.

The pipeline **does not adapt to malformed inputs**.

---

**Architecture**

```text
SharePoint (Teams Folder)
        ↓
Local Sync (Windows)
        ↓
WSL Ingestion Script
        ↓
inputs_raw/  → validation →  inputs_curated/
```

---

**Folder Contract (SharePoint)**

Each dataset has a **fixed drop location**.

```text
Inputs/

    All/
        units.csv

    QBO/
        Finance/
            cube_classification.csv

    HR/
        pay_periods.csv

    Archive/
        old_files ...
```
Only files placed in the **correct folder** are ingested.

---

**File Naming Contract**

All **old**/**archive** input files **must** follow this pattern:

```text
<file_name>_effective_<YYY-MM-DD>.<ext>
```

Example:

```text
cube_classification_effective_2026-01-01.csv
```

---

**Ingestion Rules** — *for system*
- Files are **copied**, never modified in place
- Every ingestion produces:
    - immutable copy in `inputs_raw/`
    - checksum + timestamp
    - validation report
- Only validated inputs are promoted to `inputs_curated/`

---

**Responsibility Boundary**

**Data Engineering**
- Provides folder structure and validation rules
- Implements ingestion, versioning, and error reporting
- Does not manually fix malformed inputs

**Input Providers**
- Must follow folder, naming, and schema rules
- Own correctness of the business content
- Are responsible for fixing and resubmitting invalid inputs

> If validation fails, the pipeline stops.  
> This is intentional and non-negotiable.

## Option 2 — Power Apps + Dataverse / SharePoint List (Future Control Plane)

Status  
🟡 Not Designed

---

**Overview**

Instead of file uploads, users enter data through a **controlled UI** (Power Apps).  
Data is stored in Dataverse or a SharePoint List and consumed via API.

---

**Architecture**

```text
Power App UI
      ↓
Dataverse / SharePoint List
      ↓
API Connector
      ↓
inputs_curated/
```

---

**Advantages**
- Validation at data-entry time
- Strong audit trail (who changed what, when)
- Schema enforcement by design
- Reduced ingestion errors

---

**Tradeoffs**
- Higher setup and maintenance cost
- Requires UI ownership
- Slower to change than file-based workflows

## Non-Negotiable Rules (All Methods)

1. `inputs_raw/` is **immutable**
2. `inputs_curated/` is **pipeline-owned**
3. Humans never edit curated data directly
4. Validation failures are **contract violations**, not engineering bugs
5. Pipelines fail loudly by design