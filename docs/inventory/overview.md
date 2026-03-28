# Master Data Inventory — Overview

## Summary by Module

The table below shows how many objects in each SAP module fall into each management tier.

| Module | Centrally Managed | Locally Managed | Not in SAP | Total |
|---|:---:|:---:|:---:|:---:|
| CRM | 0 | 0 | 7 | 7 |
| EH&S | 0 | 0 | 6 | 6 |
| Finance | 6 | 0 | 0 | 6 |
| HR | 0 | 0 | 12 | 12 |
| SCM / Purchasing | 11 | 2 | 0 | 13 |
| Sales & Distribution (SD) | 6 | 0 | 6 | 12 |
| Plant Maintenance (PM) | 5 | 9 | 0 | 14 |
| PP / Quality (QM) | 0 | 11 | 0 | 11 |
| Project Systems (PS) | 3 | 1 | 0 | 4 |
| Other (cross-module) | 4 | 0 | 5 | 9 |
| **Total** | **35** | **23** | **36** | **94** |

!!! note "Count Note"
    Some objects appear in multiple modules (e.g., Bill of Materials in both PM and PP/QM, Catalogs in both PM and Quality). Raw object+module combinations total 94; unique object names total 83.

---

## Summary by Tier

### Tier 1 — Centrally Managed in SAP (Data Team)
> SAP S/4HANA is the system of record. The central Data Team owns creation and maintenance.

**Key modules:** Finance, SCM, SD, PM, PS

**MDM Role:** Consumer / subscriber. MDM replicates from SAP via CPI iFlows or IDocs.

---

### Tier 2 — Locally Managed in SAP (Site Operations)
> SAP S/4HANA is the system of record. Individual sites maintain their own records.

**Key modules:** PM, PP/QM, PS, SCM (condition records)

**MDM Role:** Aggregator / visibility layer. MDM provides cross-site reporting and governance without overriding local authority.

---

### Tier 3 — Not Managed in SAP at ND
> No current system of record. Data is managed in spreadsheets, local systems, or not at all.

**Key modules:** CRM, EH&S, HR — entirely outside SAP. SD and Other — partially outside.

**MDM Role:** Lead system / system of record. MDM becomes authoritative; downstream systems (including SAP where relevant) consume from MDM.

---

## Visual Distribution

```
Centrally Managed ████████████████████░░░░░░░░░░░░░░░░░░░░  35 objects
Locally Managed   ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  23 objects
Not in SAP        ████████████████████░░░░░░░░░░░░░░░░░░░░  36 objects
```
