# Not Managed in SAP at ND

These objects have **no current system of record** in SAP. They are the highest-priority candidates for MDM as the lead system. Without MDM, these objects exist in spreadsheets, local databases, or are not formally managed at all.

!!! danger "MDM Role: Lead System / System of Record"
    MDM is the authoritative source for these objects. Downstream systems — including SAP where applicable — should consume from MDM, not the reverse.

---

## CRM

All 7 CRM objects are outside SAP. A CRM platform (e.g., Salesforce, SAP CRM) would be the likely current home, or these may be managed manually.

| Object | Description | Likely Current System | MDM Priority |
|---|---|---|---|
| Business Partners | Customer/prospect business entities | Salesforce / manual | High |
| Competitors | Competitor organizations | Salesforce / manual | Medium |
| Contacts | Contact persons at business partners | Salesforce / manual | High |
| Employees | Employee records used in CRM context | Workday / manual | High |
| MKTG Attributes | Marketing segmentation attributes | Salesforce / manual | Medium |
| Territory Management | Sales territory definitions and assignments | Salesforce / manual | Medium |
| Prospects | Pre-qualification leads/prospects | Salesforce / manual | High |

**Architecture note:** CRM Business Partners and SD BP Master (centrally managed in SAP) likely represent the **same real-world entity** at different stages of the customer lifecycle. MDM should provide a unified Business Partner object that bridges both.

---

## EH&S (Environment, Health & Safety)

All 6 EH&S objects are outside SAP. SAP's EH&S module exists but is not in use at ND. Data likely lives in specialized EH&S tools or spreadsheets.

| Object | Description | Likely Current System | MDM Priority |
|---|---|---|---|
| DOT Info & Dangerous Goods | DOT classification and hazmat transport info | Specialized EH&S tool / spreadsheet | High |
| Hazardous Materials | Hazmat substance master | Specialized EH&S tool | High |
| Phrases | Regulatory phrase library (GHS/SDS) | Specialized EH&S tool | Medium |
| Specification Data | Product specification details | Specialized EH&S tool | Medium |
| Specification Header | Header/metadata for specifications | Specialized EH&S tool | Medium |
| MSDS | Material Safety Data Sheets | Specialized EH&S tool | High |

**Regulatory note:** Dangerous Goods, MSDS, and DOT Info are compliance-critical objects. MDM for these objects must support audit trails, versioning, and regulatory approval workflows.

---

## HR (Human Resources)

All 12 HR objects are outside SAP. These are typical Workday or SuccessFactors domains.

| Object | Description | Likely Current System | MDM Priority |
|---|---|---|---|
| Add'l Payments | Additional/supplemental payment records | Workday / payroll system | Low |
| Custom Time Indicators | Custom time tracking flags | Workday / time system | Low |
| Basic Pay | Base salary/pay master | Workday | Low |
| Employees — CRM Partner | Employee records linked to CRM | Workday → Salesforce | Medium |
| Employees — Sales/Credit Rep | Employee records with sales/credit roles | Workday → SAP SD | High |
| Organizational Assignment | Org unit and position assignments | Workday | Medium |
| Planned Working Time | Work schedule definitions | Workday | Low |
| Payroll Accumulators | YTD payroll accumulators | Payroll system | Low |
| Time Transfer Specifications | Rules for time data transfers | Time system | Low |
| Time Sheet Defaults | Default time entry configurations | Time system | Low |
| Recurring Payments | Recurring pay elements | Workday / payroll | Low |
| PU12 Export Status | SAP HCM export tracking field | SAP HCM (legacy?) | Low |

**Architecture note:** Employees — Sales/Credit Rep is the highest-priority HR object for MDM because it directly feeds SAP SD (sales rep assignment on orders). This is the primary HR → SAP integration touchpoint.

---

## Sales & Distribution (SD) — Partial

6 of the 12 SD objects are outside SAP.

| Object | Description | Likely Current System | MDM Priority |
|---|---|---|---|
| Exchange Agreements | Product exchange/borrow agreements | Contract system / manual | High |
| Sales BOMs | Customer-specific bill of materials for sales | Manual / spreadsheet | High |
| Quotes | Sales quotations | Salesforce / manual | Medium |
| Output Condition Records | Document output/message determination | Manual | Low |
| EDI Partner Profile | EDI trading partner configuration | EDI system (Sterling, etc.) | Medium |
| Material Master Rev Levels | Revision level tracking for materials | Manual | Medium |

**Architecture note:** Sales BOMs and Exchange Agreements are commercial master data with no current owner — high risk of inconsistency across sites. MDM should prioritize these.

---

## Other (Cross-Module) — Partial

5 of the 9 "Other" objects are outside SAP.

| Object | Description | Likely Current System | MDM Priority |
|---|---|---|---|
| Internal Orders | CO internal order master | Manual / separate system | Medium |
| Lease & Property Master | Real estate / lease agreements | Property management system | High |
| EDI Partner Profile | (Also listed under SD — same object) | EDI system | Medium |
| Material Master Rev Levels | (Also listed under SD — same object) | Manual | Medium |

---

## Priority Matrix

| Priority | Objects | Rationale |
|---|---|---|
| **High** | Business Partners, Contacts, Prospects, DOT/Hazmat/MSDS, Employees-Sales Rep, Exchange Agreements, Sales BOMs, Lease & Property | Compliance risk, revenue impact, or no current owner |
| **Medium** | Competitors, Employees-CRM, Territory Mgmt, Spec Data, EDI Profile, Material Rev Levels | Operational impact, manageable risk today |
| **Low** | HR payroll objects, Time objects, Output Conditions | High volume, low strategic value for MDM |
