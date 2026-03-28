# Goals and Objectives

## Strategic Goals

The Sierra MDM Platform is being built to address a fundamental gap in SierraDock's data landscape: **83 master data objects** are managed across 10 SAP functional modules with no single system of record, inconsistent governance, and no event-driven mechanism to keep satellite systems in sync.

The platform targets three strategic outcomes:

| Goal | Description |
|---|---|
| **Single Source of Truth** | Every master data object has one authoritative record, one owner, and one version that all systems consume |
| **Event-Driven Data Distribution** | Changes to master data propagate automatically to all interested systems without manual exports or point-to-point integrations |
| **Governed, Quality-Controlled Data** | All master data changes go through defined workflows with RBAC enforcement, validation rules, and immutable audit trails |

---

## Objectives

### Business Objectives

| # | Objective | Measure of Success |
|---|---|---|
| BO-01 | Eliminate duplicate master data records across CRM, SAP, and HR systems | Zero duplicate golden records per object type at go-live |
| BO-02 | Provide a single API for all systems to read master data | All consumers use MDM API — no direct SAP RFC or CRM API calls for master data reads |
| BO-03 | Reduce master data onboarding time for new sites | New site data setup time reduced by 50% vs current manual process |
| BO-04 | Give business stakeholders visibility into data quality | Data quality dashboard live, reporting completeness and validity scores per object type |
| BO-05 | Ensure compliance with audit and retention requirements | 100% of master data changes captured in immutable audit log, retained 7 years |

### Technical Objectives

| # | Objective | Measure of Success |
|---|---|---|
| TO-01 | Support all 83 master data objects in the inventory | All objects onboarded to MDM by end of Phase 2 |
| TO-02 | Near-real-time propagation for critical objects | Business Partner, Material Master, Equipment changes published to AEM within 5 minutes of creation/update |
| TO-03 | Configurable scheduled delivery for batch consumers | At least 3 satellite systems consuming via nightly CPI iFlow with configurable schedule |
| TO-04 | Field-level consumer filtering | Consumer receives events only when fields they registered interest in have changed |
| TO-05 | High availability with zero data loss | 99.5% uptime; no master data event lost in transit during platform restart |
| TO-06 | RBAC enforced at object and extension level | Site coordinator cannot modify another site's records; verified by penetration test |

---

## Phased Delivery

| Phase | Scope | Target Outcome |
|---|---|---|
| **Phase 1 — Foundation** | MDM Core API, Schema Service, SAP inbound (BP + Material), MongoDB, Redis, XSUAA | MDM receives and stores S/4HANA master data; read API available |
| **Phase 2 — Fan-out** | AEM integration, Consumer Registry, CPI outbound iFlows (CRM + ERP), field-change tracking | Changes fan out to first two satellite systems via AEM + CPI |
| **Phase 3 — Non-SAP Domains** | CRM objects, EH&S objects, HR (Sales Rep), MDM as lead system for 31 non-SAP objects | MDM becomes system of record for objects currently with no owner |
| **Phase 4 — Autonomy** | Scheduled delivery, site RBAC, autonomous monitoring, AI-assisted operations | Full governance model live; autonomous monitoring and self-healing active |
| **Phase 5 — AI Enhancement** | AI schema assist, AI log analysis, AI-generated iFlow scripts, cost optimisation | AI layer reduces ongoing build and ops effort |

---

## Success Criteria

The Sierra MDM Platform is considered successful when:

1. All 83 master data objects are onboarded with defined ownership (Data Team, Site, or MDM lead)
2. No satellite system reads master data directly from S/4HANA or CRM — all reads go through MDM API or AEM events
3. Any master data change in any source system appears as a golden record update in MDM within 5 minutes
4. Data quality scores are visible per object type and trending upward quarter-over-quarter
5. The platform sustains production load without manual intervention for 30 consecutive days
