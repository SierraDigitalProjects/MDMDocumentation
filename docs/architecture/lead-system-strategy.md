# Lead System Strategy

## Principle: Management Tier Drives Architecture

The three management tiers identified in the master data inventory directly determine which system is the **lead system** (system of record) for each object domain.

```
                    ┌─────────────────────────────────────────┐
                    │           MDM PLATFORM                  │
                    │                                         │
          ┌─────────┴──────────┐         ┌────────────────────┴────────┐
          │  LEAD SYSTEM MODE  │         │    CONSUMER/SUBSCRIBER MODE │
          │                    │         │                             │
          │  MDM owns:         │         │  SAP S/4HANA owns:          │
          │  - CRM objects     │         │  - Finance objects           │
          │  - EH&S objects    │         │  - SCM objects               │
          │  - HR objects      │         │  - SD core objects           │
          │  - SD non-SAP      │         │  - PM core objects           │
          │  - Other non-SAP   │         │  - PS objects                │
          └─────────┬──────────┘         └────────────────────┬────────┘
                    │                                         │
          Downstream consumers                      Replication into MDM
          (SAP, Salesforce, etc.)                   (CPI iFlows / IDocs)
```

---

## Lead System Assignment by Module

### SAP S/4HANA as Lead System

These modules use SAP as the authoritative source. MDM replicates inbound from SAP.

| Module | Lead System | Replication Direction | Trigger |
|---|---|---|---|
| Finance (FI/CO) | SAP S/4HANA | SAP → CPI → MDM | Change pointer / BAPI |
| SCM / MM (Central) | SAP S/4HANA | SAP → CPI → MDM | MATMAS IDoc / BAPI |
| SD (Core objects) | SAP S/4HANA | SAP → CPI → MDM | DEBMAS IDoc / BAPI |
| PM (Core objects) | SAP S/4HANA | SAP → CPI → MDM | Equipment/FuncLoc BAPI |
| PS | SAP S/4HANA | SAP → CPI → MDM | PS BAPIs |

---

### MDM as Lead System

These modules have no SAP presence. MDM is authoritative; SAP (where relevant) consumes from MDM.

| Module | Lead System | Replication Direction | Trigger |
|---|---|---|---|
| CRM | MDM | MDM → CPI → SAP (if needed) | MDM event / workflow approval |
| EH&S | MDM | MDM → EH&S tool (non-SAP) | MDM event |
| HR (Sales Rep only) | MDM (fed by HCM) | HCM → MDM → SAP SD | HCM event |
| SD (non-SAP objects) | MDM | MDM → downstream consumers | MDM event |
| Other (non-SAP objects) | MDM | MDM → downstream consumers | MDM event |

---

### Split-Module Governance

Three modules have objects in **both** SAP-managed and MDM-managed tiers. These require careful object-level governance.

=== "SD — Split"

    | Object | Lead System | Notes |
    |---|---|---|
    | BP Master | SAP | Central Data Team |
    | Customer Classification | SAP | Central Data Team |
    | Customer Hierarchy | SAP | Central Data Team |
    | Standard Text | SAP | Central Data Team |
    | Pricing Condition Records | SAP | Central Data Team |
    | Freight Master Data | SAP | Central Data Team |
    | Exchange Agreements | MDM | Not in SAP |
    | Sales BOMs | MDM | Not in SAP |
    | Quotes | MDM | Not in SAP |
    | Output Condition Records | MDM | Not in SAP |
    | EDI Partner Profile | MDM | Not in SAP |
    | Material Master Rev Levels | MDM | Not in SAP |

=== "PM — Split"

    | Object | Lead System | Notes |
    |---|---|---|
    | Characteristics | SAP | Central Data Team |
    | Classes | SAP | Central Data Team |
    | Equipment | SAP | Central Data Team |
    | Functional Location | SAP | Central Data Team |
    | Engineering Change Numbers | SAP | Central Data Team |
    | Bill of Materials | SAP (site-local) | Site Operations |
    | Catalogs | SAP (site-local) | Site Operations |
    | Maintenance Plan | SAP (site-local) | Site Operations |
    | Maintenance Item | SAP (site-local) | Site Operations |
    | Task Lists | SAP (site-local) | Site Operations |
    | Standard Text | SAP (site-local) | Site Operations |
    | Work Center | SAP (site-local) | Site Operations |
    | Planner Group | SAP (site-local) | Site Operations |

=== "Other — Split"

    | Object | Lead System | Notes |
    |---|---|---|
    | Asset Master | SAP | Central Data Team |
    | Storage Bins | SAP | Central Data Team |
    | Profit Center | SAP | Central Data Team |
    | Internal Orders | MDM | Not in SAP |
    | Lease & Property Master | MDM | Not in SAP |
    | EDI Partner Profile | MDM | Not in SAP |
    | Material Master Rev Levels | MDM | Not in SAP |

---

## Business Partner Unification

The Business Partner object spans multiple modules with different management statuses. MDM must provide a **unified BP object** that resolves these into a single golden record.

```
  CRM Prospect (Not in SAP)  ──────────┐
                                        ├──► MDM Golden Record (BP)
  SD Business Partner (SAP Central) ───┤       │
                                        │       ├──► SAP SD (Customer)
  SCM Business Partners Master (SAP) ──┘       ├──► SAP MM (Vendor)
                                                └──► CRM system
```

**Key requirement:** The MDM platform must support lifecycle state transitions for BP objects (Prospect → Customer, Vendor onboarding) without creating duplicate records across source systems.

---

## SAP All-or-Nothing Modules

Three modules have **zero SAP presence** — all objects are outside SAP:

| Module | All Objects Outside SAP |
|---|---|
| CRM | Yes — 7/7 objects |
| EH&S | Yes — 6/6 objects |
| HR | Yes — 12/12 objects |

These modules should be deprioritized for SAP integration work. MDM should focus on non-SAP source system integrations (Salesforce, Workday, EH&S tools) for these domains.
