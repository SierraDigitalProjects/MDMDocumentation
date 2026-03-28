# Sierra Aura — MDM Platform

Sierra Aura is a Master Data Management platform, built to establish a single source of truth for master data across all business systems and operational sites.

Today, **83 master data objects** are spread across 10 SAP functional modules — maintained by different teams, in different systems, with no shared standard and no reliable mechanism to keep downstream consumers in sync. Sierra Aura resolves this by providing a governed, event-driven platform where every object has a defined owner, every change is traceable, and every system receives updates automatically.

---

## Management Tiers

Sierra Aura organises master data into three tiers, each reflecting how data is owned and where the system of record lives.

| Tier | Objects | What this means |
|---|:---:|---|
| **Centrally Managed in SAP** | 35 | SAP S/4HANA is authoritative. The central Data Team governs creation and changes. Sierra Aura replicates inbound via CPI. |
| **Locally Managed in SAP** | 23 | SAP is the system of record, but individual sites maintain their own records. Sierra Aura provides cross-site visibility and governance. |
| **Not in SAP (MDM Lead)** | 36 | No current system of record exists. Sierra Aura becomes the authoritative source — downstream systems, including SAP where relevant, consume from it. |

!!! tip "Highest-Value Opportunity"
    The **36 objects with no current system of record** — spanning CRM, EH&S, and HR — represent the biggest governance gap. These are managed today in spreadsheets or informal processes. Sierra Aura gives them a home.

---

## What This Site Covers

This documentation is the definitive reference for Sierra Aura. It covers:

- **Goals & Benefits** — what the platform is designed to achieve and the value it delivers
- **Scope** — the full inventory of 83 master data objects classified by module and tier
- **Requirements** — functional and non-functional requirements, plus open questions
- **Architecture** — lead system strategy, integration patterns, RBAC, and data enrichment governance
- **Technical Stack** — platform decisions across compute, event broker, data layer, and observability
- **Operations** — autonomous monitoring, alerting, self-healing, and cost of operations
- **AI in MDM** — how AI accelerates development and reduces operational overhead

---

## Modules in Scope

| Module | SAP Code | SAP Managed? |
|---|---|---|
| Customer Relationship Management | CRM | No — MDM lead |
| Environment, Health & Safety | EHS | No — MDM lead |
| Finance & Controlling | FI/CO | Yes — centrally managed |
| Human Resources | HR/HCM | No — MDM lead |
| Supply Chain & Purchasing | MM/SCM | Yes — centrally managed |
| Sales & Distribution | SD | Partial |
| Plant Maintenance | PM | Partial |
| Production Planning & Quality | PP/QM | Yes — locally managed |
| Project Systems | PS | Partial |
| Cross-module | Various | Mixed |

---

## Document Owners

| Role | Responsibility |
|---|---|
| MDM Architect | Architecture decisions, integration patterns, platform design |
| Data Team Lead | Centrally managed object governance and data stewardship |
| Site Operations | Locally managed object maintenance and coordinator oversight |
| Functional Leads | Module-specific requirements validation and sign-off |
