# Open Questions

This page tracks architectural and requirements questions that need resolution before detailed design can proceed. Each question is assigned a priority and an owner.

---

## Architecture Questions

| # | Question | Priority | Owner | Status |
|---|---|---|---|---|
| AQ-01 | Which MDM platform is selected (SAP MDG, Informatica, Reltio, Semarchy, custom)? | Critical | MDM Architect | Open |
| AQ-02 | Is SAP Integration Suite (CPI) confirmed as the sole integration middleware, or are direct API connections permitted? | High | Integration Architect | Open |
| AQ-03 | What is the SAP S/4HANA release version? This affects available IDoc types and BAPI coverage. | High | SAP Basis | Open |
| AQ-04 | Is the Business Partner unified model (S/4HANA BP) fully activated? Some deployments still use separate Customer/Vendor master. | High | SAP Functional | Open |
| AQ-05 | What is the multi-site architecture in SAP — single client, multiple clients, or system landscape with multiple S/4HANA systems? | High | SAP Basis | Open |
| AQ-06 | For locally managed objects, does "site" map to SAP Plant, Company Code, or a custom site dimension? | High | SAP Functional | Open |
| AQ-07 | Is there an existing enterprise MDM or reference data management tool that MDM would replace or co-exist with? | Medium | Enterprise Architect | Open |

---

## Data and Integration Questions

| # | Question | Priority | Owner | Status |
|---|---|---|---|---|
| DQ-01 | What is the current system of record for CRM data — Salesforce, SAP CRM, or another tool? | Critical | CRM Team | Open |
| DQ-02 | What EH&S platform is currently in use (Cority, Enablon, spreadsheets)? | High | EH&S Team | Open |
| DQ-03 | Is Workday the HR system of record, or is SAP HCM still in use? PU12 Export Status field suggests possible SAP HCM presence. | High | HR Team | Open |
| DQ-04 | For Exchange Agreements and Sales BOMs (currently not in SAP) — where does this data currently live? | High | SD/Sales Team | Open |
| DQ-05 | What is the estimated record volume per object type (Material Master, Business Partner, Equipment)? | High | Data Team | Open |
| DQ-06 | Is there an existing data migration plan for the 31 non-SAP objects? Who is responsible for initial data cleansing? | High | MDM Program Lead | Open |
| DQ-07 | Do Purchasing Condition Records and Pricing Condition Records require near-real-time sync (< 5 min) or is daily delta sufficient? | Medium | SCM/SD Teams | Open |
| DQ-08 | Are Material Master Rev Levels tracked in an engineering change management (ECM) tool, or manually? | Medium | Engineering/MM Team | Open |

---

## Governance Questions

| # | Question | Priority | Owner | Status |
|---|---|---|---|---|
| GQ-01 | How many sites are in scope? What is the expected number of Site Data Coordinators? | High | MDM Program Lead | Open |
| GQ-02 | Who is the designated Domain Steward for each non-SAP domain (CRM, EH&S, HR)? | High | Business Sponsors | Open |
| GQ-03 | Is there an enterprise IdP (Azure AD, Okta) for SSO integration, or does MDM need its own user management? | High | IT / Security | Open |
| GQ-04 | What are the regulatory drivers for EH&S data (OSHA, EPA, DOT)? Do MSDS records require e-signature for approval? | High | EH&S / Legal | Open |
| GQ-05 | What is the data retention policy — is 7 years sufficient, or do EH&S/Finance records require longer retention? | Medium | Legal / Compliance | Open |
| GQ-06 | Are there any currently running MDM or data governance initiatives that this work must align with? | Medium | Enterprise Architect | Open |

---

## Scope Questions

| # | Question | Priority | Owner | Status |
|---|---|---|---|---|
| SQ-01 | Are all 83 objects in scope for Phase 1, or will there be a phased rollout? What is the Phase 1 priority list? | Critical | MDM Program Lead | Open |
| SQ-02 | The slide shows two slides — "Next Decade's SAP Master Data Objects" vs. "SAP Master Data Objects". Is Slide 1 the future-state target and Slide 2 the current state? | High | MDM Program Lead | Open |
| SQ-03 | HR payroll objects (Basic Pay, Payroll Accumulators, etc.) — is MDM expected to manage these, or are they out of scope given they are HCM-specific? | Medium | HR / MDM Program Lead | Open |
| SQ-04 | Quotes are listed as not in SAP — is this intentional (quotes managed outside SAP entirely) or a gap to address? | Medium | SD / Sales Team | Open |

---

## Decision Log

| # | Decision | Made By | Date | Rationale |
|---|---|---|---|---|
| — | No decisions recorded yet | — | — | — |

*Decisions will be recorded here as open questions are resolved.*
