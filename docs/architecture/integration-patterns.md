# Integration Patterns

## Pattern Overview

Three integration patterns apply across the MDM object landscape, determined by the lead system and data flow direction.

| Pattern | Direction | Trigger | Volume |
|---|---|---|---|
| **A — SAP Inbound to MDM** | SAP → CPI → MDM | IDoc / change pointer | Medium-High |
| **B — MDM Outbound to SAP** | MDM → CPI → SAP | MDM workflow approval | Low-Medium |
| **C — Non-SAP to MDM** | External system → MDM | Event / API / batch | Varies |

---

## Pattern A — SAP Inbound to MDM

Used for all **centrally managed** and **locally managed** SAP objects. SAP is authoritative; MDM receives replicated data.

```
SAP S/4HANA
    │
    │  IDoc / BAPI / Change Pointer
    ▼
SAP CPI (Integration Suite)
    │
    │  Mapping + Transformation
    ▼
MDM Platform
    │
    │  Merge / Match / Publish
    ▼
MDM Golden Record
```

### Recommended IDoc/BAPI by Object Type

| Object Type | SAP Message Type / BAPI | Notes |
|---|---|---|
| Material Master | `MATMAS05` IDoc | Most critical; high volume |
| Customer / BP | `DEBMAS07` IDoc | Unified BP in S/4HANA |
| Vendor / BP | `CREMAS05` IDoc | Unified BP in S/4HANA |
| GL Account | `GLMAST` / BAPI_GL_ACC_GETLIST | |
| Cost Center | `COSMAS` IDoc | |
| Equipment | `EQUMAS` IDoc | |
| Functional Location | `IFLOTMAS` IDoc | |
| Pricing Conditions | `COND_A` / `VK11` BAPIs | High volume; delta approach |
| WBS Elements | `PRPMAS` IDoc | |

### CPI Design Considerations

- Use **delta replication** (change pointer-based) for high-volume objects (Material Master, Pricing Conditions)
- Use **full load** only for initial migration and periodic reconciliation
- Implement **idempotent processing** — duplicate IDocs must not create duplicate MDM records
- Apply **field-level mapping** in CPI to normalize SAP-specific codes to MDM canonical model
- Log all inbound messages for audit trail

---

## Pattern B — MDM Outbound to SAP

Used when MDM is the lead system and SAP needs to consume MDM data. Applies to objects such as Employees — Sales/Credit Rep where HR data feeds SAP SD.

```
MDM Platform
    │
    │  Approval workflow completed
    ▼
MDM Event / API
    │
    ▼
SAP CPI (Integration Suite)
    │
    │  Mapping + Validation
    ▼
SAP S/4HANA
    │  (BAPI / Direct API call)
    ▼
SAP Master Data Record Created/Updated
```

### Key Objects Using Pattern B

| Object | MDM → SAP Target | SAP BAPI / Transaction |
|---|---|---|
| Employees — Sales/Credit Rep | SAP SD (Credit Rep assignment) | BAPI_EMPLOYEE_GETDATA / custom |
| Business Partner (new prospect → customer) | SAP SD Customer | BAPI_BUPA_CREATE_FROM_DATA |
| Exchange Agreements | SAP SD (if applicable) | Custom |

### Governance Gate

Pattern B **must** include a workflow approval step before writing to SAP. Unapproved records should not flow into SAP.

---

## Pattern C — Non-SAP to MDM

Used for objects with no SAP presence (CRM, EH&S, HR, and non-SAP SD objects). The source is an external system (Salesforce, Workday, EH&S tool) or manual entry via MDM UI.

```
External System                    OR          MDM User Interface
(Salesforce / Workday / EH&S)                 (Manual Entry)
    │                                               │
    │  REST API / Batch File / Event                │
    ▼                                               ▼
SAP CPI (or direct API gateway)            MDM Workflow
    │                                               │
    │  Mapping + Deduplication Check               │
    └───────────────────┬───────────────────────────┘
                        ▼
                  MDM Platform
                  (Match & Merge)
                        │
                        ▼
                  MDM Golden Record
```

### Source System Map (Pattern C)

| MDM Object Domain | Likely Source System | Integration Method |
|---|---|---|
| CRM — Business Partners | Salesforce | Salesforce → CPI REST adapter |
| CRM — Contacts | Salesforce | Salesforce → CPI REST adapter |
| CRM — Prospects | Salesforce | Salesforce → CPI REST adapter |
| EH&S — Hazardous Materials | EH&S tool (Cority, Enablon, etc.) | Batch file / API |
| EH&S — MSDS | EH&S tool / DMS | Batch file |
| HR — Employees (Sales Rep) | Workday | Workday → CPI → MDM |
| HR — Org Assignment | Workday | Workday → CPI → MDM |
| Lease & Property | Property management system | Batch / API |
| EDI Partner Profile | EDI platform (Sterling, etc.) | Manual or batch |

---

## Deduplication Strategy

All three patterns must feed into the MDM **match and merge** engine.

```
Inbound Record
    │
    ▼
Match Rules Applied
(Deterministic first, then probabilistic)
    │
    ├── Exact match found → Update existing golden record
    ├── High-confidence match → Auto-merge (with audit log)
    ├── Possible match → Queue for manual review
    └── No match → Create new golden record
```

### Match Key Recommendations by Object

| Object | Deterministic Keys | Probabilistic Attributes |
|---|---|---|
| Business Partner | SAP BP number, Tax ID, DUNS | Name, address, phone |
| Material Master | SAP Material number, EAN/UPC | Description, material group |
| Employee | Employee ID (HCM), email | Name, department |
| Equipment | SAP Equipment number, serial number | Description, manufacturer |

---

## Error Handling

All patterns must implement a **dead letter queue** pattern for failed messages:

1. Message fails validation or transformation in CPI
2. Message routed to error queue with full payload and error details
3. Alert sent to MDM operations team
4. Manual reprocessing available via CPI Operations Monitor
5. Resolution recorded in audit log
