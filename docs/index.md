# Sierra MDM Platform — Architecture & Requirements

## Executive Summary

This document captures the architectural analysis and requirements derived from the SierraDock Master Data Management (MDM) initiative. The primary input was a master data object inventory identifying **83 SAP master data objects** across 10 functional modules, each classified by how they are currently maintained.

The analysis reveals three distinct management tiers that directly drive MDM architecture decisions:

| Tier | Count | Implication |
|---|---|---|
| **Centrally Managed in SAP** (Data Team) | 27 objects | SAP S/4HANA is lead system; MDM consumes via CPI |
| **Managed Locally in SAP** (Site Operations) | 25 objects | SAP is system of record; MDM needs site-scoped RBAC |
| **Not Managed in SAP at ND** | 31 objects | No current system of record; MDM becomes lead system |

!!! warning "Biggest MDM Opportunity"
    The **31 objects not currently managed in SAP** represent the highest-value MDM use case. These objects have no single system of record today and are prime candidates for MDM as the authoritative source.

---

## Scope

This document covers:

- Full inventory of master data objects by module and management status
- Architectural recommendations for lead system assignment
- Integration pattern guidance (SAP CPI / IDoc / direct)
- Role-based access control (RBAC) requirements driven by the central vs. local split
- Functional and non-functional requirements for the MDM platform

---

## Source Material

| Document | Description |
|---|---|
| `SAP Masterdata 1.pptx` | Master data object inventory with color-coded management classification |

### Legend (from source slide)

| Color | Classification |
|---|---|
| Orange (`accent3`) | Centrally Managed by Data Team |
| Light Blue (`#00B0F0`) | Managed Locally by Site Operations |
| White (`bg1`) | Not Managed in SAP at ND |

---

## Modules in Scope

| Module | SAP Module Code | In SAP? |
|---|---|---|
| CRM | CRM | No |
| EH&S | EHS | No |
| Finance | FI/CO | Yes |
| HR | HR/HCM | No |
| SCM / Purchasing | MM/SCM | Yes |
| Sales & Distribution | SD | Partial |
| Plant Maintenance | PM | Partial |
| Production Planning / Quality | PP/QM | Partial (Local only) |
| Project Systems | PS | Partial |
| Other (cross-module) | Various | Partial |

---

## Document Owners

| Role | Responsibility |
|---|---|
| MDM Architect | Architecture decisions and integration patterns |
| Data Team | Centrally managed object governance |
| Site Operations | Locally managed object governance |
| Functional Leads | Module-specific requirements validation |
